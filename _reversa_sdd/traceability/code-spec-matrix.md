# Code/Spec Matrix — DeskcommCRM

> Gerado pelo Redator (Reversa), doc_level=detalhado. Mapeia os módulos do legado às units SDD que os cobrem.
> Cobertura: 🟢 coberto por spec de unit · 🟡 coberto parcialmente (referência/amostra) · n/a sem unit.
> Granularidade: **hybrid** (14 units de domínio + casos de uso aninhados). Fonte: `code-analysis.md` (Unidades 1-14).

## Mapa módulo → unit

| Módulo/pasta do legado | Unit correspondente | Cobertura |
|------------------------|---------------------|-----------|
| `lib/agent-engine/**` | `nucleo-ia-agente/` (+ turno-inbound, guardrails-before-send, fila-de-jobs, pacing-spinning) | 🟢 |
| `lib/ai/**` | `ia-suporte/` (+ rag-conhecimento, orcamento-e-custo) | 🟢 |
| `lib/mcp/**` | `ia-suporte/servidor-mcp/` | 🟢 |
| `lib/channels/**` | `canais-mensageria/` (+ restricao-de-canal) | 🟢 |
| `lib/waha/**` | `canais-mensageria/webhook-inbound/` | 🟢 |
| `lib/messaging/**` | `canais-mensageria/` | 🟡 (media lido por referência) |
| `lib/inbox/**` | `canais-mensageria/inbox-atendimento/` | 🟢 |
| `lib/atendimento/**` | `canais-mensageria/inbox-atendimento/` | 🟢 |
| `lib/notifications/**` | `canais-mensageria/` | 🟢 |
| `lib/email/**` | `canais-mensageria/` | 🟢 |
| `lib/voice/**`, `lib/voip/**`, `lib/wacalls/**` | `voz-telefonia/` | 🟢 |
| `workers/voice-agent/**` | `voz-telefonia/` | 🟡 (audioSocketBridge por referência) |
| `lib/leads/**` | `crm-funil/` (+ funil-e-estagios, pipeline-imutavel) | 🟢 |
| `lib/pipelines/**` | `crm-funil/funil-e-estagios/` | 🟢 |
| `lib/kanban/**` | `crm-funil/` | 🟢 |
| `lib/contacts/**` | `crm-funil/` | 🟢 |
| `lib/tags/**` | `crm-funil/` | 🟢 |
| `lib/conversoes/**` | `crm-funil/` | 🟢 |
| `lib/prospecting/**` | `crm-funil/` | 🟡 (arquivos menores por referência) |
| `lib/agenda/**` | `agenda-financeiro/` | 🟢 |
| `lib/agenda/google/**` | `agenda-financeiro/` | 🟢 |
| `lib/financeiro/**` | `agenda-financeiro/` | 🟢 |
| `lib/catalogo/**` | `agenda-financeiro/` | 🟢 |
| `lib/automation/**` | `automacao-roteamento/` | 🟢 |
| `lib/routing/**` | `automacao-roteamento/` | 🟢 |
| `lib/followup/**` | `automacao-roteamento/` | 🟢 |
| `lib/escalacao/**` | `automacao-roteamento/` | 🟢 |
| `lib/auth/**` | `auth-tenancy-rbac/` (+ borda-e-sessao, rbac-requireRole, mfa) | 🟢 |
| `lib/tenants/**` | `auth-tenancy-rbac/` | 🟢 |
| `lib/team/**` | `auth-tenancy-rbac/` | 🟢 |
| `lib/users/**` | `auth-tenancy-rbac/` | 🟢 |
| `lib/impersonate/**` | `auth-tenancy-rbac/` | 🟢 |
| `proxy.ts` | `auth-tenancy-rbac/borda-e-sessao/` + `superficie-http/` | 🟢 |
| `lib/lgpd/**` | `compliance/` | 🟢 |
| `lib/legal/**` | `compliance/` | 🟢 |
| `lib/opt-out/**` | `compliance/` | 🟢 |
| `lib/retencao/**` | `compliance/` | 🟢 |
| `lib/audit/**` | `compliance/` | 🟢 |
| `lib/settings/**` | `plataforma-operacao/` | 🟢 |
| `lib/onboarding/**` | `plataforma-operacao/onboarding/` | 🟢 |
| `lib/instalacao/**` | `plataforma-operacao/` | 🟢 |
| `lib/branding/**` | `plataforma-operacao/marca-propria/` | 🟢 |
| `lib/navigation/**` | `plataforma-operacao/` | 🟢 |
| `lib/system/**` | `plataforma-operacao/` | 🟢 |
| `lib/operacao/**` | `plataforma-operacao/` | 🟢 |
| `lib/release/**` | `plataforma-operacao/` | 🟡 (citado, não lido a fundo) |
| `lib/external-db/**` | `integracoes-externas/` | 🟢 |
| `lib/nuvemshop/**` | `integracoes-externas/` | 🟢 |
| `lib/plataformas-de-anuncio/**` | `integracoes-externas/` | 🟢 |
| `lib/extensions/**` (+ `extensoes/`) | `integracoes-externas/` | 🟢 |
| `lib/event-log/**` | `eventos-tempo-real/` | 🟢 |
| `lib/realtime/**` | `eventos-tempo-real/` | 🟢 |
| `lib/relogio/**`, `lib/tempo/**` | `eventos-tempo-real/` | 🟢 |
| `workers/agent-worker/**` | `eventos-tempo-real/` | 🟢 |
| `workers/*.handler.ts` | `eventos-tempo-real/` | 🟡 (só media-persist lido como amostra) |
| `lib/api/**` | `infra-transversal-relatorios/` | 🟢 |
| `lib/supabase/**` | `infra-transversal-relatorios/` | 🟢 |
| `lib/crypto/**`, `lib/env.ts`, `lib/logger.ts` | `infra-transversal-relatorios/` | 🟢 |
| `lib/i18n/**`, `lib/schemas/**` | `infra-transversal-relatorios/` | 🟡 (schemas não campo a campo) |
| `lib/query/**`, `lib/http/**`, `lib/net/**` | `infra-transversal-relatorios/` | 🟢 |
| `lib/reports/**`, `lib/metrics/**` | `infra-transversal-relatorios/` | 🟢 |
| `app/api/**` (route handlers) | `superficie-http/` | 🟡 (amostra por superfície; ~337 rotas não lidas uma a uma) |
| `app/actions/**` (Server Actions) | `superficie-http/` | 🟡 (2 amostras lidas) |
| `supabase/migrations/**`, `supabase/baseline.sql` | — (Data Master) | n/a |
| `lib/database.types.ts` | — (gerado) | n/a |

