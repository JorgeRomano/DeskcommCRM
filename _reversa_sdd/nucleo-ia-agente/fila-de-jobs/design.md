# Caso de Uso: Fila de Jobs — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `enqueueJob` | `(db, input: EnqueueInput)` | `Promise<{job, deduped}>` |
| `claimJobs` | `(db, options: ClaimOptions)` | `Promise<JobRow[]>` |
| `completeJob` | `(db, job, inSameCommit)` | `Promise<void>` |
| `failJob` | `(db, job, err)` | `Promise<void>` |
| `cancelJob` | `(db, job)` | `Promise<void>` |
| `reapExpiredJobs` | `(db)` | `Promise<number>` |

`JobKind`: `inbound_turn | followup_turn | watchdog | flywheel | case_reply_turn | operator_turn | transactional_delivery | approved_reply`. `JobStatus`: `pending | running | done | failed | dead`. 🟢

## Fluxo Principal
1. Evento inserido em `event_log` → `enqueueJob` cria `job_queue` (ou deduplica). 🟢
2. Worker roda `rodarLoopDaFila`: consulta relógio só após rodada vazia; fail-open; nunca rejeita (shutdown gracioso). 🟢
3. `claimJobs` reivindica em dois estágios sob advisory lock. 🟢
4. Processamento → `completeJob` (guard de lease) ou `failJob` (backoff). 🟢

## Fluxos Alternativos
- Visibility-timeout: `reapExpiredJobs` recupera jobs presos. 🟢
- Cron: `fireOneDue` (`SELECT FOR UPDATE SKIP LOCKED LIMIT 1` → savepoint → `enqueueJob` + reschedule no mesmo commit). 🟢

## Dependências
- Postgres (`job_queue`, `event_log`), Supabase admin client.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Idempotência por unique + 23505 | `queue.ts` | 🟢 |
| Exactly-once por guard de lease no completeJob | `queue.ts` | 🟢 |
| Cron enfileira, nunca reimplementa a fila | `cron/scheduler.ts` | 🟢 |

## Estado Interno
- `attempts`, `locked_by`, `locked_at`, `run_after` por linha de job. 🟢

## Riscos e Lacunas
- 🟡 `CLAIM_LOCK_KEY=727258` e cap de backoff 120s confirmados no código; visibility-timeout depende de config do worker.
