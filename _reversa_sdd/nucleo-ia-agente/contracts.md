# Núcleo de IA — Agente — Contratos

> Contratos expostos/consumidos pela unit. A unit não expõe HTTP diretamente (isso é da `superficie-http`);
> seu contrato externo é (a) o toolset modelo↔runtime, (b) os `JobKind` da fila, (c) o `ChannelAdapter`.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Contrato 1 — Toolset do agente (modelo ↔ runtime)

Superfície estática de 13 tools (`AGENT_TOOL_DEFS`, `inbound-turn.ts:190-431`). Schemas largos (`.passthrough()`) para o SDK; validação real por whitelist `.strict()` em cada `apply*`. Campo extra/forjado vira erro de ENSINO ao modelo. 🟢

| Tool | Categoria | Efeito |
|------|-----------|--------|
| `get_lead_context` | read-only | Leituras org-scoped de contacts/conversations/messages/activities |
| `send_message` | write (canal) | Único caminho de envio; passa pela cadeia before-send |
| `update_lead_state` | write (CRM) | Avanço de funil validado pela máquina de estados |
| `schedule_followup` | write | 1 follow-up vivo por lead (anti-empilhamento) |
| `save_lead_note` / `get_lead_note` | write/read | Memória durável por lead com teto de índice |
| `search_knowledge` | read-only | RAG no turno; erros ensinam o modelo a não inventar |
| `request_human_handoff` | handoff | Espelha `requestHumanHandoffInputSchema` (testado) |
| `read_skill_reference` | read-only | Disclosure progressivo de skills situacionais |
| `open_human_case` / `provide_case_update` | write | Loop assíncrono IA↔humano (casos) |
| `send_template` | write (canal) | Envio de template (fora de janela 24h) |

## Contrato 2 — `JobKind` da fila

`inbound_turn | followup_turn | watchdog | flywheel | case_reply_turn | operator_turn | transactional_delivery | approved_reply`. `JobStatus`: `pending | running | done | failed | dead`. — `queue/queue.ts` 🟢

Idempotência: `enqueueJob` devolve `{deduped:true}` ao reencontrar `sourceEventId` (captura `23505`). `completeJob` exige lease válido (`status='running' AND locked_by AND locked_at=acquiredAt`), senão lança → efeito exactly-once. 🟢

## Contrato 3 — `ChannelAdapter` (agnóstico de provedor)

Interface `ChannelAdapter` (`channel-adapter.ts`). `ChannelSendResult`: `sent | already_sent | queued | blocked | failed | unavailable`. O `WahaChannelAdapter` lê o espelho `channel_session_health` e nunca fala com WAHA direto (regra dura). 🟢

## Contrato 4 — Checkpoint do turno (JSON durável)

`checkpointContentSchema` (`inbound-turn.ts:525-545`): 🟢
```
{
  commitments: string[],
  objections: string[],
  next_action: string | null,
  rolling_summary: string,
  declaracao?: DeclaracaoDoTurno   // .optional() SEM default
}
```
`declaracao === undefined` ("modelo não declarou") ≠ `{nada_a_declarar:true}` ("avaliou, nada a declarar"). Banco guarda `null` (`LeadCheckpointRow`). 🟢

## Contrato 5 — Surrogates (métricas de resultado)

Schemas Zod em `surrogates.ts`: união discriminada por `metric` — `leadReplied | stageAdvanced | stopRequested | dropoff | timeToReply`. 🟢
