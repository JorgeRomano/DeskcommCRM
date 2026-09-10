# Automação e Follow-up

> Spec SDD (híbrida — módulo de topo) · Redator (Reversa) · Nível: Detalhado
> Módulos: `lib/automation/*`, `lib/followup/*`, `lib/escalacao/*`, `lib/agenda/*`
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Visão Geral

Mecanismos de continuidade e anti-morte: motor de regras dirigido por eventos (automação), máquina de follow-up por grafo (reengajamento), escalação/pausa de IA, e proteção de agenda. Follow-up é o mecanismo que mantém a demanda viva até resolução ou encerramento (invariante 4 do Sistema Vivo). 🟢

## Responsabilidades

- Executar `automation_rules` ativas quando um evento-gatilho chega (condições em AND → ações). 🟢
- Conduzir enrollments de follow-up por um grafo publicado, com claim, idempotência e planejamento de tempo. 🟢
- Reagir a inbound/handoff (cancelar, pausar, retomar enrollments). 🟢
- Decidir quem pode assumir atendimento agora e pausar a IA por atendimento manual. 🟢
- Proteger leads com compromisso agendado da cobrança do follow-up. 🟢

## Regras de Negócio

- Anti-loop: `caused_by_rule` ou `request_id` prefixo `rule:` não reprocessa. 🟢
- Guard de `entity_kind` evita rodar a regra 2× por linhas duplicadas do event_log. 🟢
- Condições `eq/neq/contains` em AND; campo ausente + `neq` = verdadeiro. 🟢
- Consentimento é guarda dedicada (`declined_at`), não condition. 🟢
- Agregação honesta: `skipped`+`failed` contam juntos; falha vence adiamento. 🟢
- 1 enrollment vivo por lead/org (23505 → 409). 🟢
- `completeTurn` só avança se status ∈ {active, waiting_reply} (lista positiva — não sobrescreve humano). 🟢
- `action` at-most-once send; dead-man `MAX_ACTION_RECHECKS=14` (era 5, matava follow-up da noite). 🟢
- opt-out no inbound cancela tudo (hard stop LGPD); handoff pausa por contato. 🟢
- Pausa por atendimento manual: 60min, renova, nunca encurta silêncio maior. 🟢
- Proteção de agenda: falha de leitura vira `indisponivel` (adia, nunca cobra sem confirmar). 🟢

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Executar regras de automação por evento com anti-loop e postpone all-or-nothing | Must | Dado evento-gatilho, regras ativas rodam; fora da janela → adiado |
| RF-02 | Conduzir enrollment por grafo (1 por lead/org) | Must | Dado enroll de contato com pointer ativo, cria; 2º → 409 |
| RF-03 | Processar nós do grafo (wait/condition/ai_classify/match_reply/repeat/action/end) | Must | Dado nó `action`, envia at-most-once com dead-man |
| RF-04 | Reagir a inbound/handoff nos enrollments vivos | Must | Dado opt-out, cancela todos (`opted_out`); handoff pausa |
| RF-05 | Decidir disponibilidade e frase de escalação ao modelo | Should | Dado ninguém disponível, frase instrui "não prometa prazo" |
| RF-06 | Pausar IA por atendimento manual pelo canal | Should | Dada resposta `fromMe`, silencia 60min renovável |
| RF-07 | Proteger lead com compromisso agendado da cobrança | Should | Dado compromisso futuro, follow-up adia (`em_voo`) |
| RF-08 | Agregar outcomes por fluxo | Could | `conversion_rate = converted/terminal` (dead excluído) |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Consistência | Idempotência por `idempotency_key` (`${node}:${steps}`) | `lib/followup/turn-bridge.ts` | 🟢 |
| Disponibilidade | Claim falho não lança nem confunde com "nada vencido" | `lib/followup/engine.ts:runFollowupTick` | 🟢 |
| Disponibilidade | Falha de um enrollment não derruba o tick | `lib/followup/engine.ts` | 🟢 |
| Escalabilidade | Claim com lease 120s; backoff exponencial de rechecks | `lib/followup/node-handlers.ts` | 🟢 |
| Correção | Postpone all-or-nothing antes de executar ações | `lib/automation/engine.ts` | 🟢 |

## Critérios de Aceitação

```gherkin
Dado um evento lead.stage_changed e uma regra ativa que casa
Quando runAutomationForEvent roda
Então as ações executam e o desfecho é agregado honestamente (skipped/failed não viram "success")

Dado um enrollment num nó action e o worker fora do ar por 15 rechecks
Quando processNode avalia
Então o enrollment vira dead (não re-enfileira infinito, não espera para sempre)

Dado inbound de opt-out num contato com enrollments vivos
Quando applyReactivityEvent roda
Então todos os enrollments vivos são cancelados com outcome opted_out
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Motor de automação | Must | Reação central a eventos do tenant |
| Máquina de follow-up | Must | Anti-morte (invariante 4) |
| Reatividade a inbound/handoff | Must | Não responder por cima do humano / opt-out |
| Escalação + pausa manual | Should | Continuidade IA↔humano |
| Proteção de agenda | Should | Não cobrar quem tem compromisso |
| Outcomes por fluxo | Could | Laço de retorno (invariante 7) |

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/automation/engine.ts` | `runAutomationForEvent`, `buildContext` | 🟢 |
| `lib/automation/conditions.ts` | `evaluateConditions` | 🟢 |
| `lib/automation/guarda-do-contato.ts` | `checarGuardasDeContato` | 🟢 |
| `lib/followup/node-handlers.ts` | `processNode` | 🟢 |
| `lib/followup/engine.ts` | `runFollowupTick`, `processEnrollment` | 🟢 |
| `lib/followup/turn-bridge.ts` | `completeTurnForEnrollment` | 🟢 |
| `lib/followup/reactivity.ts` | `applyReactivityEvent` | 🟢 |
| `lib/followup/enroll.ts` | `enrollFollowupFlow` | 🟢 |
| `lib/followup/outcome-stats.ts` | `aggregateFollowupOutcomes` | 🟢 |
| `lib/escalacao/disponibilidade.ts` | `expectativaDeAtendimento` | 🟢 |
| `lib/agenda/protecao-followup.ts` | `protecaoDaAgenda` | 🟢 |
