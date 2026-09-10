# Dicionário de Dados — DeskcommCRM

> Gerado pelo **Arqueólogo** (Reversa) em 2026-09-10 · Nível: **Detalhado**
> Escala de confiança: 🟢 CONFIRMADO (colunas lidas em `lib/database.types.ts` ou queries) · 🟡 INFERIDO
>
> **Nota:** o **Data Master** fará a análise completa do schema (constraints, triggers, RPCs, ERD).
> Aqui estão as entidades de domínio relevantes para as regras extraídas nesta escavação.
> Colunas confirmadas contra `lib/database.types.ts` (tipos gerados do schema Supabase).

---

## Entidades centrais do CRM

### `crm_leads` 🟢 — o negócio/demanda

| Campo | Tipo | Obrig. | Notas |
|---|---|---|---|
| `id` | uuid | sim | PK |
| `organization_id` | uuid | sim | tenant (RLS) |
| `pipeline_id` | uuid | sim | FK `crm_pipelines` (ON DELETE RESTRICT) |
| `stage_id` | uuid | sim | FK `crm_stages` (ON DELETE RESTRICT) |
| `contact_id` | uuid | não | FK `contacts` |
| `title` | text | sim | título do card |
| `status` | text | sim | `open` / `won` / `lost` (escrito por trigger no fechamento) |
| `value_cents` | int | não | valor em centavos (nullable — pendência comum na conversão) |
| `currency` | text | não | ISO-4217, default `BRL` |
| `closed_at` | timestamptz | não | escrito pelo trigger `fn_crm_lead_close_on_stage` |
| `lost_reason` | text | não | obrigatório na perda |
| `owner_kind` | text | não | `user` / `ai` |
| `owner_user_id` | uuid | não | dono humano |
| `owner_agent_id` | uuid | não | dono agente |
| `source` | text | sim | `whatsapp` / `meta_ads` / `google_ads` / ... |
| `source_metadata` | jsonb | sim | atribuição de anúncio copiada no nascimento |
| `tags` | text[] | sim | inclui rótulo de anúncio (`Meta_ads`) |
| `position_in_stage` | numeric | sim | posição fracionária no kanban |
| `last_activity_at` | timestamptz | não | base do radar de risco |
| `stage_changed_at` | timestamptz | não | — |
| `custom_fields` | jsonb | sim | campos por tenant |
| `created_at` / `updated_at` | timestamptz | sim | — |

### `crm_pipelines` / `crm_stages` 🟢

- `crm_pipelines`: `id, organization_id, name, slug, position, is_default, is_archived, description, settings (jsonb: canonical_tags)`. Índices únicos: `uniq_crm_pipelines_org_slug` (não parcial), `uniq_crm_pipelines_org_default` (parcial `where is_archived=false`, imediato).
- `crm_stages`: `id, organization_id, pipeline_id, name, slug, position, is_won, is_lost, is_archived, requires_human, expected_duration_hours, agent_stage_hint, last_change_actor_kind, last_change_at`. Índices: `uniq_crm_stages_pipeline_slug` (não parcial), `uniq_crm_stages_pipeline_won`/`_lost` (parciais imediatos), `uniq_crm_stages_pipeline_hint`.
- `ETAPAS_INICIAIS` (semeadas): Novo, Em andamento, Ganho (`is_won`), Perdido (`is_lost`).

### `contacts` 🟢 — a pessoa

| Campo | Tipo | Notas |
|---|---|---|
| `id`, `organization_id` | uuid | PK + tenant |
| `name`, `display_name` | text | rótulo via `rotuloDoContato` |
| `phone_number` | text | E.164; CHECK `contacts_phone_e164_format` |
| `email`, `email_normalized` | text | match de dedup |
| `wa_identity` | text | `phone:+E164` / `lid:<digits>` / null (migration 0027) |
| `wa_lid` | text | dígitos do Linked ID (migration 0122) |
| `is_blocked`, `blocked_reason`, `blocked_at` | bool/text | opt-out (STOP) |
| `force_human` | bool | trava do contato inteiro |
| `ai_authorized_at`, `ai_authorized_reason` | timestamptz/text | elegibilidade da IA |
| `consent` | jsonb | `{marketing:{granted_at, declined_at}}` — gate LGPD |
| `is_anonymized`, `anonymized_at` | bool/timestamptz | redação LGPD (irreversível) |
| `avatar_storage_path` | text | enfileirado em `storage_redaction_queue` antes de zerar |
| `cpf_encrypted`, `cpf_hash` | text | PII cifrada |
| `source`, `source_metadata` | text/jsonb | atribuição de anúncio |
| `is_merged_into`, `merged_at` | uuid/timestamptz | dedup de contatos |
| `tags`, `custom_fields`, `locale` | text[]/jsonb/text | — |

