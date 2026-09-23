# Canais e Mensageria — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Webhook forjado (fail-open histórico) 🟢
`authenticateWahaWebhook` era fail-OPEN (buraco de forja provado com curl); agora fail-closed. Assinatura errada sempre rejeita (`bad_signature`). Default `WAHA_WEBHOOK_REQUIRE_SIGNATURE=false` porque WAHA Core não assina — a defesa de rede (Caddy) é a camada complementar.

## EC-02 — Emissão dupla de evento 🟢
NOWEB emite `message` E `message.any`. Upsert atômico via RPC fecha a corrida check-then-act; idempotência `23505` sobre `unique(organization_id, external_id)`. Ingest não emite `message.received` (o trigger emite) — medido 805 msgs com 2 eventos antes da correção.

## EC-03 — ReDoS no parse de chatId 🟢
`semSufixoDeChat` substitui `replace(/@.*$/,"")` (CodeQL js/polynomial-redos). Usa `lastIndexOf` + `indexOf`. Regex antigo era O(n²) sobre `payload.from` controlado por atacante; linear agora (1MB em 0.7ms).

## EC-04 — Timestamp em unidade desconhecida 🟢
`dataDoTimestamp` infere unidade pela magnitude (≥1e16 ns, ≥1e11 ms, else s); nunca lança `RangeError` (que derrubava o webhook).

## EC-05 — Efeito pós-entrada lança 🟢
Nenhum efeito pode lançar (mensagem já persistida; um 500 dispararia tempestade de reentrega). Opt-out falha loga `error`, o resto `warn`.

## EC-06 — Template não reabre a janela (meta_cloud) 🟢
Enviar template devolve 200+wamid, mas a Meta recusa a entrega por webhook (erro 131047) se a janela 24h está fechada; só a resposta do cliente reabre.

## EC-07 — Trocar o canal de uma conversa viva 🟢
`resolveWahaChatId`: `@lid` ANTES de phone (0122). Trocar o canal de uma conversa viva é o pior defeito, por isso a ordem é regra.

## EC-08 — Valor de silêncio ilegível 🟢
`silencioVigente`: valor ilegível é tratado como SILENCIADO (fail-closed na ação).

## EC-09 — Fronteira de serviço obsoleta 🟢
`assertCurrentServiceBoundary` lança `StaleServiceBoundaryError` em qualquer divergência (org/contato/conversa/`service_revision`, status terminal, `demanda_fechada_em`, `demandaTrocou`). Abrir a primeira demanda não é novo serviço (0222 só bump ao TROCAR de demanda).

## EC-10 — Config PUT do WAHA substitui tudo 🟢
`startSession` faz GET-before-PUT (`convergirConfigDaSessao`) porque o PUT do WAHA substitui a config INTEIRA (inclui webhooks).

## EC-11 — Push subscription morta 🟢
`enviarPushDaOrg`: 404/410 deleta a subscription morta.

## EC-12 — Ordem da espera resetando 🟢
`ORDEM_DA_ESPERA` ordena por `awaiting_since` (inbound mais antigo sem resposta), não `last_inbound_at` (que reseta a cada mensagem — issue #990/#994).
