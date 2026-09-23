# IA de Suporte — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Dependências disponíveis: `lib/agent-engine`, `lib/crypto/aes_gcm`, `lib/supabase/admin`, `@upstash/redis`, `@ai-sdk/*`, `@modelcontextprotocol/sdk`
- [ ] Tabelas: `ai_purpose_bindings`, `ai_credentials`, `ai_models`, `ai_pricing`, `ai_budgets`, `ai_knowledge_sources`, `ai_kb_versions`, `org_memory_versions/pointers`, `api_tokens`, `idempotency_keys`, `api_audit_log`
- [ ] Env: `AI_GATEWAY_API_KEY`, `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`

## Tarefas
- [ ] T-01, Implementar gateway e resolução de modelo por ponto
  - Origem no legado: `lib/ai/gateway.ts`, `lib/ai/gateway-binding.ts`
  - Critério de pronto: fallback ordenado; ZDR+tenant nos headers; provider inutilizável cai no padrão com warn; prefixo é rota
  - Confiança: 🟢
- [ ] T-02, Implementar catálogo, custo, credenciais e validadores
  - Origem no legado: `lib/ai/classifier-models.ts`, `cost.ts`, `credentials.ts`, `provider-validators.ts`, `pontos/provedores.ts`
  - Critério de pronto: `computeCost` em centavos com `Math.ceil` + cache 5min; OpenRouter valida em `/api/v1/key`; credencial cifrada
  - Confiança: 🟢
- [ ] T-03, Implementar orçamento e agregação de uso
  - Origem no legado: `lib/ai/budget/check.ts`, `usage/aggregate.ts`
  - Critério de pronto: `getBudgetStatus` nunca lança, degrada com log; `blocked_now` lê inbox; `aggregateUsage` puro com p50/p95
  - Confiança: 🟢
- [ ] T-04, Implementar pipeline de RAG
  - Origem no legado: `lib/ai/embeddings/chave.ts`, `embed.ts`, `rag/chunker.ts`, `rag/version.ts`, `rag/debounce.ts`, `knowledge/busca.ts`
  - Critério de pronto: embedding fixo 1536 (assere length); versionamento por fonte desativa-antes-de-ativar; busca com piso −1 + limiar em memória; debounce fail-open
  - Confiança: 🟢
- [ ] T-05, Implementar config de agentes, system prompt, guardrails e memória
  - Origem no legado: `guardrails-schema.ts`, `render-system-prompt.ts`, `agents.ts`, `memoria-da-org.ts`, `apply-proposal.ts`, `pacing-knobs.ts`
  - Critério de pronto: schema Zod único front+back; render preserva placeholders desconhecidos; memória versionada nunca sobrescreve; proposta com gate humano
  - Confiança: 🟢
- [ ] T-06, Implementar o núcleo do servidor MCP
  - Origem no legado: `lib/mcp/server.ts`, `types.ts`, `auth.ts`, `audit.ts`, `recusa-para-o-modelo.ts`, `uuid-de-aterro.ts`
  - Critério de pronto: pipeline por tool completo; auth Bearer com hash SHA256; actor nunca `user`; auditoria redige PII; UUID de aterro higienizado
  - Confiança: 🟢
- [ ] T-07, Implementar as ~47 tools MCP por domínio
  - Origem no legado: `lib/mcp/tools/*`, `tools/catalogo/*`, sanity-check 1:1 catálogo↔handler
  - Critério de pronto: RBAC por `requiresRole`; idempotência em escritas de mensagem; `resume_ai_attendance` só pessoa; `crm_search_products` só afirma vazio após varredura completa
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Fallback de provider e resolução por ponto
- [ ] TT-02, `computeCost` não subfatura; preços 0/0 → null
- [ ] TT-03, `getBudgetStatus` degrada sem lançar
- [ ] TT-04, `activateVersion` desativa a anterior antes
- [ ] TT-05, Bearer revogado é recusado; actor nunca `user`
- [ ] TT-06, Idempotência de `crm_send_whatsapp_message`

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Migrar `ai_kb_versions` preservando a versão ativa única por fonte

## Ordem Sugerida
1. T-01/T-02 (gateway, custo, credenciais) primeiro.
2. T-03 (orçamento) e T-04 (RAG) dependem de T-01/T-02.
3. T-06 (núcleo MCP) antes de T-07 (tools).

## Lacunas Pendentes (🔴)
- 🟢 Fonte canônica das strings de modelo (`AGENT_MODELS` × `DEFAULT_*`) é `lib/ai/gateway.ts` (`DEFAULT_BOT_MODEL = "anthropic/claude-sonnet-5"`). O `lib/ai/README.md` está **obsoleto e deve ser ignorado** na reimplementação. <!-- [Revisão] usuário confirmou 2026-09-23 -->
