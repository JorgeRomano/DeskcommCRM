# Caso de Uso: Fila de Jobs

> Sub-unit de `nucleo-ia-agente`. Event sourcing leve: `event_log` → `job_queue` drenado por workers.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Fila durável em Postgres que garante idempotência evento→job e efeito exactly-once no processamento, com claim de dois estágios para concorrência controlada. 🟢

## Responsabilidades
- Enfileirar jobs a partir de eventos com deduplicação. 🟢
- Reivindicar (claim) jobs respeitando lane única por contato e `maxConcurrency`. 🟢
- Completar/falhar/cancelar jobs com garantia de lease. 🟢
- Reagendar via cron sem reimplementar a fila. 🟢

## Regras de Negócio
- `enqueueJob`: idempotência por `unique` + captura `23505`; `sourceEventId` repetido devolve linha existente (`deduped:true`). 🟢
- `claimJobs`: `DISTINCT ON (coalesce(contact_id,id))` (uma lane por vez) + `FOR UPDATE SKIP LOCKED`, sob `pg_advisory_xact_lock(CLAIM_LOCK_KEY=727258)`. 🟢
- `completeJob`: guard `status='running' AND locked_by AND locked_at=acquiredAt`; `rowCount≠1` → throw (exactly-once). 🟢
- `failJob`: backoff `power(2,attempts-1)*10` cap 120s; `dead` → inbox `job_dead`. 🟢
- Trigger Postgres nunca faz HTTP (ADR-0002). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Enfileirar com deduplicação | Must | Mesmo `sourceEventId` devolve `{deduped:true}` |
| RF-02 | Claim de dois estágios | Must | Uma lane por contato por vez; concorrência limitada por advisory lock |
| RF-03 | Completar com garantia de lease | Must | Lease inválido lança |
| RF-04 | Backoff e dead-lettering | Should | Job estourado vira `dead` e cai no inbox |

## Critérios de Aceitação
```gherkin
Dado um evento já enfileirado
Quando enqueueJob é chamado de novo com o mesmo sourceEventId
Então devolve a linha existente com deduped=true

Dado um job reivindicado por um worker
Quando outro worker tenta completeJob com lease diferente
Então a operação lança (rowCount != 1)
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/agent-engine/queue/queue.ts` | `enqueueJob`, `claimJobs`, `completeJob`, `failJob`, `reapExpiredJobs` | 🟢 |
| `lib/agent-engine/queue/loop.ts` | `rodarLoopDaFila` | 🟢 |
| `lib/agent-engine/cron/scheduler.ts` | `scheduleCronJob`, `fireOneDue` | 🟢 |
