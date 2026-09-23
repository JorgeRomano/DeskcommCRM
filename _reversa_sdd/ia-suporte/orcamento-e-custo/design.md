# Caso de Uso: Orçamento e Custo — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `computeCost` | `({model, promptTokens?, completionTokens?, embeddingTokens?})` | `Promise<number>` (centavos) |
| `getBudgetStatus` | `(orgId)` | `Promise<BudgetStatus>` |
| `aggregateUsage` | `(rows, dailyInbounds, dailyHandoffs, range)` | `UsagePayload` |

## Fluxo Principal
1. `loadPricing()` lê `ai_pricing where superseded_at is null` (cache TTL 5min). 🟢
2. `computeCost`: fórmula `(tokens * rate)/1_000_000` por tipo, `Math.ceil`. Fallback `precoDoCatalogo` via `ai_models` (match exato; `null` se ambos 0). 🟢
3. `getBudgetStatus`: gasto via RPC `fn_gasto_de_ia_do_mes` (mesma régua do gate); falha → coluna materializada com log. `blocked_now` lê inbox. 🟢
4. `aggregateUsage`: totais, séries e `by_kind`; `percentile(sorted,p)=ceil((p/100)*len)-1`; `handoff_rate=handoffs/inbounds` a 4 casas. 🟢

## Dependências
- Supabase (`ai_pricing`, `ai_models`, `ai_budgets`, `llm_calls`, RPC de gasto).
- Consumido pelo gate de orçamento em `agent-engine/edge/llm/orcamento.ts`.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| `Math.ceil` no custo | `cost.ts` | 🟢 |
| Régua única de gasto (RPC do gate) | `budget/check.ts` | 🟢 |
| `blocked_now` lê inbox, não recalcula | `budget/check.ts` | 🟢 |

## Estado Interno
- `_pricingCache` (TTL 5min). Coluna materializada `current_month_consumed_cents`. 🟢

## Riscos e Lacunas
- 🟡 Furo de medição: `ai_pricing` casa por prefixo; gasto pode ser subestimado (`gasto_incompleto`).