### `conversations` 🟢

`id, organization_id, contact_id, channel_session_id, status (open/pending/closed/resolved/archived), assigned_to_user_id, assignee_kind (user/ai), active_ai_agent_id, active_intent, active_agent_set_at, bot_silenced_until (timestamptz | 'infinity'), last_inbound_at, last_handoff_at, last_handoff_reason, status_changed_at, service_revision, reply_context_revision`.

### `messages` 🟢

`id, organization_id, conversation_id, contact_id, direction (inbound/outbound), type (CHECK messages_type_check: text/audio/image/video/document/sticker/location/contact/reaction), body, status (queued/sending/sent/...), sent_via (ai/user/external_device), external_id (unique com org), reply_to_message_id, media_url, sent_at, created_at, metadata (jsonb: sentiment_score, raw_type)`.

---

## IA e agentes 🟢

### `ai_agents` / `ai_agent_versions`

- `ai_agents`: `id, organization_id, name, is_default, model, config (jsonb: confidence_threshold, sentiment_threshold), operation_mode, archived_at`.
- `ai_agent_versions` 🟡: versão publicada — `system_prompt`, `handoff_keywords`, `knowledge_source_ids`, `pipeline_ids` (escopo de funil), `enabled_models`.

### Fila e memória do engine

- `job_queue` 🟢: `id, organization_id, contact_id, kind (JobKind), status (pending/running/done/dead), attempts, max_attempts, created_at, lease/claim`. Sentinela do boot (migration 0050).
- `lead_checkpoints` 🟢: `commitments[], objections[], next_action, rolling_summary, seq` (por contato).
- `lead_state` 🟢: `stage` (máquina do agente), `qualification` (BANT jsonb).
- `lead_notes` 🟢: memória durável (`headline`, `body`, `supersedes`).
- `llm_calls` 🟢: telemetria por chamada (`purpose`, `provider`, `model`, tokens, `cost_cents`, `latency_ms`, `job_id`).
- `agent_inbox_items` 🟢: Central de avisos — `kind` (handoff / budget_warning / budget_exceeded / conhecimento_nao_indexado / jailbreak_escalation / followup_dead / other), `severity`, `title`, `body`, `ref_kind`, `ref_id`, `status` (open/...). Dedup por episódio aberto.
- `ai_knowledge_sources` 🟢: `id, organization_id, agent_id, source_type, name, status, is_active, source_metadata, last_index_status`.
- `ai_chunks` 🟢: RAG — `content`, `content_hash`, `token_count`, `embedding`, `metadata`, versionado.
- `flywheel_judge_verdicts` / `flywheel_distiller_proposals` 🟢: aprendizado (dedup unique; proposta com gate humano).

---

## Follow-up e automação 🟢

### `followup_flow_pointers` / `followup_flow_versions`

- pointers: `id, organization_id, name, status (active/...), active_version_id, stage_id, handoff_policy (allow/cancel/pause), trigger_config`.
- versions: `graph` (jsonb validado por `flowGraphSchema` — nós `trigger/wait/condition/ai_classify/match_reply/repeat/action/end`).

### `followup_enrollments` 🟢 (migrations 0054/0144/0145/0147)

`id, organization_id, pointer_id, version_id, contact_id, conversation_id, current_node_id, status (active/waiting_reply/paused_handoff/paused_manual/completed/cancelled/dead), outcome (converted/replied/exhausted/opted_out/handoff | null), next_eval_at, claimed_until, attempts, max_attempts, steps_taken, timing_plan (jsonb), service_boundary (jsonb), agent_id, revision, cancel_reason, started_at, completed_at`. Único: 1 enrollment vivo por lead/org (23505 → 409).

### `followup_enrollment_events` 🟢

`organization_id, enrollment_id, node_id, event_type, payload (jsonb), idempotency_key` — idempotência do motor.

### `automation_rules` / `automation_rule_runs` 🟢

- rules: `id, organization_id, name, trigger_event, conditions (jsonb: {field, op eq/neq/contains, value}[]), actions (jsonb: {type, config}[]), is_active, run_count, last_run_at`.
- runs: `organization_id, rule_id, event_id, status (adiado/success/partial/failed), actions_result (jsonb)`.

---

## Canais e agenda 🟢

### `channel_sessions`

`id, organization_id, status (WORKING/SCAN_QR_CODE/...), daily_message_limit, warmup_started_at, warmup_completed_at, is_warmup_complete, metadata (jsonb: ai_gate open/allowlist), archived_at`.

