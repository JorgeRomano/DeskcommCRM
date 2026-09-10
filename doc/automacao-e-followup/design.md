# Automação e Follow-up — Design Técnico

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Interface

### Automação 🟢

| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `runAutomationForEvent` | `(admin, row: EventRow)` | `HandlerResult` |
| `evaluateConditions` | `(conditions: RuleCondition[], context)` | `boolean` |
| `checarGuardasDeContato` | `(ctx: ActionCtx)` | `ResultadoDaGuarda` (`no_contact\|contact_blocked\|no_phone\|consent_declined`) |
| `ActionExecutor` | `{ type, postponeUntil?(ctx,cfg), execute(ctx,cfg) }` | `ActionResultDetail` |

Ações: `add-tag`, `assign-owner`, `create-or-move-lead`, `call-webhook`, `send-whatsapp`, `start-message-flow`, `send-ai-message`.

### Follow-up 🟢

| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `processNode` | `(input)` | `NodeResult` (`advance\|wait\|enqueue_turn\|recheck\|dead\|complete\|fail`) |
| `runFollowupTick` | `(deps, opts?)` | `TickSummary` |
| `completeTurnForEnrollment` | `(db, orgId, enrollmentId, nodeId, result, ...)` | `void` |
| `enrollFollowupFlow` | `(supabase, input)` | `EnrollFollowupResult` |
| `applyReactivityEvent` | `(db, clock, row: EventRow)` | `ReactivitySummary` |

Nós: `trigger`, `wait` (fixed/smart), `condition`, `ai_classify`, `match_reply`, `repeat`, `action`, `end`.

### Escalação / Agenda 🟢

- `expectativaDeAtendimento(db, tenantId, now)` → `{quem, frase}`.
- `pausarIaPorAtendimentoManual(admin, input)` → `boolean`.
- `protecaoDaAgenda(compromissos, settings, agora)` → `ProtecaoAgenda`.

## Fluxo Principal — automação 🟢

1. Evento chega (`lead.created/stage_changed/tag_added`, `contact.tag_added`, `message.received`).
2. Anti-loop + guard de `entity_kind`.
3. Carrega `automation_rules` ativas por `trigger_event`; `buildContext` (lead/contact filtrados por org).
4. Filtra por `evaluateConditions`.
5. Pré-checagem de postpone (all-or-nothing) → `adiado` + retry se fora da janela.
6. Executa ações; agrega desfecho honesto (`success/partial/failed/adiado`); audit se ≠ success.

## Fluxo Principal — follow-up 🟢

1. `runFollowupTick`: `claimDueEnrollments` (lease 120s).
2. `processEnrollment`: assert boundary/agenda; `steps_taken>80` → dead.
3. `processNode` (puro) decide; `applyResult` aplica com idempotência (`${node}:${steps}`).
4. Turnos (send/classify/plan) fecham via `completeTurnForEnrollment` (só active/waiting_reply avançam).

## Fluxos Alternativos 🟢

- **Reatividade:** inbound corta espera (`wokeEarly`); opt-out cancela tudo; handoff `allow/cancel/pause`; resolved retoma.
- **Planejamento adaptativo:** `trigger` decide `timing_plan` uma vez; `MAX_PLAN_RECHECKS=3` → segue sem plano.
- **Agenda:** `AgendaDeferredError` adia; `leitura_indisponivel` re-throw.
- **Escalação:** falha de leitura → frase conservadora (não prometer prazo).

## Dependências 🟢

- `event-log` (handlers `automation-rules`, `reactivity`, `gatilho-etapa`).
- `atendimento` (ServiceBoundary), `agenda` (proteção), `leads` (facts), `ai` (turnos de classify/send), `pacing` (janela).
- `supabase` (adapters supabase-js na web, pg no worker).

## Decisões de Design Identificadas 🟢

| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Anti-loop profundidade 1 | `runAutomationForEvent` (metadata) | 🟢 |
| Agregação honesta (skip+fail juntos) | `engine.ts` comentário + `automacao-diz-a-verdade.spec.ts` | 🟢 |
| Lista positiva de estados que avançam | `turn-bridge.ts:completeTurnForEnrollment` | 🟢 |
| Dead-man 14 rechecks (noite anti-ban) | `node-handlers.ts:MAX_ACTION_RECHECKS` | 🟢 |
| Escopo de reatividade por CONTATO | `reactivity.ts` | 🟢 |

## Estado Interno 🟢

`automation_rules`/`automation_rule_runs`; `followup_flow_pointers`/`versions`; `followup_enrollments` (status/outcome/timing_plan/service_boundary); `followup_enrollment_events` (idempotência).

## Observabilidade 🟢

- `automation_rule_runs` (status + actions_result), audit se ≠ success.
- `outcome-stats` por fluxo (converted/replied/exhausted/opted_out/handoff/in_flight).
- Logs distinguem claim falho de "nada vencido".

## Riscos e Lacunas

- 🟡 `lib/followup/silence-sweep`, `intervencao`, `timing-plan` interno, `graph-schema` detalhado não lidos.
- 🟡 Ações concretas (`actions/*`) além de `create-or-move-lead` não lidas em profundidade.
- 🟡 `lib/escalacao/{atendentes,chamados,retomada,continuidade}` e `lib/agenda/{google,horarios-livres}` não lidos.
