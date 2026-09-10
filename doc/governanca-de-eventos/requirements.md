# Governança de Eventos

> Spec SDD (híbrida — módulo de topo) · Redator (Reversa) · Nível: Detalhado
> Módulos: `lib/event-log/*` (+ integração com `workers/agent-worker` e crons)
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Visão Geral

O barramento único de eventos de domínio: `emit_event` grava a linha, triggers emitem eventos (nunca fazem HTTP), e o drain (cron + laço no worker) processa cada linha por handlers idempotentes, com claim otimista, reaper de órfãos e desfecho por precedência. Desacopla ingestão de reações (IA, automação, follow-up, LGPD, conversões, mídia, branding). 🟢

## Responsabilidades

- Registrar handlers por `event_type` e roteá-los por `consumed_by`. 🟢
- Drenar `event_log` com claim otimista (cron + laço no worker). 🟢
- Aplicar desfecho por precedência (retry > error > sucesso). 🟢
- Reagir a órfãos (`processing` >10min → `pending`). 🟢

## Regras de Negócio

- `dispatchEvent` só chama handlers cujo `key` não está em `consumed_by` e que declaram o `event_type`. 🟢
- Só drena `event_type` com handler registrado (tipos de crons dedicados ficam intocados). 🟢
- Claim otimista (`processing where pending`) torna cron + worker seguros em paralelo. 🟢
- Reaper: `processing` com `updated_at < now-10min` volta a `pending`. 🟢
- Precedência: **retry** (não incrementa attempts) > **error** (attempts++, dead em 5, backoff `2^n`) > **sucesso** (`consumed_by += ok+skipped`). 🟢
- `HandlerResult.status="retry"` exige `retry_at`. 🟢
- 15 handlers registrados em ordem deliberada (`followupReactivity` antes do LLM; `conversaoDeVenda` por último). 🟢
- Multi-tenancy: handlers usam service role → filtram `organization_id` da row. 🟢

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Registrar handlers e rotear por consumed_by + event_type | Must | Dado handler já em consumed_by, não é chamado de novo |
| RF-02 | Drenar event_log com claim otimista | Must | Dado 2 drains concorrentes, cada linha é processada uma vez |
| RF-03 | Aplicar desfecho por precedência | Must | Dado retry+error no mesmo tick, vence retry (não incrementa attempts) |
| RF-04 | Reaper de órfãos | Should | Dada linha em processing >10min, volta a pending |
| RF-05 | Dead após 5 attempts com backoff | Should | Dado erro persistente, vira dead em attempts>=5 |
| RF-06 | Laço no worker além do cron | Should | Handlers de mídia/branding rodam no ritmo do worker |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Consistência | Claim otimista (`update ... where status='pending'`) | `lib/event-log/drain.ts` | 🟢 |
| Disponibilidade | Reaper de órfãos; tick que lança vale espera ociosa | `drain.ts`, `drain-loop.ts` | 🟢 |
| Idempotência | `consumed_by[]` por handler | `dispatcher.ts` | 🟢 |
| Resiliência | Handler que lança vira `{status:error}` (não derruba o drain) | `dispatcher.ts:dispatchEvent` | 🟢 |
| Observabilidade | `detail` de skipped preservado em last_error + summary.pulados | `drain.ts` | 🟢 |

## Critérios de Aceitação

```gherkin
Dado dois processos drenando o event_log ao mesmo tempo
Quando ambos tentam reivindicar a mesma linha
Então só um reivindica (update where pending) e o outro pula

Dado um handler que retorna retry e outro que retorna error no mesmo tick
Quando o drain decide o desfecho
Então vence retry: a linha volta a pending sem incrementar attempts

Dada uma linha presa em processing há mais de 10 minutos
Quando o reaper roda
Então ela volta a pending para reprocessamento
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Registry + dispatch + drain | Must | Barramento de todo o sistema |
| Claim otimista + idempotência | Must | Cron + worker seguros em paralelo |
| Precedência de desfecho | Must | Retry não pode virar dead prematuro |
| Reaper + dead | Should | Não deixar linha presa nem retry infinito |
| Laço no worker | Should | Latência (áudio persist→derive) |

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/event-log/dispatcher.ts` | `registerHandler`, `dispatchEvent` | 🟢 |
| `lib/event-log/drain.ts` | `drainEventLog`, `backoffAt` | 🟢 |
| `lib/event-log/register-handlers.ts` | `ensureHandlersRegistered` | 🟢 |
| `lib/event-log/drain-loop.ts` | `runEventLogDrainLoop` | 🟢 |
