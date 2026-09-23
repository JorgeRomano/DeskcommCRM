# ERD Completo — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) · nível **detalhado**
> 🟢 CONFIRMADO (colunas lidas no código/`data-dictionary.md`) · 🟡 INFERIDO · 🔴 LACUNA
>
> ⚠️ O ERD físico definitivo (todas as colunas, tipos exatos, todos os índices) é trabalho do
> **Data Master**, que lê o schema. Este ERD reflete o que a Escavação e a Interpretação
> confirmaram no código. Toda tabela tenant-aware tem `organization_id uuid not null references
> organizations(id) on delete cascade` (RLS via `fn_user_org_ids()`), omitido nos blocos por
> concisão exceto onde é chave da relação.

## 1. Tenancy, auth e plataforma

```mermaid
erDiagram
    organizations ||--o{ user_organizations : "tem membros"
    organizations ||--o{ team_invites : "convida"
    organizations ||--o{ api_tokens : "emite (dsk_)"
    organizations ||--o{ support_sessions : "recebe suporte"
    platform_settings ||--|| organizations : "governa (signup_mode etc.)"

    organizations {
        uuid id PK
        text name
        text legal_name "controlador LGPD"
        text currency "ISO-4217, def BRL"
        jsonb settings "branding, vocabulary"
        text country "null = BR"
    }
    user_organizations {
        uuid user_id FK "auth.users"
        uuid organization_id FK
        text role "viewer|agent|manager|admin (NUNCA ai_operator)"
        jsonb interface_settings
        text visibility_mode "all|own_and_unassigned|own"
    }
    team_invites {
        uuid id PK
        uuid organization_id FK
        text email
        text role
        timestamptz expires_at "TTL 24h"
        timestamptz accepted_at
        timestamptz revoked_at
    }
    api_tokens {
        uuid id PK
        uuid organization_id FK
        text token_hash "SHA256 de dsk_..."
        text[] scopes "mcp:read, mcp:write, role:<r>"
        timestamptz revoked_at
        timestamptz expires_at
    }
    support_sessions {
        uuid id PK
        uuid organization_id FK
        uuid actor_user_id "platform admin"
        text access_mode "full|support_readonly"
        text status "active|expired|revoked"
        timestamptz expires_at "TTL 1h"
    }
```

## 2. CRM, funil e contatos

```mermaid
erDiagram
    contacts ||--o{ crm_leads : "gera oportunidade"
    crm_pipelines ||--o{ crm_stages : "contém etapas"
    crm_pipelines ||--o{ crm_leads : "imutável (P-01)"
    crm_stages ||--o{ crm_leads : "posiciona"
    crm_leads ||--|| crm_lead_scores : "score 0-100"
    crm_leads ||--|| crm_lead_risk_states : "bucket de risco"
    crm_leads ||--o{ crm_lead_activities : "timeline"
    crm_leads ||--o{ crm_lead_reactivations : "reativação"
    crm_leads ||--o{ crm_lead_links : "vínculo polimórfico"
    contacts ||--o{ lead_checkpoints : "memória por contato"
    contacts ||--o{ lead_state : "estado BANT"
    contacts ||--o{ demandas : "assuntos abertos"

    contacts {
        uuid id PK
        text name
        text display_name "pushName"
        text phone_number
        text wa_lid
        text email_normalized
        boolean is_blocked "opt-out"
        uuid is_merged_into "dedupe"
        boolean is_anonymized "LGPD"
        bytea cpf_encrypted "LACUNA"
        text cpf_hash "sha256"
        jsonb custom_fields
    }
    crm_leads {
        uuid id PK
        uuid contact_id FK
        uuid pipeline_id FK "ON DELETE RESTRICT, imutável"
        uuid stage_id FK "ON DELETE RESTRICT"
        text status "open|won|lost"
        bigint value_cents
        text currency
        uuid owner_user_id
        uuid owner_agent_id "ai_agents"
        text owner_kind "user|ai"
        numeric position_in_stage "fractional STEP 1000"
        text lost_reason
        text external_id "não herdado no clone"
        jsonb source_metadata "clonado_de, movido_para"
    }
    crm_pipelines {
        uuid id PK
        text slug
        boolean is_default
        boolean is_client_pipeline
        jsonb settings "vocabulary, lost_reasons, canonical_tags"
    }
    crm_stages {
        uuid id PK
        uuid pipeline_id FK
        text slug "^[a-z0-9_-]{2,40}$"
        numeric position
        boolean is_won "mutex com is_lost"
        boolean is_lost
        int expected_duration_hours "janela de risco"
    }
    crm_lead_scores {
        uuid lead_id PK
        int ai_probability "0-100"
        jsonb ai_probability_evidence "exige ancora"
        text ai_probability_band "frio|morno|quente"
    }
    crm_lead_risk_states {
        uuid lead_id PK
        text bucket "critico|em_risco|em_voo|em_dia"
        int cold_hours
        timestamptz detected_at
    }
    crm_lead_activities {
        uuid id PK
        uuid lead_id FK
        text type "~45 valores"
        text actor_kind "user|ai|system|rule|contact"
        jsonb evidence "run_ids, trace_ids, llm_call_ids"
    }
    demandas {
        uuid id PK
        uuid lead_id FK "ON DELETE SET NULL"
        uuid contact_id FK
        timestamptz aberta_em
        timestamptz fechada_em
    }
```

