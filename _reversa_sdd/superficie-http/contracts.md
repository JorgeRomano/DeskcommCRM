# Superfície HTTP — Contratos

> Contratos: superfícies de topo, envelopes por superfície, allowlist de borda.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Contrato 1 — Superfícies de topo e auth
| Superfície | Auth | Envelope |
|---|---|---|
| `app/api/v1/**` (cookie) | `requireRole` (sessão RLS) | REST `{data,meta?}`/`{error}` |
| `app/api/v1/cron/**` | Bearer `INTERNAL_CRON_SECRET`/`INTERNAL_SECRET` (fail-closed) | REST |
| `app/api/v1/webhooks/**` | HMAC + path token | REST ou 303 |
| `app/api/internal/**` | `x-internal-secret` ou Bearer `INTERNAL_SECRET` | REST |
| `app/api/mcp` | Bearer `dsk_` contra `api_tokens` | JSON-RPC 2.0 |
| `app/api/v1/` (parcial, auth-dual) | cookie OU bearer via `auth-dual.ts` | REST |

Rotas auth-dual (crescem rota a rota): `contacts$`, `messages$`, `conversations/open-with-contact$`, `conversations/[^/]+/media$`. Toda rota bearer também precisa de entrada em `public-paths.ts`. 🟢

## Contrato 2 — `proxy.ts`
Ordem: `X-Request-Id` → `x-pathname` → detecção admin → `isPublicPath` → `getUser()` → não-autenticado (`/api/*` 401 JSON, UI redirect) → impersonation `/app*` → gate `/admin/*` → matcher exceto assets. 🟢

## Contrato 3 — Envelope MCP (JSON-RPC 2.0)
`McpAuthError` → `jsonRpcError(err.mcpCode, ...)`: `-32001`/401, `-32002`/403, `-32603`/500. `X-Request-Id` ainda setado. GET/POST/DELETE → `handle`. 🟢

## Contrato 4 — Webhook de captação
`app/api/v1/webhooks/in/[token]`: org de `webhook_sources.path_token` (404 se token < 8 chars ou fonte inativa); rate-limit por token (60/min → 429 `Retry-After`); HMAC `verifyInboundSignature` (`timingSafeEqual`); secret que não decifra pula validação; idempotência por `external_id` (`uniq_crm_leads_org_source_external`, fast-path + catch 23505). Resposta `ok({lead_id})` ou 303 para form post. 🟢

## Contrato 5 — Server Actions
`"use server"`; devolvem objeto tipado (`{ok:true} | {ok:false, error, details?}`), não `NextResponse`. Usam `revalidatePath`/`redirect`. Convite (`acceptInvite`): guard é o token assinado (`verifyInviteToken`), exige match de e-mail (JWT × token), org/papel/convidador só do payload assinado. 🟢

## Contrato 6 — Convenções de wire
Envelope: sucesso `{data, meta?}` (cursor `{cursor, has_more, total}`); erro `{error:{code, message, details?}}`; `X-Request-Id` sempre. JSON snake_case; dinheiro `_cents`+`currency`; datas ISO-8601 UTC; UUID v4. Erro só via `fail()`; nenhum throw cru ao cliente; `console.log` banido. 🟢
