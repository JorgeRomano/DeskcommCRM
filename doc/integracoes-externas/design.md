# Integrações Externas — Design Técnico

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO

## Interface 🟢

| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `exchangeCodeForToken` | `(code, cfg: NuvemshopConfig)` | `TokenResult` (`{ok,accessToken,storeId,scope}` \| `{ok:false,error}`) |
| `verifyHmac` (Nuvemshop) | `(rawBody, signatureHex, clientSecret)` | `boolean` |
| `transporteDe` | `(plataforma: PlataformaDeAnuncio)` | `TransporteDeConversao \| null` |
| `transporteMeta.enviar` | `(credencial, conversao: ConversaoOffline)` | `ResultadoDeEnvio` |
| `conversaoDeVendaHandler` | `EventHandler` | escuta `lead.won`/`lead.stage_changed` |
| `registrarCaptacao` | `(admin, captacao: CaptacaoParaRegistrar)` | `Promise<void>` (nunca lança) |

### DTOs 🟢

- `ConversaoOffline { organizationId, leadId, evento:"Purchase", eventoId, ocorridoEm, cliqueDeOrigem, telefone, valorCentavos, moeda }`
- `ResultadoDeEnvio = {ok} | {transitorio, tentarEmMs?} | {permanente}`
- `CaptacaoParaRegistrar { outcome:"criado"|"duplicado"|"recusado", rejectReason?, capturedName/Phone/Email, fields, utm, origin }`

## Fluxo Principal — conversão de venda 🟢

1. Handler escuta `lead.won` E `lead.stage_changed` (payload é dica, banco é verdade — re-lê `crm_leads`).
2. Skip se status ≠ won, já enviada, ou sem atribuição de anúncio.
3. `transporteDe(plataforma)` → google_ads `null` → skip (`plataforma_sem_transporte`).
4. Exige `value_cents > 0` (senão `sem_valor`).
5. `transporteMeta.enviar`: evento >7d → permanente; monta payload (`ctwa_clid` + `ph` hasheado; `business_messaging`; `event_time` em segundos); POST à Graph API (token no header).
6. Resultado: ok → registra `sent`; transitório → retry; permanente → `error`.

## Fluxo Principal — Nuvemshop 🟢

- `buildAuthorizeUrl` (state CSRF) → callback troca `code` por token (`exchangeCodeForToken`); tokens não expiram, `user_id`=`storeId`.
- Webhook: `verifyHmac` (SHA256 hex do body cru com `client_secret`, `timingSafeEqual`, fail-closed).

## Fluxo Principal — captação 🟢

`registrarCaptacao` insere em `webhook_lead_captures` (nunca lança; org da fonte). `limitarCampos` corta valores >2000 chars e >60 campos com reticências.

## Fluxos Alternativos 🟢

- **Erro 4xx da Meta:** `classifica4xx` (613/80004 = throttle transitório; resto permanente).
- **Corpo não-JSON de erro:** texto cru preservado (gateway/WAF).
- **Campo não serializável na captação:** "[valor não serializável]".

## Dependências 🟢

- `conversoes` (`lerAtribuicao`, `registro-de-envio`), `plataformas-de-anuncio/credenciais`.
- `event-log` (handler `conversaoDeVenda`, por último), `leads` (webhook cria/move lead), `supabase`/Storage.
- Serviços: Nuvemshop, Meta Graph API, web-push.

## Decisões de Design Identificadas 🟢

| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| google_ads declarado null (não omitido) | `registry.ts` | 🟢 |
| Endpoint só em `meta/conversions.ts` (fora de lib/channels) | comentário do arquivo | 🟢 |
| Física da falha declarada (retry vs humano) | `types.ts:ResultadoDeEnvio` | 🟢 |
| Captação nunca lança (rota pública) | `captacao.ts` | 🟢 |

## Estado Interno 🟢

`webhook_lead_captures` (outcome/reject_reason/fields), `webhook_sources` (default_pipeline_id CASCADE), livro-razão de conversões (`registro-de-envio`), `tenant_integrations` (Nuvemshop) 🟡.

## Observabilidade 🟢

- Livro-razão de conversão (sent/skipped/error com motivo).
- `webhook_lead_captures` (tela "Leads recebidos") + `webhook_events_log` (forense, podado).

## Riscos e Lacunas

- 🟡 `lib/nuvemshop/{api-client,config}`, `lib/conversoes/{leitura-da-atribuicao,registro-de-envio}` não lidos em profundidade.
- 🟡 `lib/notifications/*` (pipeline push VAPID) não aprofundado.
- 🔴 Eixo de LEITURA de anúncios (métricas de conta, 0214) só visto por tipos.
