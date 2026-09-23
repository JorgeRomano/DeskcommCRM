# Caso de Uso: Pipeline Imutável — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Invariante `pipeline_id` imutável guardado no banco/CHECK

## Tarefas
- [ ] T-01, Implementar clone cross-pipeline
  - Origem no legado: `lib/leads/clonar-para-funil.ts`
  - Critério de pronto: herda `custom_fields`; não herda `external_id`/status/closed_at
  - Confiança: 🟢
- [ ] T-02, Encerrar origem com motivo canônico
  - Origem no legado: `lib/leads/motivo-da-perda.ts`, `encerramento.ts`
  - Critério de pronto: `moved_to_another_pipeline` fora da métrica; nunca ofertado na tela
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Clone tem nova identidade
- [ ] TT-02, Origem encerra fora da métrica

## Ordem Sugerida
1. T-01 → T-02.

## Lacunas Pendentes (🔴)
- Confirmar guarda de imutabilidade de `pipeline_id` no schema.
