# Voz / Telefonia — Contratos

> Contratos: (a) cliente REST WaCalls, (b) eventos SSE WaCalls, (c) ARI/AudioSocket SIP, (d) vocabulário de chamada.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Contrato 1 — `WacallsClient` (REST, 1:1 upstream)
Todo request leva `Authorization: Bearer <token>`. `/healthz` é a única rota aberta. 🟢

| Método TS | Rota upstream | Nota |
|---|---|---|
| `createSession(name)` | `POST /api/sessions` | já inicia pareamento; QR sai na `/api/events` |
| `listSessions()` | `GET /api/sessions` | |
| `deleteSession(id)` | `DELETE /api/sessions/{sid}` | sozinho não desparia |
| `logoutSession(id)` | `POST /api/sessions/{sid}/logout` | derruba vínculo com WhatsApp |
| `startCall(sid, clientId, phone)` | `POST /api/sessions/{sid}/calls` | `X-Client-Id`=dono; `record` NUNCA true (LGPD) |
| `exchangeWebrtc(sid, callId, sdpOffer)` | `POST .../calls/{id}/webrtc` | relay puro de SDP |
| `acceptCall / rejectCall / endCall` | `POST .../accept`, `.../reject`, `DELETE .../{id}` | |
| `history(sid, {limit, cursor})` | `GET .../history` | envelope `{calls, nextCursor}` (keyset) |

Regra crítica: NUNCA chamar `/pair` (troca cliente mas não refaz `s.calls`). 🟢

## Contrato 2 — Eventos SSE WaCalls (`/api/events`)
| Evento | Handler | Efeito |
|---|---|---|
| `call-list` | inline | snapshot na reconexão; única fonte com `direction` |
| `incoming` | inline | insere `voice_calls` `direction=inbound status=ringing` |
| `auth-state` | `handleAuthState` | pareamento/desvínculo |
| `call-status` | `handleCallStatus` | upsert canônico das duas direções (sem `direction` no evento) |
| `call-ended` | `handleCallEnded` | fecha chamada + efeitos |
| `session-list`/`session-qr`/`incoming-claimed`/`call-quality` | — | pura UI |

## Contrato 3 — SIP: ARI + AudioSocket
- `originateCall(params)`: entrega ao dialplan `voice-agent-out`/`s` (não Stasis); `endpoint = "PJSIP/<destino>@org-<uuid>-trunk-endpoint"`; `channelId` gerado antes e passado como `AUDIOSOCKET_UUID`. 🟢
- `connectAriEvents(onEvent)`: WS `?app=<ARI_APP>&api_key=<user:pass>&subscribeAll=true`. `AriEvent = {type, channel?, cause_txt?}`. 🟢
- AudioSocket TCP (porta 9092): 1º frame `0x01` = UUID (16 bytes) → resolve `voice_calls` por `asterisk_channel_id`. OpenAI Realtime configurada para `audio/pcmu` (μ-law); conversão no CRM. 🟢

## Contrato 4 — HTTP `/api/v1/calls`
- `POST`: cria `voice_calls provider='sip' status='ringing'`, origina via ARI; `requireSupportWrite()` antes do RBAC `manager`; sem trunk → `422 trunk_not_configured`; falha originate → `502 originate_failed`. 🟢
- `GET`: lista `provider='sip'`; `mapStatusParaApi` traduz vocabulário compartilhado (`connected→in_progress`; `ended`+reason). 🟢

## Contrato 5 — Vocabulário de chamada (`voip/call-vocabulary.ts`)
`CallDirection = outbound | inbound`. `CallStatus (API) = ringing | in_progress | completed | no_answer | busy | failed | canceled` (derivado, não coluna). `CallHandledBy = human | ai | ai_then_human`. `PhoneNumberRoutingMode = ai | human | ai_then_human`. `AiAgentChannel = whatsapp | voice`. `voice_calls.status` (`starting/ringing/connected/ended`) é vocabulário do WaCalls upstream, fora do invariante. 🟢
