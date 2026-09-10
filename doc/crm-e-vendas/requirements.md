# CRM e Vendas

> Spec SDD (híbrida — módulo de topo) · Redator (Reversa) · Nível: Detalhado
> Módulos: `lib/leads/*`, `lib/contacts/*`, `lib/pipelines/*`, `lib/kanban/*`, `lib/conversoes/*`
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Visão Geral

O núcleo de vendas: nascimento do lead a partir da conversa, ciclo de vida no funil (estágios, ganho/perda), score de fechamento por fórmula, radar de risco (anti-morte visível), kanban com posicionamento fracionário, e reporte de conversão às plataformas de anúncio. Um lead por DEMANDA, não por mensagem. 🟢

## Responsabilidades

- Criar o lead determinísticamente no ingest (funil default, primeira etapa). 🟢
- Gerir estágios/funis (criar, editar, arquivar, marcar ganho/perda). 🟢
- Encerrar demanda (ganhar/perder) via estágio terminal. 🟢
- Calcular score/probabilidade por fórmula com lastro citável. 🟢
- Classificar risco de esfriamento por janela de estágio (radar). 🟢
- Sincronizar o funil do agente com o funil nomeado do tenant. 🟢
- Reportar venda como conversão offline à plataforma de origem. 🟢

## Regras de Negócio

- Um lead ABERTO por contato; nova mensagem alimenta o existente (`ja_existe`). 🟢
- Lead vive em UM pipeline; mover entre pipelines não é suportado (clonar). 🟢 (P-01)
- `status` won/lost derivado de estágio com flag (trigger); nunca escrito à mão. 🟢 (P-02)
- `lost_reason` obrigatório na perda. 🟢 (P-03)
- Reorder por fractional indexing (`midpoint`, STEP=1000; NaN → rebalance). 🟢 (P-05)
- Score é FÓRMULA, não LLM; recusa sem checkpoint/lastro citável; faixa com histerese. 🟢
- Radar: janela de esfriamento vem do estágio (`expected_duration_hours`), não global. 🟢
- Agente move o card via `crm_stages.agent_stage_hint`; trava otimista impede atropelar humano. 🟢
- Conversão só é reportada com atribuição de anúncio; `Purchase` exige valor+moeda. 🟢
- Arquivar etapa/funil recusa quando quebraria entrada de lead (webhook/automação/negócios). 🟢

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Criar lead da conversa (idempotente por contato aberto) | Must | Dado inbound de contato sem lead aberto, cria no funil default; 2ª mensagem não cria outro |
| RF-02 | Encerrar demanda (won/lost) via estágio terminal | Must | Dado "ganhar", move para estágio `is_won`; `lost` exige motivo |
| RF-03 | Editar/arquivar estágio com validação antes do banco | Must | Dado arquivar etapa com negócios, exige destino; win/lost em 2 updates ordenados |
| RF-04 | Calcular score com lastro citável | Should | Dado lead com <2 sinais ou sem checkpoint, retorna `score=null` com motivo |
| RF-05 | Montar radar de risco por janela de estágio | Should | Dado lead frio sem follow-up, aparece como `em_risco`/`critico` |
| RF-06 | Sincronizar estágio do agente com o funil do tenant | Should | Dado passo do agente com `agent_stage_hint`, move o card; sem mapa não move |
| RF-07 | Reportar conversão de venda com atribuição | Could | Dado lead `won` com `ctwa_clid`, envia `Purchase` à Meta |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Consistência | Trava otimista no move do agente (`.eq(stage_id).select()`) | `lib/leads/agent-stage-sync.ts` | 🟢 |
| Consistência | Índices únicos parciais (won/lost/default) exigem updates em sequência | `lib/leads/stage-operations.ts` | 🟢 |
| Performance | Radar varre até `SCAN_CAP=500` leads abertos mais frios | `lib/leads/radar-de-risco.ts` | 🟢 |
| Integridade | ON DELETE RESTRICT em `crm_leads.stage_id/pipeline_id` (arquivar, não apagar) | `stage-operations.ts`, `pipeline-editing.ts` | 🟢 |
| Auditabilidade | Score com `reason` derivado de cada parcela (proíbe número sem porquê) | `lib/leads/score-formula.ts` | 🟢 |

## Critérios de Aceitação

```gherkin
Dado um contato sem lead aberto que envia a primeira mensagem
Quando garantirLeadDaConversa roda
Então nasce um lead no funil is_default, na primeira etapa não-terminal

Dado um lead sendo perdido sem lost_reason
Quando encerraDemanda é chamado
Então retorna 422 (motivo da perda obrigatório)

Dado um lead com 1 único sinal e sem checkpoint
Quando calculaScore roda
Então score=null com semSinal (sem_conteudo/sem_lastro_citavel), nunca zero inventado
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Nascimento + encerramento do lead | Must | Ciclo de vida central do CRM |
| Operações de estágio/funil | Must | Base do kanban e do roteamento |
| Score + radar | Should | Priorização e anti-morte; degrada sem derrubar |
| Sync do agente | Should | Card acompanha o agente; sem mapa não move |
| Conversão de venda | Could | Otimização de anúncio; só com atribuição |

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/leads/nascimento-do-lead.ts` | `garantirLeadDaConversa` | 🟢 |
| `lib/leads/encerramento.ts` | `encerraDemanda` | 🟢 |
| `lib/leads/stage-operations.ts` | criar/atualizar/arquivar etapa | 🟢 |
| `lib/leads/agent-stage-sync.ts` | `sincronizaEstagioDoAgente` | 🟢 |
| `lib/leads/score-formula.ts` | `calculaScore` | 🟢 |
| `lib/leads/risk-radar.ts` / `radar-de-risco.ts` | `classifyRisk`, `carregaRadarDeRisco` | 🟢 |
| `lib/leads/classificacao-inicial.ts` | `classificarLeadInicial` | 🟢 |
| `lib/kanban/card-state.ts` / `score-band.ts` / `fractional-indexing.ts` | `resolveCardState`, `resolveBand`, `midpoint` | 🟢 |
| `lib/pipelines/pipeline-editing.ts` | validação de funil | 🟢 |
| `lib/conversoes/envio.handler.ts` | `conversaoDeVendaHandler` | 🟢 |
| `lib/contacts/*` | dedup, rótulo, CSV | 🟡 |
