# Canais e Mensageria — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `capabilitiesOf` | `(provider)` | `ChannelCapabilities` (throws `unknown_channel_provider`) |
| `getAdapter` | `(provider)` | `ChannelAdapter` (fail-closed) |
| `aplicarEfeitosPosEntrada` | `(admin, entrada: EntradaDeMensagem)` | `Promise<void>` |
| `estadoDaJanela` | `(provider, lastInboundAt, agora)` | `EstadoDaJanela` |
| `sincronizarSaudeDaConexao` | `(admin, sessao, saude, apelido, origem)` | `Promise<void>` |
| `handleInboundWebhook` | `(admin, input)` | `Promise<...>` (Zernio/social) |
| `authenticateWahaWebhook` | `(input)` | `{ok, signatureVerified}` (fail-closed) |
| `comandoDaConversa` | `(fatos, agora)` | `Comando` |
| `assertCurrentServiceBoundary` | `(expected, current)` | `void` (throws `StaleServiceBoundaryError`) |

`ChannelProvider` = `waha | meta_cloud | zernio | zernio_social | wacalls`; `ProviderDeMensagem` exclui `wacalls` (voz não transporta mensagem). `ChannelAdapter`: `send(envelope)`, `resolveRecipient`, `isConfigured`, `codes`, opcionais (`fetchInboundMedia`, `sendTemplate`, `signalTyping`, `checkHealth`...). O adapter é tradutor de formato puro; janela/cap/horário ficam no `before_send`. 🟢

## Fluxo Principal — Inbound (WAHA)
1. `authenticateWahaWebhook` (fail-closed, `MIN_SECRET_LEN=16`). 🟢
2. `dispatchWahaEvent`: `message`/`message.any` → `handleInbound`/`handleOutboundFromUserPhone` (por `fromMe`); `message.ack`/`edited`/`revoked`; `session.status`. 🟢
3. `parseChatId` (guarda ReDoS `semSufixoDeChat`); `dataDoTimestamp` tolerante. 🟢
4. Upsert atômico via RPC `fn_upsert_wa_contact`/`fn_upsert_wa_conversation` (fecham corrida NOWEB). 🟢
5. `handleInbound`: INSERT message (`sent_via:"external_device"`), idempotência `23505`; `markConversation`; `audit`; `aplicarEfeitosPosEntrada`. 🟢

## Fluxo Principal — Efeitos pós-entrada (ordem é lei)
Guarda "número interno" → (1) opt-out → (4) `guardarOrigemDaPagina` → (2) `abrirDemanda` → (2b) `avaliarCampanha` → `acelerarPipelineDeEventos` → (3) `pedirDespachoDoAgente`. Nenhum efeito lança; opt-out falha loga `error`, o resto `warn`. 🟢

## Fluxos Alternativos
- Eco de envio do celular: `ehEcoDeEnvioNosso` (janela 60s) distingue eco de digitação humana; se não é eco → `pausarIaPorAtendimentoManual`. Gravar é tolerante (dupe > perda); silenciar é estrito. 🟢
- Zernio/social: assinatura verificada no `inbound.ts` (esquema por canal), não na rota. 🟢

## Dependências
- `waha/ingest.ts` → `channels/{health,pos-entrada,phone-variants}`, `escalacao`, `leads/atribuicao-de-anuncio`, `audit`. 🟢
- `channels/pos-entrada.ts` → `leads`, `opt-out/deteccao`, `ai/elegibilidade`. 🟢
- `atendimento/fronteira-server.ts` → `ai/agents/operation`, `agenda`, `agent-engine/queue`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Adapter puro; janela/cap no before-send | `channels/types.ts:169` | 🟢 |
| Ordem dos efeitos pós-entrada guardada por testes | `pos-entrada.ts:150` | 🟢 |
| Ingest não emite evento (trigger emite) | `waha/ingest.ts` | 🟢 |
| GET-before-PUT na config da sessão WAHA | `waha/client.ts` | 🟢 |
| `@lid` antes de phone no roteamento (0122) | `waha/send.ts` | 🟢 |

## Estado Interno
- `channel_sessions`, `channel_session_health` (espelho de saúde), `conversations`, `messages`, `contacts`. Comando da conversa lê fatos (silêncio, force_human, blocked, atribuição). 🟢

## Observabilidade
- `audit("message.received")` no ingest; alertas de saúde em `channel_session_health.escalated_status`. 🟢

## Riscos e Lacunas
- 🟡 Subdiretórios lidos por referência: `channels/{meta,social,zernio}/`, `messaging/media/*`, arquivos menores de `inbox/`, `atendimento/`, `notifications/`, `email/templates/`.
- 🟡 Default `WAHA_WEBHOOK_REQUIRE_SIGNATURE=false` depende de defesa de rede (Caddy) — confirmar no self-host.
- 🟡 `meta_cloud`/`zernio` reportam `isConfigured()===true` sempre; o `send` lança `*_not_configured`.
