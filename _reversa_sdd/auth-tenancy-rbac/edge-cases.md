# Auth, Tenancy e RBAC — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — PostgREST fora do ar tratado como "sem org" 🟢
Incidente 2026-07-30: restart do Docker deixou o PostgREST em `name resolution failed`; o descarte silencioso do erro fez todos os cards de admin sumirem. Agora `loadAuthUser` lança `auth_permissions_unavailable` (falha alto).

## EC-02 — Duas cópias de `@supabase/auth-js` 🟢
`ehSessaoAusente` compara por `error.name`, não `instanceof`, porque há 2.111.0 e 2.112.1 na árvore e `instanceof` só acertaria a cópia importada pelo teste.

## EC-03 — TOTP forçado na primeira tela do self-host 🟢
`install.sh` cria o dono como platform admin; a regra antiga `isPlatformAdmin || role==="admin"` forçava TOTP em toda instalação. Agora as duas origens somam e o default é não exigir; `platform_admins.mfa_required` era decorativo e passou a ser lido.

## EC-04 — Sessão aal1 chamando rota de API 🟢
O gate de MFA vivia no layout, que não roda em rota de API: uma sessão `aal1` de admin com TOTP chamava direto ~33 rotas gateadas. O gate migrou para `requireRole`, após o rank.

## EC-05 — Token de convite alheio no signup próprio 🟢
`decidirConviteDoSignup` é puro e FALHA FECHADA: `user_metadata.invite_token` é gravável pelo usuário; a autoridade é a assinatura HMAC + comparação do `payload.email` com o email confirmado. Divergente → recusa.

## EC-06 — Provisionamento externo sob replay/falha parcial 🟢
`reencontrarECompletar` só conclui por replay sobre estado COMPLETO; antes devolvia `replay:true` sobre org sem admin (INSERT do vínculo morreu por timeout) e respondia 200 "tudo certo" para sempre.

## EC-07 — Slug colidindo dois ids externos 🟢
`slugDoProvisionamento` usa hash SHA256 do `externalId` (não `slugify`, que corta em 32 chars e podia colidir num replay). Slug igual sem marcador igual → `ProvisionConflictError` 409.

## EC-08 — DoS de custo zero no login 🟢
Sem `x-forwarded-for` (kit sem proxy), a versão anterior jogava todos num balde global e 60 requisições anônimas trancavam o login inteiro. Agora `clientIp()` devolve `null` e o limite por IP não entra; o `id` (5 falhas por conta) barra brute force.

## EC-09 — Modo de cadastro no soluço do banco 🟢
`modoDeCadastro()` nunca lança; se o banco não responde vale o último valor lido com sucesso (instalação fechada continua fechada; quem nunca aplicou a migration 0253 continua aberta). Memo em `globalThis` (Next instancia o módulo 2×).

## EC-10 — Viés de módulo em recovery codes 🟢
`recovery-codes.ts` usa rejection sampling (`MAX_VALID = floor(256/31)*31`) para evitar viés; alfabeto sem 0/O, 1/I/L.

## EC-11 — Cookie de impersonation vencido com HMAC válido 🟢
`verify` checa expiração mesmo com HMAC válido (nunca confia em cookie vencido). Edge usa `constantTimeEqual` manual (sem `node:crypto`).

## EC-12 — Nome do atendente sem service role 🟢
`nomesDosAtendentes` devolve Map VAZIO (não Map de nulls) com log quando não há service role — `null` declarado, para não confundir "não consegui ler" com "não tem nome".

## EC-13 — `garantirAdminDaOrganizacao` sobre acesso removido 🟢
`23505` sai em silêncio (não vira update): um sistema de fora não ressuscita acesso que o admin da empresa removeu (linha revogada ou dono rebaixado devolve `23505`).
