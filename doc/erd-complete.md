# ERD Completo — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) · 🟢 CONFIRMADO (colunas de `lib/database.types.ts` + queries) · 🟡 cardinalidades inferidas
>
> **Nota:** o **Data Master** produzirá o ERD definitivo (constraints, triggers, índices, todas as ~213 migrations).
> Este ERD cobre as entidades de domínio centrais e seus relacionamentos observados no código.

```mermaid
erDiagram
  organizations ||--o{ user_organizations : "tem membros"
  organizations ||--o{ contacts : "possui"
  organizations ||--o{ crm_pipelines : "possui"
  organizations ||--o{ crm_leads : "possui"
  organizations ||--o{ conversations : "possui"
  organizations ||--o{ ai_agents : "possui"
  organizations ||--o{ channel_sessions : "possui"
  organizations ||--o{ event_log : "particiona"
  organizations ||--o{ api_audit_log : "particiona"
  organizations ||--o{ lgpd_requests : "recebe"
  organizations ||--o{ api_tokens : "emite"
  organizations ||--o{ automation_rules : "define"
  organizations ||--o{ followup_flow_pointers : "define"

  user_organizations }o--|| organizations : "pertence"
  platform_admins ||--o| organizations : "cross-tenant"

  crm_pipelines ||--o{ crm_stages : "contém"
  crm_pipelines ||--o{ crm_leads : "abriga"
  crm_stages ||--o{ crm_leads : "posiciona"

  contacts ||--o{ crm_leads : "origina"
  contacts ||--o{ conversations : "tem"
  contacts ||--o{ calendar_appointments : "agenda"
  contacts ||--o{ followup_enrollments : "inscrito"

  conversations ||--o{ messages : "contém"
  conversations }o--o| channel_sessions : "via"
  conversations }o--o| ai_agents : "atendida por (active_ai_agent_id)"

  crm_leads ||--o{ crm_lead_activities : "registra timeline"
  crm_leads }o--o| ai_agents : "owner_agent_id"

  ai_agents ||--o{ ai_agent_versions : "versiona"
  ai_agents ||--o{ ai_knowledge_sources : "acervo"
  ai_knowledge_sources ||--o{ ai_chunks : "indexa (RAG)"

  contacts ||--o{ lead_checkpoints : "memória do agente"
  contacts ||--o{ lead_state : "estado do agente"
  contacts ||--o{ lead_notes : "notas duráveis"

  followup_flow_pointers ||--o{ followup_flow_versions : "versiona (graph)"
  followup_flow_pointers ||--o{ followup_enrollments : "inscreve"
  followup_enrollments ||--o{ followup_enrollment_events : "trilha (idempotência)"

  automation_rules ||--o{ automation_rule_runs : "executa"

  event_log ||--o{ job_queue : "drena para"
  job_queue ||--o{ llm_calls : "gera telemetria"

  webhook_sources ||--o{ webhook_lead_captures : "capta"
  webhook_sources }o--o| crm_pipelines : "default_pipeline_id (CASCADE)"

  lgpd_requests }o--o| contacts : "sobre"
  lgpd_requests ||--o{ storage_redaction_queue : "enfileira mídia"
```

## Entidades e chaves 🟢

Ver colunas detalhadas em `data-dictionary.md`. Destaques de FK/cardinalidade:

| Relacionamento | Cardinalidade | FK / Notas |
|---|---|---|
| organization → * | 1:N | `organization_id` em toda tabela tenant-aware (RLS) |
| pipeline → stages | 1:N | `crm_stages.pipeline_id` |
| pipeline ← leads | 1:N | `crm_leads.pipeline_id` (ON DELETE RESTRICT) |
| stage ← leads | 1:N | `crm_leads.stage_id` (ON DELETE RESTRICT) |
| contact → leads | 1:N | `crm_leads.contact_id` (1 lead aberto por vez, N no histórico) |
| conversation → messages | 1:N | `messages.conversation_id`; `unique(org, external_id)` |
| agent → versions | 1:N | `ai_agent_versions.agent_id` |
| knowledge_source → chunks | 1:N | `ai_chunks` (versionado) |
| pointer → versions | 1:N | `followup_flow_versions` (graph jsonb) |
| pointer ← enrollments | 1:N | 1 enrollment vivo por lead/org |
| enrollment → events | 1:N | `idempotency_key` |
| webhook_source → pipeline | N:1 | `default_pipeline_id` (ON DELETE **CASCADE** — apagar funil apagaria a fonte) |
| lgpd_request → storage_redaction_queue | 1:N | `unique(bucket, object_path)` |

## Cardinalidades notáveis 🟢

- **contact 1:N crm_leads:** um lead ABERTO por contato, mas N no histórico (uma demanda por vez).
- **conversation N:1 channel_session:** a conversa acontece por uma sessão de número.
- **crm_lead N:1 ai_agent (owner_agent_id):** o negócio pode ter dono agente ou humano (`owner_kind`).

## 🔴 Lacunas

- FKs exatas, `ON DELETE`, índices parciais e triggers vêm das migrations — Data Master.
- Tabelas auxiliares não mapeadas aqui (ex.: `before_send_traces`, `send_ledger`, `cron_jobs`, `flywheel_*`, `demandas`/`demanda_conversas`, `orders`, `platform_support_sessions`, `platform_branding`, `attendant_availability`) — ver `data-dictionary.md`.