## Áreas de menor cobertura (candidatas a escavação adicional)
- 🔴 Toda camada de banco (RPCs security-definer, RLS, CHECKs, triggers) fica para o **Data Master** — as specs cobrem o lado que INVOCA, não o corpo das funções SQL.
- 🟡 `app/api/v1/**`: mapear cada endpoint individualmente ao aprofundar `superficie-http` (reconferir `git ls-files 'app/api/**/route.ts' | wc -l`).
- 🟡 `lib/schemas/**`: os 24 schemas de entidade precisam de conferência campo a campo ao escrever specs por entidade.
- 🟡 `workers/*.handler.ts` e `workers/voice-agent/audioSocketBridge.ts`: ler linha a linha antes de reimplementar.
- 🟡 Diretórios de `lib/ai/` lidos só por nome: `elegibilidade/`, `replies/`, `runtime/` (parcial), `skills/`, `evolution/`, `anonymize/`, `prompts/`.

## Notas de confiança
- A grande maioria das units é 🟢 (extraída diretamente do código pelo Arqueólogo e transcrita em spec pelo Redator).
- Nenhuma regra de negócio foi inventada; lacunas estão marcadas 🔴/🟡 nas specs de cada unit.
- Contagens (rotas, schemas, AUDIT_ACTIONS, NAV_CATALOG) devem ser reconferidas no disco antes de citadas como número — as specs apontam o comando em vez de fixar o valor.
