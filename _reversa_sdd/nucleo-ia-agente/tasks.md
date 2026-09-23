# Núcleo de IA — Agente — Tarefas de Implementação

> Sequência executável para reimplementar a unit a partir do legado, com rastreabilidade ao código original.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Dependências da unit disponíveis (ver `design.md`): `ai`, `channels`, `atendimento`, `leads`, `escalacao`, `prospecting`, `supabase`, `tempo`, `followup`, `mcp`
- [ ] Schema/migrations compatíveis: `lead_checkpoints`, `lead_state`, `job_queue`, `event_log`, `before_send_traces`, `llm_calls`, `outbound_copies`, `pacing_ledger`, `channel_knobs`, `agent_inbox_items`
- [ ] Env vars documentadas em `lib/agent-engine/env.ts` (janelas, caps, thresholds, `AGENT_MAX_STEPS`, `MAX_SENDS_PER_TURN`)

## Tarefas

- [ ] T-01, Implementar a fila com claim de dois estágios e efeito exactly-once
  - Origem no legado: `lib/agent-engine/queue/queue.ts`
  - Critério de pronto: `enqueueJob` deduplica por `sourceEventId` (23505); `claimJobs` respeita lane única + `FOR UPDATE SKIP LOCKED` + advisory lock; `completeJob` lança se lease inválido
  - Confiança: 🟢

- [ ] T-02, Implementar o ritual do turno inbound (abrir/loop/fechar checkpoint)
  - Origem no legado: `lib/agent-engine/agent/inbound-turn.ts:9-37`, `:1386-1556`, `:573-607`
  - Critério de pronto: turno abre com playbook+checkpoint+lead_state+histórico, loop dentro de `maxSteps`, fecha com 2ª chamada `purpose:'checkpoint'` validada por Zod
  - Confiança: 🟢

- [ ] T-03, Implementar a guarda de envio e o teto de envios por turno
  - Origem no legado: `lib/agent-engine/agent/inbound-turn.ts:2843-2962+`, `:462`
  - Critério de pronto: envio só via `send_message`; `seq >= maxSendsPerTurn` bloqueia; texto livre descartado
  - Confiança: 🟢

- [ ] T-04, Implementar a máquina de estados do funil
  - Origem no legado: `lib/agent-engine/agent/lead-state.ts:37-45`
  - Critério de pronto: só transições do grafo fixo; regressão rejeitada; `won`/`lost` terminais; upsert atômico + histórico; idempotente no mesmo estágio
  - Confiança: 🟢

- [ ] T-05, Implementar a cadeia before-send (11 gates, v7) com curto-circuito
  - Origem no legado: `lib/agent-engine/guardrails/before-send.ts`
  - Critério de pronto: `evaluateBeforeSend` puro curto-circuita no 1º veto; `runBeforeSend` pega advisory lock por número, paga atraso humano antes de conectar, grava trace durável
  - Confiança: 🟢

- [ ] T-06, Implementar pacing e spinning
  - Origem no legado: `lib/agent-engine/pacing/engine.ts`, `lib/agent-engine/spinning/engine.ts`
  - Critério de pronto: janela na tz do tenant, warmup cap por idade do número, throttle com jitter; spinning por sha256 + Jaccard sobre janela
  - Confiança: 🟢

- [ ] T-07, Implementar orçamento de IA e a costura única de LLM
  - Origem no legado: `lib/agent-engine/edge/llm/orcamento.ts`, `lib/agent-engine/edge/llm/run-model-call.ts`
  - Critério de pronto: `decidirOrcamento` com escapes ordenados e purposes isentos; `runModelCall` resolve config→binding→checa modelo→aplica orçamento→prefixo estável→`generateText`→insere `llm_calls`
  - Confiança: 🟢

- [ ] T-08, Implementar handoff humano irrevogável e casos humanos
  - Origem no legado: `lib/agent-engine/agent/human-handoff.ts:145-320`, `lib/agent-engine/agent/human-cases.ts`
  - Critério de pronto: `performHumanHandoff` seta `force_human`, silencia bot (`bot_silenced_until='infinity'`), cancela crons, cria item de inbox, timeline `handoff_triggered`
  - Confiança: 🟢

- [ ] T-09, Implementar gestão de contexto (compactação, rolling summary, poda de tools)
  - Origem no legado: `lib/agent-engine/compaction.ts`, `lib/agent-engine/prune-tool-results.ts`, `lib/agent-engine/org-memory.ts`
  - Critério de pronto: `maybeCompact` acima de `triggerMessages`; poda opera só no sufixo (preserva prefixo cacheável)
  - Confiança: 🟢

- [ ] T-10, Implementar o circuit breaker de tools e o adapter de canal
  - Origem no legado: `lib/agent-engine/agent/tool-breaker.ts`, `lib/agent-engine/edge/channel/waha-adapter.ts`
  - Critério de pronto: 3 modos de breaker; adapter lê espelho `channel_session_health` (nunca fala com WAHA direto)
  - Confiança: 🟢

- [ ] T-11, Implementar operator-turn, followup-turn, case-reply-turn e approved-reply
  - Origem no legado: `lib/agent-engine/agent/operator-turn.ts`, `followup-turn.ts`, `case-reply-turn.ts`, `approved-reply.ts`
  - Critério de pronto: operador não envia; followup cobre 3 caminhos; case-reply re-injeta turno; approved-reply reconcilia envio idempotente
  - Confiança: 🟢

## Tarefas de Teste

- [ ] TT-01, Happy path do turno inbound: abrir → send_message → fechar checkpoint (ver `requirements.md`, Critérios de Aceitação)
- [ ] TT-02, Texto livre do modelo não gera envio
- [ ] TT-03, Transição de regressão do funil é rejeitada; `won`/`lost` terminais
- [ ] TT-04, Orçamento estourado → `LlmBudgetExceededError` + aviso + handoff + re-lance
- [ ] TT-05, `enqueueJob` deduplica; `completeJob` com lease inválido lança
- [ ] TT-06, Cadeia before-send curto-circuita no gate `stop` para contato `optedOut`

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Migrar `lead_checkpoints`, `lead_state` e `before_send_traces` preservando `seq`/`job_id` (identidade da declaração)

## Ordem Sugerida
1. T-01 (fila) e T-04 (máquina de estados) primeiro — base para o restante.
2. T-02/T-03 (ritual + guarda de envio) dependem da fila.
3. T-05/T-06/T-07 (guardrails, pacing/spinning, orçamento) antes de habilitar envio real.
4. T-08 a T-11 por último (variantes de turno e resiliência).

## Lacunas Pendentes
- 🟢 Reconciliação sob perda de resposta HTTP do provedor: política é **prevenir duplicata a todo custo** (não reenviar em caso de dúvida; tolera não-entrega, nunca duplica). <!-- [Revisão] usuário confirmou 2026-09-23 -->
- 🟡 Valores exatos de env/knobs de produção (confirmar `env.ts`/`turn-knobs.ts` na instalação).
