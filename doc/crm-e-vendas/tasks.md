# CRM e Vendas — Tarefas de Implementação

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Pré-requisitos
- [ ] Tabelas `crm_pipelines`, `crm_stages`, `crm_leads`, `crm_lead_activities`, `contacts`
- [ ] Triggers `fn_crm_lead_close_on_stage`, `fn_seed_default_pipeline_for_org`
- [ ] `lead_checkpoints` / `lead_state` (fonte do score)
- [ ] Índices únicos parciais (won/lost/default) e ON DELETE RESTRICT nas FKs de lead

## Tarefas

- [ ] T-01, Implementar nascimento do lead (idempotente por contato aberto)
  - Origem no legado: `lib/leads/nascimento-do-lead.ts`
  - Critério de pronto: 1 lead aberto por contato; recusa bloqueado; funil default/1ª etapa
  - Confiança: 🟢

- [ ] T-02, Implementar encerramento (won/lost via estágio terminal)
  - Origem no legado: `lib/leads/encerramento.ts`
  - Critério de pronto: `lost` exige motivo; status/closed_at do trigger; idempotente
  - Confiança: 🟢

- [ ] T-03, Implementar operações de estágio (criar/editar/arquivar) com validação pré-banco
  - Origem no legado: `lib/leads/stage-operations.ts`, `lib/leads/stage-editing.ts`
  - Critério de pronto: win/lost em 2 updates ordenados; arquivar move negócios antes
  - Confiança: 🟢

- [ ] T-04, Implementar edição de funil (arquivar/excluir/padrão)
  - Origem no legado: `lib/pipelines/pipeline-editing.ts`
  - Critério de pronto: recusa único/padrão/com webhook/com automação; troca padrão libera antes
  - Confiança: 🟢

- [ ] T-05, Implementar score por fórmula + faixa com histerese
  - Origem no legado: `lib/leads/score-formula.ts`, `lib/kanban/score-band.ts`
  - Critério de pronto: recusa sem lastro (não zero); faixa não pisca na fronteira
  - Confiança: 🟢

- [ ] T-06, Implementar radar de risco (classificação pura + montagem)
  - Origem no legado: `lib/leads/risk-radar.ts`, `lib/leads/radar-de-risco.ts`
  - Critério de pronto: janela por estágio; buckets corretos; demandas sem próximo passo
  - Confiança: 🟢

- [ ] T-07, Implementar sincronização de estágio do agente
  - Origem no legado: `lib/leads/agent-stage-sync.ts`
  - Critério de pronto: trava otimista; motivos tipados distinguem estados
  - Confiança: 🟢

- [ ] T-08, Implementar card do kanban + posicionamento fracionário
  - Origem no legado: `lib/kanban/card-state.ts`, `lib/kanban/fractional-indexing.ts`
  - Critério de pronto: precedência da faixa ③ (um estado por vez); NaN → rebalance
  - Confiança: 🟢

- [ ] T-09, Implementar classificação inicial do lead (Respondi)
  - Origem no legado: `lib/leads/classificacao-inicial.ts`
  - Critério de pronto: desqualificado/revisao_humana/classificado (A/B/C/D); D só por critério combinado
  - Confiança: 🟢

- [ ] T-10, Implementar reporte de conversão de venda
  - Origem no legado: `lib/conversoes/envio.handler.ts`
  - Critério de pronto: só com atribuição; `Purchase` com valor+moeda; dedup `<leadId>:Purchase`
  - Confiança: 🟢

## Tarefas de Teste

- [ ] TT-01, Happy path: conversa → lead no funil default (não duplica na 2ª mensagem)
- [ ] TT-02, `lost` sem motivo retorna 422
- [ ] TT-03, Score sem lastro retorna null (não zero)
- [ ] TT-04, Radar classifica lead frio por janela de estágio
- [ ] TT-05, Move do agente com trava otimista não atropela humano

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Seed do funil default por org (`fn_seed_default_pipeline_for_org`)

## Ordem Sugerida
1. T-03/T-04 (estágio/funil) antes de T-01 (nascimento depende do funil default).
2. T-05/T-06/T-08 (score/radar/card) leem `lead_checkpoints`/`lead_state`.
3. T-07 (sync do agente) e T-10 (conversão) por último.

## Lacunas Pendentes (🔴)
- SQL dos triggers de fechamento e seed de funil.
- `lib/contacts/*` (dedup, CPF cifrado) e `lib/conversoes/*` (atribuição, registro de envio).
