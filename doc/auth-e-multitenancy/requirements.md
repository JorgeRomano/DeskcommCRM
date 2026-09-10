# Auth e Multi-tenancy

> Spec SDD (híbrida — módulo de topo) · Redator (Reversa) · Nível: Detalhado
> Módulos: `lib/auth/*`, `lib/impersonate/*`, `lib/api/*`, `lib/supabase/*`, `proxy.ts`
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Visão Geral

Autenticação, autorização (RBAC) e isolamento multi-tenant. Todo request é isolado por `organization_id` via RLS; o gate canônico `requireRole` resolve o papel efetivo do banco e aplica MFA de sessão. Superfícies não-cookie (MCP, webhooks, crons, suporte) autenticam dentro da rota. 🟢

## Responsabilidades

- Validar JWT na borda e por rota (`getUser`, nunca `getSession`). 🟢
- Resolver o papel efetivo do banco e negar com 401/403 padronizados. 🟢
- Resolver `organization_id` de fonte confiável, nunca do body. 🟢
- Aplicar MFA como política de sessão. 🟢
- Emitir/verificar tokens (convite HMAC, impersonation, MCP Bearer). 🟢
- Padronizar o contrato da API (`ok()`/`fail()`, códigos de erro). 🟢
- Aplicar rate limit por IP e por conta. 🟢

## Regras de Negócio

- Papéis: `viewer(1) < agent(2) < ai_operator(3) < manager(4) < admin(5)` + `platform_admin` (cross-tenant). 🟢
- `ai_operator` só existe em token efêmero, nunca em `user_organizations`. 🟢
- Role efetivo vem do banco (`fn_user_role_in_org`), não do snapshot do cookie. 🟢
- Sob service role (admin client), filtro manual de `organization_id` obrigatório (T-02). 🟢
- `loadAuthUser` falha ALTO (lança em erro de permissão, não degrada para sem-org). 🟢
- MFA em dívida = papel exige + fator cadastrado + sessão aal1 → 403. 🟢
- Sem IP identificável, limite por IP não entra; brute force por conta é o que barra. 🟢
- `platform_admin` bypassa rank do tenant só com `allowPlatformAdmin` e sem sessão de suporte. 🟢
- Superfícies públicas (`public-paths.ts`) bypassam auth de cookie; auth mora na rota. 🟢

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Gate de rota `requireRole(min, opts)` | Must | Dado rank insuficiente, retorna 403 forbidden_role + audit |
| RF-02 | Auth de borda (redirect/401-JSON, gate /admin, impersonation edge) | Must | Dado sem sessão em /api, retorna 401 JSON; UI → /login |
| RF-03 | Isolamento por RLS + filtro manual sob service role | Must | Dado admin client, toda query filtra organization_id da fonte |
| RF-04 | MFA de sessão | Must | Dado admin com TOTP e sessão aal1, rota gateada retorna mfa_required |
| RF-05 | Tokens (convite HMAC, impersonation, MCP Bearer) | Should | Dado token expirado/revogado, é recusado |
| RF-06 | Contrato `ok()`/`fail()` + X-Request-Id | Must | Toda rota v1 responde `{data}`/`{error:{code,message}}` |
| RF-07 | Rate limit por IP + por conta | Should | Dado 5 senhas erradas na conta, bloqueia pela janela |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Segurança | `getUser` (nunca `getSession`) no backend | `proxy.ts:66`, `lib/auth/server.ts` | 🟢 |
| Segurança | Segredos comparados com `timingSafeEqual` | `invite-token.ts`, `impersonate/cookie.ts` | 🟢 |
| Segurança | Cookie SameSite=strict, httpOnly, secure | `proxy.ts` | 🟢 |
| Disponibilidade | Falha de permissão lança (não degrada silenciosamente) | `lib/auth/server.ts:185` | 🟢 |
| Correção | Role do banco = mesma função das RLS | `fn_user_role_in_org` | 🟢 |

## Critérios de Aceitação

```gherkin
Dado um usuário agent chamando uma rota que exige manager
Quando requireRole("manager") roda
Então retorna 403 forbidden_role e grava audit authz.denied

Dado um admin com TOTP cadastrado e sessão aal1
Quando chama rota gateada por requireRole("admin")
Então retorna 403 mfa_required (política de sessão)

Dado um webhook público sem cookie
Quando passa pela borda
Então não é barrado no proxy (public-path) e a auth acontece dentro da rota (HMAC/secret)
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| requireRole + role do banco | Must | Gate de toda rota autenticada |
| Isolamento multi-tenant | Must | Segurança de dados por construção |
| Auth de borda | Must | Primeira linha; 401-JSON para API |
| MFA de sessão | Must | Contém senha vazada/phishing |
| Tokens + rate limit | Should | Superfícies não-cookie e anti-abuso |

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/auth/require-role.ts` | `requireRole` | 🟢 |
| `lib/auth/server.ts` | `loadAuthUser`, `resolveActiveOrg`, `mfaEmDivida` | 🟢 |
| `lib/auth/types.ts` | `Role`, `ROLE_RANK`, `AuthUser` | 🟢 |
| `proxy.ts` | `proxy` | 🟢 |
| `lib/auth/public-paths.ts` | `isPublicPath` | 🟢 |
| `lib/auth/rate-limit.ts` | `authRateLimited`, `contaBloqueadaPorFalhas` | 🟢 |
| `lib/auth/invite-token.ts` | `signInviteToken`, `verifyInviteToken` | 🟢 |
| `lib/impersonate/*` | cookie/support | 🟢 |
| `lib/api/wrappers.ts` / `errors.ts` | `ok`, `fail`, `ApiErrorCodes` | 🟢 |
| `lib/supabase/{browser,server,admin}.ts` | clients | 🟢 |
| `lib/mcp/auth.ts` | `validateBearerToken` | 🟢 |
