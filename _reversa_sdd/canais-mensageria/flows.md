# Canais e Mensageria — Fluxos

> Fluxos distintos não cobertos totalmente no `design.md`. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo 1 — Mensagem inbound (WAHA/QR) até o despacho do agente
```mermaid
flowchart TD
  A[POST webhook WAHA] --> B[authenticateWahaWebhook: fail-closed]
  B --> C[lerRoteamentoWaha: resolve tenant + arquiva bruto]
  C --> D[dispatchWahaEvent por tipo]
  D -->|message/message.any| E[parseChatId + dataDoTimestamp tolerante]
  E --> F[fn_upsert_wa_contact / fn_upsert_wa_conversation]
  F --> G[handleInbound: INSERT message, idempotencia 23505]
  G --> H[trigger trg_messages_emit_event emite message.received]
  G --> I[aplicarEfeitosPosEntrada em ordem]
  I --> I1[1. opt-out]
  I1 --> I2[4. guardarOrigemDaPagina]
  I2 --> I3[2. abrirDemanda]
  I3 --> I4[2b. avaliarCampanha]
  I4 --> I5[3. pedirDespachoDoAgente: ai_agent.dispatch_requested]
```

## Fluxo 2 — Eco de envio pelo celular do operador
```mermaid
flowchart TD
  A[message.any fromMe=true] --> B{ehEcoDeEnvioNosso? janela 60s}
  B -->|sim| C[grava como eco, não silencia bot]
  B -->|não| D[pausarIaPorAtendimentoManual: humano digitou]
```
Regra: gravar é tolerante (dupe > perda); silenciar é estrito (não silencia na dúvida). 🟢

## Fluxo 3 — Envio de saída (outbound) via adapter
```mermaid
flowchart LR
  A[send_message no turno] --> B[cadeia before_send: janela/cap/pacing]
  B -->|pass| C[getAdapter provider]
  C --> D[adapter.send OutboundEnvelope]
  D --> E[externalId retornado]
```
Regra: o adapter não decide janela/cap — só traduz formato e envia. 🟢

## Fluxo 4 — Saúde da conexão
```mermaid
flowchart TD
  A[observação de status] --> B{STATUS_QUE_AVISAM?}
  B -->|SCAN_QR_CODE/FAILED/STOPPED| C[avisoDaConexao: warn/critical]
  C --> D[sincronizarSaudeDaConexao: dedup por episódio]
  D --> E[só quem observou fecha o episódio]
```
🟢
