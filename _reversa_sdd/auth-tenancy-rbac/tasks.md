# Auth, Tenancy e RBAC — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Supabase Auth + `@supabase/ssr`; RPCs `fn_user_role_in_org`, `fn_is_platform_admin`, `fn_accept_team_invite`, `fn_support_context`
- [ ] Tabelas: `user_organizations`, `platform_admins`, `organizations`, `team_invites`, `api_tokens`, `user_recovery_codes`, `platform_settings`
- [ ] Env: `INTERNAL_SECRET`, `INTERNAL_CRON_SECRET`, `INVITE_TOKEN_SECRET`, `IMPERSONATE_COOKIE_SECRET`, `TENANT_PROVISIONING_SECRET`

## Tarefas
- [ ] T-01, Implementar sessão e organização ativa
  - Origem no legado: `lib/auth/server.ts`
  - Critério de pronto: `getUser()` (nunca `getSession`); falha alto; `escolherMembroAtivo` com ORDER BY
  - Confiança: 🟢
- [ ] T-02, Implementar política de MFA (cadastro vs sessão)
  - Origem no legado: `lib/auth/politica-mfa.ts`, `server.ts` (`mfaEmDivida`, `sessionAal`)
  - Critério de pronto: duas origens somam; default não exige; quem tem fator prova sempre
  - Confiança: 🟢
- [ ] T-03, Implementar o guard canônico de RBAC
  - Origem no legado: `lib/auth/require-role.ts`, `types.ts`
  - Critério de pronto: role do banco; gate de MFA após rank; `ai_operator` fora de `user_organizations`
  - Confiança: 🟢
- [ ] T-04, Implementar guard de plataforma e middleware de borda
  - Origem no legado: `lib/auth/requirePlatformAdmin.ts`, `proxy.ts`, `public-paths.ts`
  - Critério de pronto: `/admin/*` gateado; `/api/*` null → 401 JSON; allowlist ancorada com `$`
  - Confiança: 🟢
- [ ] T-05, Implementar convites (token HMAC + registro + aplicar)
  - Origem no legado: `lib/auth/invite-token.ts`, `issue-invite.ts`, `aplicar-convite.ts`, `convite-no-signup.ts`, `lib/team/convites.ts`, `convite-status.ts`
  - Critério de pronto: token stateless; registro revogável; nada do user_metadata é autoridade; email confirmado comparado
  - Confiança: 🟢
- [ ] T-06, Implementar provisionamento de tenant
  - Origem no legado: `lib/auth/provision.ts`
  - Critério de pronto: self-service idempotente; externo por (integração, id) com marcador; replay completa; slug sem marcador → 409
  - Confiança: 🟢
- [ ] T-07, Implementar cadastro, rate limit, recovery, cron auth, safe-next
  - Origem no legado: `lib/auth/politica-de-cadastro.ts`, `rate-limit.ts`, `recovery-codes.ts`, `cron-auth.ts`, `safe-next.ts`
  - Critério de pronto: leitura pegajosa; rate limit por IP+id (sem IP só id); rejection sampling; cron fail-closed; allowlist anti-redirect
  - Confiança: 🟢
- [ ] T-08, Implementar API key de org, nome do atendente e impersonation
  - Origem no legado: `lib/tenants/api-key.ts`, `lib/users/*`, `lib/impersonate/*`
  - Critério de pronto: `dsk_` com hash; papel agent + escopos; cookie HMAC server+edge; `readSupportContext` por RPC
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, `getSession` não usado no servidor
- [ ] TT-02, Erro de permissões lança (não degrada)
- [ ] TT-03, aal1 com fator → 403 mfa_required após o rank
- [ ] TT-04, Convite alheio no signup próprio é recusado (email divergente)
- [ ] TT-05, Provisionamento externo replay completa sem duplicar
- [ ] TT-06, Rate limit por id barra brute force; sem IP não tranca a instalação

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `team_invites` (migration 0238) com índice único parcial (1 pendente por email/org)

## Ordem Sugerida
1. T-01 (sessão) → T-02 (MFA) → T-03 (RBAC) → T-04 (borda).
2. T-05/T-06 (convite/provisionamento) dependem de sessão.
3. T-07/T-08 por último.

## Lacunas Pendentes (🔴)
- Funções SECURITY DEFINER e policies RLS (Data Master).
- Rate limit em crons e MCP (validar com threat-model).
