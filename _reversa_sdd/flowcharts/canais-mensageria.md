# Fluxogramas — Canais e mensageria

> 🟢 CONFIRMADO salvo indicação. Fontes: `lib/channels/**`, `lib/waha/**`, `lib/inbox/**`, `lib/atendimento/**`.
> Nível `detalhado`: um fluxo por módulo + fluxos por função com lógica não-trivial.

## Ingest WAHA — mensagem inbound vira conversa+mensagem (`lib/waha/ingest.ts`)

```mermaid
flowchart TD
  A[Webhook WAHA] --> B[authenticateWahaWebhook: HMAC SHA-512 em ordem]
  B --> C{autorizado?}
  C -- nao --> Z1[401]
  C -- sim --> D[lerRoteamentoWaha: resolve tenant + arquiva bruto]
  D --> E[conferirContratoWaha: valida envelope Zod loose]
  E --> F[dispatchWahaEvent por event/fromMe]
  F -- message fromMe=false --> G[handleInbound]
  F -- message fromMe=true --> H[handleOutboundFromUserPhone]
  F -- message.ack --> I[handleAck: ack>=2 delivered, >=3 read]
  F -- session.status --> J[handleSessionStatus + sincronizarSaudeDaConexao]
  G --> G1{parseChatId: group? nao enderecavel?}
  G1 -- group/unknown --> G2[return / trace event]
  G1 -- phone|lid --> G3[fn_upsert_wa_contact + fn_upsert_wa_conversation atomicos]
  G3 --> G4[INSERT message sent_via=external_device status=delivered]
  G4 --> G5{erro 23505 unique org+external_id?}
  G5 -- sim --> G6[dedup: info + re-acelera pipeline]
  G5 -- nao --> G7[markConversation + audit message.received]
  G7 --> G8[aplicarEfeitosPosEntrada]
  G8 --> G9[trigger trg_messages_emit_event emite o evento, NAO o ingest]
```

## `ehEcoDeEnvioNosso` — eco do nosso envio vs digitação humana (`ingest.ts:76`)

```mermaid
flowchart TD
  A[message fromMe=true] --> B[busca nossa linha outbound na mesma conversa]
  B --> C{sem external_id ainda in-flight?}
  C -- nao --> D[nao e eco]
  C -- sim --> E{mesmo corpo?}
  E -- sim --> F{dentro de JANELA_DO_ECO_MS=60000?}
  F -- sim --> G[e eco: descarta silenciosamente]
  F -- nao --> D
  E -- media --> H[prova fraca: mesmo tipo in-flight] --> C
  D --> I[pausarIaPorAtendimentoManual: silencia bot]
```
> Regra: gravar é tolerante (dupe > perda); silenciar o bot é estrito (não silencia na dúvida).

## `semSufixoDeChat` — guarda ReDoS (`ingest.ts`)

```mermaid
flowchart TD
  A[chatId controlado por atacante] --> B[lastIndexOf nos 4 terminadores de linha]
  B --> C[indexOf @ depois da ultima quebra]
  C --> D{achou @?}
  D -- sim --> E[slice ate o @ — O(n) linear]
  D -- nao --> F[string inteira]
```
> Substitui `replace(/@.*$/,"")` que era O(n^2) sobre payload.from. 1MB em 0.7ms.

## `capabilitiesOf` + janela 24h (`capabilities.ts` + `janela.ts`)

```mermaid
flowchart TD
  A[provider] --> B{esta em CHANNEL_CAPABILITIES?}
  B -- nao --> X[throw unknown_channel_provider fail-closed]
  B -- sim --> C[caps]
  C --> D{caps.freeformOutsideWindow?}
  D -- sim --> E[EstadoDaJanela=sem_restricao ex: waha]
  D -- nao --> F{ha lastInboundAt?}
  F -- nao --> G[fechada fechadaHaMs=null]
  F -- sim --> H{restanteMs>0?}
  H -- sim --> I[aberta restanteMs]
  H -- nao --> J[fechada fechadaHaMs]
```

## Efeitos pós-entrada — a ordem é a regra (`pos-entrada.ts:150`)

```mermaid
flowchart TD
  A[EntradaDeMensagem] --> B{contato do numero interno?}
  B -- sim --> Z[skip: safety-belt]
  B -- nao --> C[1 aplicarOptOut: is_blocked se stop_keyword]
  C --> D[4 guardarOrigemDaPagina antes da demanda]
  D --> E[2 abrirDemanda garantirLeadDaConversa idempotente; recusa bloqueado]
  E --> F[2b avaliarCampanha: autoriza IA se ai_gate=allowlist]
  F --> G[acelerarPipelineDeEventos]
  G --> H[3 pedirDespachoDoAgente: emite ai_agent.dispatch_requested]
  H --> I[nenhum passo pode lancar: msg ja persistida]
```

## Comando da conversa — máquina de estados (`inbox/comando-da-conversa.ts`)

```mermaid
flowchart TD
  A[FatosDoComando + agora] --> B{assigned_to_user_id?}
  B -- sim --> C[humano]
  B -- nao --> D{status em closed/archived/resolved?}
  D -- sim --> E[encerrada]
  D -- nao --> F{silencioVigente OU force_human OU is_blocked?}
  F -- sim --> G[aguardando: fila]
  F -- nao --> H{automaticoDaOrg === false?}
  H -- sim --> I[ninguem]
  H -- nao --> J[automatico]
  G --> K[motivo: blocked antes de travado antes de silencio]
```
> `silencioVigente`: valor ilegível = SILENCIADO (fail-closed). Janela 24h e status fechado ficam FORA de propósito.

## Fronteira de serviço — CAS por revisão (`atendimento/fronteira.ts`)

```mermaid
flowchart TD
  A[expected: ServiceBoundary] --> B[readCurrentServiceBoundary]
  B --> C{org/contact/conversation batem?}
  C -- nao --> X[throw StaleServiceBoundaryError]
  C -- sim --> D{service_revision igual?}
  D -- nao --> X
  D -- sim --> E{status terminal ou demanda_fechada_em?}
  E -- sim --> X
  E -- nao --> F{demandaTrocou expected vs current?}
  F -- sim --> X
  F -- nao --> G[efeito autorizado]
```
> `demandaTrocou`=false quando expected.demanda_id===null (abrir 1a demanda nao e novo serviço).

## Roteador de e-mail — por configuração, nunca por falha (`email/roteador.ts`)

```mermaid
flowchart TD
  A[sendEmail args] --> B{isSmtpConfigured getSmtpConfig?}
  B -- sim --> C[enviarPorSmtp via=smtp]
  B -- nao --> D{resendConfigurada?}
  D -- sim --> E[enviarPelaResend via=resend]
  D -- nao --> F[EmailDeliveryError not_configured]
  C --> G[EmailSendResult tag via]
  E --> G
```
> Sem fallback em falha: greylisting 4xx faria double-send e esconderia erro. Erros nao achatados para send_failed.
