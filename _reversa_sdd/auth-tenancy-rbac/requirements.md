# Auth, Tenancy e RBAC (`auth`, `tenants`, `team`, `users`, `impersonate`, `proxy.ts`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 8).

## Visão Geral
A camada que responde três perguntas de toda requisição: **quem é** (autenticação), **em que empresa está** (tenancy) e **o que pode fazer** (autorização). Multi-tenant com RLS desde o dia 1; os helpers server-side resolvem a identidade do JWT validado ANTES de tocar tabela tenant-aware, e quando usam service role filtram `user_id`/`organization_id` de fonte confiável (JWT, cookie validado, path token, segredo), nunca do body. O role efetivo vem SEMPRE do banco (`fn_user_role_in_org`/`fn_is_platform_admin`), a mesma função SECURITY DEFINER que as policies RLS usam. 🟢

## Responsabilidades
- Resolver sessão e organização ativa (`loadAuthUser`, `resolveActiveOrg`). 🟢
- Aplicar RBAC canônico por role (`requireRole`) e guard de plataforma (`requirePlatformAdmin`). 🟢
- Aplicar política de MFA (cadastro é política; provar na sessão é obrigação de quem tem fator). 🟢
- Middleware de borda (`proxy.ts`): `X-Request-Id`, valida JWT, bloqueia `/admin/*`, verifica impersonation. 🟢
- Emitir/aplicar convites (token HMAC stateless + registro `team_invites`). 🟢
- Provisionar tenant (self-service e externo idempotente). 🟢
- Rate limit da superfície de auth; API key `dsk_` de organização; acompanhamento administrativo. 🟢

## Regras de Negócio
- Sempre `getUser()` no servidor, NUNCA `getSession()`. — `auth/server.ts` 🟢
- Falha de leitura de permissões FALHA ALTO (`auth_permissions_unavailable`), não degrada para "sem org". 🟢
- MFA: as duas origens (plataforma e org) SOMAM, nunca se anulam; default é NÃO exigir. — `auth/politica-mfa.ts` 🟢
- Quem TEM fator prova SEMPRE (`mfaEmDivida` não pergunta a política). 🟢
- `requireRole`: role efetivo do banco; gate de MFA DEPOIS do rank e antes do sucesso. — `auth/require-role.ts` 🟢
- `ai_operator` é papel do agente publicado (token efêmero), nunca em `user_organizations`. 🟢
- Convite: token HMAC stateless + registro `team_invites`; nada do `user_metadata` é autoridade. — `auth/convite-no-signup.ts` 🟢
- Provisionamento externo idempotente por (integração, id externo); slug igual sem marcador → 409. — `auth/provision.ts` 🟢
- Cadastro: banco acima do `.env`; leitura pegajosa (nunca lança; vale o último valor lido). — `auth/politica-de-cadastro.ts` 🟢
- Rate limit: por IP e por identificador hasheado; sem IP identificável, o limite por IP não entra. — `auth/rate-limit.ts` 🟢
- Segredos comparados em tempo constante; tokens só como hash SHA256, plaintext uma vez. 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Validar JWT no servidor | Must | Usa `getUser()`; `getSession()` proibido |
| RF-02 | Falhar alto em erro de permissões | Must | Erro de query lança `auth_permissions_unavailable` |
| RF-03 | RBAC canônico por role do banco | Must | `requireRole` lê `fn_user_role_in_org`; rank < min → 403 |
| RF-04 | Gate de MFA em rota de API | Must | `aal1` com fator cadastrado → 403 `mfa_required` (após o rank) |
| RF-05 | Convite HMAC + registro revogável | Must | Token stateless; `team_invites` permite revogar; nada do body |
| RF-06 | Provisionamento externo idempotente | Must | Replay completa o que faltou; slug sem marcador → 409 |
| RF-07 | Rate limit da superfície de auth | Should | Por IP e por identificador hasheado; sem IP, só o `id` limita |
| RF-08 | Middleware de borda | Must | Injeta `X-Request-Id`; `/api/*` null → 401 JSON; `/admin/*` → gate |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | Comparações em tempo constante (`timingSafeEqual`) | `invite-token.ts`, `cron-auth.ts`, `impersonate/cookie.ts` | 🟢 |
| Segurança | Tokens só como hash SHA256; plaintext uma vez | `tenants/api-key.ts` | 🟢 |
| Segurança | Anti open-redirect por allowlist | `auth/safe-next.ts` | 🟢 |
| Segurança | `convite-no-signup` compara email confirmado (anti-sequestro de org) | `auth/convite-no-signup.ts` | 🟢 |
| Segurança | Cookie de sessão `SameSite=strict`, `httpOnly` | `proxy.ts` | 🟢 |
| Disponibilidade | Leitura de modo de cadastro pegajosa (não reabre no soluço do banco) | `auth/politica-de-cadastro.ts` | 🟢 |
| Segurança | Recovery codes com rejection sampling (sem viés de módulo) | `auth/recovery-codes.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado uma sessão aal1 de admin com TOTP cadastrado
Quando chama uma rota gateada por requireRole("admin")
Então recebe 403 mfa_required (após passar o rank)

Dado um erro transitório na query de permissões
Quando loadAuthUser roda
Então lança auth_permissions_unavailable (não degrada para "sem org")

Dado um provisionamento externo repetido com o mesmo id externo
Quando provisionExternalTenant roda de novo
Então completa o que faltou (replay) sem duplicar
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Sessão/JWT (RF-01/02) | Must | Base de toda requisição |
| RBAC canônico (RF-03) | Must | Autorização sem alternativa |
| Gate de MFA (RF-04) | Must | Fechou buraco de rota de API |
| Convite/provisionamento (RF-05/06) | Must | Onboarding sem lixo |
| Rate limit (RF-07) | Should | Anti brute force |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/auth/server.ts` | `loadAuthUser`, `resolveActiveOrg`, `requireAuth`, `mfaEmDivida` | 🟢 |
| `lib/auth/politica-mfa.ts` | `exigeCadastroDeMfa` | 🟢 |
| `lib/auth/require-role.ts` | `requireRole`, `roleAtLeast` | 🟢 |
| `lib/auth/requirePlatformAdmin.ts` | `requirePlatformAdmin` | 🟢 |
| `proxy.ts` | middleware de borda | 🟢 |
| `lib/auth/invite-token.ts` | token HMAC stateless | 🟢 |
| `lib/auth/provision.ts` | `ensureTenantForUser`, `provisionExternalTenant` | 🟢 |
| `lib/auth/rate-limit.ts` | `authRateLimited` | 🟢 |
| `lib/tenants/api-key.ts` | `rotateIntegrationApiKey` | 🟢 |
| `lib/impersonate/*` | `signImpersonateCookie`, `readSupportContext` | 🟢 |
