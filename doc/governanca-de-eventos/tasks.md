# Governança de Eventos — Tarefas de Implementação

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO

## Pré-requisitos
- [ ] Tabela `event_log` (status, consumed_by[], attempts, next_attempt_at, last_error, updated_at)
- [ ] RPC `emit_event` + trigger `trg_event_log_touch`
- [ ] Triggers emissores de domínio (mensagem, lead, etc.)

## Tarefas

- [ ] T-01, Implementar registry + dispatch por consumed_by/event_type
  - Origem no legado: `lib/event-log/dispatcher.ts`
  - Critério de pronto: handler em consumed_by não é chamado; handler que lança vira {status:error}
  - Confiança: 🟢

- [ ] T-02, Implementar drain com claim otimista + reaper
  - Origem no legado: `lib/event-log/drain.ts`
  - Critério de pronto: 2 drains concorrentes processam a linha uma vez; órfão >10min volta
  - Confiança: 🟢

- [ ] T-03, Implementar desfecho por precedência (retry > error > sucesso)
  - Origem no legado: `lib/event-log/drain.ts`
  - Critério de pronto: retry não incrementa attempts; error vira dead em 5; backoff 2^n
  - Confiança: 🟢

- [ ] T-04, Implementar registro idempotente dos handlers em ordem
  - Origem no legado: `lib/event-log/register-handlers.ts`
  - Critério de pronto: 15 handlers na ordem; followupReactivity antes do LLM; conversaoDeVenda por último
  - Confiança: 🟢

- [ ] T-05, Implementar laço de drain no worker (além do cron)
  - Origem no legado: `lib/event-log/drain-loop.ts`
  - Critério de pronto: imports dinâmicos não derrubam o worker; ritmo rápido/ocioso
  - Confiança: 🟢

## Tarefas de Teste

- [ ] TT-01, Claim otimista: 2 drains, linha processada uma vez
- [ ] TT-02, retry vence error no mesmo tick (sem incrementar attempts)
- [ ] TT-03, Reaper devolve órfão >10min a pending
- [ ] TT-04, Handler que lança não derruba o drain

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `event_log` com colunas de fila (status, consumed_by, attempts, next_attempt_at)

## Ordem Sugerida
1. T-01 (dispatch) e T-02/T-03 (drain) são o núcleo.
2. T-04 (registro) depende dos handlers de cada domínio existirem.
3. T-05 (laço) por último.

## Lacunas Pendentes (🔴)
- SQL de `emit_event`, `trg_event_log_touch` e triggers emissores.
- Handlers de mídia e web push — ler antes de reimplementar.
