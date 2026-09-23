# Caso de Uso: Fila de Jobs — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas `job_queue` e `event_log` disponíveis com índices únicos de idempotência

## Tarefas
- [ ] T-01, Implementar `enqueueJob` com deduplicação
  - Origem no legado: `lib/agent-engine/queue/queue.ts`
  - Critério de pronto: `sourceEventId` repetido devolve `{deduped:true}` via captura de `23505`
  - Confiança: 🟢
- [ ] T-02, Implementar `claimJobs` de dois estágios
  - Origem no legado: `queue/queue.ts`
  - Critério de pronto: `DISTINCT ON` lane + `FOR UPDATE SKIP LOCKED` sob advisory lock
  - Confiança: 🟢
- [ ] T-03, Implementar `completeJob`/`failJob`/`cancelJob`/`reapExpiredJobs`
  - Origem no legado: `queue/queue.ts`
  - Critério de pronto: lease guard no complete; backoff exponencial cap 120s; dead → inbox
  - Confiança: 🟢
- [ ] T-04, Implementar loop e cron scheduler
  - Origem no legado: `queue/loop.ts`, `cron/scheduler.ts`
  - Critério de pronto: loop fail-open; cron enfileira no mesmo commit do reschedule
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Deduplicação por sourceEventId
- [ ] TT-02, completeJob com lease inválido lança
- [ ] TT-03, Job estourado vira dead e cria inbox

## Ordem Sugerida
1. T-01 → T-02 → T-03 → T-04.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
