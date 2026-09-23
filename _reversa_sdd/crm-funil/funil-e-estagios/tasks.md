# Caso de Uso: Funil e Estágios — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Índices únicos parciais de marca exclusiva (default/client, won/lost)

## Tarefas
- [ ] T-01, Implementar edição pura de funil e etapa
  - Origem no legado: `lib/pipelines/pipeline-editing.ts`, `lib/leads/stage-editing.ts`
  - Critério de pronto: recusa ordenada; mutex ganho/perda; slug estável ao renomear
  - Confiança: 🟢
- [ ] T-02, Implementar operações com I/O e movimentadores
  - Origem no legado: `lib/leads/stage-operations.ts`, `agent-stage-sync.ts`, `handoff-stage-move.ts`, `appointment-stage-move.ts`, `agent-mapping.ts`
  - Critério de pronto: sequência sem paralelo; trava otimista; só pending/confirmed avançam
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Funil sem won → pipeline_no_won_stage
- [ ] TT-02, Marca exclusiva sem `23505`
- [ ] TT-03, Conflito humano na movimentação

## Ordem Sugerida
1. T-01 → T-02.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
