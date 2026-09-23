# IA de Suporte — Contratos

> Contratos externos: (a) catálogo de ~47 tools MCP, (b) auth Bearer MCP, (c) schema Zod de config de agente.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Contrato 1 — Auth do servidor MCP
Bearer `dsk_<prefix>_<secret>` contra `api_tokens`. `resolveApiToken` faz hash SHA256 → lookup `token_hash` → checa `revoked_at`/`expires_at`. Scopes convencionais (sem migration): `role:<r>` (default `agent`), `actor:ai_agent`, `agent_run:<uuid>`, `mcp:read`, `mcp:write`. `deriveActor` → `ai_agent` ou `api_token`, nunca `user`. Erros MCP: `-32001/401`, `-32002/403`, `-32603/500`. 🟢

## Contrato 2 — `McpToolDefinition`
```
{
  name: string,               // wire-contract imutável
  description: string,
  inputSchema,
  category: 'read' | 'write' | 'handoff',
  requiresRole: Role,
  requiresScope: 'mcp:read' | 'mcp:write',
  motivoDoVazio?: fn,         // "não achei" ≠ sucesso (issue #484)
  handler: fn
}
```
🟢

## Contrato 3 — Catálogo de tools MCP (nome · categoria · role)
`ROLE_RANK`: `viewer < agent < ai_operator < manager < admin`. 🟢

| Domínio | Tools | Regra-chave |
|---|---|---|
| contacts | `crm_search_contacts` (r·agent), `crm_get_contact` (r·agent), `crm_propose_contact_field` (w·agent) | CPF nunca em plaintext; propor não grava |
| conversations | `crm_list_conversations`, `crm_get_conversation`, `crm_get_conversation_history` (r·agent) | histórico ≤100 |
| leads | `crm_list_leads`, `crm_get_lead` (r·agent); `crm_create_lead`, `crm_update_lead`, `crm_move_lead_stage` (w·agent) | mover só no mesmo pipeline; agente vira dono |
| messages | `crm_send_whatsapp_message` (w·agent) | idempotência `idempotency_keys` TTL 24h |
| start-conversation | `crm_start_conversation_and_send` (w·manager, apenasHumano) | cold-start; `channel_session_id` obrigatório |
| governance | `crm_assign_conversation`, `crm_manage_tags` (w·agent); `crm_get_queue_status` (r·agent) | assign com optimistic lock; tags ≤40 chars, ≤20 |
| escalacao | `crm_list_available_attendants`, `crm_list_human_cases`, `crm_get_human_case` (r·agent); `crm_add_case_note`, `crm_close_human_case`, `crm_resume_ai_attendance` (w·agent) | `resume_ai_attendance` só pessoa |
| handoff | `crm_request_human_handoff` (handoff·agent) | roteamento G5: alvo → round-robin → fila |
| comercio | `crm_list_contact_orders` (r·agent), `crm_search_products` (r·agent, `motivoDoVazio`) | varredura paginada (1000×10) antes de afirmar vazio |
| evolucao | `crm_search_knowledge`, `crm_list_knowledge_sources`, `crm_list_improvement_proposals`, `crm_get_org_memory` (r·agent); `crm_save_org_memory` (w·ai_operator) | `LIMIAR_PADRAO=0.4`; IA não se auto-aprova |
| privacidade | `crm_list_privacy_requests` (r·agent) | só leitura (IA não anonimiza) |
| operacao | leituras (r·agent); escritas de stage/webhook/toggle (w·manager) | archive nunca apaga (FK RESTRICT) |
| agendamento | `crm_list_event_types`, `crm_find_free_slots`, `crm_list_appointments` (r·agent); `crm_book_appointment`, `crm_find_and_book_appointment`, `crm_reschedule/cancel/confirm`, `crm_set_appointment_outcome` (w·ai_operator) | "marcado ≠ confirmado"; recusa de negócio = resposta |
| retencao | `crm_list_followups`, `crm_list_at_risk_leads` (r·agent); `crm_schedule_followup`, `crm_cancel_followup`, `crm_enroll_followup_flow` (w·ai_operator), `crm_close_demand`, `crm_propose_reactivation` (w·agent) | prazo relativo convertido no servidor; 1 retorno vivo por cliente |

## Contrato 4 — Config declarativa de agente (Zod único front+back)
`AGENT_CONFIG_DEFAULTS`: `temperature 0.4`, `max_tokens 1024`, `context_message_window 20`, `rag_top_k 5`, `rag_similarity_threshold 0.4`, `confidence_threshold 0.6`, `voice "marin"`, `voice_speed 0.85`, `voice_model "gpt-realtime"`. `guardrailKindEnum`: `regex_output_block`, `rag_must_hit`, `regex_input_block`, `window_check`, `contact_flag` (array `.max(50)`). 🟢

## Contrato 5 — Custo e orçamento
`computeCost({model, promptTokens?, completionTokens?, embeddingTokens?})` → centavos (`Math.ceil`). `BudgetStatus`: `{monthly_limit_cents (0=sem teto), current_month_consumed_cents, pct, enforcement_mode, blocked_now, gasto_incompleto}`. 🟢
