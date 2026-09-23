# Caso de Uso: Guardrails Before-Send — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Pacing e spinning implementados (ver `../tasks.md` T-06)
- [ ] Tabela `before_send_traces` disponível

## Tarefas
- [ ] T-01, Implementar `evaluateBeforeSend` puro com 11 gates e curto-circuito
  - Origem no legado: `lib/agent-engine/guardrails/before-send.ts`
  - Critério de pronto: veto no gate N marca N+1..11 como skipped; throttle acumulado; amendBody aplicado
  - Confiança: 🟢
- [ ] T-02, Implementar `runBeforeSend` stateful
  - Origem no legado: `guardrails/before-send.ts`
  - Critério de pronto: paga atraso humano antes de conectar; advisory lock por número; grava trace; dorme throttle e envia
  - Confiança: 🟢
- [ ] T-03, Implementar os detectores de camada
  - Origem no legado: `guardrails/vazamento-interno.ts`, `human-promise.ts`, `messaging-window.ts`, `promise/`, `jailbreak/classifier.ts`
  - Critério de pronto: determinísticos onde marcado; `messaging-window` fail-closed; `jailbreak` advisory
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Contato opted-out veta no gate stop e pula os demais
- [ ] TT-02, Fora da janela 24h e não-template → `messaging_window_closed`
- [ ] TT-03, Vazamento de vocabulário interno → `internal_vocabulary_leak`

## Ordem Sugerida
1. T-01 (puro) → T-02 (stateful) → T-03 (detectores).

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
