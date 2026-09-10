# Auth e Multi-tenancy — Design Técnico

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Interface

### Gate e resolução 🟢

| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `requireRole` | `(min: Role, opts?: RequireRoleOpts)` | `RoleCheck` (`{ok:true,user,org}` \| `{ok:false,response}`) |
| `loadAuthUser` | `()` | `AuthUser \| null` |
| `resolveActiveOrg` | `(user: AuthUser)` | `ActiveOrg \| null` |
| `mfaEmDivida` | `()` | `boolean` |
| `proxy` | `(request: NextRequest)` | `NextResponse` |

### Contrato de API 🟢

| Símbolo | Assinatura |
|---------|-----------|
| `ok<T>` | `(data, opts?: {status,meta,requestId,headers})` → `NextResponse<{data,meta?}>` |
| `fail` | `(code, message, status, opts?)` → `NextResponse<{error:{code,message,details?}}>` |
| `Actor` | `user \| ai_agent \| webhook_source` (`id` ≠ `agent_id`) |
| `HandlerCtx` | `{organization_id, actor, requestId, idioma?, serviceBoundary?}` |

## Fluxo Principal — `requireRole` 🟢

1. `loadAuthUser` (JWT) → 401 se ausente.
2. Suporte inativo → 403.
3. Resolve org (cookie/`opts.organizationId`) → 403 forbidden_tenant se ausente.
4. `platform_admin` + `allowPlatformAdmin` + sem suporte → sucesso (bypass).
5. `fn_user_role_in_org` (role do banco).
6. Gate de MFA de sessão (`mfaEmDivida`) → 403 mfa_required.
7. `rank < min` → 403 forbidden_role + audit.

## Fluxo Principal — borda (`proxy.ts`) 🟢

1. Injeta `x-request-id`/`x-pathname`.
2. `isPublicPath` → segue (auth na rota).
3. `getUser` → sem user: `/api/` = 401 JSON; UI = redirect `/login?next=`.
4. `/app/*`: valida impersonation cookie edge (HMAC + expiry).
5. `/admin/*`: `fn_is_platform_admin` → senão `/admin/forbidden`.

## Fluxos Alternativos 🟢

- **Sessão de suporte:** role derivado de `access_mode` (full→admin, senão viewer); banco autoritativo.
- **MCP Bearer:** `dsk_` → SHA256 contra `api_tokens`; scope/role em `scopes`.
- **Convite:** HMAC-SHA256 stateless; role Zod-restrito; TTL 24h.
- **Erro de permissão (infra):** lança `auth_permissions_unavailable` (não degrada para sem-org).

## Dependências 🟢

- `supabase` (clients browser/server/admin; RPCs de role).
- `audit` (authz.denied), `i18n` (traduzir), `impersonate` (support), Upstash (rate limit).

## Decisões de Design Identificadas 🟢

| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Role efetivo do banco | `require-role.ts:96` (`fn_user_role_in_org`) | 🟢 |
| `getUser` nunca `getSession` | `proxy.ts:66`, doutrina CLAUDE.md | 🟢 |
| Falha ALTO em erro de permissão | `server.ts:185` | 🟢 |
| MFA como política de sessão | `mfaEmDivida` (aal1) | 🟢 |
| `fail()` aceita `(string & {})` → códigos em errors.ts | `wrappers.ts`, `errors.ts` | 🟢 |

## Estado Interno 🟢

`user_organizations` (role, revoked_at), `platform_admins` (mfa_required), `organizations` (settings.security.mfa_required, locale, legal_name), `api_tokens` (token_hash, scopes), `platform_support_sessions`.

## Observabilidade 🟢

- `api_audit_log`: authz.denied (reason required_role/mfa_required), login_failed (hashEmail).
- `X-Request-Id` em toda resposta (correlaciona com audit).

## Riscos e Lacunas

- 🔴 SQL de `fn_user_role_in_org`, `fn_is_platform_admin`, `fn_support_context`, `fn_can_view_conversation` e policies RLS não lidos (só call sites).
- 🔴 Matriz rota-a-rota completa (169 handlers) não enumerada — ver `permissions.md`.
- 🟡 `app/actions/*` (Server Actions de auth/team/settings) não lidas em profundidade.
