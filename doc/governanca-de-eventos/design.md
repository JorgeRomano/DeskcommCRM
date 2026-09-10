# Governança de Eventos — Design Técnico

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO

## Interface 🟢

| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `registerHandler` | `(handler: EventHandler)` | `void` (overwrite por key) |
| `dispatchEvent` | `(row: EventRow)` | `Promise<HandlerResult[]>` |
| `drainEventLog` | `(admin, opts?: {limit?})` | `Promise<DrainSummary>` |
| `ensureHandlersRegistered` | `()` | `void` (idempotente) |
| `runEventLogDrainLoop` | `(knobs, log, signal)` | `Promise<void>` |

### DTOs 🟢

- `EventRow { id, organization_id, event_type, entity_kind, entity_id, payload, metadata, consumed_by[], attempts, created_at? }`
- `EventHandler { key, events: string[], handle(row): Promise<HandlerResult> }`
- `HandlerResult { consumer_key, status: "ok"|"skipped"|"error"|"retry", retry_at?, detail? }`
- `DrainSummary { scanned, done, retried, failed, dead, pulados? }`

## Fluxo Principal 🟢

1. `emit_event` (RPC) grava linha `pending`; triggers de domínio emitem eventos.
2. `drainEventLog` (cron + laço no worker):
   - Reaper: `processing` >10min → `pending`.
   - Seleciona `pending` (next_attempt_at null ou ≤now), ordenado por `created_at`, `limit`.
   - Claim otimista: `update status='processing' where id=$1 and status='pending'`; se 0 linhas, pula.
   - `dispatchEvent` chama handlers não-consumidos que declaram o `event_type`.
   - Desfecho por precedência (retry > error > sucesso).

## Precedência de desfecho 🟢

| Condição | Ação |
|---|---|
| algum `retry` | `pending`; NÃO incrementa attempts; `next_attempt_at = retry_at ?? backoff(attempts+1)` |
| algum `error` (sem retry) | `attempts++`; ≥5 → `dead`; senão `pending` + `backoff(attempts)` |
| só `ok`/`skipped` | `done`; `consumed_by += keys`; `detail` de skipped preservado |

## Handlers registrados (ordem deliberada) 🟢

`followupReactivity` (antes do LLM), `aiResponse`, `aiSentiment`, `aiHandoffFromSentiment`, `ragIndexer`, `lgpdExport`, `lgpdRedact`, `automationRules`, `followupGatilhoEtapa`, `followupGatilhoCaso`, `followupGatilhoPresenca`, `mediaPersist`, `mediaDerive`, `webPushInbound`, `conversaoDeVenda` (por último — consumidor externo não segura quem escreve no banco).

## Fluxos Alternativos 🟢

- **Handler lança:** `dispatchEvent` captura e converte em `{status:error}` (não derruba o drain).
- **Órfão:** reaper devolve a `pending`.
- **Tick lança (banco fora):** vale espera ociosa (evita tempestade).
- **`carregarDeps` do laço:** imports dinâmicos para não derrubar o worker se `@/lib/env` lançar.

## Dependências 🟢

- `supabase` (admin — service role), `logger`.
- Consumido por praticamente todos os domínios (ver Spec Impact Matrix).

## Decisões de Design Identificadas 🟢

| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Barramento no mesmo Postgres (transacional, sem infra nova) | `drain.ts` | 🟢 |
| Trigger nunca faz HTTP | doutrina + `emit_event` | 🟢 |
| Claim otimista (cron + worker seguros) | `drain.ts` | 🟢 |
| Ordem de handlers deliberada | `register-handlers.ts` | 🟢 |
| Laço no worker além do cron (latência) | `drain-loop.ts` | 🟢 |

## Estado Interno 🟢

`event_log` (status pending/processing/done/dead, consumed_by[], attempts, next_attempt_at, last_error). Trigger `trg_event_log_touch`.

## Observabilidade 🟢

- `DrainSummary` (done/retried/failed/dead/pulados).
- `last_error` preserva detail de skipped e erros do tick.

## Riscos e Lacunas

- 🔴 SQL de `emit_event`, `trg_event_log_touch` e dos triggers emissores não lidos.
- 🟡 Handlers de mídia (`mediaPersist`/`mediaDerive`) e `webPushInbound` não lidos em profundidade.
