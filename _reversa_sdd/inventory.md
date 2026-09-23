# Inventário — DeskcommCRM

> Gerado pelo **Scout** (Reversa) em 2026-09-22 · nível de documentação: `completo`
> Escala de confiança: 🟢 CONFIRMADO (extraído do código) · 🟡 INFERIDO · 🔴 LACUNA

## 1. Visão geral

**DeskcommCRM** 🟢 — sistema operacional de vendas open source com agentes de IA nativos,
multi-nicho (e-commerce, clínicas, imobiliárias, infoprodutos, serviços). WhatsApp como canal
primário via WAHA. CRM exposto por MCP. Multi-tenant com RLS desde o dia 1, LGPD nativa.
Monetização por self-host em VPS (não assinatura). O produto é distribuído como código.

- **Total de arquivos versionados:** 5.647 🟢
- **Linguagem principal:** TypeScript (estrito) 🟢
- **Runtime:** Node ≥ 22 · gerenciador **pnpm 9.15.9** 🟢

## 2. Contagem por linguagem 🟢

| Linguagem | Extensões | Arquivos |
|---|---|---|
| TypeScript | `.ts`, `.tsx` | 3.904 (3.117 `.ts` + 787 `.tsx`) |
| SQL | `.sql` | 320 |
| Shell | `.sh` | 79 |
| JS/ESM | `.mjs`, `.js`, `.cjs` | 40 |
| CSS | `.css` | 2 |
| Markdown (docs) | `.md` | 422 |

## 3. Frameworks e bibliotecas principais 🟢

Majors verificadas contra `package.json` (declaração canônica é o próprio arquivo):

- **Next.js 16** — App Router, Turbopack. Middleware de borda renomeado para `proxy.ts`.
- **React 19** · **TypeScript 6** estrito
- **Tailwind CSS 4** — configuração em CSS (`app/globals.css`), não em `tailwind.config.js`
- **shadcn/ui** (`new-york`) sobre **Radix UI** · `components.json`
- **Supabase** — Postgres + Auth + Realtime + Storage; `@supabase/supabase-js` 2.x, `@supabase/ssr` 0.12.x
- **Zod 4** — validação de todo input externo
- **Vercel AI SDK** (`ai` 7.x) com `@ai-sdk/anthropic|openai|google` 4.x
- **@modelcontextprotocol/sdk** 1.x — CRM exposto por MCP
- **TanStack React Query** 5.x + Virtual + devtools
- **Upstash Redis** 1.x — rate limit / cache
- **Sentry** (`@sentry/nextjs`) 10.x
- **pg** 8.x (node-postgres) · **nodemailer** 10.x / **resend** 6.x (e-mail)
- **@react-pdf/renderer** 4.x (PDF LGPD) · **qrcode**, **web-push**, **ws**
- **@xyflow/react** (fluxos) · **recharts** (gráficos) · **@hello-pangea/dnd** (kanban)

**Testes:** Vitest 4.x (unit + invariantes de banco) · Playwright 1.x + `@axe-core/playwright` (e2e + journeys)

## 4. Estrutura de pastas (top-level relevante) 🟢

```
app/                 Next.js App Router (UI + Route Handlers)
  api/               Route handlers REST — 339 route.ts no total
    v1/              337 route.ts, REST versionado por path
    internal/        superfície x-internal-secret
    mcp/             superfície MCP (Bearer dsk_...)
  app/               UI autenticada do tenant (inbox, kanban, leads, agenda, financeiro...)
  admin/             UI de plataforma (platform admin)
  (admin)/ (public)/ route groups
  actions/           Server Actions (auth, onboarding, team, settings)
  auth/ onboarding/ get-started/ legal/ design/ team/  fluxos e páginas
lib/                 Núcleo de domínio e infraestrutura (ver seção 5)
components/          React compartilhado (shadcn/ui + composições)
hooks/               React hooks compartilhados
workers/             Workers de event_log + crons (agent-worker, voice-agent, ...)
supabase/            migrations/ (312) + baseline.sql (32.097 linhas) + config.toml
extensoes/           Extensões declarativas (catalogo.json, loja.json, pacotes/)
scripts/             CLIs de operação, QA, release, lint customizado
tests/               unit/ invariants/ e2e/ journeys/ shell/ fixtures/ helpers/
hostgator-setup-kit/ Kit de instalação/atualização de VPS
docker/ asterisk/ infra/  infraestrutura e telefonia (Asterisk/VoIP)
docs/                Doutrina, PRDs, specs, regras de negócio, runbooks
types/               tipos compartilhados
```

## 5. Módulos de domínio (`lib/`) 🟢

Núcleo de agente e IA:
- `agent-engine/` — runtime do agente (turno inbound/outbound, playbooks, guardrails, pacing, spinning, queue, flywheel, golden-candidates, cron, edge, obs, health)
- `ai/` — modelos, custo, orçamento, RAG, dispatcher, catálogo de providers
- `mcp/` — servidor MCP do CRM

Canais e comunicação:
- `channels/`, `waha/`, `messaging/`, `inbox/`, `atendimento/`, `voice/`, `voip/`, `wacalls/`, `notifications/`, `email/`

