# IA e Agentes — Tarefas de Implementação

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Pré-requisitos
- [ ] Schema do harness aplicado (`job_queue`, `lead_checkpoints`, `agent_inbox_items`, `send_ledger` — migration 0050)
- [ ] Postgres com `event_log` + RPC `emit_event`
- [ ] Chave de provedor de IA OU AI Gateway (`ANTHROPIC_API_KEY` / `AI_GATEWAY_API_KEY`)
- [ ] Chave de embedding para RAG
- [ ] `WAHA_API_BASE_URL` + `WAHA_API_KEY` para envio

## Tarefas

- [ ] T-01, Implementar o processo worker (boot, healthz, loops, dispatch por `JobKind`)
  - Origem no legado: `workers/agent-worker/main.ts`
  - Critério de pronto: `assertHarnessSchema` recusa subir sem tabelas; `/healthz` responde contagem de fila
  - Confiança: 🟢

- [ ] T-02, Implementar o turno `inbound_turn` (contexto → tools → checkpoint)
  - Origem no legado: `lib/agent-engine/agent/inbound-turn.ts`
  - Critério de pronto: só `send_message` envia; checkpoint validado por Zod persiste
  - Confiança: 🟢

- [ ] T-03, Implementar a cadeia `before-send` v6 (10 gates na ordem, advisory lock por número)
  - Origem no legado: `lib/agent-engine/guardrails/before-send.ts`
  - Critério de pronto: teste de shape trava ordem/versão; veto volta ao modelo
  - Confiança: 🟢

- [ ] T-04, Implementar pacing e spinning (decisão pura + defaults fonte única)
  - Origem no legado: `lib/agent-engine/pacing/`, `lib/agent-engine/spinning/`
  - Critério de pronto: warm-up falha fechado; spinning veta a 3ª cópia (Jaccard≥0.8)
  - Confiança: 🟢

- [ ] T-05, Implementar guardrails determinísticos de promessa/humano/jailbreak
  - Origem no legado: `guardrails/promise/engine.ts`, `guardrails/human-promise.ts`, `guardrails/jailbreak/classifier.ts`
  - Critério de pronto: promessa fora da tabela vetada; promessa-de-humano sem caso vetada
  - Confiança: 🟢

- [ ] T-06, Implementar orquestrador de handoff + triagem síncrona (G1/G3/G4)
  - Origem no legado: `lib/ai/handoff/orchestrator.ts`, `lib/ai/handoff/triggers.ts`, `regex.ts`
  - Critério de pronto: idempotência 5s; avisa o lead; `bot_silenced_until='infinity'`
  - Confiança: 🟢

- [ ] T-07, Implementar workers legados (resposta + sentimento) sobre `event_log`
  - Origem no legado: `workers/ai-response-worker.ts`, `workers/ai-sentiment-worker.ts`, `workers/ai-handoff-from-sentiment.handler.ts`
  - Critério de pronto: G3 persiste rascunho sem despachar; sentimento emite `ai.sentiment_alert`
  - Confiança: 🟢

- [ ] T-08, Implementar indexador RAG (chunking + versão + ativação)
  - Origem no legado: `workers/rag-indexer.ts`, `lib/ai/rag/chunker.ts`
  - Critério de pronto: nunca ativa versão vazia; falta de chave → retry + item na Central
  - Confiança: 🟢

- [ ] T-09, Implementar servidor MCP (Bearer + scope/role + auditoria)
  - Origem no legado: `lib/mcp/server.ts`, `lib/mcp/auth.ts`, `lib/mcp/types.ts`
  - Critério de pronto: token sem scope → -32002; higieniza uuids de aterro
  - Confiança: 🟢

- [ ] T-10, Implementar flywheel (judge + distiller com gate humano)
  - Origem no legado: `lib/agent-engine/flywheel/live.ts`
  - Critério de pronto: veredito `no` gera proposta pendente; modelo resolvido por provider
  - Confiança: 🟢

## Tarefas de Teste

- [ ] TT-01, Happy path: inbound_turn responde e persiste checkpoint (ver Critérios de Aceitação)
- [ ] TT-02, Veto de promessa fora da tabela volta razão ao modelo
- [ ] TT-03, Orçamento estourado devolve à fila humana sem descartar o job
- [ ] TT-04, Shape da cadeia before-send (ordem/versão/unicidade) — congelar como invariante
- [ ] TT-05, Spinning veta a 3ª cópia idêntica na janela

## Ordem Sugerida
1. T-01 (worker) e a fila são base de tudo.
2. T-03/T-04/T-05 (guardrails) antes de T-02 (turno) poder enviar com segurança.
3. T-06 (handoff) é compartilhado por engine e workers legados.
4. T-07/T-08/T-09/T-10 em paralelo depois do núcleo.

## Lacunas Pendentes (🔴)
- Sequência exata do laço de tools e fail-safes de `send_message.execute` (`inbound-turn.ts`) — ler linha a linha antes de reimplementar.
- Implementação concreta do `WahaChannelAdapter`.
