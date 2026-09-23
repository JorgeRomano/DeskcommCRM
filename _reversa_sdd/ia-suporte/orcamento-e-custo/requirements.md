# Caso de Uso: Orçamento e Custo

> Sub-unit de `ia-suporte`. Controle financeiro do uso de IA por org.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Calcula o custo de cada invocação em centavos e aplica o orçamento mensal por organização, avisando antes de bloquear. 🟢

## Responsabilidades
- Calcular custo por tipo de token (`computeCost`). 🟢
- Reportar status de orçamento sem lançar (`getBudgetStatus`). 🟢
- Agregar uso para o dashboard (`aggregateUsage`). 🟢

## Regras de Negócio
- Custo em centavos com `Math.ceil` (nunca subfatura); preços 0/0 → `null`. 🟢
- `getBudgetStatus` degrada da RPC para coluna materializada com log; nunca lança. 🟢
- `blocked_now` lê `agent_inbox_items kind='budget_exceeded' status='open'` (não recalcula). 🟢
- `monthly_limit_cents: 0` = sem teto. 🟢
- `gasto_incompleto` quando há `llm_calls.cost_cents is null` no mês. 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Custo em centavos sem subfaturar | Must | `computeCost` usa `Math.ceil` |
| RF-02 | Status de orçamento resiliente | Must | RPC falha → coluna materializada com log |
| RF-03 | Bloqueio lê decisão do inbox | Must | `blocked_now` reflete inbox, não recálculo |

## Critérios de Aceitação
```gherkin
Dado que a RPC de gasto falha
Quando getBudgetStatus é chamado
Então usa current_month_consumed_cents com log, sem lançar
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/ai/cost.ts` | `computeCost`, `precoDoCatalogo`, `loadPricing` | 🟢 |
| `lib/ai/budget/check.ts` | `getBudgetStatus` | 🟢 |
| `lib/ai/usage/aggregate.ts` | `aggregateUsage`, `percentile` | 🟢 |
