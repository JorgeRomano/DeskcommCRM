# Superfície HTTP — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] `lib/api/wrappers`, `lib/auth/require-role`, `lib/mcp/auth`, `lib/supabase/*`, `lib/audit`
- [ ] Env: `INTERNAL_SECRET`, `INTERNAL_CRON_SECRET`; RPC `fn_is_platform_admin`
- [ ] Tabelas: `webhook_sources`, `api_tokens`, e as tenant-aware que cada rota consulta

## Tarefas
- [ ] T-01, Implementar o middleware de borda
  - Origem no legado: `proxy.ts`, `lib/auth/public-paths.ts`
  - Critério de pronto: 9 passos em ordem; `/api/*` 401 JSON; `/admin/*` gate antecipado; allowlist ancorada
  - Confiança: 🟢
- [ ] T-02, Implementar a receita de route handler (padrão)
  - Origem no legado: amostras `contact-tags`, `cron/snooze-watcher`, `internal/agents/run`, `webhooks/in/[token]`
  - Critério de pronto: Zod → guard → org confiável → query → audit → `ok()`/`fail()`; nenhum throw cru ao cliente
  - Confiança: 🟢
- [ ] T-03, Implementar as superfícies não-cookie
  - Origem no legado: `app/api/v1/cron/*`, `app/api/v1/webhooks/*`, `app/api/internal/*`, `app/api/mcp/route.ts`
  - Critério de pronto: cron fail-closed + audit condicional; webhook org do path token + HMAC + idempotência; internal secret; MCP JSON-RPC
  - Confiança: 🟢
- [ ] T-04, Implementar os grupos REST de `app/api/v1/**`
  - Origem no legado: `app/api/v1/**` (~48 grupos)
  - Critério de pronto: cada grupo segue a receita; envelope canônico; RBAC por rota; reconferir contagem com `git ls-files`
  - Confiança: 🟡
- [ ] T-05, Implementar as Server Actions
  - Origem no legado: `app/actions/**` (~51 arquivos)
  - Critério de pronto: `"use server"`; Zod → guard → mutação → audit → `revalidatePath`/`redirect`; objeto tipado; convite por token assinado + match de e-mail
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, `/api/*` sem sessão → 401 JSON com X-Request-Id
- [ ] TT-02, Webhook HMAC inválido → 401 invalid_signature
- [ ] TT-03, Cron sem mutação não audita
- [ ] TT-04, `external_id` repetido não duplica captação
- [ ] TT-05, MCP `McpAuthError` → JSON-RPC (-32001/-32002/-32603)

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `webhook_sources` com `path_token` e índice único de captação (`uniq_crm_leads_org_source_external`)

## Ordem Sugerida
1. T-01 (borda) → T-02 (receita) primeiro.
2. T-03 (não-cookie) e T-04 (grupos REST) em paralelo por superfície.
3. T-05 (Server Actions) por último.

## Lacunas Pendentes (🔴)
- Mapear cada endpoint de `v1` ao escrever a spec de superfície (as ~337 rotas não foram lidas uma a uma).
- Tabelas/RLS que os handlers assumem (Data Master).
