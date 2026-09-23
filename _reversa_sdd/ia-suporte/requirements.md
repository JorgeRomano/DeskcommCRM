# IA de Suporte (`lib/ai` + `lib/mcp`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 2) e `.reversa/context/modules.json`.

## Visão Geral
Duas metades complementares em torno do núcleo (`lib/agent-engine`): 🟢
- **`lib/ai`** — a plataforma de IA por tenant: resolução de modelo/credencial (Vercel AI Gateway com fallback de provider), catálogo de modelos e custo em centavos, orçamento mensal por org, pipeline de RAG (ingestão → chunking → embedding → busca top-K com citações), config declarativa de agentes (Zod único front+back), guardrails de camada de IA, handoff bot→humano, memória versionada da org, dispatcher legado (`@deprecated`) e agregação de uso.
- **`lib/mcp`** — o servidor MCP que expõe o CRM inteiro como ~47 tools para o modelo, com auth Bearer `dsk_`, escopos convencionais, RBAC por `requiresRole`, auditoria com redação de PII, e a semântica de "recusa para o modelo" e "UUID de aterro".

## Responsabilidades
- Resolver modelo/credencial por ponto (binding → credencial da org → padrão da instalação). 🟢
- Calcular custo em centavos (`Math.ceil`, nunca subfatura) e aplicar orçamento mensal por org. 🟢
- Executar o pipeline de RAG: chunking, embedding (modelo fixo 1536 dims), versionamento por fonte, busca top-K com piso/limiar e citações. 🟢
- Prover config declarativa de agentes (schema Zod único front+back) e render do system prompt. 🟢
- Expor o CRM como servidor MCP com RBAC, idempotência, auditoria redigida e recusa para o modelo. 🟢
- Manter privacidade por tenant (ZDR/zero-retention no gateway). 🟢

