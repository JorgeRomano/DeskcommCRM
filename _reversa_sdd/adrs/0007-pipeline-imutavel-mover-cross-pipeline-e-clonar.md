# ADR-0007 — Pipeline de um lead é imutável; mover entre funis é clonar

> ADR **retroativo** (Detetive/Reversa). Confiança: 🟢 CONFIRMADO.
> Evidência: `data-dictionary.md` (Unidade 5), regra P-01, commit `951fdcdc4` (prioridade do nome do contato).

## Status
Aceito (vigente).

## Contexto
Um lead vive dentro de um funil (pipeline) com etapas próprias, vocabulário próprio e métricas
próprias. Deixar um lead "mudar de funil" mantendo a mesma linha destrói a integridade das métricas
do funil de origem (a conversão contaria em dois lugares) e mistura históricos de etapas que não
existem no destino.

## Decisão
**`crm_leads.pipeline_id` é imutável (P-01).** FK `ON DELETE RESTRICT` em `pipeline_id` e `stage_id`.
Mover um lead para outro funil é **clonar**: nasce um novo lead no destino (herda `custom_fields`
inteiro; `external_id` **não** é herdado, pois é único por origem). O lead original é fechado com
`lost_reason = 'moved_to_another_pipeline'`, motivo que é **excluído das métricas** e **nunca
ofertado** na UI. `source_metadata` registra `clonado_de` / `movido_para`. A tool MCP
`crm_move_lead_stage` só move **dentro do mesmo pipeline**.

## Alternativas consideradas
1. **`pipeline_id` mutável** — rejeitado: corrompe métricas de conversão do funil de origem; etapas
   incompatíveis entre funis.
2. **Mover sem registrar rastro** — rejeitado: perde a linhagem (quem veio de onde).
3. **Clonar + fechar com motivo excluído das métricas (escolhida).**

## Consequências
- **Positivas:** métricas de funil íntegras; linhagem preservada (`clonado_de`/`movido_para`);
  posição no funil por indexação fracionária (`STEP=1000`) sem reordenação em massa.
- **Negativas / custo:** duplicação de linha (dois leads para uma jornada); o `external_id` não
  herdado exige atenção de quem integra sistemas externos; consumidores de relatório precisam saber
  filtrar `moved_to_another_pipeline`.
- **Regra vizinha:** lead `lost` exige `lost_reason` do vocabulário canônico ∪ `settings.lost_reasons`.
