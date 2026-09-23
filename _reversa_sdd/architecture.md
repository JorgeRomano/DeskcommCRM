# Arquitetura — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) — fase de Interpretação · nível **detalhado**
> Escala de confiança: 🟢 CONFIRMADO (extraído do código/artefatos) · 🟡 INFERIDO · 🔴 LACUNA
> Fontes: `inventory.md`, `dependencies.md`, `code-analysis.md`, `data-dictionary.md`, `domain.md`,
> `state-machines.md`, `permissions.md`, `adrs/`, `docker-compose.prod.yml`, `.reversa/context/`.
>
> Diagramas C4 detalhados: `c4-context.md`, `c4-containers.md`, `c4-components.md`.
> Modelo de dados completo: `erd-complete.md`. Matriz de impacto: `traceability/spec-impact-matrix.md`.
> Topologia de infraestrutura: `deployment.md`.

---

## 1. Visão geral

**DeskcommCRM** 🟢 é um sistema operacional de vendas open-source com agentes de IA nativos,
multi-nicho (e-commerce, clínicas, imobiliárias, infoprodutos, serviços), com **WhatsApp como canal
primário** e o **CRM inteiro exposto por MCP**. É **multi-tenant com RLS desde o dia 1** e **LGPD
nativa**. A monetização é **self-host em VPS** — o produto é distribuído como código, e quem instala
numa VPS *é* o usuário. Consequência arquitetural direta: **uma mudança que só funciona na máquina do
dev e quebra no clone fresco é bug de produto**, e nada que exija edição manual de arquivo na VPS é
aceitável (ver ADR-0009).

Escala do sistema (🟢, do inventário):

| Métrica | Valor |
|---|---|
| Arquivos versionados | 5.647 |
| TypeScript (`.ts`/`.tsx`) | 3.904 |
| Migrations SQL | 312 (+ `baseline.sql` de 32.097 linhas) |
| Route handlers REST | 339 (`app/api`), 337 em `app/api/v1` |
| Tools MCP | ~47 |
| Testes | 1.486 unit · 254 invariantes de banco · 142 e2e |

## 2. Estilo arquitetural

🟢 O sistema é um **monólito modular Next.js com plano de dados no Postgres/Supabase** e um
**plano de trabalho assíncrono orientado a eventos** (event sourcing leve). Não é microserviços: a
UI e os route handlers vivem no mesmo processo Next 16 (App Router). O que é assíncrono sai por
`event_log` + workers drenados por cron, nunca por trigger fazendo HTTP (ADR-0002).

Pilares (cada um com ADR retroativo):

- **Multi-tenancy com RLS desde o dia 1** (ADR-0001). Toda tabela tenant-aware tem
  `organization_id` e RLS via `fn_user_org_ids()`/`fn_user_role_in_org()`. Service role
  (`lib/supabase/admin.ts`) bypassa RLS e filtra `organization_id` manualmente.
- **Event sourcing leve** (ADR-0002). `event_log` + workers por cron. Trigger nunca faz HTTP.
- **Ritual do turno do agente** (ADR-0003). Cada job vira uma sessão LLM fresca com abertura/loop/
  fechamento por checkpoint durável.
- **Guardrails determinísticos before-send** (ADR-0004). Cadeia versionada de 11 gates entre a
  decisão do modelo e o canal; texto direto do modelo nunca é enviado.
- **Restrição de canal** (ADR-0005). Feature pergunta *capability*, nunca nomeia provedor
  (gate `pnpm lint:channels`).
- **Webhooks fail-closed + efeitos pós-entrada ordenados** (ADR-0006).
- **Pipeline imutável** (ADR-0007). Mover lead cross-pipeline é clonar.
- **Marca própria resolve do banco** (ADR-0008), nunca do `.env`.
- **Packaging: imagem publicada, bump sem editar a VPS** (ADR-0009).
- **CRM como servidor MCP com RBAC e recusa-para-o-modelo** (ADR-0010).

## 3. Camadas e responsabilidades

