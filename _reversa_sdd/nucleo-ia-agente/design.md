# Núcleo de IA — Agente — Design Técnico

> `design.md` — foca no COMO a unit é construída, com base no código legado lido.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface

Entradas: jobs da fila (`job_queue`), drenados de `event_log`. Saídas: mensagens no canal (via adapter), mutações no CRM, checkpoints e traces duráveis.

| Símbolo | Assinatura | Retorno | Observação |
|---------|-----------|---------|------------|
| `runAgentTurn` | `(db, cfg, input)` | `Promise<TurnResult>` | Entrada compartilhada por inbound/followup/case-reply | 🟢 |
| `applyLeadStateUpdate` | `(db, cfg, raw)` | `Promise<LeadStateUpdateResult>` | Valida transição da máquina de estados | 🟢 |
| `performHumanHandoff` | `(db, input)` | `Promise<void>` | Handoff irrevogável | 🟢 |
| `evaluateBeforeSend` | `(initial: GateContext, gates: Gate[])` | `{body, trace, veto, throttleWaitMs}` | Puro, curto-circuita no 1º veto | 🟢 |
| `runBeforeSend` | `(args)` | `Promise<BeforeSendResult>` | Stateful: advisory lock, trace durável, envio | 🟢 |
| `decidePacing` | `(input: PacingInput)` | `PacingDecision` | Janela/warmup/throttle | 🟢 |
| `decideSpinning` | `(input: SpinningInput)` | `SpinningDecision` | sha256 exato + Jaccard | 🟢 |
| `enqueueJob` | `(db, input: EnqueueInput)` | `Promise<{job, deduped}>` | Idempotência evento→job | 🟢 |
| `claimJobs` | `(db, options: ClaimOptions)` | `Promise<JobRow[]>` | Claim de dois estágios | 🟢 |
| `completeJob` | `(db, job, inSameCommit)` | `Promise<void>` | Guard de lease (exactly-once) | 🟢 |
| `runModelCall` | `(db, cfg, input, deps)` | `Promise<ModelCallResult>` | Única costura de LLM | 🟢 |
| `decidirOrcamento` | `(entrada: EntradaDeOrcamento)` | `Veredito` | Escapes ordenados + limiar | 🟢 |
| `wrapToolsWithBreaker` | `(tools, thresholds)` | `ToolSet` | Circuit breaker de tools | 🟢 |

Tools do agente (`AGENT_TOOL_DEFS`, superfície estática de 13, prefixo estável de cache): `get_lead_context`, `send_message`, `update_lead_state`, `schedule_followup`, `save_lead_note`, `get_lead_note`, `search_knowledge`, `request_human_handoff`, `read_skill_reference`, `open_human_case`, `provide_case_update`, `send_template`. — `inbound-turn.ts:190-431` 🟢

## Fluxo Principal

O ritual do turno (`agent/inbound-turn.ts`, cabeçalho `:9-37`): 🟢

1. **Abertura** — `loadPlaybook` (system, por ponteiro) + checkpoint anterior de `lead_checkpoints` (commitments/objections/next_action + rolling summary) + `lead_state` (estágio do funil) + últimas N mensagens via `get_lead_context`. Montagem em `buildOpeningMessage` → `ritualBlocks` (`:1386-1556`).
2. **Loop do modelo** — o modelo chama tools livremente dentro de `maxSteps` (`AGENT_MAX_STEPS`). Enviar é sempre `send_message`; texto direto é descartado. `update_lead_state` marca avanços validados por `lead-state.ts`.
3. **Fechamento** — 2ª chamada com `purpose:'checkpoint'` devolve somente o JSON do checkpoint, validado por Zod (`checkpointContentSchema`, `:525-545`) e persistido (`:573-607`, `:3976-4003`).
4. **Pós-checkpoint** — enfileira `operator_turn` (fire-and-forget) se o papel estiver ligado (`decidirSeEnfileiraOperador` `:1311-1319`, enqueue `:4030-4065`).

