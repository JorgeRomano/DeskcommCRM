# Auth e Multi-tenancy — Tarefas de Implementação

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Pré-requisitos
- [ ] Supabase Auth + RLS habilitada em toda tabela tenant-aware
- [ ] RPCs `fn_user_role_in_org`, `fn_is_platform_admin`, `fn_support_context`, `fn_user_org_ids`
- [ ] Tabelas `user_organizations`, `platform_admins`, `api_tokens`, `platform_support_sessions`
- [ ] Segredos: `INTERNAL_SECRET`, `IMPERSONATE_COOKIE_SECRET` (≥32), `INVITE_TOKEN_SECRET`
- [ ] Upstash Redis (ou fallback em memória) para rate limit

## Tarefas

- [ ] T-01, Implementar `requireRole` (role do banco + gate de MFA + 401/403 padronizados)
  - Origem no legado: `lib/auth/require-role.ts`
  - Critério de pronto: rank do banco; MFA aal1 → mfa_required; audit authz.denied
  - Confiança: 🟢

- [ ] T-02, Implementar `loadAuthUser`/`resolveActiveOrg` (falha ALTO)
  - Origem no legado: `lib/auth/server.ts`
  - Critério de pronto: erro de permissão lança; org default por `accepted_at` sem cookie
  - Confiança: 🟢

- [ ] T-03, Implementar middleware de borda
  - Origem no legado: `proxy.ts`, `lib/auth/public-paths.ts`
  - Critério de pronto: 401 JSON em /api; gate /admin; impersonation edge (Web Crypto)
  - Confiança: 🟢

- [ ] T-04, Implementar clients Supabase (browser/server/admin)
  - Origem no legado: `lib/supabase/*`
  - Critério de pronto: `getUser` (nunca getSession); admin bypassa RLS com filtro manual
  - Confiança: 🟢

- [ ] T-05, Implementar contrato de API (`ok`/`fail` + códigos)
  - Origem no legado: `lib/api/wrappers.ts`, `lib/api/errors.ts`, `handlers/types.ts`
  - Critério de pronto: `{data}`/`{error}` + X-Request-Id; códigos declarados em errors.ts
  - Confiança: 🟢

- [ ] T-06, Implementar tokens (convite HMAC, impersonation Node+Edge, MCP Bearer)
  - Origem no legado: `lib/auth/invite-token.ts`, `lib/impersonate/*`, `lib/mcp/auth.ts`
  - Critério de pronto: `timingSafeEqual`; expiry checado; revogado recusado
  - Confiança: 🟢

- [ ] T-07, Implementar rate limit (IP + conta)
  - Origem no legado: `lib/auth/rate-limit.ts`
  - Critério de pronto: sem IP → só conta; brute force por conta bloqueia
  - Confiança: 🟢

- [ ] T-08, Implementar sessão de suporte (impersonation) autoritativa no banco
  - Origem no legado: `lib/impersonate/support.ts`
  - Critério de pronto: `requireSupportWrite` antes de service role; expirada bloqueia
  - Confiança: 🟢

## Tarefas de Teste

- [ ] TT-01, agent → rota de manager retorna 403 forbidden_role
- [ ] TT-02, admin aal1 com TOTP → mfa_required
- [ ] TT-03, isolamento cross-tenant (RLS) — invariante de banco
- [ ] TT-04, token MCP sem scope → -32002
- [ ] TT-05, 5 senhas erradas na conta bloqueiam pela janela

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `user_organizations` com CHECK de papéis humanos (exclui ai_operator)

## Ordem Sugerida
1. T-04 (clients) e T-05 (contrato) são base.
2. T-01/T-02/T-03 (gate + borda) sobre eles.
3. T-06/T-07/T-08 (tokens/rate limit/suporte) por último.

## Lacunas Pendentes (🔴)
- SQL das RPCs de role/suporte e as policies RLS (obrigatório antes de reimplementar o isolamento).
- Matriz rota-a-rota completa (169 handlers).
- Server Actions (`app/actions/*`).
