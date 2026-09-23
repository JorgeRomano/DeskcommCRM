# Eventos e Tempo Real (`event-log`, `realtime`, `relogio`, `tempo` + `workers`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 12).

## Visão Geral
Filas duráveis no mesmo Postgres, drenadas por um worker 24/7 e (como rede de segurança) por tiques HTTP de cron/relógio; trigger de Postgres NUNCA faz HTTP. Cobre o `event_log` (event sourcing leve com fan-out para handlers registrados), o agendador `cron_jobs` (que ao disparar ENFILEIRA um job, nunca reimplementa a fila), o worker 24/7 (`workers/agent-worker`), o relógio HTTP (rede de segurança sem contêiner scheduler), o relógio/fuso injetável e o Supabase Realtime. 🟢

## Responsabilidades
- Registrar handlers de evento e drenar o `event_log` com claim otimista. 🟢
- Reaper de eventos presos em `processing`; backoff e dead-lettering. 🟢
- Rodar o worker 24/7 com laços condicionais e shutdown gracioso. 🟢
- Agendar crons por contato (tz-aware, anti-thundering-herd). 🟢
- Prover o relógio HTTP de segurança e o relógio/fuso injetável (DST-safe). 🟢
- Centralizar o canal broadcast de alertas (Supabase Realtime). 🟢

## Regras de Negócio
- Trigger de Postgres nunca faz HTTP (event sourcing leve). — ADR-0002 🟢
- Drain com claim otimista (`update status='processing' where status='pending'`); torna worker + cron seguros em paralelo. — `event-log/drain.ts` 🟢
- Evento órfão em `processing` CONTA como tentativa (conserta o laço do PDF envenenado, 313× em 2026-09-15). 🟢
- `MAX_ATTEMPTS=5`; `backoffAt(attempts)=2^attempts` MINUTOS (difere do backoff da job_queue). 🟢
- Handlers registrados EM ORDEM; retry não conta attempt; error conta e vira `dead` no limite. 🟢
- Cron ao disparar ENFILEIRA (`fireOneDue` → `enqueueJob` + reschedule no mesmo commit). — `cron/scheduler.ts` 🟢
- `staggerOffsetMs` (FNV-1a do contact_id) anti-thundering-herd, sem estado. — `cron/schedule.ts` 🟢
- Relógio injetável: `agora` é sempre parâmetro; fuso falha ABERTA para `America/Sao_Paulo`. — `tempo/*`, `agenda/fuso.ts` 🟢
- Worker: veto permanente de negócio (`LlmBudgetExceededError.terminal`) → cancelJob (não N×5 alertas críticos). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Drenar event_log com claim otimista | Must | Só handlers cujo tipo casa e key não está em `consumed_by`; claim evita corrida |
| RF-02 | Reaper de processing preso | Must | Evento órfão conta tentativa; 1ª volta reprocessa no mesmo tique |
| RF-03 | Backoff e dead-lettering | Must | `2^attempts` min; `dead` no limite → inbox `event_dead` |
| RF-04 | Worker 24/7 com laços condicionais | Must | Laços ligados por env; shutdown gracioso com grace |
| RF-05 | Cron enfileira (não reimplementa fila) | Must | `fireOneDue` enfileira + reschedule no mesmo commit |
| RF-06 | Relógio HTTP de segurança | Should | `executarTickDoRelogio` roda as 4 tarefas, falha de uma não derruba |
| RF-07 | Fuso DST-safe, falha aberta | Must | `instanteDe` nunca Invalid Date; fuso inválido → padrão |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Disponibilidade | Loop fail-open (tick que explode mantém espera ociosa) | `event-log/drain-loop.ts` | 🟢 |
| Disponibilidade | Imports dinâmicos no laço (env não mata o worker) | `event-log/drain-loop.ts` | 🟢 |
| Disponibilidade | `assertHarnessSchema` recusa boot sem tabelas essenciais | `workers/agent-worker/main.ts` | 🟢 |
| Escalabilidade | Stagger determinístico anti-thundering-herd | `cron/schedule.ts` | 🟢 |
| Observabilidade | `/healthz` reporta `event_log_drain` nos ramos 200 e 503 | `workers/agent-worker/main.ts` | 🟢 |
| Corretude | Origem do dreno via AsyncLocalStorage (worker vs request) | `event-log/origem-do-dreno.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado um evento e um worker + um cron rodando em paralelo
Quando ambos tentam processar
Então o claim otimista garante que só um o pega

Dado um evento preso em processing além do stale
Quando o reaper roda
Então incrementa attempts (órfão conta) e reprocessa

Dado um cron devido
Quando fireOneDue dispara
Então enfileira um job e reagenda no mesmo commit
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Drain + reaper + backoff (RF-01/02/03) | Must | Coração do event sourcing |
| Worker 24/7 (RF-04) | Must | Processa os laços |
| Cron enfileira (RF-05) | Must | Agendamento sem reimplementar fila |
| Relógio HTTP (RF-06) | Should | Rede de segurança sem scheduler |
| Fuso DST-safe (RF-07) | Must | Correção de horário |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/event-log/dispatcher.ts` | `registerHandler`, `dispatchEvent` | 🟢 |
| `lib/event-log/drain.ts` | `drainEventLog`, `avisarEventoMorto` | 🟢 |
| `lib/event-log/drain-loop.ts` | `runEventLogDrainLoop` | 🟢 |
| `lib/event-log/register-handlers.ts` | `ensureHandlersRegistered` | 🟢 |
| `workers/agent-worker/main.ts` | `startWorker`, `runJob` | 🟢 |
| `lib/agent-engine/cron/schedule.ts` | `nextCronTime`, `staggerOffsetMs`, `computeNextRunAt` | 🟢 |
| `lib/agent-engine/cron/scheduler.ts` | `scheduleCronJob`, `fireOneDue`, `tickCron` | 🟢 |
| `lib/relogio/executar.ts` | `executarTickDoRelogio` | 🟢 |
| `lib/tempo/fusos.ts`, `agora.ts` | `fusoUtilizavel`, `renderAgora`, `isoLocalComOffset` | 🟢 |
| `lib/realtime/channels.ts` | `alertsPlatform` | 🟢 |
