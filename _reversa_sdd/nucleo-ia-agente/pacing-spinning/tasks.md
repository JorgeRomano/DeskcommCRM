# Caso de Uso: Pacing e Spinning — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas `channel_knobs`, `pacing_ledger`, `outbound_copies` disponíveis

## Tarefas
- [ ] T-01, Implementar `decidePacing` (janela/warmup/throttle) puro
  - Origem no legado: `lib/agent-engine/pacing/engine.ts`, `defaults.ts`
  - Critério de pronto: janela na tz do tenant via Intl; warmup cap por idade; throttle com jitter
  - Confiança: 🟢
- [ ] T-02, Implementar seam de pacing (store + leitor CRM-side)
  - Origem no legado: `pacing/store.ts`, `pacing/ledger-supabase.ts`
  - Critério de pronto: mesma regra em dois leitores; leitor CRM-side fail-open, sem janela
  - Confiança: 🟢
- [ ] T-03, Implementar `decideSpinning`
  - Origem no legado: `spinning/engine.ts`, `spinning/defaults.ts`, `spinning/store.ts`
  - Critério de pronto: sha256 + Jaccard sobre janela; veto `mass_identical` ≥ threshold
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Fora da janela → outside_window
- [ ] TT-02, Número novo limitado pelo warmup cap
- [ ] TT-03, Cópia idêntica repetida → mass_identical

## Ordem Sugerida
1. T-01 → T-02 → T-03.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