| Camada | Onde | Responsabilidade |
|---|---|---|
| **Borda** | `proxy.ts` | Injeta `X-Request-Id`/`x-pathname`, autentica sessão antes da rota, resolve impersonation. Allowlist em `lib/auth/public-paths.ts`. |
| **Superfície HTTP** | `app/api/v1/**` (337 rotas), `app/api/internal/`, `app/api/mcp/`, Server Actions em `app/actions/` | Zod → guard → org de fonte confiável → query (RLS ou filtro manual) → `audit()` → `ok()`/`fail()`. |
| **UI** | `app/app/` (tenant), `app/admin/` (plataforma) | Server Components por default; `"use client"` só com estado/evento/browser. |
| **Domínio** | `lib/*` (agent-engine, ai, mcp, channels, leads, agenda, financeiro, followup, ...) | Regras de negócio puras e serviços de aplicação. |
| **Plano de trabalho** | `workers/` + `lib/agent-engine/queue`, `lib/event-log` | Drena `event_log`/fila de jobs por cron; roda o turno do agente. |
| **Dados** | Supabase Postgres, `supabase/migrations/`, `baseline.sql` | Schema versionado; RLS; RPCs `SECURITY DEFINER`. |
| **Infra transversal** | `lib/api`, `lib/supabase`, `lib/crypto`, `lib/audit`, `lib/logger`, `lib/env`, `lib/i18n` | Wrappers de resposta, clients canônicos, cifra AES-GCM, trilha de auditoria, log estruturado. |

## 4. Fluxos-chave

### 4.1 Rota autenticada de tenant 🟢
`request → proxy.ts (X-Request-Id, sessão) → Zod valida input → guard (requireRole /
requirePlatformAdmin / secret) → organization_id de fonte confiável → query (RLS ou filtro manual
de org) → audit() se mutação → ok()/fail()`.

### 4.2 Turno do agente de IA 🟢
`inbound WhatsApp → webhook HMAC + idempotência → event_log → worker → runAgentTurn (RAG + tools
MCP) → guardrails before-send (11 gates) → adapter de canal (WAHA/Meta/Zernio) → handoff humano se o
gatilho disparar`. Entrada em `lib/agent-engine/agent/inbound-turn.ts`. O fechamento do turno é uma
2ª chamada de LLM só com o JSON do checkpoint (sempre acontece).

### 4.3 Superfícies não-cookie 🟢
Cada uma com guard próprio, nunca o cookie de sessão: `cron/` (Bearer `INTERNAL_CRON_SECRET`,
fail-closed), `internal/` (`x-internal-secret`), `mcp/` (Bearer `dsk_...` contra `api_tokens`),
`webhooks/` (HMAC + path token), e parte de `v1/` via `lib/api/auth-dual.ts` (cookie OU bearer).

## 5. Integrações externas (resumo)

Detalhe em `c4-context.md`. Principais sistemas externos:

| Sistema | Papel | Protocolo | Entrada no código |
|---|---|---|---|
| **WAHA** (WhatsApp NOWEB) | canal WhatsApp por QR (primário) | HTTP + webhook HMAC | `lib/waha/`, `channels/adapters/waha.ts` |
| **Meta Cloud API** | WhatsApp oficial | HTTP + webhook | `channels/adapters/meta-cloud.ts` |
| **Zernio / Zernio Social** | BSP | HTTP + webhook | `channels/adapters/zernio.ts` |
| **WaCalls** | voz sobre WhatsApp | SSE + HTTP | `lib/wacalls/` |
| **Asterisk / SIP (VoIP)** | telefonia (ARI + AudioSocket) | ARI/WS + AudioSocket TCP | `lib/voip/`, `workers/voice-agent/` |
| **Vercel AI Gateway** | roteia LLM (Anthropic primário, OpenAI embeddings, Google) | HTTP (AI SDK) | `lib/ai/gateway.ts` |
| **OpenRouter / DeepSeek** | providers de fallback/BYOK | HTTP | `lib/ai/provider-validators.ts` |
| **Google Calendar** | agenda bidirecional (OAuth) | REST v3 + OAuth | `lib/agenda/google/` |
| **Nuvemshop** | e-commerce (OAuth) | REST + OAuth | `lib/nuvemshop/` |
| **Meta Ads / Google Ads** | conversões offline | REST | `lib/plataformas-de-anuncio/` |
| **Resend / SMTP** | e-mail transacional | REST / SMTP | `lib/email/` |
| **Web Push (VAPID)** | notificações push | Web Push | `lib/notifications/` |
| **Upstash Redis** (ou `srh`+redis no self-host) | rate limit / cache / debounce | REST | `lib/ai/dispatcher/rate-limit.ts` |
| **Sentry** | observabilidade | HTTP (tunnel `/monitoring`) | `sentry.*.config.ts`, `instrumentation.ts` |
| **Banco externo do cliente** | leitura de dados de negócio | Postgres (pg) | `lib/external-db/` |

