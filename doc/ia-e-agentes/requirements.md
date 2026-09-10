# IA e Agentes

> Spec SDD (organização híbrida — módulo de topo) · Redator (Reversa) · Nível: Detalhado
> Módulos do legado: `lib/agent-engine/*`, `lib/ai/*`, `lib/mcp/*`, `workers/agent-worker`, `workers/ai-*`, `workers/rag-indexer`
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Visão Geral

O núcleo de IA atende conversas de WhatsApp por agentes configuráveis, com dois runtimes convivendo: o **agent-engine** rico (fila Postgres, turnos com tools, guardrails determinísticos, RAG, pacing/spinning anti-ban, flywheel de aprendizado) e os **workers legados** (resposta e sentimento sobre `event_log`) para organizações sem versão de agente publicada. Toda saída ao cliente passa por uma cadeia versionada de guardrails. 🟢

## Responsabilidades

- Executar o turno do agente (ler contexto → decidir tools → enviar → checkpoint). 🟢
- Vetar deterministicamente cada envio (opt-out, LGPD, anti-ban, promessa, disclosure, etc.). 🟢
- Classificar sentimento e disparar handoff bot→humano. 🟢
- Indexar conhecimento (FAQ/documentos/catálogo) para RAG versionado. 🟢
- Expor operações do CRM como tools MCP a clientes externos autenticados. 🟢
- Aprender com turnos reais (judge + distiller) sob gate humano. 🟢

## Regras de Negócio

- Enviar é SEMPRE tool call (`send_message`); texto direto do modelo é descartado. 🟢
- A cadeia `before-send` é versionada (`BEFORE_SEND_CHAIN_VERSION` v6) e vigiada por teste de shape. 🟢
- Veto de gate volta ao modelo como erro instrutivo (ele reescreve no turno seguinte). 🟢
- Orçamento de IA estourado devolve a conversa à fila humana (não descarta o job — `terminal===true` → `cancelJob`, não `failJob`). 🟢
- Handoff idempotente por 5s com mesma razão; avisa o lead antes de silenciar; `bot_silenced_until='infinity'`. 🟢
- RAG: nunca ativa versão vazia; falta de chave de embedding → `retry` (não `skipped`) + item na Central. 🟢
- Pacing/spinning: números anti-ban vivem só em `pacing/defaults.ts`/`spinning/defaults.ts` (lint proíbe literais fora). 🟢
- Flywheel: propostas só viram comportamento quando o dono publica (gate humano). 🟢
- Multi-tenancy: workers usam service role → toda query filtra `organization_id` da row do evento/job, nunca do body. 🟢

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Processar `inbound_turn`: sessão fresca, ler playbook+checkpoint+state+mensagens, decidir tools, fechar com checkpoint | Must | Dado um job `inbound_turn`, o agente responde via `send_message` e persiste checkpoint validado por Zod |
| RF-02 | Vetar envio pela cadeia de 10 gates na ordem v6 | Must | Dado um corpo que viola um gate, o envio é bloqueado e a razão volta ao modelo |
| RF-03 | Triagem síncrona sem LLM (G1 humano, G4 legal, G4 stage) antes de invocar o modelo | Must | Dado "quero falar com atendente", dispara handoff sem chamar o LLM |
| RF-04 | Classificar sentimento e emitir `ai.sentiment_alert` abaixo do limiar | Should | Dada mensagem inbound negativa, grava `sentiment_score` e emite alerta |
| RF-05 | Indexar fonte de conhecimento em `ai_chunks` versionados | Should | Dada uma FAQ atualizada, cria versão nova e ativa só se ≥1 chunk gravado |
| RF-06 | Expor tools MCP autenticadas por Bearer com scope/role | Should | Dado token sem `mcp:write`, tool de escrita retorna -32002 |
| RF-07 | Rodar flywheel judge+distiller sobre turnos reais | Could | Dado turno com fato durável perdido, gera proposta pendente de aprovação |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Disponibilidade | Worker com graceful shutdown (drena até `SHUTDOWN_GRACE_MS`) + reaper de jobs órfãos | `workers/agent-worker/main.ts` | 🟢 |
| Segurança | Serialização por número via `pg_advisory_xact_lock` na cadeia de envio | `guardrails/before-send.ts` | 🟢 |
| Escalabilidade | Fila durável `job_queue` com lease/claim; flywheel OFF por knob | `agent-engine/queue`, `flywheel/live.ts` | 🟢 |
| Performance | Teto de sends por turno (`DEFAULT_MAX_SENDS_PER_TURN=3`); pacing throttle 1.2s+jitter | `agent/inbound-turn.ts`, `pacing/defaults.ts` | 🟢 |
| Observabilidade | `before_send_traces` por gate; `llm_calls` por chamada (custo/tokens/latência) | `guardrails/before-send.ts`, `lib/ai/log-invocation` | 🟢 |

## Critérios de Aceitação

```gherkin
Dado um lead que escreve no WhatsApp e uma org com agente publicado
Quando o job inbound_turn é processado
Então o agente lê contexto, decide send_message, a cadeia before-send aprova e a mensagem sai pelo canal

Dado que o agente tenta prometer "R$1" fora da tabela versionada
Quando a candidata passa pelo promiseGate
Então o envio é vetado e a razão instrutiva volta ao modelo para reescrita

Dado que o teto de gasto de IA da org foi atingido
Quando qualquer chamada de modelo do turno é feita
Então a conversa é devolvida à fila humana (handoff orcamento_de_ia) e o job não é descartado
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Turno do agente + cadeia before-send | Must | Caminho crítico de toda resposta ao cliente |
| Triagem síncrona de handoff | Must | Segurança: pedido de humano/menção legal não pode depender de LLM |
| Sentimento + handoff | Should | Alimenta o gate G2; degrada sem derrubar o bot |
| RAG | Should | Qualidade da resposta; sem ele o agente diz que vai confirmar |
| MCP | Should | Superfície externa; não é o caminho quente |
| Flywheel | Could | Melhoria contínua; gate humano, roda agendado |

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `workers/agent-worker/main.ts` | `startWorker` | 🟢 |
| `lib/agent-engine/agent/inbound-turn.ts` | `AGENT_TOOL_DEFS`, turno | 🟢 |
| `lib/agent-engine/guardrails/before-send.ts` | cadeia de gates | 🟢 |
| `lib/agent-engine/pacing/engine.ts` | `decidePacing` | 🟢 |
| `lib/agent-engine/spinning/engine.ts` | `decideSpinning` | 🟢 |
| `lib/agent-engine/flywheel/live.ts` | `runFlywheelOnce` | 🟢 |
| `workers/ai-response-worker.ts` | `processMessageReceived` | 🟢 |
| `workers/ai-sentiment-worker.ts` | `processSentiment` | 🟢 |
| `workers/rag-indexer.ts` | `processRagIndexer` | 🟢 |
| `lib/ai/handoff/orchestrator.ts` | `triggerHandoff` | 🟢 |
| `lib/mcp/server.ts`, `lib/mcp/auth.ts` | `createMcpServer`, `validateBearerToken` | 🟢 |
