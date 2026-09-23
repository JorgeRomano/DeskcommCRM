# Canais e Mensageria (`channels`, `waha`, `messaging`, `inbox`, `atendimento`, `notifications`, `email`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 3).

## Visão Geral
Camada agnóstica de canal que leva mensagem entre o CRM e os provedores (WhatsApp via WAHA/QR, Meta Cloud API oficial, Zernio/BSP) e as superfícies internas de mensageria (inbox, fronteira de atendimento, notificações, e-mail). A lei que rege a unit é o **invariante de restrição de canal**: nenhuma feature fora de `lib/channels/` pode nomear um provedor — features perguntam *o que o canal permite* (capabilities), nunca *quem ele é*. Gate: `pnpm lint:channels`. 🟢

## Responsabilidades
- Definir `ChannelProvider`, `ChannelCapabilities` e o contrato `ChannelAdapter`. 🟢
- Transportar mensagens de saída via 3 adaptadores (waha, meta_cloud, zernio). 🟢
- Ingerir webhooks inbound (WAHA, Zernio) com autenticação por canal e idempotência. 🟢
- Aplicar os efeitos pós-entrada em ORDEM (opt-out → origem → demanda → campanha → despacho). 🟢
- Derivar a janela 24h por leitura, monitorar saúde/estado da conexão. 🟢
- Gerir o comando da conversa (inbox) e a fronteira IA↔humano (atendimento). 🟢
- Entregar notificações (in-app/push) e e-mail (SMTP/Resend por configuração). 🟢

## Regras de Negócio
- `capabilitiesOf`/`getAdapter`/`transportaMensagem` são fail-closed (lançam `unknown_channel_provider`). — `capabilities.ts:189`, `index.ts` 🟢
- `organizationId` no `OutboundEnvelope` vem de fonte confiável, nunca do body (issue #236). 🟢
- meta_cloud: enviar template NÃO reabre a janela 24h (só a resposta do cliente reabre); Meta recusa entrega por webhook erro 131047. 🟢
- Efeitos pós-entrada em ordem fixa (guardada por testes); nenhum efeito pode lançar (mensagem já persistida). — `pos-entrada.ts:150` 🟢
- Opt-out é o passo 1 porque `abrirDemanda` recusa contato bloqueado. 🟢
- Janela 24h derivada a cada leitura (sem coluna de expiração — evita segunda verdade rançosa). — `janela.ts:58` 🟢
- WAHA webhook é fail-closed: assinatura errada → `bad_signature`; exigida e ausente → `signature_required`. — `webhook-auth.ts` 🟢
- Ingest não emite `message.received` — o trigger `trg_messages_emit_event` emite (evita emissão dupla). — `ingest.ts` 🟢
- Idempotência de mensagem por `unique(organization_id, external_id)` + captura `23505`. 🟢
- E-mail: SMTP↔Resend por CONFIGURAÇÃO, nunca por falha (fallback em falha faria double-send). — `email/roteador.ts` 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Resolver capabilities/adapter por provider, fail-closed | Must | Provider desconhecido lança `unknown_channel_provider` |
| RF-02 | Enviar mensagem via adapter puro (sem lógica de janela/cap) | Must | Adapter só traduz formato; janela/cap ficam no before-send |
| RF-03 | Ingerir webhook inbound autenticado e idempotente | Must | Assinatura errada → `bad_signature`; `external_id` repetido não duplica |
| RF-04 | Aplicar efeitos pós-entrada em ordem, sem lançar | Must | Opt-out primeiro; falha de efeito loga, não derruba webhook |
| RF-05 | Derivar janela 24h por leitura | Should | `estadoDaJanela` retorna `aberta`/`fechada`/`sem_restricao` sem coluna persistida |
| RF-06 | Monitorar saúde/estado e alertar | Should | `SCAN_QR_CODE`/`FAILED`/`STOPPED` geram alerta; só quem observou fecha episódio |
| RF-07 | Gerir comando da conversa (inbox) | Must | `comandoDaConversa` decide humano/automatico/ninguem/aguardando/encerrada |
| RF-08 | Roteador de e-mail por configuração | Should | SMTP se configurado, senão Resend; erro não achatado |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | HMAC de webhook (`timingSafeEqual`), fail-closed | `waha/ingest.ts` (`verifyHmacSha512`), `channels/inbound.ts` | 🟢 |
| Segurança | Guarda SSRF por allowlist de host ao buscar mídia | `adapters/meta-cloud.ts`, `messaging/media`, `automation/outbound-*` | 🟢 |
| Segurança | Guarda ReDoS no parse de chatId | `waha/ingest.ts` (`semSufixoDeChat`) | 🟢 |
| Segurança | Erros de canal carregam STATUS, nunca o BODY (PII) | `waha/client.ts` | 🟢 |
| Disponibilidade | Timestamp tolerante (nunca lança `RangeError`) | `waha/ingest.ts` (`dataDoTimestamp`) | 🟢 |
| Disponibilidade | Push 404/410 deleta subscription morta | `notifications/web_push.ts` | 🟢 |
| Performance | Timeouts: texto 15s, mídia 30s | `waha/client.ts` | 🟢 |
| Escalabilidade | `CONVERSAS_IGNORADAS` empurradas ao `config.ignore` (376/395 MB descartado) | `waha/client.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado um provider desconhecido
Quando capabilitiesOf é chamado
Então lança unknown_channel_provider (fail-closed)

Dado um webhook WAHA com assinatura errada
Quando authenticateWahaWebhook avalia
Então rejeita com bad_signature

Dado dois eventos WAHA (message e message.any) para a mesma mensagem
Quando handleInbound processa
Então o segundo é idempotente (23505) e não duplica

Dado um template enviado no meta_cloud
Quando o cliente ainda não respondeu
Então a janela 24h NÃO reabre
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Abstração de canal fail-closed (RF-01/02) | Must | Invariante de restrição de canal |
| Ingest autenticado e idempotente (RF-03) | Must | Porta de entrada de toda mensagem |
| Efeitos pós-entrada ordenados (RF-04) | Must | Corretude de opt-out e demanda |
| Comando da conversa (RF-07) | Must | Decide quem atende |
| Janela 24h / saúde (RF-05/06) | Should | Operação com defaults |
| Roteador de e-mail (RF-08) | Should | Canal secundário |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/channels/capabilities.ts` | `capabilitiesOf`, `transportaMensagem`, `CHANNEL_CAPABILITIES` | 🟢 |
| `lib/channels/index.ts` | `getAdapter`, `ADAPTERS` | 🟢 |
| `lib/channels/pos-entrada.ts` | `aplicarEfeitosPosEntrada` | 🟢 |
| `lib/channels/janela.ts` | `estadoDaJanela` | 🟢 |
| `lib/channels/health.ts` | `avisoDaConexao`, `sincronizarSaudeDaConexao` | 🟢 |
| `lib/waha/webhook-auth.ts` | `authenticateWahaWebhook` | 🟢 |
| `lib/waha/ingest.ts` | `dispatchWahaEvent`, `handleInbound`, `handleOutboundFromUserPhone` | 🟢 |
| `lib/inbox/comando-da-conversa.ts` | `comandoDaConversa`, `silencioVigente` | 🟢 |
| `lib/atendimento/fronteira-server.ts` | `assertCurrentServiceBoundary` | 🟢 |
| `lib/email/roteador.ts` | `transporteDeEmail` | 🟢 |