CRM e vendas:
- `leads/`, `pipelines/`, `kanban/`, `contacts/`, `tags/`, `agenda/`, `financeiro/`, `catalogo/`, `conversoes/`, `prospecting/`, `followup/`, `escalacao/`, `routing/`, `automation/`, `reports/`, `metrics/`

Plataforma e governança:
- `auth/`, `tenants/`, `team/`, `users/`, `settings/`, `onboarding/`, `instalacao/`, `branding/`, `impersonate/`, `navigation/`, `audit/`, `system/`, `operacao/`, `release/`

Compliance e dados:
- `lgpd/`, `legal/`, `opt-out/`, `retencao/`, `event-log/`, `realtime/`, `external-db/`, `nuvemshop/`, `plataformas-de-anuncio/`

Infra transversal:
- `api/` (`wrappers.ts` `ok()`/`fail()`, `errors.ts`), `supabase/` (browser/server/admin clients), `crypto/`, `net/`, `http/`, `i18n/`, `query/`, `schemas/`, `relogio/`, `tempo/`, `env.ts`, `logger.ts`

## 6. Pontos de entrada 🟢

| Entry point | Tipo |
|---|---|
| `app/layout.tsx`, `app/page.tsx` | App Router raiz |
| `proxy.ts` | Middleware de borda Next 16 (auth, `X-Request-Id`, `x-pathname`) |
| `app/api/v1/**/route.ts` | 337 route handlers REST versionados |
| `app/api/mcp/`, `app/api/internal/` | superfícies não-cookie |
| `workers/agent-worker/main.ts` | worker de agente (event_log) |
| `instrumentation.ts`, `instrumentation-client.ts` | boot de observabilidade (Sentry) |

Scripts npm relevantes: `dev`, `build`, `start`, `lint`, `typecheck`, `test:unit`, `test:db`,
`test:e2e`, `test:journeys`, `test:shell`, `gov:verify`, `worker`, `dev:crons`.

## 7. Superfícies HTTP não-cookie (cada uma com guard próprio) 🟢

| Superfície | Autenticação |
|---|---|
| `app/api/v1/cron/` (38 rotas) | Bearer `INTERNAL_CRON_SECRET`, fail-closed |
| `app/api/internal/` | header `x-internal-secret` |
| `app/api/mcp/` | Bearer `dsk_...` contra `api_tokens` |
| `app/api/v1/webhooks/` (9 rotas) | HMAC + path token |
| parte de `app/api/v1/` | cookie OU bearer via `lib/api/auth-dual.ts` |

## 8. Distribuição das rotas REST (`app/api/v1/`) 🟢

Maiores grupos: `ai` (88), `cron` (38), `admin` (29), `agenda` (25), `conversations` (21),
`leads` (18), `team`/`contacts`/`pipelines` (14 cada), `financeiro` (12), `voice`/`extensions` (11),
`webhooks` (9), `channel-sessions` (9), `system`/`channels` (8). Total v1: 337 · total app/api: 339.

## 9. CI/CD 🟢

`.github/workflows/`: `ci.yml`, `e2e.yml`, `perf.yml`, `publish-image.yml`, `release.yml`,
`relogio.yml`, `acolhida.yml`, `vigia-de-colisao.yml`.
Checks obrigatórios na `main` (conforme AGENTS.md): `verify`, `build-and-size`, `invariants`,
`e2e`, `imagens-ok`.

## 10. Banco de dados (superficial — Data Master fará o detalhe) 🟢

- `supabase/migrations/` — **312** migrations SQL versionadas + `MANIFEST.md`
- `supabase/baseline.sql` — **32.097** linhas (dump + apêndice idempotente); é o que `install.sh`/`update.sh` aplicam no self-host
- `lib/database.types.ts` — tipos **gerados** do schema (não editar à mão)
- Multi-tenant: `organization_id` + RLS em toda tabela tenant-aware; helpers `fn_user_org_ids()` / `fn_user_role_in_org()`

## 11. Testes 🟢

| Camada | Comando | Arquivos |
|---|---|---|
| Unit (Vitest, jsdom) | `pnpm test:unit` | 1.486 `*.test.ts(x)` no total do repo |
| Invariantes de banco (Docker) | `pnpm test:db` | 254 em `tests/invariants/` |
| E2E (Playwright + axe-core) | `pnpm test:e2e` | 142 specs em `tests/e2e/` |
| Kit self-host (bash) | `pnpm test:shell` | scripts em `tests/shell/` |
| Jornadas de canal | `pnpm test:journeys` | `tests/journeys/` |

## 12. Empacotamento / distribuição 🟢

Produto self-host: `Dockerfile` (+ `.worker`, `.scheduler`, `.voice-agent`), 5 arquivos
`docker-compose*.yml`, `Caddyfile`, `hostgator-setup-kit/`. Imagens publicadas pelo CI
(`publish-image.yml`); nenhum serviço de produção constrói na máquina do cliente.

## 13. Sinais para organização das specs 🟢

Dois sinais fortes coexistem → sugestão **hybrid**:
- Roteamento REST centralizado e versionado (`app/api/v1/**`, 337 route.ts)
- Pastas top-level de domínio (`lib/*`, `app/app/*`)

Detalhes estruturados em `.reversa/context/surface.json`.