## 3. Conversa, canal e escalação

```mermaid
erDiagram
    channel_sessions ||--o{ conversations : "transporta"
    contacts ||--o{ conversations : "participa"
    conversations ||--o{ messages : "contém"
    conversations ||--o{ passagens_de_atendimento : "escala"
    conversations ||--o{ human_cases : "abre caso"
    contacts ||--o{ agent_inbox_items : "gera aviso"

    channel_sessions {
        uuid id PK
        text provider "waha|meta_cloud|zernio|zernio_social|wacalls"
        text status "STARTING|SCAN_QR_CODE|WORKING|STOPPED|FAILED"
        text wacalls_session_id
        timestamptz archived_at
    }
    conversations {
        uuid id PK
        uuid contact_id FK
        uuid channel_session_id FK
        text status "open|pending|claimed|ai_handling|closed|..."
        text assignee_kind "human|ai"
        uuid assigned_to_user_id
        timestamptz bot_silenced_until
        timestamptz awaiting_since
        int service_revision "CAS"
    }
    messages {
        uuid id PK
        uuid conversation_id FK
        text direction "inbound|outbound"
        text external_id "wamid"
        boolean ai_generated
        jsonb media
    }
    passagens_de_atendimento {
        uuid id PK
        uuid conversation_id FK
        uuid caso_id
        text motor "engine|crm"
        text origem "13 valores"
        text motivo_codigo "9 valores"
        boolean cliente_avisado
    }
    human_cases {
        uuid id PK
        uuid conversation_id FK
        text status "awaiting_human|awaiting_lead|resolved|escalated|cancelled"
    }
    agent_inbox_items {
        uuid id PK
        text kind "voice_call_missed|budget_exceeded|..."
        text severity
        text ref_kind
        uuid ref_id
    }
```

## 4. Agente de IA, orçamento e conhecimento (RAG)

```mermaid
erDiagram
    organizations ||--o{ ai_agents : "publica"
    organizations ||--|| ai_budgets : "teto mensal"
    organizations ||--o{ llm_calls : "consome"
    organizations ||--o{ knowledge_sources : "base por tenant"
    knowledge_sources ||--o{ knowledge_versions : "versiona"
    knowledge_versions ||--o{ knowledge_chunks : "indexa (embedding)"
    organizations ||--o{ idempotency_keys : "dedup 24h"

    ai_agents {
        uuid id PK
        text operation_mode "automatic|assisted"
        text system_prompt
        text channel "whatsapp|voice"
        int max_steps
        int rag_top_k "def 5"
    }
    ai_budgets {
        uuid organization_id PK
        bigint monthly_limit_cents "0 = sem teto"
        text enforcement_mode
        timestamptz enforcement_effective_at
    }
    llm_calls {
        uuid id PK
        text purpose "bot_respond|embedding_generate|..."
        bigint cost_cents "Math.ceil, null = incompleto"
        int total_tokens
    }
    knowledge_sources {
        uuid id PK
        text type "faq|documento|conversas|catalogo"
    }
    knowledge_versions {
        uuid id PK
        uuid knowledge_source_id FK
        boolean is_active "uma ativa por fonte"
    }
    knowledge_chunks {
        uuid id PK
        uuid knowledge_source_id FK
        vector embedding "text-embedding-3-small, 1536 dims"
        text content
    }
```

## 5. Automação e follow-up (event sourcing do fluxo)

