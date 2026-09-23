# Superfície HTTP — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Estrutura
`app/api/` tem três superfícies: `internal/` (1 rota, `x-internal-secret`), `mcp/` (1 rota, Bearer `dsk_`), `v1/` (~337 rotas, ~48 grupos). Reconferir: `git ls-files 'app/api/**/route.ts' | wc -l`. Grupos de `v1`: admin, ads, agenda, ai, anuncios, attendants, audit, auth, automation-rules, calls, channels, contacts, conversations, cron, demandas, extensions, external-db, financeiro, health, leads, lead-captures, lgpd, marca, messages, metrics, notifications, onboarding, phone-numbers, pipelines, products, prospecting, reports, settings, system, tags, tasks, team, tenants, voice, voip, webhook-sources, webhooks (e outros). 🟢

## Fluxo Principal — `proxy.ts` (Edge, em ordem)
1. `X-Request-Id` (lê ou gera). 2. `x-pathname` (resposta + request encaminhada). 3. Detecção admin (host `admin.`/pathname `/admin`; ramo por host é NOOP no self-host). 4. Curto-circuito `isPublicPath`. 5. `createServerClient` + `getUser()` (nunca `getSession()`). 6. Não-autenticado: `/api/` → 401 JSON; UI → redirect. 7. Impersonation em `/app*` (HMAC no Edge, sem DB). 8. Gate `/admin/*` (`fn_is_platform_admin`). 9. Matcher exceto assets. 🟢

## Fluxo Principal — Receita de handler (6 passos)
Zod valida input externo → guard canônico (`requireRole`/`requirePlatformAdmin`/secret/HMAC) → `organization_id` de fonte confiável → query (RLS pelo client de sessão, ou filtro manual de org com service role) → `audit()` se mutação → `ok()`/`fail()`. GET só-leitura omite passos 1 e 5. 🟢

## Fluxos Alternativos (por superfície)
- Cron: Bearer fail-closed; `createAdminClient` cross-tenant; claim atômico; audit condicional. 🟢
- Internal: `x-internal-secret` OU Bearer com `timingSafeEq`; Zod; `runAgent` em try/catch; `maxDuration=300`. 🟢
- MCP: `validateBearerToken`; `McpAuthError` → JSON-RPC 2.0; transport por request. 🟢
- Webhook: org do path token; rate-limit por token; HMAC `timingSafeEqual`; secret que não decifra pula validação (disponibilidade > defesa opcional, precedente WAHA); idempotência por `external_id`. 🟢
- Server Actions: `"use server"`; Zod → guard (`loadAuthUser`/token) → mutação → audit → `revalidatePath`/`redirect`; devolve objeto tipado. 🟢

## Dependências
- `proxy.ts` → `lib/auth/public-paths`, `lib/impersonate/cookie-edge`, RPC `fn_is_platform_admin`. 🟢
- Handlers → `lib/api/wrappers`, `lib/auth/require-role`, `lib/mcp/auth` (mcp/auth-dual), `lib/supabase/*`, `lib/audit`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| `/api/*` responde 401 JSON, não HTML | `proxy.ts` | 🟢 |
| Receita de 6 passos como padrão | convenções + amostras | 🟢 |
| Webhook: org do path token, nunca do body | `webhooks/in/[token]` | 🟢 (ADR-0006) |
| MCP em JSON-RPC, não REST | `app/api/mcp/route.ts` | 🟢 |
| Convite: guard é o token assinado, exige match de e-mail | `actions/team/acceptInvite.ts` | 🟢 |
| Handler service-role filtra org manualmente (sem gate automático) | convenções | 🟢 |

## Observabilidade
- `X-Request-Id` correlaciona com audit; erro de DB logado com requestId (cliente recebe frase de produto). 🟢

## Riscos e Lacunas
- 🟡 As ~337 rotas de `v1` não foram lidas uma a uma (amostra representativa por superfície + contagens estruturais); reconferir contagem com `git ls-files` e mapear cada endpoint ao escrever a spec de superfície.
- 🔴 Tabelas/RLS que os handlers assumem (`webhook_sources`, `api_tokens`, `conversations`, `contacts`, `crm_leads`) — Data Master.
