# Inventário — DeskcommCRM

> Gerado pelo **Scout** (Reversa) em 2026-09-10
> Escala de confiança: 🟢 CONFIRMADO (extraído do código) · 🟡 INFERIDO · 🔴 LACUNA

## 1. Visão geral

🟢 **DeskcommCRM** — sistema operacional de vendas open source com agentes de IA nativos, WhatsApp como canal primário (via WAHA), multi-tenant com RLS, LGPD by-design. Distribuído como código para self-host em VPS (fonte: `package.json`, `AGENTS.md`, `VISION.md`).

- **Tipo:** aplicação web full-stack monolítica (Next.js App Router) + workers assíncronos.
- **Arquitetura de multi-tenancy:** Postgres com Row-Level Security (RLS) desde o dia 1.
- **Modelo de distribuição:** self-host (imagens Docker publicadas), não SaaS por assinatura.

## 2. Contagem de arquivos por linguagem 🟢

Exclusões aplicadas: `node_modules`, `.git`, `.reversa`, `_reversa_sdd`, `dist`, `build`, `coverage`, `.next`, `.agents`, `.kiro`.

| Linguagem / Tipo | Extensão | Arquivos |
|---|---|---|
| TypeScript | `.ts` | 2258 |
| TypeScript React | `.tsx` | 603 |
| Imagens (evidência/testes) | `.png` | 351 |
| Markdown (docs) | `.md` | 295 |
| SQL (migrations/baseline) | `.sql` | 215 |
| JSON | `.json` | 77 |
| Shell | `.sh` | 32 |
| Texto | `.txt` | 22 |
| YAML (CI/config) | `.yml` / `.yaml` | 19 |
| CSV | `.csv` | 8 |
| Snapshots aprovados | `.approved` | 6 |
| TOML | `.toml` | 5 |
| PDF | `.pdf` | 5 |
| HTML | `.html` | 5 |
| Outros (mjs, css, ps1, js…) | — | ~15 |

**Total de arquivos (sem exclusões):** ~3928.
**Linguagem principal:** TypeScript (estrito, TS 6).

## 3. Tecnologias e frameworks 🟢

Fonte: `package.json`, `next.config.ts`, `tsconfig.json`.

- **Runtime:** Node ≥ 22 (`.nvmrc` = 22; `engines.node`).
- **Framework web:** Next.js 16 (App Router) com `output: standalone` para Docker.
- **UI:** React 19, Tailwind 4, shadcn/ui, Radix UI, Phosphor/Lucide icons.
- **Linguagem:** TypeScript 6 (modo estrito), Zod 4 para validação.
- **Backend/Dados:** Supabase (Postgres + Auth + Realtime + Storage), `pg`, Upstash Redis.
- **IA:** Vercel AI SDK (`ai`), `@ai-sdk/anthropic|openai|google`, `@modelcontextprotocol/sdk`, `gpt-tokenizer`.
- **Canal WhatsApp:** WAHA (engine NOWEB) — integração via `lib/waha/`.
- **E-mail:** Resend. **Push:** `web-push`. **PDF:** `@react-pdf/renderer`.
- **Observabilidade:** Sentry (`@sentry/nextjs` 10).
- **Gerenciador de pacotes:** pnpm 9.15.9 (`packageManager`).
- **Testes:** Vitest 4 (unit), Playwright 1 (E2E), Testing Library, axe-core.

## 4. Pontos de entrada 🟢

| Caminho | Tipo |
|---|---|
| `app/layout.tsx` | App entry (root layout, resolve marca) |
| `app/page.tsx` | Rota raiz |
| `app/app/layout.tsx` | Layout da UI autenticada do tenant |
| `app/(admin)/`, `app/admin/` | UI de plataforma (admin) |
| `proxy.ts` | Middleware do Next 16 (auth de borda, `X-Request-Id`) |
| `instrumentation.ts` / `instrumentation-client.ts` | Instrumentação Sentry |
| `app/api/v1/**/route.ts` | Route handlers REST versionados |
| `app/api/internal/`, `app/api/mcp/`, `app/api/v1/cron/` | Superfícies não-cookie |
| `workers/agent-worker/main.ts` | Worker do agente de IA (via `pnpm worker`) |
| `scripts/dev-crons.ts` | Runner de crons em dev |

## 5. Superfície de API 🟢

- **~253** arquivos `route.ts` sob `app/api/` (route handlers).
- **API v1** (versionada por path) sob `app/api/v1/`, organizada por recurso:
  `admin`, `ads`, `agenda`, `ai`, `attendants`, `audit`, `auth`, `automation-rules`,
  `channel-sessions`, `channels`, `contacts`, `conversation-tags`, `conversations`,
  `cron`, `demandas`, `health`, `integrations`, `lead-captures`, `leads`, `lgpd`,
  `marca`, `mcp`, `message-templates`, `messages`, `metrics`, `notifications`,
  `onboarding`, `pipelines`, `products`, `reports`, `settings`, `system`, `tasks`,
  `team`, `webhook-sources`, `webhooks`.
- **Padrão de handler** (🟢 confirmado em `app/api/v1/leads/`): valida input com Zod →
  guard (`requireRole` / secret) → query com `organization_id` explícito → `ok()`/`fail()`.

## 6. Server Actions e UI 🟢

