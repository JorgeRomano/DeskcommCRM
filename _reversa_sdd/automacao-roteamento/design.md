# Automação e Roteamento — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface (destaques)
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `runAutomationForEvent` | `(admin, row)` | `Promise<HandlerResult>` |
| `evaluateConditions` | `(conditions, context)` | `boolean` (AND) |
| `decideRouting` | `(input)` | `assign \| skip \| requeue` |
| `selectRoundRobin` | `(eligibles)` | atendente |
| `processNode` | `(input)` | `NodeResult` (advance/wait/enqueue_turn/recheck/dead/complete/fail) |
| `runFollowupTick` | `(deps, opts)` | tick de cron |
| `prepararLinhaDaPassagem` | `(p)` | `LinhaDaPassagem` (pura) |
| `avaliarDevolucao` | `(c, sel)` | decisão de devolução |

## Fluxo Principal — Motor de regras (`automation/engine.ts`)
1. Anti-loop (profundidade 1). 2. Guard de `entity_kind`. 3. Seleção de regras ativas. 4. Evento dirigido (`rule_id`). 5. `buildContext` (contato do compromisso vem da linha de agora). 6. `evaluateConditions` (AND). 7. Pré-checagem de postpone all-or-nothing. 8. Execução por ação. 9. Status derivado (`failed`>`partial`>`adiado`>`success`). 🟢

## Fluxo Principal — Roteamento
`runRoutingWorker` drena `conversation.routing_requested`; claim CAS `pending→processing`; `processEvent` resolve config por Zod, carrega elegíveis, `decideRouting`, executa `assign` via `fn_channel_routing_claim` + `adotarLeadsDoContato` (rodízio distribui conversa; adota o lead sem dono). 🟢

## Fluxo Principal — Follow-up (grafo)
`runFollowupTick` → `claimDueEnrollments` (RPC, lease 120s) → `processEnrollment`: assere fronteiras, carrega grafo pinado + nó + `LeadFacts` + eventos, `processNode`, `applyResult` (evento idempotente `${node}:${steps}`, patch, `enqueue_turn`). Estado reconstruído por `followup_enrollment_events`; nó entrado 2× (`resolveWaitPhase`). 🟢

## Fluxo Principal — Escalação
Passagem: `prepararLinhaDaPassagem` (pura, tetos por coluna, briefing separa citação de paráfrase) → `registrarPassagem` (nunca lança, sem dedup). Devolução: `devolverAtendimentoAoAgente` (6 passos, solta as 3 travas incl. `force_human=false`, emite `ai.handoff_resolved`). Automática: `avaliarDevolucao` conta do último sinal humano. 🟢

## Fluxos Alternativos
- Webhook 3xx → falha (`redirect:"manual"`, nunca seguido); retry 3× (1s/5s), timeout 10s. 🟢
- Reatividade do follow-up por status (nunca carrega o grafo): inbound cancela/acorda; handoff pausa/retoma. 🟢
- Aviso ao suporte: pipeline de ~16 passos, reivindica antes da rede; do passo 15 tudo falha aberto. 🟢

## Dependências
- `automation` → `event-log/dispatcher`, `schemas/webhooks`, `agent-engine/pacing`, `messages/_handler`, `followup/enroll`. 🟢
- `routing` → `event_log`, `schemas/routing`, RPC `fn_channel_routing_claim`, `Intl`. 🟢
- `followup` → RPC `fn_claim_due_followup_enrollments`/`fn_publish_followup_flow_version`, `pg`/`supabase-js`, `agent-engine/queue`. 🟢
- `escalacao` → `passagens_de_atendimento`, `fn_conversation_assign`, `fn_passagem_devolvida`, `event_log`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Anti-loop de profundidade 1 (cadeia regra→regra fica p/ v2) | `automation/engine.ts` | 🟢 |
| Consentimento como gate fixo (não condition) lendo `declined_at` | `guarda-do-contato.ts` | 🟢 |
| Rodízio real derivado de `conversation_assignment_events` | `routing/decide.ts` | 🟢 |
| Plantão calculado por leitura (removido auto-offline sem emissor) | `routing/eligibility.ts` | 🟢 |
| Estado do follow-up por event sourcing | `followup/node-handlers.ts` | 🟢 |
| Passagem = fato imutável; briefing anti-injection | `escalacao/passagem.ts`, `briefing-da-passagem.ts` | 🟢 |

## Estado Interno
- `automation_rules`, `automation_rule_runs`; `conversation_assignment_events`; `followup_enrollments`, `followup_enrollment_events`; `passagens_de_atendimento`; `channel_knobs`/`pacing_ledger`. 🟢

## Observabilidade
- `audit()` só em run não-`success`; avisos de Central (`message_send_stuck`, `aviso_de_caso_nao_entregue`). 🟢

## Riscos e Lacunas
- 🟡 `throttle.ts`: ramo do cap de `channel_session_warmup` inalcançável (sem escritor; `sent` sempre 0) — documentado.
- 🟡 Migração de nó v1→v2 do follow-up exige reescrever arestas no mesmo instante (`classEdgeMatch` não tem a cortesia do canvas).
- 🟡 RPCs (`fn_channel_routing_claim`, `fn_claim_due_followup_enrollments`, `fn_conversation_assign`) vivem no baseline — Data Master.
