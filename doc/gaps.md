# Lacunas — DeskcommCRM

> Gerado pelo **Revisor** (Reversa) em 2026-09-10 · Nível: Detalhado (categorizado por severidade)
> Lacunas que permaneceram após a revisão. As que dependem de decisão sua estão em `questions.md`.

## Crítico (bloqueia reimplementação fiel)

| Lacuna | Onde | Endereçamento |
|---|---|---|
| SQL das RPCs (`fn_user_role_in_org`, `fn_is_platform_admin`, `emit_event`, `fn_claim_due_followup_enrollments`, `fn_lgpd_cascade_redact_contact`, `fn_crm_lead_close_on_stage`, `fn_seed_default_pipeline_for_org`, etc.) | transversal (auth, event-log, followup, lgpd, crm) | **Data Master** (a partir de `supabase/migrations/`) |
| Policies RLS por tabela | multi-tenancy | **Data Master** |
| Triggers de banco (fechamento, emissão de eventos, touch) | crm, event-log | **Data Master** |
| Modelo de dados completo (constraints, índices parciais, FKs, ~213 migrations) | ERD | **Data Master** |

## Moderado (comportamento parcialmente inferido)

| Lacuna | Onde | Nota |
|---|---|---|
| ~~Dias exatos do SLA LGPD (7/15)~~ | lgpd | ✅ RESOLVIDO (usuário): D+7 / D+15, sem exceção |
| Regras AT-02/AT-05/AT-08 (claim, notas, idle) | canais/atendimento | 🟡 seguem não confirmadas (AT-04 ✅ corrigido: supervisor lê E responde) |
| ~~Retenção de audit (5 anos/piso 90d)~~ | lgpd/audit | ✅ RESOLVIDO (usuário): 5 anos, sem cold/S3 |
| Laço de tools e fail-safes de `inbound-turn.ts` (>600 linhas) | ia-e-agentes | ler linha a linha antes de reimplementar |
| `WahaChannelAdapter` (envio concreto) | canais/ia | não lido em detalhe |
| `lib/messaging/*`, `lib/inbox/*` (mídia, comandos) | canais | não lidos |
| `lib/contacts/*` (dedup, CPF cifrado), `lib/conversoes/*` (atribuição) | crm | não lidos em profundidade |
| Ações da automação (`actions/*`) além de create-or-move-lead | automação | não lidas |
| `lib/followup/{silence-sweep,intervencao,timing-plan}` | followup | não lidos |
| `lib/agenda/{google,horarios-livres}` | agenda | integração Google não lida |
| `lib/notifications/*` (push VAPID) | integrações | 🔜 usuário quer unit própria (próxima extração) |

## Cosmético (não bloqueia)

| Lacuna | Onde | Nota |
|---|---|---|
| Módulos metrics/reports/catalogo/settings/retencao/operacao/tarefas | traceability | 🔜 usuário quer documentar (próxima extração) |
| OpenAPI completo dos 166 handlers | api | ✅ usuário: amostra + padrão é suficiente |
| Kit `hostgator-setup-kit/` | onboarding/deployment | instalação assistida não analisada |
| Telas (`app/app/*`) e tokens de design | UI | Visor / Design System (agentes independentes) |
| Provisão do Supabase (gerenciado vs self-hosted) | deployment | não confirmado no compose |
| Estratégia de backup do Postgres/Storage | deployment | não confirmada |

## Agentes independentes recomendados (para reduzir lacunas críticas)

- **Data Master** — resolve todas as lacunas críticas de banco (RPCs, RLS, triggers, ERD definitivo, ~213 migrations).
- **Visor** — telas via screenshots (se houver evidência visual).
- **Design System** — tokens de design a partir de `docs/design-system/` / CSS.
