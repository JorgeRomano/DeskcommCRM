# Caso de Uso: RBAC (`requireRole`) — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal
1. `loadAuthUser()` → 401 `unauthenticated` se null. 🟢
2. Support não-`active` → 403 `forbidden`. 🟢
3. Org: `opts.organizationId` (org do recurso) ou `resolveActiveOrg`; ausente → 403 `forbidden_tenant`. 🟢
4. `allowPlatformAdmin && is_platform_admin && !support` → bypass. 🟢
5. `rpc("fn_user_role_in_org", {p_org})`; erro → 500 `internal_error`. 🟢
6. Gate de MFA: `rank >= min && mfaEmDivida()` → audit + 403 `mfa_required`. 🟢
7. `rank < min` → audit `authz.denied` (fire-and-forget) + 403 `forbidden_role`. 🟢
8. Sucesso: `{ ok:true, user, org: {...org, role: effectiveRole} }`. 🟢

## Papéis
`viewer(1) | agent(2) | ai_operator(3) | manager(4) | admin(5)`. `ai_operator` só no token efêmero. `VisibilityMode` restringe só `agent`; a RLS (`fn_can_view_conversation`) garante. 🟢

## Dependências
- `auth/server` (`loadAuthUser`, `mfaEmDivida`), RPC `fn_user_role_in_org`, `lib/audit`.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Guard único (matriz advisória proibida; gate `lint:role-rank`) | `auth/require-role.ts` | 🟢 |
| Gate de MFA após o rank | `auth/require-role.ts` | 🟢 |
| `roleAtLeast` não decide 401/403 sozinho | `auth/types.ts` | 🟢 |

## Riscos e Lacunas
- 🔴 `fn_user_role_in_org` e `fn_role_at_least` vivem no baseline (Data Master).
