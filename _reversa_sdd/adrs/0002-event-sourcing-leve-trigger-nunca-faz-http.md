# ADR-0002 — Event sourcing leve; trigger Postgres nunca faz HTTP

> ADR **retroativo** (Detetive/Reversa). Confiança: 🟢 CONFIRMADO.
> Evidência: `code-analysis.md` (1.11, 1.14, 3.2, Unidade 12), `event-log/*`, `workers/`.

## Status
Aceito (vigente).

## Contexto
O sistema precisa reagir a fatos (mensagem recebida, lead movido, compromisso criado) disparando
trabalho assíncrono (turno do agente, follow-up, automação, integração de anúncio). A tentação
óbvia é o trigger do Postgres chamar um webhook/worker direto. Isso acopla a transação do banco à
disponibilidade da rede e transforma um `INSERT` numa chamada HTTP que pode travar, repetir ou
falhar dentro do commit.

## Decisão
**Event sourcing leve:** todo fato vira uma linha em `event_log`; **workers drenados por cron**
consomem os eventos fora da transação. O **trigger Postgres nunca faz HTTP** — só escreve o evento.
`message.received` é emitido pelo trigger `trg_messages_emit_event`, **não** pelo código de ingest
(que antes emitia — medido 805 mensagens com 2 eventos, emissão dupla). Idempotência de consumo por
`consumed_by[]` (keys dos handlers); idempotência evento→job por `unique(organization_id,
source_event_id)` + captura de `23505`.

## Alternativas consideradas
1. **Trigger chama webhook/worker (pg_net, http)** — rejeitado: acopla commit à rede; retentativa e
   falha ficam dentro da transação; risco de tempestade de reentrega.
2. **Fila externa dedicada (SQS/RabbitMQ/Redis Streams)** — rejeitado como dependência mandatória:
   contradiz o self-host enxuto (mais um serviço para o operador manter). Upstash Redis é usado, mas
   para rate-limit/debounce, não como barramento de eventos primário.
3. **`event_log` + workers por cron (escolhida)** — só Postgres + processos worker; drena em lotes,
   com backoff e `MAX_ATTEMPTS=5`, `PROCESSING_STALE_MS=10min`.

## Consequências
- **Positivas:** transações curtas e confiáveis; retentativa e observabilidade fora do commit;
  exactly-once na fila do agente (`completeJob` com guarda de lease); sobrevive a self-host modesto.
- **Negativas / custo:** latência de drenagem (cron tick); complexidade de idempotência espalhada
  por consumidor; `created_at` do evento é opcional de propósito e quem descarta por idade precisa
  falhar ABERTO se faltar (RN-44).
- **Derivada:** a coalescência de rajada de inbound (debounce 8s) virou módulo próprio, fora do
  drain (commit `328d83985`), para separar "o que juntar" de "quando drenar".
