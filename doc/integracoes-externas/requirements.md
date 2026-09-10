# Integrações Externas

> Spec SDD (híbrida — módulo de topo) · Redator (Reversa) · Nível: Detalhado
> Módulos: `lib/nuvemshop/*`, `lib/plataformas-de-anuncio/*`, `lib/webhooks/*`, `lib/notifications/*`, `lib/conversoes/*`
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Visão Geral

Conecta o CRM a sistemas externos: e-commerce Nuvemshop (OAuth + webhooks), plataformas de anúncio (conversão offline Meta; Google declarado ausente), captação de leads via webhook, e notificações push. Todo webhook entrante verifica HMAC (fail-closed). 🟢

## Responsabilidades

- OAuth e verificação de webhook da Nuvemshop. 🟢
- Reportar conversão de venda à plataforma de origem (registry de transportes). 🟢
- Registrar durável a captação de leads via webhook. 🟢
- Entregar notificações push (VAPID). 🟡

## Regras de Negócio

- Nuvemshop: tokens não expiram; `user_id` = `storeId`; webhook HMAC-SHA256 (`x-linkedstore-hmac-sha256`), fail-closed. 🟢
- Conversão só reportada com atribuição de anúncio; `Purchase` exige valor+moeda; evento >7d recusado. 🟢
- `google_ads` declarado `null` no registry (sem extrator de gclid) — invariante 4. 🟢
- Meta: `action_source: business_messaging` + `messaging_channel: whatsapp`; token no header, nunca query; `event_time` em segundos. 🟢
- Erro de conversão classificado transitório (5xx, throttle 613) vs permanente (token, evento velho). 🟢
- Captação: nunca lança (rota pública sem sessão); org sempre da fonte, nunca do body; teto 60 campos × 2000 chars. 🟢
- Dedup de conversão em duas camadas: índice do livro-razão + `eventoId=<leadId>:Purchase` na plataforma. 🟢

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | OAuth Nuvemshop (troca de code por token) | Should | Dado code válido, retorna `{accessToken, storeId}`; falhas tipadas |
| RF-02 | Verificar HMAC de webhook Nuvemshop (fail-closed) | Must | Dado HMAC inválido, retorna false |
| RF-03 | Reportar conversão de venda por transporte | Could | Dado lead won com atribuição Meta, envia Purchase; sem valor → skip |
| RF-04 | Registrar captação de lead via webhook | Should | Dado formulário, grava `webhook_lead_captures` (criado/duplicado/recusado) |
| RF-05 | Entregar notificação push | Could | Dado inbound, entrega web push VAPID conforme prefs 🟡 |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Segurança | HMAC com `timingSafeEqual`, fail-closed (Nuvemshop) | `lib/nuvemshop/oauth.ts:verifyHmac` | 🟢 |
| Segurança | Token de conversão no header, nunca query | `lib/plataformas-de-anuncio/meta/conversions.ts` | 🟢 |
| Robustez | Física da falha declarada (ok/transitorio/permanente) | `lib/plataformas-de-anuncio/types.ts` | 🟢 |
| Robustez | Captação nunca lança; corta valor grande (não recusa linha) | `lib/webhooks/captacao.ts` | 🟢 |
| Performance | Timeout de 10s no envio de conversão | `meta/conversions.ts:TEMPO_LIMITE_MS` | 🟢 |

## Critérios de Aceitação

```gherkin
Dado um webhook Nuvemshop com assinatura HMAC inválida
Quando verifyHmac roda
Então retorna false (fail-closed) e o evento não é processado

Dado um lead won com atribuição Meta e valor > 0
Quando o handler de conversão roda
Então envia Purchase com ctwa_clid e telefone hasheado; sem atribuição → skip

Dado um formulário com um campo de 5000 caracteres
Quando registrarCaptacao roda
Então o valor é cortado com reticências e a linha é registrada (não recusada)
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| HMAC de webhooks | Must | Segurança de superfície pública |
| Captação de leads | Should | Entrada de leads por formulário |
| OAuth Nuvemshop | Should | Integração de e-commerce |
| Conversão de venda | Could | Otimização de anúncio; só com atribuição |
| Push | Could | Notificação; degrada sem derrubar |

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/nuvemshop/oauth.ts` | `exchangeCodeForToken`, `verifyHmac` | 🟢 |
| `lib/plataformas-de-anuncio/registry.ts` | `transporteDe` | 🟢 |
| `lib/plataformas-de-anuncio/types.ts` | `ConversaoOffline`, `ResultadoDeEnvio` | 🟢 |
| `lib/plataformas-de-anuncio/meta/conversions.ts` | `transporteMeta.enviar` | 🟢 |
| `lib/conversoes/envio.handler.ts` | `conversaoDeVendaHandler` | 🟢 |
| `lib/webhooks/captacao.ts` | `registrarCaptacao`, `limitarCampos` | 🟢 |
| `lib/notifications/*` | pipeline push VAPID | 🟡 |
