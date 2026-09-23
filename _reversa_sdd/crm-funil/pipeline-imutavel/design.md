# Caso de Uso: Pipeline Imutável — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal
1. Requisição de mudança de funil chega. 🟢
2. `clonar-para-funil.ts` cria um novo `crm_leads` no funil destino, herdando `custom_fields` inteiro. 🟢
3. Não herda `external_id`, status nem `closed_at` (identidade nova). 🟢
4. Encerra a origem com `moved_to_another_pipeline` (via `decideMotivoDaPerda`), fora das métricas (0266). 🟢

## Dependências
- `leads/encerramento`, `leads/motivo-da-perda`.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| `pipeline_id` imutável — clonar em vez de mover | `leads/clonar-para-funil.ts` | 🟢 (ADR-0007) |
| Motivo de saída canônico fora da métrica | `leads/motivo-da-perda.ts` | 🟢 |

## Riscos e Lacunas
- 🟡 Confirmar que nenhum caminho de escrita permite `update pipeline_id` direto (invariante deve ser guardado no banco/CHECK).