## Regras de Negócio
- Fallback de provider do mais específico ao mais genérico: gateway Vercel → OpenRouter → Anthropic → OpenAI → `null`. — `lib/ai/gateway.ts:75-97` 🟢
- Privacy-by-default: injeta `X-AI-Gateway-Zero-Retention` e `X-AI-Gateway-Tenant-Id` em toda chamada. — `gateway.ts:108` 🟢
- Resolução de modelo por ponto: binding → credencial da org ativa+validada → padrão; provider/chave inutilizável cai no padrão com `warn`, nunca silencioso. — `gateway-binding.ts:60` 🟢
- Custo em centavos com `Math.ceil`; fallback de preço via `ai_models` devolve `null` se ambos os preços são 0. — `cost.ts` 🟢
- OpenRouter valida chave contra `/api/v1/key` (não `/models` público). — `provider-validators.ts:118-210` 🟢
- `getBudgetStatus` nunca lança: degrada da RPC para coluna materializada com log; `blocked_now` lê o inbox, não recalcula. — `budget/check.ts:170` 🟢
- Embedding: modelo FIXO `text-embedding-3-small` 1536 dims; length divergente lança. — `embed.ts` 🟢
- RAG versão por fonte: `activateVersion` desativa a anterior ANTES de ativar (índice único uma-ativa-por-fonte). — `rag/version.ts:135` 🟢
- Debounce de indexação falha ABERTA. — `rag/debounce.ts` 🟢
- Busca de conhecimento usa piso −1 na RPC e filtra o limiar em memória (expõe o melhor candidato reprovado). — `knowledge/busca.ts:70` 🟢
- Guardrails: 9 conferências de saída não se desligam; só `semantic_promise` tem escolha (+1 consulta/mensagem). — `guardrails/lista-de-conferencia.ts:66` 🟢
- MCP auth Bearer `dsk_`: hash SHA256 contra `token_hash`, checa revoked/expired; actor é `ai_agent` ou `api_token`, NUNCA `user`. — `mcp/auth.ts` 🟢
- `motivoDoVazio`: "não achei" não é sucesso; desce para `api_audit_log.metadata.motivo`. — `mcp/types.ts:32` 🟢
- Dispatcher é `@deprecated`; orgs em `ai_dispatch_mode='external'` são puladas (G6-02). — `dispatcher/index.ts` 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Resolver modelo por ponto com fallback ordenado | Must | Provider inutilizável cai no padrão com `warn`, nunca silencioso |
| RF-02 | Calcular custo em centavos sem subfaturar | Must | `computeCost` usa `Math.ceil`; preços 0/0 → `null` |
| RF-03 | Aplicar orçamento mensal sem lançar | Must | `getBudgetStatus` degrada para coluna materializada com log |
| RF-04 | Executar RAG com versionamento por fonte | Must | `activateVersion` desativa a anterior antes; só 1 ativa por fonte |
| RF-05 | Buscar conhecimento distinguindo "vazio" de "fraco" | Should | `melhorSimilaridade` reflete o melhor candidato reprovado |
| RF-06 | Autenticar MCP por Bearer com RBAC | Must | Token revogado/expirado é recusado; actor nunca é `user` |
| RF-07 | Auditar chamadas MCP com redação de PII | Must | `authorization/api_key/token/password/cpf` redigidos; strings >500 truncadas |
| RF-08 | Traduzir recusa por papel sem vazar vocabulário interno | Should | Recusa não menciona "role"/"permissão" |
| RF-09 | Idempotência em escritas de mensagem MCP | Must | `crm_send_whatsapp_message` deduplica via `idempotency_keys` TTL 24h |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | ZDR/zero-retention por tenant no gateway | `gateway.ts:108` | 🟢 |
| Segurança | Credencial cifrada (aes_gcm); plaintext só no retorno | `credentials.ts:56` | 🟢 |
| Segurança | Redação de PII na auditoria MCP | `mcp/audit.ts:36` | 🟢 |
| Segurança | UUID de aterro higienizado antes do handler | `mcp/uuid-de-aterro.ts:120` | 🟢 |
| Performance | Cache de preços TTL 5min | `cost.ts:23` | 🟢 |
| Performance | Validação de provider com timeout 5s, sem retry | `provider-validators.ts:39` | 🟢 |
| Disponibilidade | Debounce de indexação fail-open | `rag/debounce.ts` | 🟢 |
| Escalabilidade | Varredura paginada de produtos com prova de fim | `mcp/tools/comercio.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado uma org com credencial de provider inválida
Quando resolverModeloDoPonto é chamado
Então cai no padrão da instalação com logger.warn (nunca silencioso)

Dado um índice RAG com versão ativa por fonte
Quando activateVersion ativa uma nova versão
Então a versão anterior é desativada ANTES (só 1 ativa por fonte)

Dado um Bearer dsk_ revogado
Quando validateBearerToken avalia
Então a autenticação é recusada

Dado crm_send_whatsapp_message chamado duas vezes com a mesma idempotency key
Quando processado
Então o segundo retorna deduplicated:true (TTL 24h)
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Resolução de modelo/credencial (RF-01) | Must | Toda chamada de IA depende disso |
| Custo/orçamento (RF-02/03) | Must | Controle financeiro, sem fallback silencioso |
| RAG versionado (RF-04) | Must | Base de conhecimento do agente |
| Auth/RBAC MCP (RF-06) | Must | Superfície de ataque grande, sem alternativa |
| Auditoria redigida (RF-07) | Must | LGPD/segurança |
| Busca distinguindo vazio/fraco (RF-05) | Should | Qualidade de RAG, com fallback |
| Dispatcher legado | Won't | `@deprecated`, fora do caminho quente |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/ai/gateway.ts` | `resolveLanguageModel`, `gatewayHeaders` | 🟢 |
| `lib/ai/gateway-binding.ts` | `resolverModeloDoPonto` | 🟢 |
| `lib/ai/cost.ts` | `computeCost`, `precoDoCatalogo` | 🟢 |
| `lib/ai/budget/check.ts` | `getBudgetStatus` | 🟢 |
| `lib/ai/embed.ts`, `lib/ai/rag/*` | `embedText`, `chunkText`, `createKnowledgeVersion`, `activateVersion` | 🟢 |
| `lib/ai/knowledge/busca.ts` | `buscarConhecimento` | 🟢 |
| `lib/mcp/server.ts` | `createMcpServer` | 🟢 |
| `lib/mcp/auth.ts` | `validateBearerToken`, `resolveApiToken`, `ensureRole` | 🟢 |
| `lib/mcp/audit.ts` | `auditMcpToolCall` | 🟢 |
