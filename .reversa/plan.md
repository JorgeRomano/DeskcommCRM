# Plano de Exploração — DeskcommCRM

> Criado pelo Reversa em 2026-09-22
> Marque cada tarefa com ✅ quando concluída.
> Você pode editar este plano antes de iniciar: adicione, remova ou reordene tarefas conforme necessário.

---

## Fase 1: Reconhecimento 🔍

- [x] **Scout** — Mapeamento de estrutura de pastas e tecnologias ✅
- [x] **Scout** — Análise de dependências e gerenciadores de pacotes ✅
- [x] **Scout** — Identificação de entry points, CI/CD e configurações ✅

## Decisão de organização das specs 🗂️

> Entre o Scout e o Arqueólogo, o Reversa pergunta como você quer organizar as specs (por módulo, caso de uso, endpoint, híbrida, por features ou customizada). A escolha fica persistida em `.reversa/config.toml` na seção `[specs]` e não será reperguntada em execuções futuras. Para reapresentar o menu, remova manualmente a seção.

## Fase 2: Escavação 🏗️

> Módulos reais agrupados por domínio pelo Scout (2026-09-22). Cada linha é uma unidade de análise do Arqueólogo.

- [x] **Arqueólogo** — Núcleo de IA: `lib/agent-engine/` (turno inbound/outbound, playbooks, guardrails, pacing, spinning, queue, flywheel) ✅
- [x] **Arqueólogo** — IA de suporte: `lib/ai/` (modelos, custo, orçamento, RAG, dispatcher) + `lib/mcp/` ✅
- [x] **Arqueólogo** — Canais e mensageria: `lib/channels/`, `lib/waha/`, `lib/messaging/`, `lib/inbox/`, `lib/atendimento/`, `lib/notifications/`, `lib/email/` ✅
- [x] **Arqueólogo** — Voz/telefonia: `lib/voice/`, `lib/voip/`, `lib/wacalls/` ✅
- [x] **Arqueólogo** — CRM e funil: `lib/leads/`, `lib/pipelines/`, `lib/kanban/`, `lib/contacts/`, `lib/tags/`, `lib/conversoes/`, `lib/prospecting/` ✅
- [x] **Arqueólogo** — Agenda e financeiro: `lib/agenda/`, `lib/financeiro/`, `lib/catalogo/` ✅
- [x] **Arqueólogo** — Automação e roteamento: `lib/automation/`, `lib/routing/`, `lib/followup/`, `lib/escalacao/` ✅
- [x] **Arqueólogo** — Auth, tenancy e RBAC: `lib/auth/`, `lib/tenants/`, `lib/team/`, `lib/users/`, `lib/impersonate/`, `proxy.ts` ✅
- [x] **Arqueólogo** — Compliance: `lib/lgpd/`, `lib/legal/`, `lib/opt-out/`, `lib/retencao/`, `lib/audit/` ✅
- [x] **Arqueólogo** — Plataforma e operação: `lib/settings/`, `lib/onboarding/`, `lib/instalacao/`, `lib/branding/`, `lib/navigation/`, `lib/system/`, `lib/operacao/` ✅, `lib/release/`
- [x] **Arqueólogo** — Integrações externas: `lib/external-db/`, `lib/nuvemshop/`, `lib/plataformas-de-anuncio/`, `lib/extensions/` (+ `extensoes/`) ✅
- [x] **Arqueólogo** — Eventos e tempo real: `lib/event-log/`, `lib/realtime/`, `lib/relogio/`, `lib/tempo/` + `workers/` ✅
- [x] **Arqueólogo** — Infra transversal e relatórios: `lib/api/`, `lib/supabase/`, `lib/reports/`, `lib/metrics/`, `lib/query/`, `lib/schemas/`, `lib/http/`, `lib/net/`, `lib/crypto/`, `lib/i18n/` ✅
- [x] **Arqueólogo** — Superfície HTTP: `app/api/v1/**` (roteamento, guards, wrappers) e Server Actions `app/actions/` ✅

## Fase 3: Interpretação 🧠

- [x] **Detetive** — Arqueologia Git e ADRs retroativos ✅
- [x] **Detetive** — Regras de negócio implícitas e máquinas de estado ✅
- [x] **Detetive** — Matriz de permissões (RBAC/ACL) ✅
- [x] **Arquiteto** — Diagramas C4 (Contexto, Containers, Componentes) ✅
- [x] **Arquiteto** — ERD completo e integrações externas ✅
- [x] **Arquiteto** — Spec Impact Matrix ✅

## Fase 4: Geração 📝

- [x] **Redator** — Specs SDD por componente (14 units híbridas: nucleo-ia-agente, ia-suporte, canais-mensageria, voz-telefonia, crm-funil, agenda-financeiro, automacao-roteamento, auth-tenancy-rbac, compliance, plataforma-operacao, integracoes-externas, eventos-tempo-real, infra-transversal-relatorios, superficie-http) ✅
- [x] **Redator** — OpenAPI (`openapi/deskcomm-api.yaml`) ✅
- [x] **Redator** — User Stories (`user-stories/`: turno-do-agente, handoff-e-retomada, lgpd-direitos-do-titular, onboarding-e-marca) ✅
- [x] **Redator** — Code/Spec Matrix (`traceability/code-spec-matrix.md`) ✅

## Fase 5: Revisão ✅

- [ ] **Revisor** — Revisão cruzada de specs
- [ ] **Revisor** — Resolução de lacunas com o usuário
- [ ] **Revisor** — Relatório de confiança final

---

## Agentes Independentes

> Execute estes agentes quando os recursos estiverem disponíveis — podem rodar em qualquer fase.

- [ ] **Visor** — Análise de interface via screenshots
- [ ] **Data Master** — Análise completa do banco de dados
- [ ] **Design System** — Extração de tokens de design
- [ ] **Tracer** — Análise dinâmica (requer sistema acessível)

---

## Próximo passo

Após o Time de Descoberta concluir e o `_reversa_sdd/` estar populado, você pode disparar um dos fluxos seguintes:

- `/reversa-migrate`: orquestrador do **Time de Migração** (Paradigm Advisor → Curator → Strategist → Designer → Screen Translator → Inspector). Gera as specs do sistema novo. Saída em `_reversa_sdd/migration/` e `_reversa_sdd/screens/`.
- `/reversa-reconstructor`: gera plano bottom-up para reimplementar o software a partir das specs do legado (uma tarefa por sessão).
