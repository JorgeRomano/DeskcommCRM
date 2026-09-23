# Spec Impact Matrix — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) · nível **detalhado** · 🟢 CONFIRMADO salvo nota
> Qual componente impacta qual. Serve para prever o raio de mudança antes de tocar código
> (o "delta sobre o legado" do ciclo forward). Baseada em `code-analysis.md` (seções
> "Dependências") e `modules.json`.

## 1. Matriz de dependência entre módulos

Linha **depende de** coluna (✓ = dependência de código confirmada). Módulos agrupados por domínio.

| ↓ depende de → | agent-engine | ai | mcp | channels | supabase | api | auth | event-log | audit | crypto | tempo |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **agent-engine** | — | ✓ | ✓ | ✓ | ✓ | | | ✓ | ✓ | | ✓ |
| **ai** | ✓ | — | | | ✓ | | | | ✓ | ✓ | |
| **mcp** | | ✓ | — | | ✓ | ✓ | ✓ | | ✓ | | |
| **channels** | | | | — | ✓ | | | ✓ | ✓ | | ✓ |
| **leads/pipelines/kanban** | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ |
| **agenda** | | ✓ | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **financeiro** | | | | | ✓ | ✓ | ✓ | | ✓ | | ✓ |
| **followup** | ✓ | ✓ | | ✓ | ✓ | | | ✓ | ✓ | | ✓ |
| **automation/routing** | ✓ | ✓ | | ✓ | ✓ | | ✓ | ✓ | ✓ | ✓ | ✓ |
| **escalacao** | ✓ | | | ✓ | ✓ | | | ✓ | ✓ | | ✓ |
| **prospecting** | ✓ | ✓ | | ✓ | ✓ | | | ✓ | ✓ | | ✓ |
| **lgpd/legal/retencao** | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **voice/voip/wacalls** | | ✓ | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **external-db/nuvemshop/ads** | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **superfície HTTP (app/api)** | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | | ✓ |

> `supabase`, `api`, `audit`, `crypto`, `tempo`, `env`, `logger` são **infra transversal**: quase
> tudo depende deles, e eles dependem de quase nada. Mudança neles tem raio máximo.

## 2. Componentes de raio de impacto alto (tocar exige cautela)

| Componente | Arquivo | Quem quebra se mudar | Gate que protege |
|---|---|---|---|
| Cadeia `before_send` | `agent-engine/guardrails/before-send.ts` | Todo envio de saída do agente | `BEFORE_SEND_CHAIN_VERSION`, testes de gate |
| Contrato de canal | `channels/types.ts`, `capabilities.ts` | Todos os adapters e o pós-entrada | `pnpm lint:channels` |
| `ok()`/`fail()` | `lib/api/wrappers.ts` | Toda resposta HTTP + `X-Request-Id` | convenção + testes |
| `requireRole` / rank | `lib/auth/require-role.ts`, `auth/types.ts` | Todo guard RBAC + MCP | `lint:role-rank`, invariantes |
| Clients Supabase | `lib/supabase/{server,admin}.ts` | Todo acesso a dados (RLS vs service role) | invariantes de RLS (`test:db`) |
| Fila de jobs | `agent-engine/queue/queue.ts` | Todo turno do agente + follow-up | testes de idempotência/lease |
| `event_log` dispatcher | `lib/event-log/dispatcher.ts` | Todo efeito assíncrono | invariantes de governança G1–G6 |
| Schema do funil | `crm_leads`/`crm_stages`/`crm_pipelines` | Kanban, score, risco, métricas, MCP | migration + baseline + MANIFEST + `test:db` |
| Resolvedor de marca | `lib/branding/*` | `app/layout.tsx` → todas as telas | `tests/unit/branding.test.ts` |
| Contrato MCP (nomes de tool) | `lib/mcp/tools/*` | Agentes/integrações externas (wire-contract) | nome imutável + auditoria |

## 3. Impacto por tipo de mudança

| Se você mexe em… | Precisa também… | Gate obrigatório |
|---|---|---|
| Schema (qualquer tabela) | migration versionada + apêndice em `baseline.sql` + linha no `MANIFEST.md` | `pnpm test:db` |
| Tabela tenant-aware | testar RLS / isolamento cross-tenant | `pnpm test:db` (invariantes) |
| Qualquer mutação de API | chamar `audit()` (fire-and-forget) | DoD |
| Input externo (body/query/path) | validar com Zod | DoD |
| UI ou fluxo de usuário | prova visual (e2e/journey) | `pnpm test:e2e` |
| `Dockerfile*` / `docker-compose*` / `hostgator-setup-kit/` | rodar o kit | `pnpm test:shell` |
| Env var nova | adicionar em `.env.example` **e** `lib/env.ts` | DoD |
| Função nova em `public` (Postgres) | `revoke execute ... from public, anon` + `grant` | `test:db` |
| Rota com `auth-dual` | entrada em `lib/auth/public-paths.ts` | senão `proxy.ts` devolve 401 |
| Feature que muda comportamento visível ao operador | fragmento em `.changes/` (efeito no operador) | `pnpm release:conferir` |

## 4. Cadeias de propagação notáveis 🟢

- **Turno do agente:** `event_log` → `queue` → `inbound-turn` → `edge/llm` (AI Gateway) →
  `guardrails/before-send` → `channels/adapters` → `messages`. Mudar qualquer elo afeta a entrega.
- **Ingestão WhatsApp:** `webhooks/` (HMAC) → `waha/ingest` → `channels/pos-entrada` (ordem fixa:
  opt-out → origem → demanda → campanha → pipeline → despacho) → `event_log`. Ordem é lei (RN-21).
- **Score/risco:** `crm_lead_activities` → `fn_update_last_activity_at` → `risk-radar` /
  `score-formula` → `crm_lead_scores`/`crm_lead_risk_states` → realtime (kanban).
- **Agenda ↔ Google:** `calendar_appointments` (revision) ↔ `google/sync-model` (merge 3-pontas) ↔
  `calendar_external_events`. Conflito vira `google_conflict` pendente.
- **LGPD redact:** `lgpd_requests` → `redact-cascade`/`cascata` → `storage_redaction_queue` →
  `contacts.is_anonymized`. Irreversível; a IA nunca dispara.

## 5. Superfícies de contrato externo (mudança = breaking) 🟢

| Contrato | Onde | Consumidor externo |
|---|---|---|
| Nomes de tool MCP | `lib/mcp/tools/*` | Agentes/integrações via `dsk_` |
| Formato de webhook aceito | `webhooks/`, `waha/envelope.ts` | WAHA/Meta/Zernio |
| JSON da API v1 (snake_case) | `app/api/v1/**` | Integrações do cliente / `auth-dual` |
| `baseline.sql` | `supabase/baseline.sql` | `install.sh`/`update.sh` na VPS |
| Imagens Docker publicadas | `docker-compose.prod.yml` | Operador da VPS |
| Fragmentos de release | `.changes/` → `CHANGELOG.md` | Quem lê antes de `update.sh` |

## Nota de confiança
🟢 As dependências vêm das seções "Dependências" de cada unidade em `code-analysis.md` e de
`modules.json`. Os gates vêm do `AGENTS.md`/`CLAUDE.md`. 🟡 A matriz da seção 1 é uma projeção
lógica: uma célula vazia não garante ausência total de acoplamento indireto via infra transversal.