## 6. Dívidas técnicas identificadas 🔴🟡

Do `code-analysis.md` e `domain.md`:

- 🔴 **Catálogo de modelos divergente.** `AGENT_MODELS` (`claude-sonnet-4-6`/`-haiku-4-5`/`-opus-4-7`
  em `guardrails-schema.ts`) diverge de `DEFAULT_*` (`claude-sonnet-5`/`-haiku-4-5` em `gateway.ts`).
  Impacta qual modelo o agente realmente usa — precisa de fonte única.
- 🔴 **READMEs obsoletos.** `lib/ai/README.md` e `PORT-NOTES.md` descrevem arquivos/modelos/versões
  que não existem mais (afirmam Zod v3 / `ai ^6`). Não usar como fonte de regra.
- 🔴 **`sale_orders`/`sale_items`** têm colunas exatas em lacuna; invariantes de imutabilidade e
  comissão congelada são 🟡 inferidos do comportamento (pendente do Data Master).
- 🟡 **`inbound-turn.ts` com ~4.352 linhas** concentra o ritual do turno inteiro — complexidade alta
  num único arquivo (o coração do sistema).
- 🟡 **Motivos de escalação legados** (`ORIGENS_DA_PASSAGEM` `legado_*`) podem ser resíduo de
  migração — confirmar se ainda são caminhos vivos.
- 🟡 **Dispatcher de IA `@deprecated`** (`lib/ai/dispatcher/`) coexiste com o runtime nativo; orgs
  em `ai_dispatch_mode='external'` são puladas (G6-02).
- 🟡 **Rate limit ausente em crons e MCP** (cobre login/signup/reset/convite e webhook de captação +
  dispatcher). Detalhe no threat-model do repositório.

## 7. Modelo de dados (resumo)

O ERD completo está em `erd-complete.md`. Núcleos de entidade:

- **Tenancy & auth:** `organizations`, `user_organizations`, `team_invites`, `api_tokens`,
  `support_sessions`, `platform_settings`.
- **CRM & funil:** `contacts`, `crm_leads`, `crm_pipelines`, `crm_stages`, `crm_lead_scores`,
  `crm_lead_risk_states`, `crm_lead_activities`, `crm_lead_links`, `demandas`, `lead_checkpoints`,
  `lead_state`.
- **Conversa & canal:** `conversations`, `messages`, `channel_sessions`, `agent_inbox_items`,
  `passagens_de_atendimento`, `human_cases`.
- **Agente & IA:** `ai_agents`, `ai_budgets`, `llm_calls`, `knowledge_sources`,
  `knowledge_versions`, `knowledge_chunks`, `outbound_copies`, `idempotency_keys`.
- **Automação & follow-up:** `automation_rules`, `followup_flow_pointers`, `followup_flow_versions`,
  `followup_enrollments`, `followup_enrollment_events`.
- **Agenda & financeiro:** `calendar_appointments`, `calendar_event_types`, `calendar_connections`,
  `calendar_external_events`, `attendant_availability`, `calendar_availability_exceptions`,
  `financial_accounts`, `payment_methods`, `account_plans`, `commission_rules`, `recurring_entries`,
  `sale_orders`, `sale_items`, `catalog_products`.
- **Voz:** `voice_calls`, `org_voice_calls`, `voip_trunk_settings`.
- **Plano de trabalho:** `event_log`, `agent_jobs` (fila), `cron_jobs`.
- **Compliance:** `lgpd_requests`, `storage_redaction_queue`, `audit_log`/`api_audit_log`, `consents`,
  `conversion_ledger`.
- **Integrações:** `external_db_connections`, tabelas de OAuth (`platform_google_oauth`),
  credenciais de anúncio, `extensions`/catálogo declarativo.

---

## Escala de confiança aplicada

Os artefatos deste agente refletem o que os agentes anteriores confirmaram (🟢) e sinalizam
inferências (🟡) e lacunas (🔴). As lacunas de dados finas (colunas de `sale_orders`, ERD físico
completo) ficam para o **Data Master**, que tem acesso ao schema.
