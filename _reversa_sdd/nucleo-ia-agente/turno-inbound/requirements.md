# Caso de Uso: Turno Inbound

> Sub-unit de `nucleo-ia-agente`. O ritual de um turno disparado por mensagem recebida do lead.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Processa o job `inbound_turn`: uma mensagem do lead vira uma sessão de agente que abre com contexto, deixa o modelo agir por tools e fecha com checkpoint durável. 🟢

## Responsabilidades
- Carregar o corpo canônico do inbound pelo id exato (`loadInboundBodyForJob`, evita corrida). 🟢
- Montar a abertura (`buildOpeningMessage` → `ritualBlocks`). 🟢
- Orquestrar o loop de tools dentro de `maxSteps`. 🟢
- Fechar com a 2ª chamada `purpose:'checkpoint'` e persistir. 🟢
- Enfileirar `operator_turn` pós-checkpoint se o papel estiver ligado. 🟢

## Regras de Negócio
- Enviar é sempre `send_message`; texto livre descartado. 🟢
- Máx. `maxSendsPerTurn` (default 3) envios físicos por turno. 🟢
- `latestCheckpoint` (por seq) abre o turno; `checkpointDoJob` (por job_id) é lido pelo Operador. 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Executar o ritual completo abrir→loop→fechar | Must | Há checkpoint persistido ao fim do turno |
| RF-02 | Respeitar o teto de envios por turno | Must | `seq >= maxSendsPerTurn` bloqueia novos envios |
| RF-03 | Carregar inbound pelo id exato | Must | Corpo lido é `corpoDaMensagem`, nunca `body` cru |

## Critérios de Aceitação
```gherkin
Dado um job inbound_turn
Quando processado
Então abre com playbook+checkpoint+lead_state+histórico
E fecha com checkpoint JSON validado por Zod
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/agent-engine/agent/inbound-turn.ts` | `runAgentTurn`, `loadInboundBodyForJob`, `buildOpeningMessage` | 🟢 |
