# ADR 0006 — `event_log` como barramento único de eventos de domínio

> ADR retroativo · Status: **Aceito** · Confiança: 🟢 CONFIRMADO (`lib/event-log/`, `docs/doctrine/sistema-vivo.md`)

## Contexto

O invariante 1 do Sistema Vivo exige que toda peça tenha entrada e saída, e o invariante 3 que toda mutação vire atividade. Muitas reações precisam acontecer depois de uma mensagem/venda/mudança de etapa (responder, classificar sentimento, indexar RAG, LGPD, automação, follow-up, reportar conversão) — acoplar tudo no caminho síncrono da ingestão o tornaria frágil e lento.

## Decisão

Um **barramento único `event_log`** com:
- `emit_event` (RPC) grava a linha; triggers de banco emitem eventos de domínio (nunca fazem HTTP).
- `dispatchEvent` roteia por `event_type` e `consumed_by` (idempotência de retry por handler).
- `drainEventLog` (cron + laço no worker) com **claim otimista** (`processing where pending`) — torna cron e worker seguros em paralelo — e reaper de órfãos (>10min).
- Desfecho por precedência: **retry** (não incrementa attempts) > **error** (attempts++, dead em 5, backoff `2^n`) > **sucesso** (`consumed_by += ok+skipped`).
- 15 handlers registrados em ordem deliberada (`followupReactivity` antes do LLM; `conversaoDeVenda` por último — consumidor externo não pode segurar quem escreve no banco).

## Alternativas consideradas

1. **Chamadas síncronas na ingestão.** Rejeitada: uma exceção viraria 500 para o provider e tempestade de reentregas; acoplaria a ingestão à Meta estar no ar.
2. **Fila externa (Redis/SQS).** Rejeitada para o self-host: `event_log` no mesmo Postgres não adiciona dependência de infra e é transacional com a mutação.
3. **Trigger fazendo HTTP.** Proibido por doutrina (trigger nunca faz HTTP).

## Consequências

- **Positivas:** desacoplamento; idempotência; auditável; cada handler falha isolado sem derrubar os outros.
- **Negativas:** latência de drain (mitigada pelo laço no worker além do cron — a cadeia persist→derive de áudio levava 103–188s só pelo cron).
- **Nota:** o `event_log` não tinha reaper (o `job_queue` sempre teve) — acrescentado para não deixar linhas presas em `processing`.
