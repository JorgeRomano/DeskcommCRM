# Núcleo de IA — Agente (`lib/agent-engine`)

> `requirements.md` — foca no QUE a unit faz, não no como.
> Escala de confiança: 🟢 CONFIRMADO (lido no código) · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte primária: `_reversa_sdd/code-analysis.md` (Unidade 1) e `.reversa/context/modules.json`.

## Visão Geral

Runtime do agente de IA como **worker de fila**. Cada job da fila vira uma **sessão LLM fresca**: todo o estado do run (sequência de envio, outcomes, mensagens) vive no closure da invocação, o que garante isolamento entre leads por construção. O runtime **impõe um ritual** ao turno (abertura com playbook + checkpoint + estado do lead + histórico; loop em que o modelo decide as tools; fechamento com uma segunda chamada que devolve apenas o JSON do checkpoint) e coloca **guardrails determinísticos** entre a decisão do modelo e o canal. 🟢

## Responsabilidades

- Processar os jobs de turno do agente (`inbound_turn`, `followup_turn`, `operator_turn`, `case_reply_turn`, `approved_reply`) drenados de `event_log`/`job_queue`. 🟢
- Impor o ritual do turno: abrir (contexto), loop de tools, fechar com checkpoint JSON durável. 🟢
- Garantir que **enviar é sempre um tool call `send_message`**; texto direto do modelo nunca chega ao canal (é descartado). 🟢
- Aplicar a cadeia `before-send` (11 gates versionados) entre a decisão do modelo e o adapter de canal. 🟢
- Gerir a máquina de estados do funil (`lead-state`), aceitando só transições válidas. 🟢
- Gerir handoff humano de primeira classe, casos humanos (loop IA↔humano) e escalação. 🟢
- Controlar orçamento de IA por org (avisar antes, bloquear com handoff, nunca bloquear sem aviso prévio). 🟢
- Aplicar pacing (janela de horário + warmup + throttle) e spinning (anti cópia em massa). 🟢
- Gerir contexto do modelo (compactação, rolling summary, poda determinística de resultados de tool). 🟢
- Prover a única costura de LLM (`edge/llm/run-model-call.ts`) e o adapter de canal (WAHA). 🟢

## Regras de Negócio

- Enviar é sempre um tool call `send_message`; texto direto do modelo nunca é enviado (descartado). — `lib/agent-engine/agent/inbound-turn.ts:22-31` 🟢
- Funil só avança para o próximo estágio válido; regressão é rejeitada; `won`/`lost` são terminais. — `lib/agent-engine/agent/lead-state.ts:37-45` 🟢
- Orçamento nunca bloqueia sem aviso prévio no mês; purposes `connection_test`/`jailbreak_detect`/`promise_semantic` são isentos. — `lib/agent-engine/edge/llm/orcamento.ts` 🟢
- Cadeia before-send versionada (v7) de 11 gates sob advisory lock por número; qualquer gate pode vetar. — `lib/agent-engine/guardrails/before-send.ts` 🟢
- Idempotência evento→job por `unique` + captura de `23505`; efeito exactly-once via guard de lease no `completeJob`. — `lib/agent-engine/queue/queue.ts` 🟢
- Máximo de 3 envios físicos por turno (`DEFAULT_MAX_SENDS_PER_TURN`); teto configurável por `MAX_SENDS_PER_TURN`. — `lib/agent-engine/agent/inbound-turn.ts:462` 🟢
- Fechamento do turno usa uma 2ª chamada de modelo (`purpose:'checkpoint'`) porque essa chamada SEMPRE acontece; uma tool dependeria de o modelo lembrar de chamá-la. — `inbound-turn.ts:573-607` 🟢
- Schemas das tools são largos (`.passthrough()`) para o SDK, mas a validação real é whitelist `.strict()` dentro de cada `apply*`: campo extra/forjado vira erro de ENSINO ao modelo, nunca strip silencioso. 🟢
- `declaracao` do checkpoint é `.optional()` sem default: `undefined` ("modelo não declarou") ≠ `{nada_a_declarar:true}` ("avaliou, nada a declarar"). 🟢
- Operador muta o CRM mas NUNCA fala com o lead (separação por ausência de `send_message` no toolset). 🟢

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Processar `inbound_turn` executando o ritual completo (abrir → loop → fechar checkpoint) | Must | Dado um job `inbound_turn`, quando processado, então há um checkpoint persistido em `lead_checkpoints` e no máximo `maxSendsPerTurn` envios físicos |
| RF-02 | Só permitir envio via tool `send_message`; descartar texto livre do modelo | Must | Texto direto (não tool call) do modelo não gera mensagem no canal |
| RF-03 | Validar transições de funil pela máquina de estados fixa | Must | Uma transição de regressão (ex.: `qualified`→`contacted`) é rejeitada; `won`/`lost` não transicionam |
| RF-04 | Executar a cadeia `before-send` antes de tomar conexão, curto-circuitando no 1º veto | Must | Um contato `optedOut` produz veto `contato_bloqueado` e nenhum envio |
| RF-05 | Garantir idempotência evento→job e efeito exactly-once no completeJob | Must | Reenfileirar o mesmo `sourceEventId` devolve a linha existente (`deduped:true`); `completeJob` com lease inválido lança |
| RF-06 | Aplicar orçamento de IA: avisar ao atingir limiar, bloquear com handoff quando estourar | Must | Ao estourar o teto, o turno lança `LlmBudgetExceededError`, avisa o lead com texto de código e roda `performHumanHandoff` |
| RF-07 | Aplicar pacing (janela/warmup/throttle) e spinning (anti cópia em massa) | Should | Fora da janela do tenant → veto `outside_window`; N cópias idênticas na janela → veto `mass_identical` |
| RF-08 | Gerir handoff humano irrevogável e casos humanos (loop IA↔humano) | Must | `performHumanHandoff` seta `force_human=true`, silencia o bot, cancela crons e cria item de inbox |
| RF-09 | Gerir contexto: compactação/rolling summary e poda determinística de resultados de tool | Should | Acima de `triggerMessages`, `maybeCompact` gera rolling summary; rounds antigos viram stub de poda |
| RF-10 | Circuit breaker de tools por-run (exact/same-tool/idempotent-no-progress) | Should | Repetição de tool sem progresso abre o breaker e ensina o modelo |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Segurança | Args e resultados de tool nunca logados (só hash + contagem) | `lib/agent-engine/agent/tool-breaker.ts` | 🟢 |
| Segurança | Higienização de chaves/segredos nas mensagens do provedor (`sk-`/`AIza`/`Bearer`/`api-key`) | `lib/agent-engine/edge/llm/run-model-call.ts` (`redigirMensagemDoProvedor`) | 🟢 |
| Segurança | Egress do modelo com `allowlistedFetch` fail-closed, re-checa host em redirect | `lib/agent-engine/edge/egress.ts` | 🟢 |
| Escalabilidade | Fila com claim de dois estágios (`DISTINCT ON` lane + `FOR UPDATE SKIP LOCKED`) e `pg_advisory_xact_lock` para `maxConcurrency` | `lib/agent-engine/queue/queue.ts` | 🟢 |
| Disponibilidade | Backoff exponencial `power(2,attempts-1)*10` cap 120s; job vira `dead` e cai no inbox | `lib/agent-engine/queue/queue.ts` (`failJob`) | 🟢 |
| Disponibilidade | Loop da fila fail-open, nunca rejeita (shutdown gracioso) | `lib/agent-engine/queue/loop.ts` | 🟢 |
| Performance | Prefixo de prompt estável para cache; `run_cache_read_ratio` medido | `lib/agent-engine/edge/llm/run-model-call.ts`, `obs/metrics.ts` | 🟢 |
| Escalabilidade | Prefixo cacheável preservado pela poda (opera só no sufixo) | `lib/agent-engine/prune-tool-results.ts` | 🟢 |

