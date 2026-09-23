# Eventos e Tempo Real — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas: `event_log`, `job_queue`, `cron_jobs`, `agent_inbox_items`; trigger `trg_event_log_touch`
- [ ] Env (`lib/agent-engine/env.ts`): `EVENT_LOG_DRAIN_*`, `CRON_TICK_INTERVAL_MS`, `QUEUE_*`, `HEALTH_PORT`
- [ ] Migration 0050 (harness schema); `INTERNAL_SECRET` (relógio)

## Tarefas
- [ ] T-01, Implementar dispatcher e drain do event_log
  - Origem no legado: `lib/event-log/dispatcher.ts`, `drain.ts`, `origem-do-dreno.ts`, `aviso-de-evento-morto.ts`
  - Critério de pronto: claim otimista; reaper (órfão conta attempt, 1ª volta reprocessa); backoff `2^attempts` min; dead → inbox
  - Confiança: 🟢
- [ ] T-02, Implementar o laço de drain e registro de handlers
  - Origem no legado: `lib/event-log/drain-loop.ts`, `register-handlers.ts`
  - Critério de pronto: imports dinâmicos; fail-open; `MARCA_LACO_CARREGADO`; handlers na ordem
  - Confiança: 🟢
- [ ] T-03, Implementar o worker 24/7
  - Origem no legado: `workers/agent-worker/main.ts`
  - Critério de pronto: `assertHarnessSchema`; laços condicionais por env; `/healthz`+`/metrics`; veto terminal → cancelJob; shutdown gracioso
  - Confiança: 🟢
- [ ] T-04, Implementar o agendador cron
  - Origem no legado: `lib/agent-engine/cron/schedule.ts`, `scheduler.ts`
  - Critério de pronto: `parseCronExpr` (regra Vixie dom/dow); stagger FNV-1a; `fireOneDue` enfileira + reschedule no mesmo commit
  - Confiança: 🟢
- [ ] T-05, Implementar o relógio HTTP e relógio/fuso
  - Origem no legado: `lib/relogio/*`, `lib/tempo/*`, `lib/agenda/fuso.ts`
  - Critério de pronto: 4 tarefas (falha de uma não derruba); `aplicarRespostasQueChegaram`; fuso falha aberta; `instanteDe` DST 2 passadas
  - Confiança: 🟢
- [ ] T-06, Implementar o canal de realtime
  - Origem no legado: `lib/realtime/channels.ts`
  - Critério de pronto: `alertsPlatform` centraliza o nome do canal broadcast
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Claim otimista: só um processa
- [ ] TT-02, Órfão conta tentativa; 1ª volta reprocessa
- [ ] TT-03, Cron enfileira + reagenda no mesmo commit
- [ ] TT-04, Stagger determinístico (FNV-1a)
- [ ] TT-05, `instanteDe` nunca Invalid Date (spring-forward → max)

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `event_log` com `consumed_by`/`attempts`/`next_attempt_at`; trigger `trg_event_log_touch`

## Ordem Sugerida
1. T-01/T-02 (event-log) e T-04 (cron) primeiro.
2. T-03 (worker) orquestra os laços.
3. T-05/T-06 (relógio/realtime) por último.

## Lacunas Pendentes (🔴)
- Tabelas/triggers e RPCs (Data Master).
- Corpos dos `workers/*.handler.ts` (ler antes de reimplementar cada handler).
