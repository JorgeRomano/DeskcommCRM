# Caso de Uso: Orçamento e Custo — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas `ai_pricing`, `ai_models`, `ai_budgets`, `llm_calls` e RPC `fn_gasto_de_ia_do_mes`

## Tarefas
- [ ] T-01, Implementar cálculo de custo com cache
  - Origem no legado: `lib/ai/cost.ts`
  - Critério de pronto: `Math.ceil`; cache TTL 5min; preços 0/0 → null
  - Confiança: 🟢
- [ ] T-02, Implementar status de orçamento resiliente
  - Origem no legado: `lib/ai/budget/check.ts`
  - Critério de pronto: RPC → coluna materializada com log; `blocked_now` lê inbox; nunca lança
  - Confiança: 🟢
- [ ] T-03, Implementar agregação de uso
  - Origem no legado: `lib/ai/usage/aggregate.ts`
  - Critério de pronto: p50/p95, `handoff_rate` a 4 casas, séries e `by_kind`
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, `computeCost` não subfatura
- [ ] TT-02, `getBudgetStatus` degrada sem lançar
- [ ] TT-03, `percentile` correto para p50/p95

## Ordem Sugerida
1. T-01 → T-02 → T-03.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante; documentar limitação de medição por prefixo de preço.