> Inferido a partir do código. Validar com equipe de operações.

## Critérios de Aceitação

```gherkin
Dado um job inbound_turn com um lead válido
Quando o runtime processa o turno
Então abre com playbook + checkpoint anterior + lead_state + histórico
E o modelo decide tools dentro de maxSteps
E o turno fecha com uma segunda chamada que persiste o JSON do checkpoint

Dado que o modelo produz texto livre em vez de chamar send_message
Quando o turno termina
Então nenhuma mensagem é enviada ao canal (texto descartado)

Dado que o orçamento mensal de IA da org está estourado
Quando um turno tenta chamar o modelo (purpose não isento)
Então lança LlmBudgetExceededError
E avisa o lead com texto de código (sem tokens)
E executa performHumanHandoff
E re-lança para a fila decidir o destino do job

Dado um contato com force_human=true (optedOut)
Quando a cadeia before-send avalia o envio
Então o gate 'stop' veta com contato_bloqueado
E nenhum gate posterior executa (curto-circuito)
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Ritual do turno e checkpoint durável (RF-01) | Must | Caminho crítico de todo turno do agente |
| Envio só via `send_message` (RF-02) | Must | Regra de segurança de canal sem fallback |
| Máquina de estados do funil (RF-03) | Must | Integridade do CRM sem alternativa |
| Cadeia before-send (RF-04) | Must | Guardrail entre modelo e canal, sem fallback |
| Idempotência/exactly-once (RF-05) | Must | Evita envio duplicado e trabalho repetido |
| Orçamento com handoff (RF-06) | Must | Controle de custo e continuidade do atendimento |
| Pacing/spinning (RF-07) | Should | Anti-ban; há defaults e degradação |
| Compactação/poda (RF-09) | Should | Otimização de contexto com fallback |
| Circuit breaker de tools (RF-10) | Could | Resiliência acionada em loop de erro |

> Prioridade inferida por frequência de chamada e posição na cadeia de dependências.

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/agent-engine/agent/inbound-turn.ts` | `runAgentTurn`, `comHandoffSeOrcamentoAcabar`, `loadInboundBodyForJob` | 🟢 |
| `lib/agent-engine/agent/lead-state.ts` | `applyLeadStateUpdate` | 🟢 |
| `lib/agent-engine/agent/human-handoff.ts` | `performHumanHandoff`, `isLeadInHandoff` | 🟢 |
| `lib/agent-engine/agent/operator-turn.ts` | handler `operator_turn`, `apuraDonoDaPromessa` | 🟢 |
| `lib/agent-engine/guardrails/before-send.ts` | `evaluateBeforeSend`, `runBeforeSend` | 🟢 |
| `lib/agent-engine/pacing/engine.ts` | `decidePacing` | 🟢 |
| `lib/agent-engine/spinning/engine.ts` | `decideSpinning` | 🟢 |
| `lib/agent-engine/queue/queue.ts` | `enqueueJob`, `claimJobs`, `completeJob`, `failJob` | 🟢 |
| `lib/agent-engine/edge/llm/run-model-call.ts` | `runModelCall` | 🟢 |
| `lib/agent-engine/edge/llm/orcamento.ts` | `decidirOrcamento` | 🟢 |
| `lib/agent-engine/agent/tool-breaker.ts` | `wrapToolsWithBreaker` | 🟢 |
