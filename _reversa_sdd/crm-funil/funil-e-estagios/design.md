# Caso de Uso: Funil e Estágios — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal
1. Edição pura valida (mutex ganho/perda, desmarcar sem substituta é proibido). 🟢
2. `stage-operations.ts` executa em sequência: validar → liberar antiga → marcar nova → mover negócios → arquivar. 🟢
3. Movimentadores usam trava otimista (`.eq("stage_id", origem).select("id")`); emitem `stage_changed` + `emit_event lead.stage_changed entity_kind='crm_lead'`. 🟢

## Detalhes
- `ETAPAS_INICIAIS` = Novo/Em andamento/Ganho/Perdido. `SLUG_MIN=2`, `SLUG_MAX=40`; slug não muda ao renomear. 🟢
- `agent-mapping.ts`: `PASSOS_QUE_PRECISAM_DE_ETAPA = [new, contacted, qualifying, qualified, negotiating]` (won/lost têm coluna própria). `diffParaUpdates` faz UNSETs primeiro. 🟢
- `conflitoDoBanco` traduz `23505`/`23514` em pt-BR (409). 🟢

## Dependências
- `leads/stage-editing`, `kanban/fractional-indexing`, `leads/motivo-da-perda` (`decideMotivoDaPerda`).

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Recusa na ordem que o usuário resolve | `pipelines/pipeline-editing.ts` | 🟢 |
| Libera antes de marcar (índices imediatos) | `pipeline-editing.ts`, `stage-editing.ts` | 🟢 |
| I/O em sequência, nunca paralelo | `stage-operations.ts` | 🟢 |
| Só `pending`/`confirmed` avançam o card | `appointment-stage-move.ts` | 🟢 |

## Riscos e Lacunas
- 🟡 `regrasQueApontamPara` é defensivo porque `actions` (jsonb) não tem schema.
