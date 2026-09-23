# ADR-0004 — Cadeia determinística `before_send` entre o modelo e o canal

> ADR **retroativo** (Detetive/Reversa). Confiança: 🟢 CONFIRMADO.
> Evidência: `code-analysis.md` (1.9), `agent-engine/guardrails/before-send.ts`.

## Status
Aceito (vigente).

## Contexto
Um LLM pode prometer o que não deve (preço, desconto, humano), copiar mensagem em massa (risco de
ban no WhatsApp), falar fora da janela permitida, vazar vocabulário interno ou falar com quem pediu
opt-out. Confiar no prompt para evitar isso é confiar no não-determinístico numa fronteira que tem
consequência legal (LGPD), comercial (promessa) e operacional (ban de número).

## Decisão
Toda decisão `send_message` do modelo passa por uma **cadeia declarativa e versionada** de 11 gates
(`BEFORE_SEND_CHAIN_VERSION = 7`), avaliada em ordem fixa: `stop`, `lgpd`, `pacing`,
`messaging_window`, `spinning`, `promise`, `semantic_promise`, `case_promise`,
`internal_vocabulary`, `agenda_stall`, `disclosure`. `evaluateBeforeSend` é **puro** e curto-circuita
no 1º veto; `runBeforeSend` é stateful (paga o atraso humano antes de tomar conexão — issue #654 —,
pega `pg_advisory_xact_lock` por número de canal, grava trace durável em `before_send_traces`).
**9 das 10 conferências não se desligam**; só `semantic_promise` é opcional (custa +1 consulta).

## Alternativas consideradas
1. **Guardrails só no prompt** — rejeitado: não-determinístico; sem trace; sem veto duro.
2. **Pós-processamento não-versionado (if/else espalhado)** — rejeitado: impossível auditar qual
   regra vetou o quê e em que versão.
3. **Cadeia declarativa versionada, avaliação pura + execução stateful (escolhida).**

## Consequências
- **Positivas:** veto determinístico e auditável (`before_send_traces` com a versão da cadeia);
  anti-ban armado por default (`spinningEnforced` ARMADO, `messagingWindow` ausente = fechado);
  atraso humano cobrado sem segurar conexão; serialização read-then-act por número.
- **Negativas / custo:** custo de latência (atraso humano) e de consulta (semantic_promise,
  jailbreak) por mensagem; assimetria de defaults dos flags opcionais é sutil e fácil de errar
  (`internalVocabularyEnforced`/`agenda` default NO-OP vs `spinning` ARMADO).
- **Regra de produto derivada:** enviar template **não** reabre a janela de 24h (só a resposta do
  cliente reabre); a Meta recusa entrega por webhook erro 131047 (RN-20).
