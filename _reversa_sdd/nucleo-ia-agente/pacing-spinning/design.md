# Caso de Uso: Pacing e Spinning — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `decidePacing` | `(input: PacingInput)` | `PacingDecision` (puro) |
| `decideSpinning` | `(input: SpinningInput)` | `SpinningDecision` (puro) |

## Fluxo Principal — Pacing
1. Checa janela de horário na tz do tenant (`fusoDaJanela`: canal > org > default). 🟢
2. Se `!banRisk`, allow direto. 🟢
3. Warmup cap por idade do número; `effectiveCap = min(warmup, crmDailyLimit)`. 🟢
4. Throttle com jitter. Wall-clock via `Intl` (DST-safe). 🟢

## Fluxo Principal — Spinning
1. Allowlist bypass. 🟢
2. sha256 exato + similaridade Jaccard sobre janela (`outbound_copies` normalizado + hash). 🟢
3. `matchCount ≥ repetitionThreshold` → veto `mass_identical`. 🟢

## Dependências
- `pacing/store.ts` (seam Postgres: `channel_knobs`, `pacing_ledger`), `spinning/store.ts` (`outbound_copies`).
- Consumido pelo gate `pacing` (#3) e `spinning` (#5) da cadeia before-send.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Engine puro, store como seam | `pacing/engine.ts`, `pacing/store.ts` | 🟢 |
| Leitor CRM-side com mesma regra, fail-open, sem janela de horário | `pacing/ledger-supabase.ts` | 🟢 |
| Throttle é espera, não veto | `guardrails/before-send.ts` (gate 3) | 🟢 |

## Riscos e Lacunas
- 🟡 `crmDailyLimit` vem da org; a interação warmup×daily deve ser validada na instalação.
