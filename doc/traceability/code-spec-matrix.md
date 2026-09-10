# Code/Spec Matrix — DeskcommCRM

> Gerado pelo **Redator** (Reversa) em 2026-09-10 · Nível: Detalhado
> Por arquivo/módulo do legado, qual unit de spec o cobre. 🟢 coberto · 🟡 parcial · n/a sem unit

## Cobertura por módulo do legado

| Módulo / arquivo do legado | Unit de spec | Cobertura |
|---|---|---|
| `lib/agent-engine/*` | `ia-e-agentes/` | 🟢 |
| `lib/ai/*` (workers, RAG, handoff, elegibilidade) | `ia-e-agentes/` | 🟢 |
| `lib/mcp/*` | `ia-e-agentes/` (+ auth em `auth-e-multitenancy/`) | 🟢 |
| `workers/agent-worker/*`, `workers/ai-*`, `workers/rag-indexer` | `ia-e-agentes/` | 🟢 |
| `lib/waha/*` | `canais-e-mensageria/` | 🟢 |
| `lib/channels/*` | `canais-e-mensageria/` | 🟢 |
| `lib/atendimento/*` | `canais-e-mensageria/` | 🟢 |
| `lib/messaging/*`, `lib/inbox/*` | `canais-e-mensageria/` | 🟡 (não lidos em profundidade) |
| `lib/leads/*` | `crm-e-vendas/` | 🟢 |
| `lib/kanban/*` | `crm-e-vendas/` | 🟢 |
| `lib/pipelines/*` | `crm-e-vendas/` | 🟢 |
| `lib/conversoes/*` | `crm-e-vendas/` (+ `integracoes-externas/`) | 🟡 |
| `lib/contacts/*` | `crm-e-vendas/` | 🟡 |
| `lib/automation/*` | `automacao-e-followup/` | 🟢 |
| `lib/followup/*` | `automacao-e-followup/` | 🟢 |
| `lib/escalacao/*` | `automacao-e-followup/` (+ `canais-e-mensageria/`) | 🟡 |
| `lib/agenda/*` | `automacao-e-followup/` | 🟡 |
| `lib/auth/*` | `auth-e-multitenancy/` | 🟢 |
| `lib/impersonate/*` | `auth-e-multitenancy/` | 🟢 |
| `lib/api/*` | `auth-e-multitenancy/` | 🟢 |
| `lib/supabase/*` | `auth-e-multitenancy/` | 🟢 |
| `proxy.ts` | `auth-e-multitenancy/` | 🟢 |
| `app/api/v1/*` (166 handlers) | `auth-e-multitenancy/` (padrão) + `openapi/api-v1.yaml` | 🟡 (amostra) |
| `app/actions/*` | `auth-e-multitenancy/` | 🟡 |
| `lib/lgpd/*`, `workers/lgpd-*` | `lgpd-legal-e-auditoria/` | 🟢 |
| `lib/legal/*` | `lgpd-legal-e-auditoria/` | 🟢 |
| `lib/audit/*` | `lgpd-legal-e-auditoria/` | 🟢 |
| `lib/branding/*`, `lib/branding.ts` | `lgpd-legal-e-auditoria/` | 🟢 |
| `lib/event-log/*` | `governanca-de-eventos/` | 🟢 |
| `lib/nuvemshop/*` | `integracoes-externas/` | 🟡 |
| `lib/plataformas-de-anuncio/*` | `integracoes-externas/` | 🟢 |
| `lib/webhooks/*` | `integracoes-externas/` | 🟡 |
| `lib/notifications/*` | `integracoes-externas/` | 🟡 (push não aprofundado) |
| `lib/onboarding/*` | `onboarding-e-instalacao/` | 🟢 |
| `lib/instalacao/*` | `onboarding-e-instalacao/` | 🟡 |

## Módulos sem unit dedicada (n/a — candidatos a análise adicional)

| Módulo | Nota |
|---|---|
| `lib/metrics/*`, `lib/reports/*` | n/a — métricas/relatórios (transversal; cobertos indiretamente) |
| `lib/catalogo/*` | n/a — catálogo de produtos |
| `lib/settings/*` | n/a — configurações (transversal) |
| `lib/retencao/*`, `lib/operacao/*` | n/a — retenção/operação |
| `lib/tarefas/*`, `lib/tempo/*`, `lib/relogio/*` | n/a — tarefas/tempo |
| `lib/navigation/*`, `lib/ui/*`, `lib/i18n/*` | n/a — infra de UI |
| `lib/crypto/*`, `lib/net/*`, `lib/http/*`, `lib/query/*` | n/a — utilitários |
| `lib/realtime/*`, `lib/system/*`, `lib/release/*` | n/a — infra |
| `app/app/*` (UI de telas) | n/a — front-end (Visor/Design System cobririam) |
| `supabase/migrations/*`, `supabase/baseline.sql` | n/a — **Data Master** |
| `components/*` | n/a — componentes React |

## Cobertura estimada

- **Módulos de domínio centrais (`lib/*` de negócio):** ~9 grupos → **9 units** (🟢 no núcleo, 🟡 nos auxiliares não lidos em profundidade).
- **Superfície de API v1:** padrão documentado + amostra OpenAPI (🟡 — 166 handlers não enumerados um a um).
- **Cobertura estimada do que foi extraído:** ~80% dos módulos de negócio mapeados a alguma unit; infra/UI/utilitários ficam n/a (fora do escopo de reimplementação de regra de negócio).

## Lacunas transversais (🔴 — para Data Master / Visor / Design System)

- SQL das RPCs, triggers e policies RLS (~213 migrations) → **Data Master**.
- Telas (`app/app/*`) e tokens de design → **Visor** / **Design System**.
- Matriz rota-a-rota completa dos 166 handlers v1.
