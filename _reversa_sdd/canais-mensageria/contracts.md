# Canais e Mensageria — Contratos

> Contratos: (a) `ChannelAdapter`/`OutboundEnvelope`, (b) matriz de capabilities, (c) webhooks inbound.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Contrato 1 — `ChannelAdapter`
```
{
  provider: ChannelProvider,
  resolveRecipient, isConfigured,
  send(envelope): Promise<{externalId}>,
  codes,
  // opcionais:
  fetchProfilePictureUrl?, echoExternalIds?, resolvePhoneForIdentity?,
  templates?, signalTyping?, checkHealth?, fetchInboundMedia?, sendTemplate?
}
```
Regra: adapter é tradutor de formato puro — nenhuma lógica de janela 24h / cap / horário. 🟢

## Contrato 2 — `OutboundEnvelope`
Estende `ChannelTenantScope {organizationId}` (obrigatório, fonte confiável, nunca do body). Campos: `beforeSend?()` (re-valida origem após preparo async), `sessionRef`, `to`, `kind`, `body?`, `media?`, `contact?`, `providerConversationId?` (thread Zernio), `replyToExternalId?` (wamid citado). 🟢

## Contrato 3 — Matriz de Capabilities (`CHANNEL_CAPABILITIES`)
| Provider | freeform | requiresTemplates | banRisk | groups | Observação |
|---|---|---|---|---|---|
| waha | true | false | true | full | "Falo quando quiser, mas WhatsApp bane se abusar" |
| meta_cloud | false | true | false | — | `minIntervalMs 6000`, `voiceNote opus-only`, `costPerMessage true`; template não reabre janela (erro 131047) |
| zernio | (perfil próprio) | — | — | — | endereça por `providerConversationId` |
| zernio_social | false | (sem templates) | — | none | — |

`DEFAULT_CHANNEL_PROVIDER = "waha"`. `capabilitiesOf` fail-closed. Exaustividade garantida em compilação (`ProviderNaoClassificado extends never`). 🟢

## Contrato 4 — Webhook inbound WAHA
`authenticateWahaWebhook(input)`: (1) assinatura presente e errada → `bad_signature`; (2) exigida e ausente → `signature_required`; (3) ausente e não exigida → `{ok, signatureVerified:false}`. `verifyHmacSha512` (SHA-512 hex + `timingSafeEqual`). Eventos: `message`, `message.any`, `message.ack`, `message.edited`, `message.revoked`, `session.status`. 🟢

## Contrato 5 — Webhook inbound Zernio
`handleInboundWebhook(admin, input)`: assinatura verificada no handler (`verifyZernioSignature` em `x-zernio-signature`, fail-closed, `MIN_SECRET_LEN=16`). Erros: `unauthorized | provider_mismatch | invalid_json | contrato_violado`. 🟢

## Contrato 6 — Roteador de e-mail
`transporteDeEmail()` = SMTP se `isSmtpConfigured`, senão Resend (por configuração, nunca por falha). `EmailDeliveryError` distingue `dominio_nao_verificado`, `sender_rejected`, `rate_limited`... (não achatado). `getSmtpConfig()` prefere `platform_smtp_settings` (id=1, senha cifrada) sobre `env.SMTP_*`. 🟢
