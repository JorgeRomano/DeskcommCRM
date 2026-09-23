# Caso de Uso: Pipeline Imutável (clone cross-pipeline)

> Sub-unit de `crm-funil`. `pipeline_id` de um negócio nunca muda; mover entre funis é clonar + encerrar origem.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Garante a invariante P-01: o `pipeline_id` de um `crm_leads` é imutável. Trocar de funil não edita o campo, cria um clone no funil destino e encerra a origem. 🟢

## Responsabilidades
- Clonar o negócio no funil destino herdando o que faz sentido. 🟢
- Encerrar a origem com motivo canônico fora da métrica. 🟢

## Regras de Negócio
- `pipeline_id` imutável (P-01). 🟢
- Clone herda `custom_fields` inteiro; NÃO herda `external_id`/status/closed_at. 🟢
- Origem encerra com `moved_to_another_pipeline` (canônico, excluído das métricas, nunca ofertado na tela). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | `pipeline_id` imutável | Must | Update direto de `pipeline_id` não é permitido |
| RF-02 | Clone herda custom_fields, não herda identidade | Must | Clone tem novos `external_id`/status/closed_at |
| RF-03 | Origem encerra fora da métrica | Must | Motivo `moved_to_another_pipeline`, excluído das métricas |

## Critérios de Aceitação
```gherkin
Dado um negócio que precisa mudar de funil
Quando clonar-para-funil executa
Então cria um clone no destino
E encerra a origem com moved_to_another_pipeline (fora da métrica)
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/leads/clonar-para-funil.ts` | clone cross-pipeline | 🟢 |
| `lib/leads/motivo-da-perda.ts` | `moved_to_another_pipeline` canônico | 🟢 |
