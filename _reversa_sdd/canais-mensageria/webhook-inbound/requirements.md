# Caso de Uso: Webhook Inbound

> Sub-unit de `canais-mensageria`. A porta de entrada de toda mensagem recebida.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Recebe, autentica, roteia e persiste eventos de webhook dos provedores (WAHA, Zernio), garantindo idempotência e disparando os efeitos pós-entrada. 🟢

## Responsabilidades
- Autenticar o webhook fail-closed por canal. 🟢
- Rotear eventos por tipo (`message`, `message.ack`, `session.status`...). 🟢
- Persistir mensagem com idempotência e disparar efeitos pós-entrada. 🟢

## Regras de Negócio
- WAHA: assinatura errada → `bad_signature`; exigida e ausente → `signature_required`. 🟢
- Ingest não emite `message.received` (trigger emite). 🟢
- Idempotência por `unique(organization_id, external_id)` + `23505`. 🟢
- Guarda ReDoS e timestamp tolerante (nunca derruba o webhook). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Autenticar fail-closed | Must | Assinatura errada rejeitada |
| RF-02 | Rotear eventos por tipo | Must | `message`/`ack`/`status` tratados por handler próprio |
| RF-03 | Persistir com idempotência | Must | `external_id` repetido não duplica |

## Critérios de Aceitação
```gherkin
Dado um webhook WAHA autenticado
Quando handleInbound processa a mensagem
Então persiste com idempotência e dispara efeitos pós-entrada
E não emite message.received (o trigger emite)
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/waha/webhook-auth.ts` | `authenticateWahaWebhook` | 🟢 |
| `lib/waha/ingest.ts` | `dispatchWahaEvent`, `handleInbound` | 🟢 |
| `lib/channels/inbound.ts` | `handleInboundWebhook` (Zernio) | 🟢 |