Guarda de envio (`send_message.execute`, `:2843-2962+`): `claimsCurrentInboundIsEmpty` (guarda de falso-vazio) → teto `seq >= maxSendsPerTurn` → detecção de promessa fora da tabela → cadeia `runBeforeSend`. `seq` só avança em tentativa real de envio. 🟢

## Fluxos Alternativos

- **Orçamento estourado:** `comHandoffSeOrcamentoAcabar` (`:672-745`) envolve o turno inteiro; em `LlmBudgetExceededError` lê briefing do checkpoint durável (sem LLM), avisa o lead (texto de código), roda `performHumanHandoff` e re-lança. 🟢
- **Reply/Draft assistido:** `draft-reply.ts` (rascunho sob demanda, sem envio), `approved-reply.ts` (envia rascunho aprovado, `reconcileAcceptedSend` idempotente). 🟢
- **Followup:** `followup-turn.ts` — três caminhos: flow-driven, template determinístico (sem LLM, variante por hash), ou `runAgentTurn` normal. Não usa `sendInBubbles`/atraso humano. 🟢
- **Case reply:** `case-reply-turn.ts` — resposta humana a caso aberto re-injeta um turno; `resolved` é terminal. 🟢
- **Operador:** `operator-turn.ts` — muta o CRM, nunca fala; curto-circuita se `nada_a_declarar:true`; roda se `declaracao===null`. 🟢

## Dependências

- `ai` — resolução de modelo/credencial, custo, orçamento, RAG. 🟢
- `channels` — adapter e capabilities de canal (invariante de restrição de canal). 🟢
- `atendimento`, `leads`, `escalacao`, `prospecting` — CRM-side. 🟢
- `supabase` — persistência (admin client no edge). 🟢
- `tempo`, `followup`, `mcp` — relógio, agendamento, tools do CRM. 🟢

## Decisões de Design Identificadas

| Decisão | Evidência no código | Confiança |
|---------|---------------------|-----------|
| Sessão LLM fresca por job; estado no closure (isolamento por construção) | `agent/inbound-turn.ts` | 🟢 |
| Fechamento por 2ª chamada de checkpoint (sempre acontece) em vez de tool | `inbound-turn.ts:573-607` | 🟢 |
| Cadeia before-send declarativa e versionada (`BEFORE_SEND_CHAIN_VERSION = 7`) | `guardrails/before-send.ts` | 🟢 |
| Advisory lock por número (`pg_advisory_xact_lock(hashtext(channelSessionId))`) serializa read-then-act | `guardrails/before-send.ts` | 🟢 |
| Atraso humano pago ANTES de tomar conexão (issue #654) | `guardrails/before-send.ts` | 🟢 |
| Validação real das tools por whitelist `.strict()`, erro vira ensino | `inbound-turn.ts` | 🟢 |
| Claim de dois estágios + advisory lock para `maxConcurrency` | `queue/queue.ts` | 🟢 |

## Estado Interno

- **Por-run (closure):** sequência de envio (`seq`), outcomes, mensagens, estado do circuit breaker de tools. 🟢
- **Durável:** `lead_checkpoints` (checkpoint por seq/job_id), `lead_state` (estágio do funil), `before_send_traces` (trace da cadeia), `llm_calls` (usage/cost/latency), `outbound_copies` (spinning), `pacing_ledger`/`channel_knobs` (pacing). 🟢
- `checkpointDoJob` (filtra por `job_id`) vs `latestCheckpoint` (mais recente por seq) — o Operador N lê a declaração N. 🟢

## Observabilidade

- Logger estruturado (`obs/logger.ts`); `recordRunMetrics` agrega `llm_calls` por run em `metrics`, inclui `run_cache_read_ratio`; `evaluateCacheHitAlert`; `metricsSnapshot`. 🟢
- Traces da cadeia before-send persistidos em `before_send_traces`. 🟢

## Riscos e Lacunas

- 🟡 Valores exatos de env vars (janelas, caps, thresholds) vêm de `env.ts`/`turn-knobs.ts`; conferir defaults na instalação antes de reimplementar.
- 🔴 Comportamento de reconciliação sob falhas parciais de rede na entrega (perda de resposta HTTP do provedor de canal) depende de validação com casos reais além do que `reconcileAcceptedSend` cobre.
