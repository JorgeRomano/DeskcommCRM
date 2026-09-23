# CRM e Funil — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas centrais e RPCs/triggers do baseline (`fn_nascer_lead_da_conversa`, `fn_crm_lead_close_on_stage`, `fn_update_last_activity_at`)
- [ ] Índices únicos parciais (`uniq_crm_stages_pipeline_won`, marca exclusiva de pipeline)

## Tarefas
- [ ] T-01, Implementar scoring citável e histerese de faixa
  - Origem no legado: `lib/leads/score-formula.ts`, `kanban/score-band.ts`, `score-writer.ts`
  - Critério de pronto: pesos e teto; lastro citável obrigatório; null apaga score; histerese degrau-a-degrau
  - Confiança: 🟢
- [ ] T-02, Implementar Risk Radar (classificador + since + workers + seed)
  - Origem no legado: `lib/leads/risk-radar.ts`, `risk-since.ts`, `risk-worker.ts`, `risk-seed.ts`, `radar-de-risco.ts`
  - Critério de pronto: janela por estágio; since = cruzamento do limiar (nunca futuro); só escreve quando muda
  - Confiança: 🟢
- [ ] T-03, Implementar edição de funil/etapa e movimentadores de card
  - Origem no legado: `lib/pipelines/pipeline-editing.ts`, `leads/stage-editing.ts`, `stage-operations.ts`, `agent-stage-sync.ts`, `handoff-stage-move.ts`, `appointment-stage-move.ts`
  - Critério de pronto: recusa ordenada; libera antes de marcar; trava otimista → conflito_humano
  - Confiança: 🟢
- [ ] T-04, Implementar ciclo de vida (nascimento, encerramento, motivo, reativação, clone, correção)
  - Origem no legado: `lib/leads/nascimento-do-lead.ts`, `encerramento.ts`, `motivo-da-perda.ts`, `reactivation.ts`, `clonar-para-funil.ts`, `correcao-humana.ts`, `active-lead.ts`
  - Critério de pronto: nascimento idempotente; motivo obrigatório em lost; `moved_to_another_pipeline` fora da métrica; `pipeline_id` imutável
  - Confiança: 🟢
- [ ] T-05, Implementar atividade/timeline
  - Origem no legado: `lib/leads/activity-vocabulary.ts`, `activity-emitter.ts`, `timeline-query.ts`, `timeline-grouping.ts`, `activity-write-failure.ts`
  - Critério de pronto: vocabulário fechado (compilador é o gate); ator ai sem lastro vira system; agrupamento por janela
  - Confiança: 🟢
- [ ] T-06, Implementar Kanban (card-state, dono, local echo, filtros, vocabulário)
  - Origem no legado: `lib/kanban/*`
  - Critério de pronto: precedência estrita do card-state; indexação fracionária com rebalance; local echo por ciclo de mutação
  - Confiança: 🟢
- [ ] T-07, Implementar contatos, etiquetas, conversões e prospecção
  - Origem no legado: `lib/contacts/*`, `lib/tags/cor-da-etiqueta.ts`, `lib/conversoes/*`, `lib/prospecting/*`
  - Critério de pronto: dedup union-find; CPF por hash; conversão idempotente por atribuição; prospecção ~4× mais lenta, falha fechada
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Score sem lastro → null
- [ ] TT-02, Risco por estágio com janela própria
- [ ] TT-03, Funil sem won não fecha
- [ ] TT-04, Nascimento idempotente (3 mensagens → 1 lead)
- [ ] TT-05, Trava otimista → conflito_humano
- [ ] TT-06, Conversão idempotente por `<leadId>:Purchase`

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Seed de risco (`risk-seed.ts`) para todo negócio aberto, sem timeline falsa

## Ordem Sugerida
1. T-03 (funil/etapa) e T-04 (ciclo de vida) como base.
2. T-01/T-02 (score/risco) dependem de estágios e checkpoints.
3. T-05/T-06/T-07 por último.

## Lacunas Pendentes (🔴)
- Provisionar cifra de CPF at-rest (`encrypt_cpf`).
- Validar constraints/triggers do baseline com o Data Master antes de reimplementar.
