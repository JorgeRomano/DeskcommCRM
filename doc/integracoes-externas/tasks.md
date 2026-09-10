# Integrações Externas — Tarefas de Implementação

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO

## Pré-requisitos
- [ ] `event_log` + handler `conversaoDeVenda`
- [ ] Tabelas `webhook_lead_captures`, `webhook_sources`, livro-razão de conversão
- [ ] Credenciais Nuvemshop (`client_id`/`client_secret`), Meta (dataset/token) cifradas
- [ ] VAPID keys para push 🟡

## Tarefas

- [ ] T-01, Implementar OAuth + HMAC da Nuvemshop
  - Origem no legado: `lib/nuvemshop/oauth.ts`
  - Critério de pronto: tokens não expiram; HMAC fail-closed; `user_id`=storeId
  - Confiança: 🟢

- [ ] T-02, Implementar registry de transportes de conversão
  - Origem no legado: `lib/plataformas-de-anuncio/registry.ts`, `types.ts`
  - Critério de pronto: google_ads null declarado; `ehPlataformaConhecida`
  - Confiança: 🟢

- [ ] T-03, Implementar transporte Meta (Conversions API)
  - Origem no legado: `lib/plataformas-de-anuncio/meta/conversions.ts`
  - Critério de pronto: business_messaging; token no header; classifica 4xx transitório/permanente
  - Confiança: 🟢

- [ ] T-04, Implementar handler de conversão de venda
  - Origem no legado: `lib/conversoes/envio.handler.ts`
  - Critério de pronto: escuta lead.won/stage_changed; re-lê banco; só com atribuição+valor
  - Confiança: 🟢

- [ ] T-05, Implementar registro durável de captação
  - Origem no legado: `lib/webhooks/captacao.ts`
  - Critério de pronto: nunca lança; teto 60×2000; outcome tipado
  - Confiança: 🟢

- [ ] T-06, Implementar pipeline de notificação push (VAPID)
  - Origem no legado: `lib/notifications/*`
  - Critério de pronto: emit → policy/prefs → deliver → web push
  - Confiança: 🟡

## Tarefas de Teste

- [ ] TT-01, HMAC Nuvemshop inválido → false
- [ ] TT-02, Conversão sem atribuição → skip; sem valor → sem_valor
- [ ] TT-03, Erro 613 da Meta → transitório (retry)
- [ ] TT-04, Captação corta campo grande sem recusar a linha

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `webhook_sources.default_pipeline_id` ON DELETE CASCADE
- [ ] TM-02, Cifragem de tokens OAuth at-rest (L-09)

## Ordem Sugerida
1. T-05 (captação) é entrada de lead independente.
2. T-01 (Nuvemshop) e T-02/T-03/T-04 (conversão) em paralelo.
3. T-06 (push) por último.

## Lacunas Pendentes (🔴)
- `lib/nuvemshop/{api-client,config}`, `lib/conversoes/{leitura-da-atribuicao,registro-de-envio}`.
- `lib/notifications/*` (pipeline completo).
- Eixo de leitura de métricas de anúncio (0214).
