# IA de Suporte — Design Técnico

> `design.md` — foca no COMO. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `resolveLanguageModel` | `(model: ModelId)` | `LanguageModel \| null` |
| `resolverModeloDoPonto` | `(purpose, organizationId, padrao)` | `Promise<ModeloResolvido \| null>` |
| `computeCost` | `(input: ComputeCostInput)` | `Promise<number>` (centavos) |
| `getBudgetStatus` | `(orgId)` | `Promise<BudgetStatus>` |
| `embedText` | `(content, opts: EmbedOptions)` | `Promise<EmbedResult>` |
| `chunkText` | `(text, opts?: ChunkOptions)` | `string[]` |
| `createKnowledgeVersion` / `activateVersion` | `(params)` | `Promise<...>` |
| `buscarConhecimento` | `(supabase, p: ParametrosDaBusca, deps?)` | `Promise<ResultadoDaBusca>` |
| `createMcpServer` | `(auth: McpAuthResult, requestId)` | `McpServer` |
| `validateBearerToken` | `(authHeader)` | `Promise<McpAuthResult>` |
| `auditMcpToolCall` | `(input)` | `Promise<void>` |

Constantes: `DEFAULT_BOT_MODEL="anthropic/claude-sonnet-5"`, `DEFAULT_CLASSIFIER_MODEL="anthropic/claude-haiku-4-5"`, `DEFAULT_EMBEDDING_MODEL="openai/text-embedding-3-small"`. `PROVEDORES` = anthropic, openai, google, openrouter, deepseek. `ROLE_RANK`: `viewer < agent < ai_operator < manager < admin`. 🟢

## Fluxo Principal — Resolução de modelo
1. `resolverModeloDoPonto`: binding habilitado em `ai_purpose_bindings` → credencial ativa+validada da org → padrão da instalação. 🟢
2. `idParaOProvider`: prefixo é rota (OpenRouter recebe id inteiro; outros sem prefixo; id de outro provider → `null`, freio PR #151). 🟢
3. `resolveLanguageModel` faz fallback do gateway Vercel → OpenRouter → Anthropic → OpenAI → `null`. 🟢
4. `gatewayHeaders` injeta ZDR + tenant id. 🟢

## Fluxo Principal — RAG
1. `chunkText`: parágrafo (`\n\n+`) → sub-split por sentença → overlap de 200 chars. 🟢
2. `embedText` com modelo fixo 1536 dims (assere length). 🟢
3. `createKnowledgeVersion` (status `building`, `version_number=max+1`) → `markVersionReady` → `activateVersion` (desativa a anterior antes). 🟢
4. `buscarConhecimento`: RPC `fn_buscar_trechos_das_fontes` com piso −1 e filtra limiar em memória. 🟢

## Fluxo Principal — MCP (pipeline por tool)
1. `higienizarUuidsDeAterro(args)`. 🟢
2. Monta `McpContext` (org de `ctx`, nunca do arg; admin client service-role). 🟢
3. `ensureScope` + `ensureRole`. 🟢
4. `tool.handler(args, ctx)`. 🟢
5. `motivoDoVazio` → `success` da auditoria. 🟢
6. `auditMcpToolCall` (redação de PII). 🟢
7. Devolve `content[]` + `structuredContent`. 🟢

## Dependências
- `lib/ai` → `lib/agent-engine/*` (pacing, edge/llm), `lib/crypto/aes_gcm`, `lib/supabase/admin`, `@upstash/redis`, `@ai-sdk/{anthropic,openai,google}`, `ai`. 🟢
- `lib/mcp` → handlers REST `app/api/v1/*/_handler`, `lib/schemas`, `lib/routing`, `lib/escalacao`, `lib/followup`, `lib/leads`, `lib/operacao`, `lib/agenda`, `lib/catalogo/busca`, `lib/messaging`, `lib/ai/knowledge/busca`, `@modelcontextprotocol/sdk`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Fallback de provider em cascata | `gateway.ts:75-97` | 🟢 |
| Chave da org nunca cruza com modelo alheio (freio PR #151) | `gateway-binding.ts:190` | 🟢 |
| Custo com `Math.ceil` + cache de preços TTL 5min | `cost.ts` | 🟢 |
| Versionamento RAG desativa-antes-de-ativar | `rag/version.ts:135` | 🟢 |
| Actor MCP nunca é `user` (evita quebrar FKs e furar gate) | `mcp/auth.ts` | 🟢 |
| Recusa para o modelo sem vazar vocabulário interno | `mcp/recusa-para-o-modelo.ts` | 🟢 |

## Estado Interno
- `_pricingCache` (TTL 5min). Versões de índice RAG e ponteiros em `ai_knowledge_sources`. Memória da org versionada (`org_memory_versions` + `org_memory_pointers`). 🟢

## Observabilidade
- `logInvocation` fire-and-forget via `queueMicrotask` em `llm_calls` (0130 aposentou `ai_invocations`). `InvocationKind`: `bot_respond|sentiment_classify|triage_classify|embedding_generate`. 🟢

## Riscos e Lacunas
- 🔴 `lib/ai/README.md` é placeholder obsoleto (menciona arquivos/modelos inexistentes) — ignorar.
- 🟡 Dois catálogos de string de modelo divergentes: `AGENT_MODELS` (`-4-6/-4-5/-4-7`) × `DEFAULT_*` (`-5`). Confirmar qual é canônico.
- 🟡 Furo de medição de custo (`gasto_incompleto`): `ai_pricing` casa por prefixo, gasto medido pode ser menor que o real.
- 🟡 Diretórios lidos só por nome nesta passagem: `elegibilidade/`, `replies/`, `runtime/` (parcial), `skills/`, `evolution/`, `anonymize/`, `prompts/`.
