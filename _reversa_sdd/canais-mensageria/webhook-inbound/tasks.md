# Caso de Uso: Webhook Inbound — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] RPCs de upsert e trigger de emissão disponíveis

## Tarefas
- [ ] T-01, Implementar autenticação fail-closed
  - Origem no legado: `lib/waha/webhook-auth.ts`, `lib/channels/inbound.ts`
  - Critério de pronto: 3 regras em ordem; HMAC com `timingSafeEqual`
  - Confiança: 🟢
- [ ] T-02, Implementar roteamento e persistência idempotente
  - Origem no legado: `lib/waha/ingest.ts`, `envelope.ts`
  - Critério de pronto: dispatch por tipo; guarda ReDoS/timestamp; idempotência 23505; não emite evento
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Assinatura errada → bad_signature
- [ ] TT-02, message + message.any não duplicam
- [ ] TT-03, 1MB de chatId processa linear (ReDoS)

## Ordem Sugerida
1. T-01 → T-02.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
