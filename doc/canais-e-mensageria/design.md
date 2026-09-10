# Canais e Mensageria — Design Técnico

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Interface

### Ingestão (`lib/waha/ingest.ts`) 🟢

| Símbolo | Assinatura | Retorno | Observação |
|---------|-----------|---------|------------|
| `parseChatId` | `(chatId: string)` | `ChatIdentity` | phone/lid/group/unknown |
| `verifyHmacSha512` | `(rawBody, signatureHeader\|null, secret)` | `boolean` | timingSafeEqual, fail-closed |
| `resolveMessageType` | `(p: WahaPayload)` | `string` | type → chave NOWEB → MIME → text |
| `telefoneAlternativoDe` | `(p: WahaPayload)` | `string\|null` | E.164 de `remoteJidAlt` |

### Cliente/envio (`lib/waha/client.ts`, `send.ts`) 🟢

| Símbolo | Assinatura | Observação |
|---------|-----------|------------|
| `WahaClient.sendMessage` | `(session, chatId, text, replyTo?)` | `reply_to` só se presente; lança `waha_<status>` |
| `WahaClient.sendMedia` | `(session, chatId, plan)` | teto 30s (ffmpeg convert) |
| `WahaClient.setPresence` | `(session, chatId, "typing"\|"recording"\|"paused")` | sessão no CAMINHO |
| `resolveWahaChatId` | `(ResolveWahaChatIdInput)` | grupo → lid → wa_identity lid → phone@c.us |
| `getWahaClient` | `()` | `null` se env não configurado (UI mostra banner) |

### Janela e fronteira 🟢

- `estadoDaJanela(provider, lastInboundAt, agora)` → `sem_restricao | aberta{restanteMs} | fechada{fechadaHaMs}`.
- `assertCurrentServiceBoundary(expected, current)` → lança `StaleServiceBoundaryError`.

## Fluxo Principal (ingestão) 🟢

1. Webhook chega em `/api/v1/webhooks/waha` (global) ou `/waha/[token]` (per-tenant).
2. Verifica HMAC-SHA512 (fail-closed quando exigido).
3. `parseChatId` resolve identidade; grupo → skip, unknown → evento de anomalia.
4. `fn_upsert_wa_contact` / `fn_upsert_wa_conversation` (atômico, evita corrida `message`×`message.any`).
5. `estamparAtribuicaoDoContato` (anúncio no 1º toque).
6. Insere `messages` (`WA_TYPE_MAP` para o tipo; `unique(org, external_id)` = idempotência).
7. Se inbound: `aplicarEfeitosPosEntrada` → opt-out → nascimento do lead → campanha → despacho (`ai_agent.dispatch_requested`).
8. Trigger `trg_messages_emit_event` emite `message.received`.

## Fluxos Alternativos 🟢

- **Eco de envio próprio:** `ehEcoDeEnvioNosso` (janela 60s) evita silenciar a IA por eco.
- **Resposta `fromMe` humana:** `pausarIaPorAtendimentoManual` (60min).
- **Grupo/desconhecido:** não vira contato; unknown emite `whatsapp.chat_id_not_recognized`.
- **WAHA fora do ar:** `wahaFriendlyError` classifica (ENOTFOUND ≠ container caído); teto de relógio evita socket pendurado.

## Dependências 🟢

- `event-log` (emite `message.received`, `ai_agent.dispatch_requested`, anomalias).
- `leads` (`garantirLeadDaConversa`, atribuição de anúncio).
- `opt-out` (`ehPedidoDeOptOut`).
- `escalacao` (`pausarIaPorAtendimentoManual`).
- `ai/elegibilidade` (autorização por campanha).
- `supabase` (admin — service role).
- WAHA (REST + webhook), engine NOWEB.

## Decisões de Design Identificadas 🟢

| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Resolução atômica via RPC (anti-corrida NOWEB) | migration 0027; `handleInbound` | 🟢 |
| Corte manual de sufixo (anti-ReDoS) | `semSufixoDeChat`, `telefoneAlternativoDe` | 🟢 |
| `lid:` antes do telefone no envio | `resolveWahaChatId` (migration 0122) | 🟢 |
| `message.any` (não `message`) no webhook | `docker-compose.prod.yml` WHATSAPP_HOOK_EVENTS | 🟢 |
| Erro do WAHA sem corpo (PII) | `WahaSessionError`, `wahaFriendlyError` | 🟢 |

## Estado Interno 🟢

- `messages` (direction, type, status, sent_via, external_id, reply_to_message_id).
- `conversations` (status, bot_silenced_until, last_inbound_at, service_revision).
- `channel_sessions` (status WAHA, metadata.ai_gate).

## Observabilidade 🟢

- `event_log`: `message.received`, `whatsapp.chat_id_not_recognized`.
- `audit`: `contact.blocked`.
- Logs estruturados sem corpo/telefone.

## Riscos e Lacunas

- 🟡 `lib/messaging/*` (media, presença, contact-card) e `lib/inbox/*` (comandos da conversa) não lidos em profundidade.
- 🟡 Canal Zernio e adapters Meta (`lib/channels/zernio`, `meta`) referenciados, não aprofundados.
- 🔴 SQL das RPCs de upsert não lido.