```mermaid
erDiagram
    organizations ||--o{ automation_rules : "regras"
    organizations ||--o{ followup_flow_pointers : "fluxos"
    followup_flow_pointers ||--o{ followup_flow_versions : "versiona grafo"
    followup_flow_pointers ||--o{ followup_enrollments : "inscreve contatos"
    followup_enrollments ||--o{ followup_enrollment_events : "event sourcing"

    automation_rules {
        uuid id PK
        jsonb conditions "AND, field/op/value"
        jsonb actions "add_tag|assign_owner|call_webhook|send_*|start_flow"
    }
    followup_flow_pointers {
        uuid id PK
        text status
        uuid active_version_id
        jsonb trigger_config
        jsonb handoff_policy
    }
    followup_flow_versions {
        uuid id PK
        jsonb graph "nodes 2..60, edges ..120"
    }
    followup_enrollments {
        uuid id PK
        uuid pointer_id FK
        uuid contact_id FK "unique(org,pointer,contact) vivo"
        text current_node_id
        text status "active|waiting_reply|dormente|paused_*|completed|cancelled|dead"
        int steps_taken
        text outcome
    }
    followup_enrollment_events {
        uuid id PK
        uuid enrollment_id FK
        text node_id
        text idempotency_key "unique por enrollment"
        jsonb payload
    }
```

## 6. Agenda, financeiro e catálogo

```mermaid
erDiagram
    calendar_event_types ||--o{ calendar_appointments : "molde"
    calendar_connections ||--o{ calendar_appointments : "google sync"
    calendar_connections ||--o{ calendar_external_events : "eventos do Google"
    contacts ||--o{ calendar_appointments : "com quem"
    financial_accounts ||--o{ payment_methods : "meio"
    financial_accounts ||--o{ recurring_entries : "recorrência"
    account_plans ||--o{ recurring_entries : "classifica"
    calendar_appointments ||--o{ sale_orders : "comanda (idempotente por appointment)"
    sale_orders ||--o{ sale_items : "itens"
    commission_rules ||--o{ sale_items : "comissão congelada"
    catalog_products ||--o{ sale_items : "produto"

    calendar_appointments {
        uuid id PK
        uuid event_type_id FK
        text status "pending|confirmed|cancelled|completed|no_show"
        text location_kind "in_person|phone|whatsapp|video_link|google_meet"
        int revision "otimista"
        uuid google_connection_id FK
        jsonb google_base_projection "merge 3-pontas"
    }
    calendar_event_types {
        uuid id PK
        int duration_minutes
        int buffer_before_minutes
        int buffer_after_minutes
        int minimum_notice_minutes
        boolean requires_confirmation
    }
    calendar_connections {
        uuid id PK
        uuid user_id FK
        text provider "google_calendar"
        text status "connecting|healthy|token_expired|..."
        bytea oauth_refresh_token_encrypted
    }
    financial_accounts {
        uuid id PK
        text kind "cash|bank|other"
        bigint opening_balance_cents
        text currency
    }
    commission_rules {
        uuid id PK
        uuid attendant_user_id
        uuid event_type_id
        numeric percent "0-100"
    }
    sale_orders {
        uuid id PK
        uuid appointment_id FK "abertura idempotente"
    }
    sale_items {
        uuid id PK
        uuid sale_order_id FK
        uuid event_type_id
        int quantity
        bigint unit_price_cents
        numeric commission_percent "congelado"
    }
    catalog_products {
        uuid id PK
        text codigo "identidade estável ≤60"
        bigint preco_cents
        boolean controla_estoque
    }
```

## 7. Voz e telefonia

```mermaid
erDiagram
    organizations ||--|| org_voice_calls : "opt-in voz WhatsApp"
    organizations ||--|| voip_trunk_settings : "trunk SIP"
    organizations ||--o{ voice_calls : "chamadas (wacalls + sip)"
    channel_sessions ||--o{ voice_calls : "wacalls"
    contacts ||--o{ voice_calls : "peer"

    voice_calls {
        uuid id PK
        text provider "wacalls|sip"
        text wacalls_call_id "unique(org, id)"
        text asterisk_channel_id "AudioSocket UUID"
        text direction "outbound|inbound"
        text status "starting|ringing|connected|ended"
        text end_reason "cru do upstream"
        text peer_phone
        text handled_by "human|ai|ai_then_human"
        bigint duration_ms
        jsonb transcript "{speaker,text,ts}[]"
    }
    org_voice_calls {
        uuid organization_id PK
        boolean enabled "null/ausente = desligado"
        timestamptz risco_aceito_em
    }
    voip_trunk_settings {
        uuid organization_id PK
        text host
        int port
        bytea password_encrypted "AES-GCM"
        text endpoint_name "org-<uuid>-trunk-endpoint"
        boolean is_active
    }
```