### `channel_knobs` 🟡

Override anti-ban por número: `number_activated_at`, `spinning_knobs (jsonb)`, colunas de pacing (NULL cai no default).

### `calendar_appointments` 🟢

`id, organization_id, contact_id, revision, starts_at, ends_at, status (pending/confirmed/...), meeting_state`.

---

## Governança, LGPD e auditoria 🟢

### `event_log` 🟢 — barramento

`id, organization_id, event_type, entity_kind, entity_id, payload (jsonb), metadata (jsonb: request_id, caused_by_rule, source), consumed_by (text[]), attempts, status (pending/processing/done/dead), next_attempt_at, last_error, created_at, updated_at`. RPC `emit_event`. Trigger `trg_event_log_touch`.

### `lgpd_requests` 🟢

`id, organization_id, request_type (data_request/redact/store_redact), source (nuvemshop/admin_panel/api), contact_id, external_customer_id, status (received/processing/completed/failed/pending_review), scope (contact/tenant), attempts, received_at, due_at (dias úteis BR), completed_at, request_payload (jsonb: delivery, progress), result (jsonb), error_message, cascaded_to, emergency`.

### `storage_redaction_queue` 🟢

`bucket, object_path` (unique) — mídia a apagar após redação LGPD.

### `api_audit_log` 🟢

`action (AUDIT_ACTIONS), actor_user_id, actor_api_token_id, actor_auth_session_id, organization_id, resource_type, resource_id, metadata, request_id, actor_ip, actor_user_agent, bypassed_rls, acting_as_platform_admin, created_at`.

### `webhook_lead_captures` / `webhook_events_log` 🟢

- captures: registro durável (`outcome` criado/duplicado/recusado, `reject_reason` sem CHECK, campos PII limitados a 60×2000).
- events_log: arquivo forense, podado por cron (corpo D+7, linha D+90 — migrations 0163/0174).

### `webhook_sources` 🟢

`id, organization_id, name, default_pipeline_id (ON DELETE CASCADE), secret`.

---

## Multi-tenancy, papéis e plataforma 🟢

### `organizations`

`id, display_name, slug, locale, legal_name (controlador LGPD), cnpj, dpo_email, privacy_policy_url, status (active/redacted), redacted_at, settings (jsonb: agenda, branding, security.mfa_required, campanhas_whatsapp)`.

### `user_organizations`

`user_id, organization_id, role (viewer/agent/manager/admin — CHECK exclui ai_operator), interface_settings, invited_at, accepted_at, revoked_at, created_at`. RPC `fn_user_role_in_org`.

### `platform_admins`

`user_id, scope, mfa_required, revoked_at`. RPC `fn_is_platform_admin`.

### `platform_branding`

Marca da instalação: `app_name, logo_url, logo_path, accent_hex`. (semente/piso em `.env`: `APP_NAME`, `APP_LOGO_URL`, `APP_ACCENT_HEX`.)

### `api_tokens`

`id, organization_id, token_hash (SHA256 \x hex), scopes (text[]: role:/actor:/agent_run:/mcp:read/mcp:write), revoked_at, expires_at, last_used_at`.

### `attendant_availability`

`organization_id, user_id, is_available, capacity, schedule (jsonb com timezone)` — roteamento/escalação.

### Outras tabelas relevantes 🟡

`crm_lead_activities` (timeline; `actor_agent_id` FK), `demandas`/`demanda_conversas` (atendimento), `orders` (payload jsonb, redação preserva valores), `platform_support_sessions` (impersonation), `send_ledger` (idempotência de envio), `cron_jobs` (follow-up agendado kind='at'), `before_send_traces` (auditoria de guardrails).

---

## RPCs (funções SECURITY DEFINER) referenciadas 🟢

`emit_event`, `fn_user_role_in_org`, `fn_is_platform_admin`, `fn_support_context`, `fn_support_callback_write_allowed`, `fn_can_view_conversation`, `fn_user_org_ids`, `fn_upsert_wa_contact`, `fn_upsert_wa_conversation`, `fn_mark_conversation_message`, `fn_crm_lead_close_on_stage`, `fn_seed_default_pipeline_for_org`, `fn_emit_event_on_lead_change`, `fn_claim_due_followup_enrollments`, `fn_lgpd_cascade_redact_contact`, `fn_meet_delivery_policy`, `fn_gasto_de_ia_do_mes`, `fn_role_at_least`.

> 🔴 **LACUNA:** o SQL interno dessas funções e as policies RLS não foram lidos nesta escavação (só os call sites). O **Data Master** deve documentá-los a partir de `supabase/migrations/` e `supabase/baseline.sql`.
