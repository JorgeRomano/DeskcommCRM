# Canais e Mensageria — Contratos Externos

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO

## 1. Webhook de entrada (WAHA → CRM) 🟢

| Item | Valor |
|---|---|
| Rota | `POST /api/v1/webhooks/waha` (global) e `POST /api/v1/webhooks/waha/[token]` (per-tenant) |
| Auth | HMAC-SHA512 no header (`WAHA_HMAC_SECRET`); fail-closed quando exigido |
| Público | sim (bypassa auth de cookie — `public-paths.ts`); auth mora na rota |
| Eventos aceitos | `message.any`, `message.ack`, `message.edited`, `message.revoked`, `session.status`, `state.change` |
| Idempotência | `unique(organization_id, external_id)` → 23505 = 200 sem duplicar |
| Resposta | 200 sempre que possível (evita reentrega em massa); erro de efeito não vira 5xx |

**Payload (`WahaEnvelope`/`WahaPayload`, schema Zod em `lib/waha/envelope.ts`):** 🟡 campos usados: `from`, `body`, `type`, `fromMe`, `media.{url,mimetype}`, `_data.{notifyName,pushName,message,key.remoteJidAlt,key.participantAlt}`.

## 2. Envio (CRM → WAHA) 🟢

| Operação | Endpoint WAHA | Notas |
|---|---|---|
| Texto | `POST /api/sendText` | `{session, chatId, text, reply_to?}`; teto 15s |
| Mídia | `POST /api/{endpoint}` | `convert:true` em vídeo/áudio; teto 30s |
| Presença | `POST /api/{session}/presence` | sessão no CAMINHO; `typing/recording/paused` |
| Vcard | `POST /api/sendContactVcard` | exige `checkContactExists` antes (nono dígito BR) |
| Sessões | create/start/stop/logout/delete | `WahaSessionError{operation, httpStatus}` |

- Auth: header `X-Api-Key` (plaintext); o container compara com `sha512:` do hash.
- Erros expõem só `waha_<operation>_<status>`, nunca o corpo.

## 3. Eventos internos emitidos 🟢

| Evento | Quando | Consumidores |
|---|---|---|
| `message.received` | mensagem persistida (trigger) | workers de IA, sentimento, reatividade de follow-up |
| `ai_agent.dispatch_requested` | inbound de contato elegível | drain do engine → `inbound_turn` |
| `whatsapp.chat_id_not_recognized` | identidade desconhecida | observabilidade (anomalia) |

## 4. Estado da janela (interno → UI) 🟢

`EstadoDaJanela = { tipo: "sem_restricao" } | { tipo: "aberta", restanteMs } | { tipo: "fechada", fechadaHaMs: number\|null }`. `LIMIAR_URGENTE_MS = 2h`.

## 5. Restrição de canal (doutrina) 🟢

`docs/doctrine/restricao-de-canal.md`: nenhuma feature nomeia o provider fora de `lib/channels/`. Restrição não aplicável = `skipped: not_applicable` no trace (invariante 4), nunca omitida. Auto-restrição (anti-ban) × hetero-restrição (janela 24h) não se generalizam.
