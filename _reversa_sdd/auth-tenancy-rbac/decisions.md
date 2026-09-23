# Auth, Tenancy e RBAC — Decisões

> Referências: `_reversa_sdd/adrs/`. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## D-01 — Role efetivo do banco, nunca do cookie 🟢
`requireRole` lê `fn_user_role_in_org` — a mesma função SECURITY DEFINER que as policies RLS usam (fonte única de verdade). Membership revogada falha fechada. Ver ADR-0001 (multi-tenancy com RLS).

## D-02 — `getUser()` sempre, `getSession()` nunca 🟢
`getSession()` confia no cookie sem revalidar; toda leitura de sessão valida o JWT no servidor. — `auth/server.ts`.

## D-03 — MFA: cadastro é política, prova é sessão 🟢
`exigeCadastroDeMfa` soma plataforma+org (default não exige); `mfaEmDivida` obriga quem TEM fator a provar sempre, sem perguntar a política. — `auth/politica-mfa.ts`.

## D-04 — Gate de MFA dentro de `requireRole` 🟢
Layout não roda em rota de API; o gate migrou para o guard, posicionado após o rank (403 por falta de papel não revela estado de MFA de quem nem chegaria lá).

## D-05 — Falha alto em erro de permissões 🟢
Erro de infra na leitura de permissões não vira decisão de autorização — lança `auth_permissions_unavailable`. Incidente 2026-07-30 justificou. — `auth/server.ts`.

## D-06 — `ai_operator` fora de `user_organizations` 🟢
O papel do agente publicado existe só no token efêmero; a RLS segue intacta (o agente não é usuário). `fn_role_at_least` no banco não conhece o papel, e está certo.

## D-07 — Convite stateless + registro revogável 🟢
Token HMAC não requer linha para emitir; a linha `team_invites` existe para listar, dizer se o email saiu e permitir revogar (cancelar um token stateless exige algo contra o que verificar).

## D-08 — Provisionamento externo idempotente com marcador 🟢
Marcador `{integration, external_id}` em settings + app_metadata prova a origem; slug igual sem marcador → 409 (é outra empresa, devolver a chave dela seria entregar dados de terceiro).

## D-09 — Leitura de modo de cadastro pegajosa 🟢
`modoDeCadastro()` nunca lança e vale o último valor lido: instalação fechada não reabre no soluço do banco; quem nunca aplicou a migration não fecha. — `auth/politica-de-cadastro.ts`.

## D-10 — Rate limit por IP e por identificador 🟢
Duas contagens; sem IP identificável (kit sem proxy) o limite por IP não entra (evita DoS de custo zero); o `id` (5 falhas por conta) é o que barra brute force e não é configurável. — `auth/rate-limit.ts`.
