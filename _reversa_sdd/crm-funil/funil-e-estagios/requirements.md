# Caso de Uso: Funil e Estágios

> Sub-unit de `crm-funil`. Edição de funil/etapa e movimentação de card com integridade.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Regras puras de edição de funil e etapa (criar, renomear, marcar ganho/perda, arquivar) e os três movimentadores de card, todos com trava otimista. 🟢

## Responsabilidades
- Validar arquivamento/exclusão de funil e etapa na ordem que o usuário resolve. 🟢
- Manter marca exclusiva (default/client, ganho/perda) com índices imediatos. 🟢
- Mover cards com trava otimista e emitir eventos. 🟢

## Regras de Negócio
- Funil sem etapa de ganho não fecha (`/win` → `pipeline_no_won_stage`). 🟢
- `updatesDeMarcaExclusiva`/`updatesDeMarcacao`: libera o anterior ANTES de marcar o novo. 🟢
- Só `pending`/`confirmed` de compromisso avançam o card (`appointment-stage-move`). 🟢
- Trava otimista: 0 linhas = `conflito_humano`; negócio não-open = `lead_fechado`. 🟢
- Arquivada não se edita (furaria índice parcial). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Validar arquivamento ordenado | Must | Recusa: único ativo → padrão → fonte webhook → automação |
| RF-02 | Marca exclusiva sem colisão | Must | Libera anterior antes de marcar (sem `23505`) |
| RF-03 | Movimentar com trava otimista | Must | Conflito humano detectado |

## Critérios de Aceitação
```gherkin
Dado um funil sem etapa de ganho
Quando /win é chamado
Então responde pipeline_no_won_stage

Dado dois marcadores exclusivos
Quando updatesDeMarcaExclusiva roda
Então o anterior é liberado antes de o novo ser marcado
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/pipelines/pipeline-editing.ts` | `validarArquivamento`, `updatesDeMarcaExclusiva` | 🟢 |
| `lib/leads/stage-editing.ts` | `validarMarcacao`, `updatesDeMarcacao` | 🟢 |
| `lib/leads/stage-operations.ts` | operações com I/O em sequência | 🟢 |
| `lib/leads/agent-stage-sync.ts` | movimentador do agente | 🟢 |
