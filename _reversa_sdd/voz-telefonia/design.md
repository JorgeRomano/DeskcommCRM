# Voz / Telefonia — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `chamadaDeVozLigada` | `(escolhaDaOrg, instalacaoOferece)` | `boolean` (puro) |
| `estadoDaVoz` | `(...)` | `{ligada, instalacaoOferece, escolhaDaOrg, motivo}` |
| `exigirVozLigada` | `(...)` | `null \| Response` (3 códigos) |
| `WacallsClient(baseUrl, apiToken)` | classe | métodos 1:1 com upstream |
| `runVoiceCallsBridgeLoop` | `(pool, cfg, log, signal)` | loop de vida (sai no abort) |
| `podeEncerrar` | `(call, userId)` | `boolean` |
| `resolverNumeroDiscavel` | `(...)` | `{numero, fonte}` |
| `originateCall` | `(params)` | chamada SIP de saída |
| `guardarTrunk` | `(p)` | persiste trunk cifrado |

## Fluxo Principal — WaCalls (voz WhatsApp)
1. Opt-in de duas perguntas (`instalacaoOferece && (escolhaDaOrg ?? false)`). 🟢
2. Guarda de rota (`exigirVozLigada`, falha fechada). 🟢
3. `createSession` já pareia (QR sai na `/api/events`); nunca `/pair`. 🟢
4. Ponte SSE `runVoiceCallsBridgeLoop`: `pumpSse` → `despacharEventoWacalls` por tipo (`call-list`, `incoming`, `auth-state`, `call-status`, `call-ended`). 🟢
5. `handleCallStatus` infere sentido (declarado > `dono ? outbound : inbound`); `on conflict` só reescreve `direction` se declarado. 🟢
6. `handleCallEnded`: fecha chamada, `devolverAVozDaIa`, chamada perdida → inbox, `event_log voice_call.ended`, 3 tipos de atividade. 🟢

## Fluxo Principal — SIP (telefonia por IA)
1. `POST /api/v1/calls`: `requireSupportWrite()` antes do RBAC `manager`; `channelId` gerado antes do originate. 🟢
2. `originateCall` entrega ao dialplan (`voice-agent-out`, extension `s`), que chama `AudioSocket(uuid, voice-agent:9092)`. 🟢
3. Worker `voice-agent`: entrada via `handleStasisStart` (resolve org, cria contato/lead, UUID separado, `continueDialplan`). 🟢
4. AudioSocket TCP: 1º frame `0x01` = UUID; acha `voice_calls`; `getActiveVoiceAgent`; RAG em paralelo; `AudioSocketCallBridge` (OpenAI Realtime); μ-law ↔ PCM16. 🟢
5. `finalizeAudioSocketCall` grava `ended`, `duration_ms`, `transcript`. 🟢

## Fluxos Alternativos
- Número discável falha aberta (disca cadastro). 🟢
- `handleChannelDestroyed`: fecha chamada SIP de saída presa em `ringing` → `ended/timeout`. 🟢
- Contato SIP novo nasce `source:'voip'` (nunca `fn_upsert_wa_contact`). 🟢

## Dependências
- `lib/voice/*` → `lib/channels/*`, `lib/wacalls/client`. 🟢
- `lib/wacalls/events-bridge.ts` → `lib/channels/phone-variants`, `lib/leads/agent-activity`, `pg`. 🟢
- `workers/voice-agent` → `lib/voip/ariClient`, `lib/ai/agents`, `lib/ai/knowledge/busca`, `./audioSocketBridge`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Opt-in por `&&`, não `??` (não liga para todas as orgs) | `voice/opt-in.ts` | 🟢 |
| Guarda de voz falha fechada; desligar não passa pela guarda | `voice/guarda.ts` | 🟢 |
| Nunca `/pair` (quebra `s.calls`) | `wacalls/client.ts` | 🟢 |
| `originateCall` ao dialplan, não Stasis (limitação externalMedia) | `voip/ariClient.ts` | 🟢 |
| `channelId` gerado antes do originate (fecha race com StasisStart) | `voip/ariClient.ts`, `calls/route.ts` | 🟢 |
| Silêncio de IA com teto 2h anti-morte | `wacalls/events-bridge.ts` | 🟢 |

## Estado Interno
- `voice_calls` (unificada), `org_voice_calls` (opt-in), `voip_trunk_settings` (trunk cifrado). Ponte mantém 1 conexão SSE persistente. 🟢

## Observabilidade
- `event_log voice_call.ended` (registro, status `done`); atividades de contato (`voice_call`/`voice_call_missed`/`voice_call_unanswered`); transcript SIP. 🟢

## Riscos e Lacunas
- 🔴 `asterisk/pjsip.conf`/`extensions.conf` e `fn_resolve_inbound_number` vivem fora de `lib/` (aplicados manualmente por VPS, migration 0349) — topologia Asterisk é lacuna para validação humana.
- 🟡 `workers/voice-agent/audioSocketBridge.ts` lido só por referência nesta passagem.
- 🟡 Ponte WebRTC humano→navegador do SIP não implementada ("esqueleto"); `mode:'human'` grava dono mas não abre áudio.
- 🟡 Ligação encerrada com ponte SSE caída não vem no snapshot; linha segue aberta até o teto de 2h.
