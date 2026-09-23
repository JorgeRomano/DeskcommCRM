# Eventos e Tempo Real — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface (destaques)
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `registerHandler` | `(handler: EventHandler)` | registra por key (hot-reload-friendly) |
| `dispatchEvent` | `(row)` | roda handlers aplicáveis em try/catch |
| `drainEventLog` | `(admin, {limit=50})` | `DrainSummary` |
| `runEventLogDrainLoop` | `(...)` | laço fail-open |
| `fireOneDue` | `(...)` | dispara 1 cron (enfileira + reschedule) |
| `nextCronTime` | `(expr, tz, afterMs)` | próximo instante (tz-aware) |
| `executarTickDoRelogio` | `()` | roda as 4 tarefas do relógio HTTP |
| `instanteDe` | `(parede, fuso)` | `Date` (DST 2 passadas) |

`EventRow`: `{id, organization_id, event_type, entity_kind, entity_id, payload, metadata, consumed_by:string[], attempts, created_at?}`. `HandlerResult`: `{consumer_key, status:"ok"|"skipped"|"error"|"retry", retry_at?, detail?}`. 🟢

## Fluxo Principal — Drain
1. `handledTypes` = eventos dos handlers; vazio → retorna. 🟢
2. Reaper de `processing` preso (`updated_at < now()-PROCESSING_STALE_MS`); órfão conta attempt; 1ª volta reprocessa no mesmo tique. 🟢
3. Select `status='pending' AND next_attempt_at<=now() AND type in handledTypes`. 🟢
4. Claim otimista por linha (`if !claimed continue`). 🟢
5. `dispatchEvent`: retry (volta pending, não conta), error (attempts+1, dead no limite), success/skipped (done). 🟢

## Fluxo Principal — Worker 24/7
`Sentry.init` antes dos imports → `assertHarnessSchema` (recusa boot sem tabelas) → `createHealthzServer` → `startWorker` (pool, seed, reap, comportamento) → laços condicionais por env (drain CRM, event-log, watchdog, voz, health, flywheel, cron, worker) → shutdown gracioso (abort + grace). 🟢

## Fluxo Principal — Cron
`scheduleCronJob` (stagger no 1º next_run_at) → `fireOneDue` (`for update skip locked` → savepoint → `enqueueJob` → reschedule no mesmo commit) → `tickCron` (até batchSize, uma transação cada). `computeNextRunAt`: 'every' colapsa runs perdidos; 'cron' recalcula do agora. 🟢

## Fluxos Alternativos
- Relógio HTTP (`executarTickDoRelogio`): 4 tarefas envolvidas por `uma()` (falha loga, não lança); `aplicarRespostasQueChegaram` fecha o buraco do SIM antecipado. 🟢
- Fuso: `fusoUtilizavel` falha aberta para o padrão; `instanteDe` 2 passadas (spring-forward → `Math.max`). 🟢

## Dependências
- `event-log` → `agent_inbox_items`, trigger `trg_event_log_touch`, handlers de `workers/*.handler.ts`. 🟢
- `worker` → `lib/agent-engine/{queue,cron,edge,health,flywheel}`, `lib/wacalls/events-bridge`, `pg`. 🟢
- `cron` → `job_queue` (enfileira), `Intl`. 🟢
- `realtime` → Supabase Realtime broadcast. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Trigger nunca faz HTTP (event sourcing leve) | ADR-0002 | 🟢 |
| Claim otimista torna worker+cron seguros em paralelo | `event-log/drain.ts` | 🟢 |
| Evento órfão conta tentativa | `event-log/drain.ts` | 🟢 |
| Imports dinâmicos no laço (env não mata o worker) | `drain-loop.ts` | 🟢 |
| Cron enfileira, nunca reimplementa a fila | `cron/scheduler.ts` | 🟢 |
| Relógio/fuso injetável, falha aberta | `tempo/*`, `agenda/fuso.ts` | 🟢 |

## Estado Interno
- `event_log`, `job_queue`, `cron_jobs`, `agent_inbox_items`, `followup_enrollments`. Knobs em `lib/agent-engine/env.ts`. 🟢

## Observabilidade
- `MARCA_LACO_CARREGADO` (string de contrato do gate de publicação); `/healthz` + `/metrics`; `avisarEventoMorto` com dedup. 🟢

## Riscos e Lacunas
- 🔴 `event_log`, `job_queue`, `cron_jobs`, trigger `trg_event_log_touch` e RPCs — Data Master.
- 🟡 Corpos individuais dos `workers/*.handler.ts` (só `media-persist` lido como amostra); ordem/keys vêm de `register-handlers.ts`.
