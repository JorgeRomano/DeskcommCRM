# Fluxograma — Ingestão de mensagem (WhatsApp → CRM)

> Gerado pelo **Arqueólogo** · Módulo: `lib/waha/ingest.ts` + `lib/channels/pos-entrada.ts` 🟢

## Visão geral do fluxo

```mermaid
flowchart TD
  A[Webhook WAHA] --> B{HMAC exigido?}
  B -- sim --> C[verifyHmacSha512<br/>timingSafeEqual]
  C -- inválido --> Z1[401 fail-closed]
  B -- não / válido --> D[parseChatId]
  D --> E{tipo de identidade}
  E -- group @g.us --> Z2[skip — não vira contato]
  E -- unknown --> Z3[emit whatsapp.chat_id_not_recognized]
  E -- phone/lid --> F[upsertContact via RPC atômica<br/>fn_upsert_wa_contact]
  F --> G[estamparAtribuicaoDoContato<br/>anúncio no 1º toque]
  G --> H[upsertConversation<br/>fn_upsert_wa_conversation]
  H --> I{é eco de envio nosso?<br/>ehEcoDeEnvioNosso janela 60s}
  I -- sim --> Z4[não re-silencia — era envio da IA/user]
  I -- fromMe humano --> J[pausarIaPorAtendimentoManual<br/>60min, renova, nunca encurta]
  H --> K[insert messages<br/>type via WA_TYPE_MAP]
  K --> L{inbound?}
  L -- sim --> M[aplicarEfeitosPosEntrada]
  L -- não --> N[trigger emite message.received se aplicável]
  M --> O[trg_messages_emit_event → message.received]
```

## `aplicarEfeitosPosEntrada` — a ordem é regra de negócio

```mermaid
flowchart TD
  A[mensagem inbound gravada] --> B[1. aplicarOptOut]
  B --> B1{ehPedidoDeOptOut?}
  B1 -- sim --> B2[grava is_blocked + audit contact.blocked]
  B1 -- não --> C
  B2 --> C[2. abrirDemanda]
  C --> C1[garantirLeadDaConversa]
  C1 --> C2{bloqueado?}
  C2 -- sim --> C3[motivo: contato_bloqueado — não cria]
  C2 -- não --> C4{já há lead open?}
  C4 -- sim --> C5[motivo: ja_existe]
  C4 -- não --> C6[cria crm_leads no funil is_default]
  C6 --> D[2b. avaliarCampanha]
  D --> D1{ai_gate = allowlist E casa campanha?}
  D1 -- sim --> D2[autorizarContatoParaIA]
  D1 -- não --> E
  D2 --> E[3. acelerarPipeline + pedirDespachoDoAgente]
  E --> E1[emit ai_agent.dispatch_requested]
```

**Invariante:** cada passo falha "para dentro" (log estruturado), NUNCA lança — uma exceção viraria 500 para o provider e uma tempestade de reentregas. Inverter opt-out↔lead faria quem pediu para sair virar oportunidade no funil.
