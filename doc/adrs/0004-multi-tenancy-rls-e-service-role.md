# ADR 0004 — Multi-tenancy por RLS com filtro manual sob service role

> ADR retroativo · Status: **Aceito** · Confiança: 🟢 CONFIRMADO (`docs/business-rules` T-01/T-02, `lib/supabase/`, `lib/auth/`)

## Contexto

Produto multi-tenant self-host sobre Supabase/Postgres. Precisa isolar dados de tenants por construção, mas muitos caminhos (webhooks públicos, crons, workers) não têm sessão de usuário e usam o cliente service-role, que **bypassa RLS**.

## Decisão

- **RLS habilitada em toda tabela tenant-aware** (`organization_id` + policy de isolamento) — regra T-01, hard constraint.
- **Role efetivo resolvido pela mesma função das RLS** (`fn_user_role_in_org`), nunca pelo snapshot do cookie.
- **Sob service role, filtro manual obrigatório** de `organization_id` resolvido de fonte confiável (cookie validado / JWT / webhook secret / path token), **nunca do body** (T-02). 89 dos 169 handlers usam service role.
- `platform_admin` é o único papel cross-tenant (T-04).
- `loadAuthUser` **falha ALTO**: lança em erro de permissão em vez de degradar para "sem organização" (degradar permissão em silêncio parece decisão de autorização e é defeito de infra — medido 2026-07-30, seis diagnósticos errados).

## Alternativas consideradas

1. **Confiar só na RLS (sem filtro manual).** Rejeitada: o service role bypassa RLS; sem filtro, um webhook vazaria cross-tenant.
2. **Não usar service role (tudo com client de usuário).** Rejeitada: webhooks/crons/workers não têm sessão; e o audit de login falho precisa escrever sem sessão.
3. **Resolver tenant do body/header do request.** Rejeitada: superfície de forja — a fonte tem de ser o cookie validado/JWT/secret/path token.

## Consequências

- **Positivas:** isolamento por construção + defesa em profundidade; auditável (`bypassed_rls=true`).
- **Negativas:** o filtro manual é responsabilidade do autor do handler (sem gate automático) — risco recorrente; a doutrina cobra em code review.
- **Enforcement:** teste de isolamento obrigatório por tabela (`tests/invariants/`), linter SQL no CI, job `invariants` (obrigatório na `main`).
