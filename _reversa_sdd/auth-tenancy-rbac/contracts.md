# Auth, Tenancy e RBAC — Contratos

> Contratos: (a) guards de autorização, (b) token de convite, (c) API key de org, (d) cookie de impersonation, (e) superfícies não-cookie.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Contrato 1 — `requireRole`
```ts
const authz = await requireRole("manager", { requestId, organizationId? });
if (!authz.ok) return authz.response; // 401/403 já montado
// authz.org.role = role efetivo do banco
```
Respostas: 401 `unauthenticated`, 403 `forbidden`, 403 `forbidden_tenant`, 403 `forbidden_role`, 403 `mfa_required`, 500 `internal_error`. `allowPlatformAdmin` faz bypass do rank do tenant. 🟢

## Contrato 2 — Papéis (`auth/types.ts`)
`Role = viewer(1) | agent(2) | ai_operator(3) | manager(4) | admin(5)`. `PAPEIS_HUMANOS` (CHECK de `user_organizations`) só tem os quatro humanos; `ai_operator` só no token efêmero do agente. `VisibilityMode = all | own_and_unassigned | own` (restringe só `agent`; a RLS garante). 🟢

## Contrato 3 — Token de convite (HMAC stateless)
`<base64url(body)>.<base64url(hmac_sha256)>`. Payload: `{invite_id, email, organization_id, role, exp, iat?, invited_by?, interface_settings?}`. TTL 24h. `timingSafeEqual` após checar comprimento. Secret: `INVITE_TOKEN_SECRET → INTERNAL_SECRET → "dev-fallback"` (fallback inalcançável em produção). Não requer linha no banco para emitir; a revogação é pela linha `team_invites`. 🟢

## Contrato 4 — API key de organização (`dsk_`)
`dsk_<prefix>_<secret>`, SHA256 em `token_hash` (plaintext nunca persistido). Escopos: `["mcp:read", "mcp:write", "role:agent", <integrationScope>]`. Sem `actor:ai_agent` de propósito (`deriveActor` → `api_token`). Reemitir REVOGA a anterior (nunca duas vivas para o mesmo par org/integração). 🟢

## Contrato 5 — Cookie de impersonation
`<base64url(payload)>.<base64url(hmac_sha256)>`, HttpOnly + Secure + `SameSite=Lax`, TTL 1h. Server assina (`node:crypto`), Edge só verifica (Web Crypto, `constantTimeEqual` por XOR). Aditivo à sessão — não troca o usuário Supabase. `IMPERSONATE_COOKIE_SECRET` < 32 chars → lança (caller responde 503). 🟢

## Contrato 6 — Superfícies não-cookie (`public-paths.ts`)
"Público" = o proxy não decide; muitas têm guard próprio dentro:
| Superfície | Auth |
|---|---|
| `app/api/v1/cron/` | Bearer `INTERNAL_CRON_SECRET` (fail-closed) |
| `app/api/internal/` | `x-internal-secret` |
| `app/api/mcp/` | Bearer `dsk_` contra `api_tokens` |
| `app/api/v1/webhooks/` | HMAC + path token |
| `app/api/v1/` (parcial) | cookie OU bearer via `auth-dual.ts` |
| callbacks OAuth | `state` HMAC (`INTERNAL_SECRET`) + nonce único |

Regra: adicionar path aqui remove a checagem de borda — só com guard próprio dentro da rota. 🟢

## Contrato 7 — Cron auth
`autorizaCron(req)`: `Authorization: Bearer <secret>` ou `x-cron-secret`, tempo constante contra `INTERNAL_CRON_SECRET` e `INTERNAL_SECRET`. Fail-closed (sem token/secret → false). 🟢
