# Caso de Uso: Borda e Sessão — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal — `proxy.ts`
1. Injeta `x-request-id` + `x-pathname`. 🟢
2. `isPublicPath(pathname)` → passa sem auth. 🟢
3. `createServerClient` (cookie `sb-deskcomm-auth`, `SameSite=strict`, `httpOnly`) → `getUser()`. Null: `/api/*` 401 JSON; UI redirect `/login?next=`. 🟢
4. `/app/*`: verifica cookie de impersonation (HMAC + expiry no Edge, sem DB); inválido → deleta o cookie de apresentação. 🟢
5. `/admin/*` (exceto `/admin/forbidden`): `fn_is_platform_admin`; erro/false → redirect (gate antecipado). 🟢

## Fluxo Principal — Sessão
1. `getUser()` valida JWT; `ehSessaoAusente` distingue deslogado de falha transitória. 🟢
2. `Promise.all` de `platform_admins` + `user_organizations`; erro → lança. 🟢
3. `escolherMembroAtivo(memberships, cookieOrg)`. 🟢

## Dependências
- `@supabase/ssr`, RPC `fn_is_platform_admin`, `lib/impersonate/cookie-edge`.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| `/api/*` responde 401 JSON, não HTML | `proxy.ts` | 🟢 |
| Allowlist ancorada com `$` | `public-paths.ts` | 🟢 |
| Uma resposta para escopo e idioma | `auth/server.ts` (`escolherMembroAtivo`) | 🟢 |

## Riscos e Lacunas
- 🟡 Callbacks OAuth dependem do `state` HMAC porque o cookie `SameSite=strict` não viaja de outro site.