## 8. Plano de trabalho (eventos, fila, cron)

```mermaid
erDiagram
    organizations ||--o{ event_log : "emite eventos"
    organizations ||--o{ agent_jobs : "enfileira"
    agent_jobs ||--o| event_log : "dedup source_event_id"
    cron_jobs ||--o{ agent_jobs : "agenda"

    event_log {
        uuid id PK
        text event_type "message.received etc."
        text entity_kind
        uuid entity_id
        text[] consumed_by "handlers que consumiram"
        int attempts "max 5"
        timestamptz created_at "OPCIONAL de propósito"
    }
    agent_jobs {
        uuid id PK
        uuid contact_id "lane da fila"
        text kind "inbound_turn|followup_turn|watchdog|flywheel|..."
        uuid source_event_id "unique + 23505"
        text status "pending|running|done|failed|dead"
        int attempts "max 5"
        timestamptz run_after
        text locked_by "lease"
    }
    cron_jobs {
        uuid id PK
        text kind "at|every|cron"
        bigint interval_ms
        text cron_expr "5 campos"
        text tz "def UTC"
        timestamptz next_run_at "stagger FNV-1a"
    }
```

## 9. Compliance e integrações

```mermaid
erDiagram
    organizations ||--o{ lgpd_requests : "pedidos do titular"
    lgpd_requests ||--o{ storage_redaction_queue : "apaga mídia"
    organizations ||--o{ audit_log : "trilha ~330 ações"
    contacts ||--o{ consents : "consentimento"
    crm_leads ||--o{ conversion_ledger : "conversões offline"
    organizations ||--o{ external_db_connections : "banco do cliente"

    lgpd_requests {
        uuid id PK
        text request_type "data_request|redact|store_redact"
        text scope "contact|tenant"
        text status "received|processing|completed|failed|pending_review"
        timestamptz due_at "SLA D+5/D+10"
        jsonb request_payload "PII"
    }
    storage_redaction_queue {
        uuid id PK
        text status "pending|processing|deleted|failed|skipped"
        int attempts "max 3"
    }
    audit_log {
        uuid id PK
        text action "AUDIT_ACTIONS"
        uuid actor_user_id
        uuid actor_api_token_id
        boolean bypassed_rls
        jsonb metadata "PII redigida"
    }
    consents {
        uuid contact_id FK
        timestamptz marketing_declined_at "recusa registrada"
        timestamptz marketing_granted_at
    }
    conversion_ledger {
        uuid id PK
        text plataforma "meta_ads|google_ads"
        text evento "Purchase"
        text evento_id "<leadId>:Purchase (idempotência)"
        text status "sent|skipped|error"
        bigint valor_centavos
    }
    external_db_connections {
        uuid id PK
        text host
        text ssl_mode "disable..verify-full"
        bytea password "AES-GCM"
        int max_rows
    }
```

## Cardinalidades e regras notáveis 🟢

- `crm_leads.pipeline_id` / `stage_id`: **`ON DELETE RESTRICT`** — pipeline é imutável; mover é
  clonar (P-01, ADR-0007).
- `followup_enrollments`: unique parcial **um vivo por (org, pointer, contact)**.
- `crm_lead_scores` / `crm_lead_risk_states`: **1:1** com o lead (PK = `lead_id`); apagadas quando o
  valor vira null.
- `agent_jobs.source_event_id`: **unique** + captura `23505` = idempotência evento→job.
- `knowledge_versions`: índice único **uma ativa por fonte** (activate desativa a anterior antes).
- `sale_orders.appointment_id`: abertura **idempotente** por compromisso.
- `voice_calls`: **`unique(organization_id, wacalls_call_id)`**; `id` interno é o que vaza ao
  frontend, nunca `wacalls_call_id`.

## Lacunas para o Data Master 🔴
- Colunas físicas exatas de `sale_orders`/`sale_items`, `conversations`, `messages`, `ai_agents`
  (só os campos lidos no código estão aqui).
- Tipos SQL exatos, defaults e a lista completa de índices (o código expõe só um subconjunto).
- Tabelas de plataforma/onboarding/branding não tocadas nas unidades analisadas.
