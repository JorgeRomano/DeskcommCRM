# Spec Impact Matrix — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) em 2026-09-10 · 🟢 CONFIRMADO / 🟡 INFERIDO
>
> Qual componente impacta qual. Use para prever o raio de uma mudança antes de tocar o código.
> Ler: linha **A** → coluna marcada = "mudar A afeta B".

## 1. Matriz de dependência entre módulos

| ↓ impacta →       | event-log | auth | api | supabase | waha | channels | leads | kanban | followup | automation | ai | agent-engine | lgpd | branding | conversoes |
|-------------------|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| **event-log**     | —  |    |    | ●  |    |    |    |    | ●  | ●  | ● | ●  | ●  | ●  | ●  |
| **auth**          |    | —  | ●  | ●  |    |    |    |    |    |    |    | ●  |    |    |    |
| **api**           |    |    | —  |    |    |    |    |    |    |    |    |    |    |    |    |
| **supabase**      | ●  | ●  |    | —  | ●  | ●  | ●  | ●  | ●  | ●  | ● | ●  | ●  | ●  | ●  |
| **waha**          |    |    |    | ●  | —  | ●  | ●  |    |    |    | ● | ●  |    |    |    |
| **channels**      |    |    |    |    | ●  | —  | ●  |    | ●  |    | ● | ●  |    |    |    |
| **leads**         | ●  |    |    | ●  |    |    | —  | ●  | ●  | ●  |    | ●  | ●  |    | ●  |
| **kanban**        |    |    |    |    |    |    | ●  | —  |    |    |    |    |    |    |    |
| **followup**      | ●  |    |    | ●  |    |    | ●  |    | —  |    | ● | ●  |    |    |    |
| **automation**    | ●  |    |    | ●  |    |    | ●  |    | ●  | —  |    |    |    |    |    |
| **ai**            | ●  |    |    | ●  | ●  | ●  | ●  |    | ●  |    | — | ●  |    |    |    |
| **agent-engine**  | ●  |    |    | ●  | ●  | ●  | ●  |    | ●  |    | ● | —  |    |    |    |
| **lgpd**          | ●  |    |    | ●  |    |    | ●  |    |    |    |    |    | —  | ●  |    |
| **branding**      |    |    |    | ●  |    |    |    |    |    |    |    |    | ●  | —  |    |
| **conversoes**    | ●  |    |    | ●  |    |    | ●  |    |    |    |    |    |    |    | —  |

● = acoplamento direto observado (import/consumo de evento/tabela).

## 2. Componentes de alto impacto (mexer com cuidado) 🟢

| Componente | Por que é crítico | Blast radius |
|---|---|---|
| `lib/event-log` | barramento de tudo | automação, follow-up, IA, LGPD, conversões, mídia, branding |
| `lib/supabase/admin` | bypassa RLS | 89 handlers + todos os workers |
| `lib/auth/require-role` | gate de toda rota | 169 handlers |
| `lib/agent-engine/guardrails/before-send` | único caminho de saída do agente | toda mensagem do agente ao cliente |
| `lib/leads/*` (nascimento, encerramento, stage) | núcleo do CRM | kanban, follow-up, conversões, radar, agente |
| `crm_leads` / `crm_stages` (schema) | tabelas centrais | leads, kanban, follow-up, automação, conversões |
| `lib/api/errors` | catálogo de códigos | todo consumidor da API |
| Triggers (`fn_crm_lead_close_on_stage`, `emit_event`) | escrevem status/eventos | todo o fluxo pós-mutação |

## 3. Impacto por tipo de mudança 🟢

| Mudança | Verificar / rodar | Artefato de gate |
|---|---|---|
| Schema (tabela/coluna/RLS) | migration + baseline (idempotente) + MANIFEST; `pnpm test:db` | job `invariants` |
| Regra de RBAC / auth | `requireRole`, RLS, `tests/invariants` | job `invariants` |
| Guardrail de saída | bumpar `BEFORE_SEND_CHAIN_VERSION` + `before-send-chain-shape.test.ts` | `verify` |
| Número de pacing/spinning | só em `pacing/defaults.ts` (lint proíbe fora) | `lint-pacing` |
| Máquina de follow-up | `node-handlers`, `engine`, `turn-bridge`; `followup-engine.test.ts` | `invariants` |
| UI / fluxo de usuário | `pnpm test:e2e` com evidência visual | `e2e` (obrigatório) |
| Artefato do self-host (Docker/compose/kit) | `pnpm test:shell` + `publish-image` | `imagens-ok` |
| Marca (branding) | `branding.test.ts` (allowlist só encolhe) | `verify` |
| Conversão de anúncio | `conversoes/*`, transporte Meta; não pode bloquear venda | — |

## 4. Cadeias de propagação notáveis 🟢

- **Mudança de etapa de lead** → trigger emite `lead.stage_changed` → automação + follow-up (gatilho-etapa) + conversão (venda) reagem. Emitir com `entity_kind` errado = evento para ninguém.
- **Mensagem inbound** → ingestão → `pos-entrada` (opt-out → lead → despacho) → `message.received` → workers de IA + sentimento + reatividade de follow-up.
- **Handoff** → `conversations` + activity + move card + `emit_event` + realtime + inbox item; reatividade de follow-up pausa enrollments do contato.
- **Redação LGPD** → enfileira mídia ANTES → RPC cascata → muta 5 tabelas → evento → storage cleanup.
