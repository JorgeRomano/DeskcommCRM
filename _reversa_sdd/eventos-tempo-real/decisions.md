# Eventos e Tempo Real — Decisões

> Referências: `_reversa_sdd/adrs/`. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## D-01 — Trigger de Postgres nunca faz HTTP 🟢
Event sourcing leve: `event_log`/`job_queue` são drenados por worker + cron; o trigger só emite a linha. Ver ADR-0002.

## D-02 — Claim otimista para paralelismo seguro 🟢
`update status='processing' where status='pending'` torna worker-loop + cron/relógio HTTP seguros rodando ao mesmo tempo. — `event-log/drain.ts`.

## D-03 — Evento órfão conta tentativa 🟢
O reaper incrementa attempts de eventos presos em `processing`; sem isso um crash reprocessava para sempre (313× em 2026-09-15). Exceção: 1ª volta reprocessa no mesmo tique. — `event-log/drain.ts`.

## D-04 — Backoff em minutos, diferente da job_queue 🟢
`backoffAt(attempts)=2^attempts` MINUTOS (expoente é attempts direto); a job_queue usa `power(2,attempts-1)*10` segundos. São deliberadamente distintos.

## D-05 — Imports dinâmicos no laço do worker 🟢
A cadeia termina em `@/lib/env`, que lança no topo do módulo; import estático mataria o worker inteiro. `carregarDepsDoLaco` em try/catch nunca lança. — `event-log/drain-loop.ts`.

## D-06 — Cron enfileira, nunca reimplementa a fila 🟢
`fireOneDue` enfileira um job e reagenda no mesmo commit (`for update skip locked` + savepoint). — `cron/scheduler.ts`.

## D-07 — Stagger determinístico anti-thundering-herd 🟢
`staggerOffsetMs` (FNV-1a do contact_id) espalha o disparo sem estado; ajustável por `windowMs`. — `cron/schedule.ts`.

## D-08 — Relógio/fuso injetável, falha aberta 🟢
`agora` é sempre parâmetro (nunca `new Date()` interno); fuso inválido degrada para `America/Sao_Paulo` em vez de tela branca. — `tempo/*`, `agenda/fuso.ts`.

## D-09 — Veto de negócio ≠ incidente de sistema no worker 🟢
`err.terminal===true` (`LlmBudgetExceededError`) → cancelJob + warn; senão failJob + error+Sentry. Impede que bloqueio de orçamento gere N×5 alertas críticos. — `workers/agent-worker/main.ts`.
