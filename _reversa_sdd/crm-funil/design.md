# CRM e Funil — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface (destaques)
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `calculaScore` | `(sinais, {checkpointId})` | `{score\|null, reason, semSinal?}` |
| `classifyRisk` | `(entrada)` | `RiskBucket` (`critico\|em_risco\|em_voo\|em_dia`) |
| `resolveBand` | `(score, faixaAnterior)` | faixa com histerese |
| `resolveActiveLeadForContact` | `(...)` | rota \| `no_open_lead` \| `ambiguous_open_leads` |
| `resolveCardState` | `(CardInput)` | estado do card (precedência estrita) |
| `resolveLeadOwner` | `(...)` | humano \| agente \| ninguém |

## Fluxo Principal — Scoring (fórmula, não modelo)
Pesos: `BASE=30`, `+12/compromisso` (teto 3), `−8/objeção` (teto 3), `+5/BANT` (teto 4), risco por bucket (`em_dia +10 / em_voo 0 / em_risco −10 / critico −20`). `MINIMO_DE_SINAIS=2`; recência não conta; `score = clamp(0,100, soma+BASE)`. `reason` derivado do cálculo (número sem porquê é impossível). Negócio fechado → `null`. 🟢

## Fluxo Principal — Máquina do funil
- Edição de funil/etapa: `validarArquivamento` recusa na ordem que o usuário resolve; `updatesDeMarcaExclusiva` libera o anterior ANTES de marcar o novo (índices imediatos). 🟢
- Operações com I/O em sequência (nunca paralelo): validar → liberar antiga → marcar nova → mover negócios → arquivar. 🟢
- Três movimentadores de card com trava otimista: `agent-stage-sync`, `handoff-stage-move`, `appointment-stage-move` (só `pending`/`confirmed` avançam). 🟢

## Fluxo Principal — Ciclo de vida
Nascimento (idempotente, advisory lock, RPC `fn_nascer_lead_da_conversa`) → funil → score/risco (workers) → encerramento (trigger `fn_crm_lead_close_on_stage`) → reativação (proposta ao esfriar, vence sozinha) → clone (imutabilidade de `pipeline_id`). 🟢

## Fluxos Alternativos
- Vários leads abertos → `ambiguous_open_leads` (não escolhe). 🟢
- Reativação vencida não mexe no bucket (inação foi do time, não silêncio do cliente). 🟢
- Conversão: escuta `lead.won` e `lead.stage_changed`; re-lê `crm_leads` (payload é dica). 🟢

## Dependências
- `leads` (scoring) → `agenda/protecao-followup`, `kanban/score-band`, `contacts/rotulo-do-contato`. 🟢
- `leads` (lifecycle) → `api`, `audit`, `i18n`, `atendimento`, `agent-engine/agent/lead-state`. 🟢
- `conversoes` → `event-log/dispatcher`, `plataformas-de-anuncio`. 🟢
- `prospecting` → `agent-engine/pacing`, `ai/elegibilidade`, `messages/_handler`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Score é fórmula (reason derivado do cálculo) | `leads/score-formula.ts` | 🟢 |
| Janela de risco por estágio, não global | `leads/risk-radar.ts` | 🟢 |
| `pipeline_id` imutável (clone cross-pipeline) | `leads/clonar-para-funil.ts` | 🟢 (ADR-0007) |
| Conversão como handler de evento (não chamada em encerramento) | `conversoes/envio.handler.ts` | 🟢 |
| Prospecção com ritmo derivado dos knobs da casa | `prospecting/ritmo-da-esteira-fria.ts` | 🟢 |

## Estado Interno
- `crm_leads`, `crm_stages`, `crm_pipelines`, `crm_lead_scores`, `crm_lead_risk_states`, `crm_lead_reactivations`, `crm_lead_activities`, `lead_checkpoints`, `lead_state`, `demandas`, `contacts`, `prospecting_*`, `conversion_ledger`. 🟢

## Observabilidade
- Timeline (`crm_lead_activities`, vocabulário fechado ~45 tipos); `activity-write-failure` (decisão falha alto, rastro falha baixo mas conta). 🟢

## Riscos e Lacunas
- 🔴 Cifra de CPF at-rest (`encrypt_cpf` RPC) não provisionada — hoje só `cpf_hash`.
- 🟡 `maxScoreConhecido=100` (Respondi) é INFERIDO de 2 amostras.
- 🟡 Constraints/triggers (`fn_nascer_lead_da_conversa`, `fn_crm_lead_close_on_stage`, `fn_update_last_activity_at`) vivem no baseline — validação é do Data Master.
