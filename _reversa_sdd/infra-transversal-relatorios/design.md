# Infra Transversal e Relatórios — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface (destaques)
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `ok<T>` | `(data, opts)` | `NextResponse` (`{data, meta?}`, X-Request-Id) |
| `fail` | `(code, message, status, opts)` | `NextResponse` (`{error}`) |
| `createClient` (server) | `()` | client RLS (async) |
| `createAdminClient` | `()` | client service-role (singleton) |
| `resolveAuthDual` | `(req, {requestId, resource, role, scope})` | `AuthDual` |
| `comIdempotencia<T>` | `(entrada)` | `DesfechoIdempotente` (executou/replay/conflito/em_curso) |
| `encryptKey`/`decryptKey` | `(...)` | `{ciphertext,iv,tag,last4}` / plaintext |
| `traduzir` | `(texto, idioma)` | tradução ou o próprio texto |

## Fluxo Principal — Resposta e clients
`ok`/`fail`/`noContent` sempre setam `X-Request-Id` (= opts ou `randomUUID`). `createClient` (server) usa cookie + `getUser()`; `browser` singleton com Realtime token via `/api/v1/auth/realtime-token`; `admin` service-role com filtro manual de org. `cookieSecure` do protocolo da URL. 🟢

## Fluxo Principal — Auth dual
Bearer → `validateBearerToken` → `ensureScope`+`ensureRole` → org do token, admin client, `via:"token"`. Senão → `requireRole` (mesmo gate), client RLS, `via:"session"`. A rota bearer também precisa de entrada em `public-paths`. 🟢

## Fluxo Principal — Idempotência
`hashDoCorpo` (SHA-256 do JSON ordenado) → lê linha viva (`classificar`: hash difere→conflito, `status_code null`→em_curso, senão→replay) → sem linha, INSERT de RESERVA (60s, índice único decide) → `23505` re-lê (vencida reescreve com guard de posse) → `executar()`; lança→libera reserva; ok→grava recibo terminal na mesma linha filtrada por hash. 🟢

## Fluxo Principal — Env
Zod `safeParse(process.env)` no import; tiers `requiredAlways`/`required`(só prod)/`diasDeRetencao`; leniência de build (`NEXT_PHASE=phase-production-build` semeia placeholders), boot real lança. Knobs digitados usam `z.string()`. 🟢

## Dependências
- `api` → `lib/mcp/auth` (auth-dual), `idempotency_keys`, `crypto`. 🟢
- `supabase` → `@supabase/ssr`, `@supabase/supabase-js`, `next/headers`. 🟢
- `reports`/`metrics` → RPCs `fn_activity_report`/`fn_atrito_metrics`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| `fail` não impõe o catálogo (ramo `string & {}`) | `api/wrappers.ts` | 🟢 |
| Código de erro nunca renomeado (versiona por /v2/) | `api/errors.ts` | 🟢 |
| `cookieSecure` do protocolo, não de NODE_ENV | `supabase/cookie-secure.ts` | 🟢 |
| Idempotência reserva antes do efeito (corrida fechada) | `api/idempotency.ts` | 🟢 |
| Env com `z.string()` para knobs digitados | `env.ts` | 🟢 |
| Mutação nunca retenta em timeout | `api/client.ts` | 🟢 |
| Eficiência sempre pareada com dano | `metrics/atrito.ts` | 🟢 |

## Estado Interno
- `idempotency_keys` (reserva + recibo TTL 24h); view `ai_provider_credentials_safe` (só last4); memo da chave AES. 🟢

## Observabilidade
- Logger JSON estruturado (`{level, msg, ts, ...ctx}`); `X-Request-Id` correlaciona com audit. 🟢

## Riscos e Lacunas
- 🔴 RPCs `fn_activity_report`/`fn_atrito_metrics`, tabela `idempotency_keys`, view `_safe`, policy `idempotency_tenant` — Data Master.
- 🟡 Os 24 schemas de entidade em `lib/schemas/` vistos pelo registro/validação, não campo a campo — reconferir ao escrever spec por entidade.
