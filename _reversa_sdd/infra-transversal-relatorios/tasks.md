# Infra Transversal e Relatórios — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] `@supabase/ssr`; tabela `idempotency_keys`; view `ai_provider_credentials_safe`
- [ ] Env: `AI_CRED_AES_KEY`, `NEXT_PUBLIC_SUPABASE_*`, `SUPABASE_SERVICE_ROLE_KEY`
- [ ] RPCs `fn_activity_report`, `fn_atrito_metrics`

## Tarefas
- [ ] T-01, Implementar wrappers e catálogo de erros
  - Origem no legado: `lib/api/wrappers.ts`, `errors.ts`, `types.ts`, `recusa.ts`
  - Critério de pronto: `X-Request-Id` em toda resposta; `respostaDeRecusa` re-lança não-ApiError
  - Confiança: 🟢
- [ ] T-02, Implementar os três clients Supabase
  - Origem no legado: `lib/supabase/{server,browser,admin,cookie-secure}.ts`
  - Critério de pronto: server `getUser()`; admin bypassa RLS; `cookieSecure` do protocolo; Realtime token via endpoint
  - Confiança: 🟢
- [ ] T-03, Implementar auth dual e idempotência
  - Origem no legado: `lib/api/auth-dual.ts`, `idempotency.ts`
  - Critério de pronto: org do token/RBAC; reserva antes do efeito; corrida → 409; hash desconhecido → conflito
  - Confiança: 🟢
- [ ] T-04, Implementar cripto e contrato de env
  - Origem no legado: `lib/crypto/aes_gcm.ts`, `lib/env.ts`
  - Critério de pronto: plaintext nunca sai (só last4); env com `z.string()` para knobs; lança no boot real, leniente no build
  - Confiança: 🟢
- [ ] T-05, Implementar logger, i18n e schemas
  - Origem no legado: `lib/logger.ts`, `lib/i18n/*`, `lib/schemas/*`
  - Critério de pronto: JSON estruturado sem PII; i18n fallback legível; `validateRequest` lança ApiError 400/422
  - Confiança: 🟢
- [ ] T-06, Implementar cliente de fetch, query, http e net
  - Origem no legado: `lib/api/client.ts`, `lib/query/client.ts`, `lib/http/ip-do-cliente.ts`, `lib/net/alcance.ts`
  - Critério de pronto: mutação nunca retenta; retry só 429/503; `ipDoCliente` não autoriza; `classificarFalhaDeAlcance` distingue config de serviço-fora
  - Confiança: 🟢
- [ ] T-07, Implementar relatórios e métricas
  - Origem no legado: `lib/reports/atividades.ts`, `lib/metrics/atrito.ts`
  - Critério de pronto: `origemDoAtor` (contact = automatico); eficiência pareada com dano; `null` nunca `0`
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Toda resposta seta X-Request-Id
- [ ] TT-02, Idempotência: corrida → 409 idempotency_in_progress
- [ ] TT-03, Mutação não retenta em timeout
- [ ] TT-04, Env com knob inválido não deixa contêiner healthy com 500 (usa z.string)
- [ ] TT-05, Cripto: plaintext nunca devolvido; só last4

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `idempotency_keys` (migration 0321) com índice único de reserva

## Ordem Sugerida
1. T-01/T-02 (base HTTP + clients) primeiro.
2. T-03/T-04 (auth dual, idempotência, cripto, env) dependem da base.
3. T-05/T-06/T-07 por último.

## Lacunas Pendentes (🔴)
- RPCs/tabela/view/policy (Data Master).
- Reconferir os 24 schemas de entidade campo a campo.
