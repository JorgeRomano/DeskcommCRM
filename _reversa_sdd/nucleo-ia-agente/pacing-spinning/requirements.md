# Caso de Uso: Pacing e Spinning

> Sub-unit de `nucleo-ia-agente`. Anti-ban: janela de envio + warmup + throttle (pacing) e detecção de cópia em massa (spinning).
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Duas defesas determinísticas contra bloqueio de número: pacing controla QUANDO e QUANTO enviar; spinning detecta envio da mesma mensagem em massa. 🟢

## Responsabilidades
- Aplicar janela de horário na timezone do tenant. 🟢
- Aplicar warmup cap por idade do número e throttle com jitter. 🟢
- Detectar cópia idêntica/similar em janela e vetar `mass_identical`. 🟢

## Regras de Negócio
- `decidePacing`: (1) janela → `outside_window`; (2) se `!banRisk`, allow; (3) warmup `effectiveCap=min(warmup, crmDailyLimit)` → `warmup_cap`/`daily_cap`; (4) throttle. Wall-clock via `Intl` (DST-safe). 🟢
- `PACING_DEFAULTS`: throttle 1200ms, jitter 800ms, janela 7–22h, allowSunday true, tz America/Sao_Paulo, warmup `[{0,20},{4,50},{8,100},{15,200},{31,null}]`. 🟢
- Spinning: allowlist → sha256 exato + Jaccard ≥ threshold; `matchCount ≥ repetitionThreshold` → veto. `SPINNING_DEFAULTS`: windowSize 20, similarity 0.8, repetition 2. 🟢
- Leitor CRM-side (`ledger-supabase.ts`) usa a MESMA regra, nunca lança (fail-open), não aplica janela de horário. 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Vetar envio fora da janela do tenant | Should | Envio às 23h (janela 7–22h) → `outside_window` |
| RF-02 | Aplicar warmup cap por idade do número | Should | Número novo (idade 0) limita a 20/dia |
| RF-03 | Vetar cópia idêntica em massa | Should | 2 cópias idênticas na janela → `mass_identical` |

## Critérios de Aceitação
```gherkin
Dado um número com banRisk fora da janela 7-22h
Quando decidePacing avalia
Então retorna outside_window

Dado duas mensagens idênticas na janela de spinning
Quando decideSpinning avalia a segunda
Então veta com mass_identical
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/agent-engine/pacing/engine.ts` | `decidePacing` | 🟢 |
| `lib/agent-engine/pacing/defaults.ts` | `PACING_DEFAULTS` | 🟢 |
| `lib/agent-engine/spinning/engine.ts` | `decideSpinning` | 🟢 |
| `lib/agent-engine/spinning/defaults.ts` | `SPINNING_DEFAULTS` | 🟢 |
