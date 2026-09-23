# ADR-0006 — Webhooks fail-closed e efeitos pós-entrada ordenados que nunca lançam

> ADR **retroativo** (Detetive/Reversa). Confiança: 🟢 CONFIRMADO.
> Evidência: `code-analysis.md` (3.2, 3.6), `waha/webhook-auth.ts`, `channels/pos-entrada.ts`.

## Status
Aceito (vigente).

## Contexto
O webhook de canal é a porta de entrada da mensagem do cliente. Duas armadilhas: (1) aceitar webhook
forjado (o WAHA Core não assina por padrão, e havia um buraco de forja provado com curl); (2) deixar
um efeito pós-entrada (opt-out, abrir demanda, despachar agente) lançar exceção — a mensagem já está
persistida, então um 500 dispara tempestade de reentrega do provedor.

## Decisão
- **Autenticação fail-closed:** `authenticateWahaWebhook` verifica HMAC com `timingSafeEqual`,
  `MIN_SECRET_LEN = 16`. Assinatura presente e errada → **sempre** rejeita. Exigida e ausente →
  rejeita. Ausente e não-exigida → segue mas loga que não verificou. (Era fail-OPEN.) A defesa de
  rede (proxy não publicando a rota global) é a camada que não depende do secret. Zernio verifica a
  assinatura **na camada de canal**, não na rota (o esquema é específico por canal).
- **Efeitos pós-entrada com ORDEM fixa e nenhum pode lançar:** opt-out (passo 1) → origem da página →
  abrir demanda → avaliar campanha → acelerar pipeline → pedir despacho do agente. Opt-out precede
  porque abrir demanda recusa contato bloqueado. Opt-out que falha loga `error`; o resto, `warn`.

## Alternativas consideradas
1. **Fail-open no webhook (aceitar sem assinatura)** — rejeitado: forja provada; abandonado.
2. **Efeitos inline no ingest do WAHA (estado anterior)** — rejeitado: só rodavam no WAHA (medido
   806 no QR, 0 no oficial); movidos para trás do seam `aplicarEfeitosPosEntrada`.
3. **Efeitos que propagam exceção** — rejeitado: 500 → reentrega em tempestade.

## Consequências
- **Positivas:** forja barrada; efeitos idênticos em todos os canais; entrada resiliente (mensagem
  nunca se perde por falha de efeito secundário).
- **Negativas / custo:** o default `WAHA_WEBHOOK_REQUIRE_SIGNATURE=false` existe porque o WAHA Core
  não assina — a segurança recai sobre a camada de rede, que é responsabilidade do operador da VPS;
  efeitos que "só logam" podem mascarar falha silenciosa se ninguém observar o log.
- **Robustez adicional:** guarda ReDoS em `semSufixoDeChat` (substituiu regex O(n²) sobre entrada de
  atacante) e timestamp tolerante que nunca lança `RangeError` (antes derrubava o webhook).
