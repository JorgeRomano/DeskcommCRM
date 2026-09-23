# Caso de Uso: Turno Inbound — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal
1. `loadInboundBodyForJob(db, {tenantId, conversationId, inboundMessageId})` carrega a linha canônica. 🟢
2. Abertura: `loadPlaybook` + checkpoint anterior + `lead_state` + `get_lead_context`. Montagem em `ritualBlocks` (`inbound-turn.ts:1386-1556`). 🟢
3. Loop: modelo chama tools dentro de `maxSteps`; `send_message.execute` (`:2843+`) aplica guarda de falso-vazio → teto → promessa → `runBeforeSend`. 🟢
4. Fechamento: 2ª chamada `purpose:'checkpoint'` → `parseCheckpointText` (`:1355-1382`) → persiste (`:573-607`, `:3976-4003`). 🟢
5. Pós-checkpoint: `decidirSeEnfileiraOperador` (`:1311-1319`) → enqueue `operator_turn` (`:4030-4065`). 🟢

## Fluxos Alternativos
- Orçamento estourado: `comHandoffSeOrcamentoAcabar` (`:672-745`) intercepta e faz handoff.
- Falso-vazio: até `MAX_VETOS_DE_FALSO_VAZIO=2` vetos antes de aceitar.

## Dependências
- Cadeia before-send, orçamento/LLM (`edge/llm`), `get-lead-context`, fila (`operator_turn`).

## Estado Interno
- `seq` (sequência de envio, só avança em tentativa real), outcomes, mensagens — no closure.

## Riscos e Lacunas
- 🟡 `maxSteps`/`maxSendsPerTurn` vêm de env; confirmar defaults na instalação.
