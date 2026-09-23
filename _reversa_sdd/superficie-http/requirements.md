# Superfície HTTP (`app/api/**` route handlers + Server Actions `app/actions/`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 14).

## Visão Geral
A borda de entrada: os route handlers REST sob `app/api/` e as Server Actions sob `app/actions/`. O `proxy.ts` (middleware de borda do Next 16) autentica a sessão antes da rota; cada superfície não-cookie tem guard próprio. `app/api/` tem três superfícies de topo: `internal/`, `mcp/`, `v1/`. 🟢

## Responsabilidades
- Autenticar na borda (`proxy.ts`): `X-Request-Id`, `x-pathname`, JWT, gate `/admin/*`, impersonation. 🟢
- Servir os route handlers REST versionados por path (`app/api/v1/**`, ~48 grupos). 🟢
- Isolar as superfícies não-cookie (cron, webhooks, internal, mcp, auth-dual), cada uma com guard próprio. 🟢
- Servir Server Actions (`"use server"`) espelhando a receita, devolvendo objeto tipado. 🟢
- Manter a allowlist de borda (`public-paths.ts`). 🟢

## Regras de Negócio
- Receita de route handler (nesta ordem): Zod → guard canônico → org de fonte confiável → query (RLS ou filtro manual) → audit se mutação → `ok()`/`fail()`. 🟢
- `proxy.ts`: `/api/*` sem sessão → 401 JSON (nunca redirect HTML); `/admin/*` gate antecipado por `fn_is_platform_admin`. 🟢
- Superfície não-cookie tem guard próprio; `public-paths` só diz que "o proxy não decide". 🟢
- Cron fail-closed (Bearer `INTERNAL_CRON_SECRET`/`INTERNAL_SECRET`); audit condicional (só quando muta). 🟢
- Webhook: org do PATH TOKEN, nunca do body; HMAC `timingSafeEqual`; idempotência por `external_id`. 🟢
- MCP responde envelope JSON-RPC 2.0, não o envelope REST. 🟢
- Server Action de convite: guard é o TOKEN assinado (não papel); exige match de e-mail; nada do body. 🟢
- Erro só via `fail()` na borda; nenhum `throw` cru chega ao cliente; `console.log` banido. 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Middleware de borda | Must | `X-Request-Id` sempre; `/api/*` null → 401 JSON; `/admin/*` gate |
| RF-02 | Receita de route handler | Must | Zod → guard → org confiável → query → audit → `ok()`/`fail()` |
| RF-03 | Superfícies não-cookie com guard próprio | Must | Cron Bearer fail-closed; webhook HMAC + path token; internal secret; mcp bearer |
| RF-04 | Idempotência de captação | Must | `external_id` repetido não duplica (23505 fast-path) |
| RF-05 | MCP em JSON-RPC 2.0 | Must | `McpAuthError` → `jsonRpcError` (-32001/-32002/-32603) |
| RF-06 | Server Actions tipadas | Should | Devolve `{ok:true} \| {ok:false, error, details?}`; `revalidatePath`/`redirect` |
| RF-07 | Allowlist de borda ancorada | Must | Padrões com `$` para sub-path não nascer público |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | Segredos comparados em tempo constante (`timingSafeEq`) | `internal/agents/run`, webhook | 🟢 |
| Segurança | Rate-limit por token no webhook (60/min) | `webhooks/in/[token]` | 🟢 |
| Segurança | Claim atômico no cron (dois ticks não duplicam) | `cron/snooze-watcher` | 🟢 |
| Segurança | Handler service-role filtra org manualmente (sem gate automático) | convenções | 🟢 |
| Corretude | Nenhum throw cru chega ao cliente (try/catch + `fail`) | `internal/agents/run`, webhook | 🟢 |
| Observabilidade | Erro de DB logado com requestId; cliente recebe frase de produto | convenções | 🟢 |

## Critérios de Aceitação
```gherkin
Dado uma requisição a /api/v1/* sem sessão
Quando o proxy avalia
Então responde 401 unauthenticated em JSON com X-Request-Id

Dado um webhook com assinatura HMAC inválida
Quando a rota valida
Então audita e responde 401 invalid_signature

Dado um cron tick que não mudou nada
Quando o handler termina
Então NÃO audita (varredura sem mutação não é evento)
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Borda (RF-01) | Must | Autentica antes da rota |
| Receita (RF-02) | Must | Padrão de todo handler |
| Não-cookie (RF-03) | Must | Cada superfície com guard próprio |
| Idempotência webhook (RF-04) | Must | Captação sem duplicata |
| MCP JSON-RPC (RF-05) | Must | Contrato do protocolo |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `proxy.ts` | `proxy(request)` | 🟢 |
| `lib/auth/public-paths.ts` | `isPublicPath`, `PUBLIC_PATHS` | 🟢 |
| `app/api/v1/contact-tags/route.ts` | handler cookie-authed (amostra) | 🟢 |
| `app/api/v1/cron/snooze-watcher/route.ts` | handler cron (amostra) | 🟢 |
| `app/api/internal/agents/run/route.ts` | handler internal (amostra) | 🟢 |
| `app/api/mcp/route.ts` | handler MCP (JSON-RPC) | 🟢 |
| `app/api/v1/webhooks/in/[token]/route.ts` | handler webhook (amostra) | 🟢 |
| `app/actions/settings/updateProfile.ts`, `team/acceptInvite.ts` | Server Actions (amostra) | 🟢 |
