# Arquitetura — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) em 2026-09-10 · Nível: **Detalhado**
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA
>
> Síntese arquitetural. Diagramas C4 detalhados em `c4-context.md`, `c4-containers.md`,
> `c4-components.md`; ERD em `erd-complete.md`; impacto em `traceability/spec-impact-matrix.md`;
> infraestrutura em `deployment.md`.

## 1. Visão geral 🟢

DeskcommCRM é um **monólito Next.js 16 (App Router) + worker 24/7**, distribuído como imagens Docker para self-host em VPS. Processa mensagens de WhatsApp (via WAHA) para operar um CRM de vendas multi-tenant com agentes de IA nativos, isolado por RLS no Postgres (Supabase) desde o dia 1, LGPD by-design.

**Dois planos de execução:**
- **Plano web** — Next.js (rotas API, Server Actions, UI). Cliente `supabase-js`. Identidade do JWT, org de fonte confiável.
- **Plano worker** — `workers/agent-worker/main.ts`. Cliente `pg.Pool` puro. Fila durável `job_queue` + loops (drain, cron, watchdog, health, flywheel).

**Barramento:** `event_log` (Postgres) desacopla ingestão de reações (IA, automação, follow-up, LGPD, conversões). Triggers emitem eventos; nunca fazem HTTP.

## 2. Estilo arquitetural 🟢

- **Monólito modular** com fronteiras de domínio em `lib/<dominio>/` e API versionada por recurso (`app/api/v1/<recurso>/route.ts`).
- **Event-driven** internamente (`event_log` + handlers idempotentes).
- **Regra pura + adapter de I/O**: lógica de negócio testável sem banco (`classifyRisk`, `processNode`, `decidePacing`, `calculaScore`, gates), com adapters `supabase-js` (web) e `pg` (worker).
- **Guardrails determinísticos** entre o modelo e o canal (cadeia `before-send`).
- **Fail-closed em segurança, fail-open em telemetria**.

## 3. Containers principais 🟢

| Container | Tecnologia | Responsabilidade |
|---|---|---|
| `app` | Next.js 16 standalone (Node ≥22) | UI + API REST + Server Actions + middleware de borda |
| `worker` | tsx + pg.Pool | Agent-engine: fila, turnos, drain, cron, watchdog, flywheel |
| `scheduler` | crond + curl | Dispara crons (1 min) contra a API interna |
| `waha` | WAHA NOWEB | Gateway WhatsApp (sessões, envio, webhooks) |
| `redis` + `srh` | Redis 7 + serverless-redis-http | Rate limit, debounce (REST Upstash-compatível) |
| `caddy` | Caddy 2 | HTTPS (Let's Encrypt), único que publica portas |
| Supabase | Postgres + Auth + Realtime + Storage | Dados (RLS), autenticação, realtime, mídia |

## 4. Componentes de domínio 🟢

Agrupados por área (detalhe em `code-analysis.md` e `c4-components.md`):
- **IA/agentes:** `lib/agent-engine/*` (turnos, guardrails, pacing, spinning, flywheel), `lib/ai/*` (workers legados, RAG, handoff), `lib/mcp/*`.
- **Canais:** `lib/waha/*`, `lib/channels/*`, `lib/atendimento/*`.
- **CRM:** `lib/leads/*`, `lib/contacts/*`, `lib/pipelines/*`, `lib/kanban/*`, `lib/conversoes/*`.
- **Automação/follow-up:** `lib/automation/*`, `lib/followup/*`, `lib/escalacao/*`, `lib/agenda/*`.
- **Governança:** `lib/auth/*`, `lib/lgpd/*`, `lib/legal/*`, `lib/audit/*`, `lib/event-log/*`, `lib/branding/*`.
- **Integrações:** `lib/nuvemshop/*`, `lib/plataformas-de-anuncio/*`, `lib/webhooks/*`, `lib/notifications/*`.

## 5. Integrações externas 🟢

| Integração | Direção | Protocolo | Notas |
|---|---|---|---|
| WAHA (WhatsApp) | bidirecional | REST + webhook HMAC-SHA512 | engine NOWEB; `message.any` (não `message`) |
| Supabase | bidirecional | Postgres/REST/Realtime | RLS; service role bypassa |
| Upstash Redis | saída | REST (via srh) | rate limit/debounce; fallback em memória |
| Provedores de IA | saída | HTTP (Vercel AI SDK) | Anthropic/OpenAI/Google; resolvido por painel de provedores |
| Resend | saída | REST | e-mail transacional (LGPD, convites) — com marca da org |
| Nuvemshop | entrada | OAuth + webhook HMAC-SHA256 | e-commerce; tokens não expiram |
| Meta Ads | saída | Conversions API (Graph) | conversão offline `Purchase` (ctwa_clid) |
| Google Ads | — | — | declarado `null` (sem extrator de gclid) |
| Sentry | saída | HTTP | erros; `beforeSend` higieniza PII |
| web-push | saída | Web Push VAPID | notificações |

## 6. Dívidas técnicas identificadas 🟡

1. **Dois runtimes de IA** (agent-engine + workers legados): correções precisam ser feitas nos dois lados (vários `fix(ia)` corrigem "a cópia que ficou para trás"). 🟢
2. **89/169 handlers usam service role sem gate automático** para o filtro de `organization_id` — responsabilidade do autor. 🟢 (AGENTS.md)
3. **`Idempotency-Key` implementado em 1 rota** apesar do contrato prometer nos POSTs de criação. 🟢 (AGENTS.md)
4. **Rate limit** cobre login/signup/reset/convite + webhook de captação + dispatcher; **crons e MCP seguem sem**; fallback em memória sem Upstash. 🟢 (AGENTS.md)
5. **`dispatcher` legado** (`lib/ai/dispatcher`) `@deprecated` mas presente. 🟢
6. **Invariante 7 (todo laço se fecha) parcial** — só o flywheel fecha o laço; demais classes de decisão ainda não declaram retorno. 🟢 (doutrina)
7. **`vps-fresh-onboarding` E2E fora do CI** — a jornada de instalação fresca (o produto que se vende) não tem gate. 🟢 (AGENTS.md)
8. **`lib/database.types.ts`** (6.1k linhas) gerado — não editar à mão.

## 7. Qualidade e testes 🟢

- **~1036** arquivos de teste; **~101** specs E2E (Playwright); invariantes de banco (RLS/RBAC/governança) em `tests/invariants/`.
- Gates obrigatórios na `main`: `verify` (typecheck+lint+unit), `build-and-size`, `invariants` (test:db), `e2e`, `imagens-ok`.
- `gov:verify` não cobre `test:db` nem `test:e2e` — precisam ser rodados à parte.
