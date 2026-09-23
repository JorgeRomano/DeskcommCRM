# Fluxogramas — Eventos e tempo real

> Gerado pelo Arqueólogo (Reversa) — Unidade 12.
> Cobre `lib/event-log/`, `lib/realtime/`, `lib/relogio/`, `lib/tempo/`, `lib/agent-engine/cron/`, `workers/`.

---

## 1. Drain do event_log — `drainEventLog` (`event-log/drain.ts`)

```mermaid
flowchart TD
  A[drainEventLog admin, limit=50] --> B[handledTypes = eventos<br/>dos handlers registrados]
  B --> C{handledTypes vazio?}
  C -- sim --> CX[retorna resumo vazio]
  C -- não --> D[Reaper: status='processing'<br/>E updated_at < now-10min]
  D --> E[attempts+1; reclaim otimista]
  E --> E1{primeiraVolta<br/>attempts==0?}
  E1 -- sim --> E2[next_attempt_at=null<br/>reprocessa no mesmo tique]
  E1 -- não --> E3[backoffAt: 2^attempts min]
  E2 --> F
  E3 --> F[Select pending E next_attempt_at<=now<br/>E type in handledTypes, order created_at]
  F --> G[Por linha: claim otimista<br/>update processing where pending]
  G --> H{claimed?}
  H -- não --> HX[continue — outra instância pegou]
  H -- sim --> I[dispatchEvent row]
  I --> J{particiona resultados}
  J -->|retry| J1[pending, NÃO conta attempt,<br/>next_attempt_at = retry_at]
  J -->|error| J2[attempts+1; dead no limite;<br/>avisarEventoMorto se dead]
  J -->|ok/skipped| J3[done; detail do skip<br/>em last_error + summary.pulados]
```

---

## 2. Laço do worker vs. rede de segurança do cron

```mermaid
flowchart LR
  subgraph Worker24x7[Worker 24/7]
    A[runEventLogDrainLoop] --> B[carregarDepsDoLaco<br/>imports DINÂMICOS]
    B --> C{cadeia montou?}
    C -- não --> CX[log.error + aviso degradado<br/>readiness=false]
    C -- sim --> D[log MARCA_LACO_CARREGADO<br/>string de contrato do gate]
    D --> E[loop: drain → proximaEspera<br/>feitos>0 rápido, senão ocioso]
  end
  subgraph RedeSeguranca[Rede de segurança]
    F[relogio tick HTTP 1x/min] --> G[executarTickDoRelogio]
    G --> H[event-log-drain + followup +<br/>routing + recover-stuck]
  end
  E -. mesmo drain, claim otimista .-> H
```

---

## 3. Ciclo de vida de um cron — `fireOneDue` (`agent-engine/cron/scheduler.ts`)

```mermaid
flowchart TD
  A[tickCron: até batchSize] --> B[fireOneDue]
  B --> C[SELECT enabled E next_run_at<=now<br/>FOR UPDATE SKIP LOCKED limit 1]
  C --> D{cron vencido?}
  D -- não --> DX[empty — para o tick]
  D -- sim --> E[savepoint fire]
  E --> F[enqueueJob kind, leadId]
  F --> G[computeNextRunAt spec]
  G --> H{next é null?}
  H -- sim, 'at' --> H1[desabilita cron]
  H -- não --> H2[next_run_at = próximo + stagger]
  H1 --> I[commit → 'fired']
  H2 --> I
  F -. erro .-> J[rollback to savepoint fire]
  J --> K[applyFailure]
  K --> L{permanente OU<br/>attempts>=max?}
  L -- sim --> L1[desabilita + inbox job_dead 1x]
  L -- não --> L2[backoff exponencial]
```

---

## 4. Cron tz-aware com DST — `nextCronTime` (`agent-engine/cron/schedule.ts`)

```mermaid
flowchart TD
  A[nextCronTime expr, tz, afterMs] --> B[parseCronExpr: 5 campos<br/>7→0 em dow]
  B --> C[Intl.DateTimeFormat tz<br/>hourCycle h23, weekday short]
  C --> D[candidate = próximo minuto cheio]
  D --> E[wallClockAt: parede naquele fuso]
  E --> F{matches?<br/>Vixie: dom+dow ambos<br/>restritos → OR, senão AND}
  F -- sim --> G[return candidate]
  F -- não --> H[candidate += 1 min]
  H --> I{estourou<br/>366 dias?}
  I -- sim --> IX[lança: expressão impossível]
  I -- não --> E
```

---

## 5. Boot e shutdown do worker — `startWorker` (`workers/agent-worker/main.ts`)

```mermaid
flowchart TD
  A[startWorker] --> B[assertHarnessSchema<br/>recusa boot sem tabelas 0050]
  B --> C[seedPlatformPlaybook +<br/>reapExpiredJobs no boot]
  C --> D[carregarComportamentoPorPool<br/>ANTES dos laços]
  D --> E[createHealthzServer<br/>/healthz + event_log_drain]
  E --> F[loopsAbort = AbortController]
  F --> G[Liga laços condicionais por env:<br/>drain, event-log, watchdog WAHA,<br/>voz WACALLS, health, flywheel, cron, fila]
  G --> H{SIGTERM/SIGINT}
  H --> I[shuttingDown=true<br/>limpa timers, fecha server]
  I --> J[loopsAbort.abort<br/>await todos os laços]
  J --> K{drain em voo<br/>vs SHUTDOWN_GRACE_MS}
  K -->|grace ganha| K1[log.error + exit 1]
  K -->|drain termina| K2[pool.end + exit limpo]
```

---

## 6. Resolução de hora de parede num fuso — `instanteDe` (`agenda/fuso.ts`)

```mermaid
flowchart TD
  A[instanteDe parede, fuso] --> B[alvo = Date.UTC da parede]
  B --> C[1ª passada: offset lido no palpite]
  C --> D{ehAHoraPedida?}
  D -- sim --> DX[retorna primeiro]
  D -- não --> E[2ª passada: offset no instante achado]
  E --> F{ehAHoraPedida?}
  F -- sim --> FX[retorna segundo]
  F -- não --> G[hora inexistente spring-forward<br/>retorna Math.max primeiro, segundo<br/>= compatible, nunca Invalid Date]
```
