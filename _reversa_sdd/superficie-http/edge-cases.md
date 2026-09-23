# Superfície HTTP — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Redirect HTML para consumidor JSON 🟢
`/api/*` sem sessão responde JSON 401 `{error:{code:"unauthenticated"}}` com `x-request-id`, nunca redirect HTML (que quebraria um cliente que espera JSON).

## EC-02 — Sub-path nascendo público de carona 🟢
Padrões de `public-paths` são ancorados com `$` de propósito (ex.: `/api/v1/contacts$`), para que um sub-path futuro não herde a dispensa de auth de borda.

## EC-03 — Cron duplicando disparo 🟢
Claim atômico (ex.: `.update(...).not("snooze_until","is",null)` é o lock): dois ticks concorrentes não disparam duplo. Audit condicional: varredura que não mudou nada não é mutação.

## EC-04 — Secret de webhook que não decifra 🟢
Se o secret não decifra (`hmacSkipped`), a validação HMAC é PULADA (disponibilidade > defesa opcional, precedente WAHA); mas assinatura presente e inválida sempre rejeita (`invalid_signature`).

## EC-05 — Captação duplicada 🟢
Idempotência por `external_id` (`uniq_crm_leads_org_source_external`): fast-path lê a linha existente, catch de `23505` cobre a corrida.

## EC-06 — Callback OAuth sem cookie 🟢
O navegador volta de outro site e o cookie `sameSite:strict` não viaja; a identidade vem do `state` assinado (HMAC de `INTERNAL_SECRET`) com nonce de uso único. A rota fica em `public-paths` (proxy não decide), com o guard dentro.

## EC-07 — Throw cru chegando ao cliente 🟢
Internal envolve `runAgent` em try/catch; webhook captura `ApiError` e mapeia `fail`, re-lançando só o inesperado; erro de DB é logado com o requestId e o cliente recebe frase de produto.

## EC-08 — MCP com envelope REST 🟢
`app/api/mcp` responde JSON-RPC 2.0 (`jsonRpcError`), não `{error:{code}}` REST; misturar os envelopes quebraria o cliente MCP.

## EC-09 — Convite alheio aceito por outra conta 🟢
`acceptInvite` exige match de e-mail entre o user do JWT e o token (`email_mismatch`); org/papel/convidador só do payload assinado, nada do body.

## EC-10 — `cookieSecure` por NODE_ENV 🟢
`cookieSecure` deriva do PROTOCOLO da URL: self-host em HTTP puro com `NODE_ENV=production` não descarta o cookie (evita loop de login).

## EC-11 — Handler service-role sem filtro de org 🟢
`createAdminClient` bypassa RLS; SEM gate automático para o filtro de `organization_id` — responsabilidade de quem escreve o handler (resolver de fonte confiável, nunca do body).

## EC-12 — Idade do PDF/validação em rota longa 🟡
`internal/agents/run` define `maxDuration=300` e `runtime="nodejs"` para acomodar execuções longas do agente sem timeout de borda.
