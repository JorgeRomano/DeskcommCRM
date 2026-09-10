# Integrações Externas — Contratos

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO

## 1. Nuvemshop (OAuth + webhook) 🟢

| Item | Valor |
|---|---|
| Authorize | `GET {NUVEMSHOP_AUTH_BASE}/apps/{appId}/authorize?state=<csrf>` |
| Token | `POST {NUVEMSHOP_AUTH_BASE}/apps/authorize/token` (`grant_type=authorization_code`) |
| Resposta token | `{access_token, scope, user_id}` — `user_id` É o `storeId`; tokens não expiram |
| Webhook sig | header `x-linkedstore-hmac-sha256` = HMAC-SHA256 hex do body cru com `client_secret` |
| Callback público | `/api/v1/integrations/nuvemshop/callback` (public-path, state HMAC) |
| Verificação | `timingSafeEqual`, fail-closed |

## 2. Meta Conversions API (saída) 🟢

| Item | Valor |
|---|---|
| Endpoint | `POST https://graph.facebook.com/v22.0/{datasetId}/events` |
| Auth | `Authorization: Bearer <accessToken>` (header, nunca query) |
| Evento | `Purchase` (único hoje) |
| Identidade | `user_data.ctwa_clid` + `ph` (telefone SHA256 em array) |
| `action_source` | `business_messaging` + `messaging_channel: whatsapp` |
| `event_time` | segundos (não ms) |
| `custom_data` | `{ value: cents/100, currency: UPPER }` |
| Dedup | `event_id = <leadId>:Purchase` |
| Limites | evento >7 dias recusado; timeout 10s |
| Erros | 5xx/613/80004 → transitório; token/evento velho → permanente |

## 3. Webhook de captação de lead (entrada) 🟢

| Item | Valor |
|---|---|
| Rota | `POST /api/v1/webhooks/...` (per-source token) 🟡 |
| Auth | assinatura por fonte (`webhook_sources.secret`) 🟡 |
| Registro | `webhook_lead_captures` (`outcome: criado/duplicado/recusado`, `reject_reason`) |
| Limites | 60 campos × 2000 chars (corta com reticências, não recusa) |
| Idempotência | por fonte + dedup de contato 🟡 |
| Org | resolvida da FONTE (token), nunca do body |

## 4. Web Push (saída) 🟡

- VAPID; pipeline `emit → policy/prefs → deliver → web_push`. Detalhe não lido em profundidade.

## 5. Eventos internos consumidos/emitidos 🟢

| Evento | Papel |
|---|---|
| `lead.won` / `lead.stage_changed` | consumidos por `conversaoDeVendaHandler` |
| `nuvemshop.product_synced` | consumido pelo rag-indexer (catálogo) |
| `message.received` | consumido por `webPushInbound` 🟡 |

## Notas de segurança 🟢

- Todo webhook entrante verifica HMAC (fail-closed).
- Tokens OAuth cifrados at-rest (L-09); token de conversão nunca em query string.
- `google_ads` sem transporte (declarado `null`) — restrição registrada, não omitida (invariante 4).
