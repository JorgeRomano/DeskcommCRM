# Auth, Tenancy e RBAC — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface (destaques)
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `loadAuthUser` | `()` | `Promise<AuthUser \| null>` (memoizada por request) |
| `resolveActiveOrg` | `(authUser)` | `ActiveOrg \| null` |
| `requireRole` | `(min: Role, opts)` | `Promise<RoleCheck>` |
| `requirePlatformAdmin` | `()` | `PlatformAdminContext` |
| `exigeCadastroDeMfa` | `({role, isPlatformAdmin, plataformaExige, empresaExige})` | `boolean` |
| `mfaEmDivida` | `()` | `boolean` |
| `provisionExternalTenant` | `(input)` | tenant (idempotente) |
| `authRateLimited` | `(action, identifier, limits)` | resultado |

Papéis: `Role = viewer(1) | agent(2) | ai_operator(3) | manager(4) | admin(5)`. `ai_operator` só no token efêmero do agente publicado. `VisibilityMode = all | own_and_unassigned | own`. 🟢

## Fluxo Principal — Sessão
1. `getUser()` valida o JWT no servidor. 🟢
2. `ehSessaoAusente` distingue estado normal (deslogado) de falha transitória (loga). 🟢
3. `Promise.all`: `platform_admins` + `user_organizations` (2 embeds do mesmo `organizations`), ordenado por `accepted_at` nulls-first, depois `organization_id`. 🟢
4. Erro em qualquer query → lança `auth_permissions_unavailable` (falha alto). 🟢
5. `escolherMembroAtivo(memberships, cookieOrg)` decide org ativa (a mesma resposta para escopo de dados e idioma). 🟢

## Fluxo Principal — RBAC (`requireRole`)
1. `loadAuthUser` → 401 se null. 2. Support não-active → 403. 3. Org resolvida (`opts.organizationId` do recurso ou ativa) → 403 `forbidden_tenant` se ausente. 4. Bypass de platform admin (se `allowPlatformAdmin`). 5. Role efetivo via `fn_user_role_in_org` (banco). 6. Gate de MFA (após rank, antes do sucesso). 7. `rank < min` → 403 `forbidden_role`. 8. Sucesso. 🟢

## Fluxo Principal — Borda (`proxy.ts`)
Injeta `x-request-id`/`x-pathname` → `isPublicPath` passa sem auth → `getUser()` (null: `/api/*` 401 JSON, UI redirect) → `/app/*` verifica cookie de impersonation (HMAC no Edge, sem DB) → `/admin/*` `fn_is_platform_admin` (gate antecipado). 🟢

## Fluxos Alternativos
- Convite: token HMAC `<body>.<sig>` + registro `team_invites` (status derivado); `aplicarConvite` checa revogação e grava `accepted_at`; `fn_accept_team_invite` monta o vínculo (org/papel do token, nada do body). 🟢
- Provisionamento externo: marcador `{integration, external_id}` em settings + app_metadata; `reencontrarECompletar` só conclui por replay completo. 🟢
- Cadastro: `modoDeCadastro()` pegajoso (globalThis, TTL 30s, geração contra lost-update). 🟢

## Dependências
- `auth` → Supabase Auth (`@supabase/ssr`), RPCs `fn_user_role_in_org`/`fn_is_platform_admin`, `lib/ai/dispatcher/rate-limit`, `crypto`. 🟢
- `impersonate` → `node:crypto` (server), Web Crypto (Edge), RPC `fn_support_context`. 🟢
- `tenants/api-key` → `api_tokens`, SHA256. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Role efetivo do banco (mesma fn das policies RLS) | `auth/require-role.ts` | 🟢 |
| MFA soma plataforma+org; default não exige | `auth/politica-mfa.ts` | 🟢 |
| Gate de MFA movido para `requireRole` (layout não roda em API) | `auth/require-role.ts` | 🟢 |
| Falha alto em erro de permissões | `auth/server.ts` | 🟢 |
| Provisionamento externo idempotente com marcador | `auth/provision.ts` | 🟢 |
| Rate limit por IP + identificador; sem IP só `id` | `auth/rate-limit.ts` | 🟢 |

## Estado Interno
- `user_organizations`, `platform_admins`, `organizations`, `team_invites`, `api_tokens`, `user_recovery_codes`, `platform_settings`. Cookie `active_org`, cookie de impersonation (HMAC). 🟢

## Observabilidade
- `audit()` em `authz.denied` (fire-and-forget), `member.invited`, `tenant.created_*`, `token.created/revoked` (actorUserId null para ação de máquina). 🟢

## Riscos e Lacunas
- 🔴 Funções SECURITY DEFINER (`fn_user_role_in_org`, `fn_is_platform_admin`, `fn_accept_team_invite`, `fn_support_context`) e policies RLS vivem no baseline — Data Master.
- 🟡 Secret HMAC de convite com fallback `"dev-fallback"` (inalcançável em produção).
- 🔴 Rate limit ausente em crons e MCP (limitação conhecida — validar com threat-model).
- 🟡 Fallback de `com-nome-do-atendente.ts` candidato a código morto se nunca disparar.
