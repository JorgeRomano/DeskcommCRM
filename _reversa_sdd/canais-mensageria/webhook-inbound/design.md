# Caso de Uso: Webhook Inbound — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal (WAHA)
1. `authenticateWahaWebhook` (fail-closed, `MIN_SECRET_LEN=16`, `verifyHmacSha512` SHA-512 + `timingSafeEqual`). 🟢
2. `lerRoteamentoWaha` (mínimo p/ resolver tenant + arquivar bruto); `conferirContratoWaha` (contrato cheio). Schema `looseObject` (`.nullish()`) — loose evita descartar mensagem. 🟢
3. `dispatchWahaEvent` roteia por tipo. 🟢
4. `parseChatId` (guarda ReDoS), `dataDoTimestamp` (tolerante). 🟢
5. Upsert atômico via RPC; `handleInbound` INSERT + idempotência `23505`; `aplicarEfeitosPosEntrada`. 🟢

## Fluxos Alternativos
- `message.any fromMe`: `handleOutboundFromUserPhone` distingue eco (janela 60s) de humano.
- `message.ack`/`edited`/`revoked`: handlers próprios; `bareWaMessageId` (slice após último `_`).
- Zernio: `handleInboundWebhook` verifica assinatura no handler (esquema por canal).

## Dependências
- RPCs `fn_upsert_wa_contact`/`fn_upsert_wa_conversation`, trigger `trg_messages_emit_event`, `channels/pos-entrada`.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Fail-closed com default de assinatura false + defesa de rede | `waha/webhook-auth.ts` | 🟢 |
| Schema loose para não descartar mensagem | `waha/envelope.ts` | 🟢 |
| Upsert atômico fecha corrida NOWEB | `waha/ingest.ts` | 🟢 |

## Riscos e Lacunas
- 🟡 Confirmar configuração de rede (Caddy) que complementa o default de assinatura.