- `app/actions/` — Server Actions (auth, onboarding, team, settings).
- `app/app/` — UI autenticada do tenant. Módulos de tela:
  `activities`, `ads`, `agenda`, `ai`, `analise`, `audit`, `connections`, `contacts`,
  `crm`, `inbox`, `integrations`, `kanban`, `leads`, `lgpd`, `metrics`, `pipelines`,
  `products`, `radar`, `settings`, `tasks`, `team`, `templates`, `webhooks`.
- `components/` — biblioteca de componentes compartilhados.

## 7. Camada de domínio (`lib/`) 🟢

Módulos de domínio e infraestrutura identificados em `lib/`:

`agenda`, `agent-engine`, `ai`, `api`, `atendimento`, `audit`, `auth`, `automation`,
`branding`, `catalogo`, `channels`, `contacts`, `conversoes`, `crypto`, `email`,
`escalacao`, `event-log`, `followup`, `http`, `i18n`, `impersonate`, `inbox`,
`instalacao`, `kanban`, `leads`, `legal`, `lgpd`, `mcp`, `messaging`, `metrics`,
`navigation`, `net`, `notifications`, `nuvemshop`, `onboarding`, `operacao`, `opt-out`,
`pipelines`, `plataformas-de-anuncio`, `query`, `realtime`, `release`, `relogio`,
`reports`, `retencao`, `routing`, `schemas`, `sentry`, `settings`, `supabase`, `system`,
`tarefas`, `tempo`, `types`, `ui`, `users`, `waha`, `webhooks`.

Núcleos infra canônicos citados em `AGENTS.md`:
- `lib/api/wrappers.ts` — `ok()` / `fail()`.
- `lib/auth/require-role.ts` — `requireRole()` (RBAC canônico).
- `lib/supabase/{browser,server,admin}.ts` — clients (admin bypassa RLS).
- `lib/logger.ts` — log estruturado (console.log proibido).

### Runtime do agente de IA — `lib/agent-engine/` 🟢

Subsistemas: `agent`, `cron`, `db`, `edge`, `flywheel`, `golden-candidates`,
`guardrails`, `health`, `obs`, `pacing`, `playbooks`, `queue`, `spinning`,
mais `channel-adapter.ts` e `surrogates.ts`.

## 8. Workers assíncronos (`workers/`) 🟢

Consomem `event_log` + crons:
- `agent-worker/main.ts` (worker principal do agente).
- `ai-response-worker`, `ai-sentiment-worker`, `ai-handoff-from-sentiment`.
- `lgpd-export-worker`, `lgpd-redact-worker`.
- `media-derive-worker`, `media-persist-worker`, `storage-cleanup-worker`.
- `rag-indexer`.

## 9. Banco de dados (superficial) 🟢

Análise detalhada será feita pelo **Data Master**. Sinais encontrados:
- `supabase/migrations/*.sql` — **~213** migrations versionadas (nunca editar as aplicadas).
- `supabase/baseline.sql` — schema aplicado pelo self-host (apêndice idempotente obrigatório).
- `supabase/migrations/MANIFEST.md` — manifesto de migrations.
- `supabase/config.toml`, `supabase/templates/`.
- `lib/database.types.ts` — tipos gerados do schema (não editar à mão).
- ORM: acesso via cliente Supabase / `pg`, **RLS** como mecanismo central de tenancy.

## 10. Cobertura de testes 🟢

- **Frameworks:** Vitest 4 (unit) e Playwright 1 (E2E); Testing Library + axe-core.
- **~1036** arquivos `*.test.ts(x)` / `*.spec.ts` no repositório.
- **~101** specs Playwright em `tests/e2e/*.spec.ts`.
- Invariantes de banco em `tests/invariants/` (RLS/isolamento, RBAC, governança G1–G6).
- Comandos: `pnpm test:unit`, `pnpm test:db` (Docker), `pnpm test:e2e`, `pnpm gov:verify`.

## 11. CI/CD, Docker e configuração 🟢

- **Workflows** (`.github/workflows/`): `ci.yml`, `e2e.yml`, `perf.yml`, `publish-image.yml`,
  `release.yml`, `relogio.yml`, `acolhida.yml`.
- **Checks obrigatórios** (branch protection main): `verify`, `build-and-size`, `invariants`, `e2e`, `imagens-ok`.
- **Docker:** `Dockerfile`, `Dockerfile.worker`, `Dockerfile.scheduler`; composes
  `docker-compose.yml`, `.prod.yml`, `.build.yml`, `.traefik.yml`.
- **Config:** `next.config.ts`, `tsconfig.json`, `tsconfig.typecheck.json`, `eslint.config.mjs`,
  `postcss.config.mjs`, `playwright.config.ts`, `vitest.config.ts`, `vitest.db.config.ts`,
  `.env.example`, `.env.hostgator.example`, `Caddyfile`, `vercel.ts`.
- **Kit de self-host:** `hostgator-setup-kit/`.

## 12. Documentação e governança existentes 🟢

O repositório já tem doutrina escrita (relevante para o Detective/Architect):
- `CLAUDE.md`, `AGENTS.md`, `ARCHITECTURE.md`, `VISION.md`, `CHANGELOG.md`.
- `docs/` (prd, specs, business-rules, doctrine, runbooks, architecture, threat-model…).
- `.specs/`, `plan/`, `tasks/`, múltiplos `HANDOFF-*.md`.

> **Nota:** conforme `AGENTS.md`, este é um repo com PRDs, specs e regras de negócio já
> escritos. Os agentes seguintes devem cruzar o código com esses documentos e **nunca
> inventar** regra de negócio ou comportamento não confirmado.
