# Análise de Código — DeskcommCRM

> Gerado pelo Arqueólogo (Reversa) — fase de Escavação
> Nível de documentação: **detalhado**
> Escala de confiança: 🟢 CONFIRMADO (lido no código) · 🟡 INFERIDO (padrão) · 🔴 LACUNA (validação humana)

Este documento consolida a análise técnica módulo a módulo do projeto legado, agrupada por
domínio conforme o plano de escavação. Cada unidade traz propósito, fluxo de controle (funções
principais), algoritmos não-triviais, estruturas de dados, metadados/configuração e dependências.

---

## Unidade 1 — Núcleo de IA: `lib/agent-engine/`

### Propósito 🟢
Runtime do agente de IA como **worker de fila**. Cada job vira uma **sessão LLM fresca**: todo o
estado do run (sequência de envio, outcomes, mensagens) vive no closure da invocação — isolamento
entre leads por construção. O runtime **impõe um ritual** (abrir com playbook + checkpoint + estado
do lead + histórico; modelo decide tools; fechar com uma 2ª chamada que devolve só o JSON do
checkpoint). Guardrails determinísticos ficam entre a decisão do modelo e o canal.

### 1.1 O ritual do turno (`agent/inbound-turn.ts`, ~4352 linhas) 🟢

Handler do job `inbound_turn`. Ritual imposto pelo runtime (cabeçalho `inbound-turn.ts:9-37`):

1. **Abertura** — `loadPlaybook` (system, por ponteiro) + checkpoint anterior de `lead_checkpoints`
   (commitments/objections/next_action + rolling summary) + `lead_state` (estágio do funil) +
   últimas N mensagens via `get_lead_context`. Montagem: `buildOpeningMessage` → `ritualBlocks`
   (`inbound-turn.ts:1386-1556`).
2. **Loop do modelo** — modelo chama tools livremente dentro de `maxSteps` (`AGENT_MAX_STEPS`).
   Enviar é SEMPRE `send_message` (tool call); texto direto do modelo NUNCA é enviado (descartado).
   `update_lead_state` marca avanços; a máquina de estados no código valida (`lead-state.ts`).
3. **Fechamento** — 2ª chamada com `purpose:'checkpoint'` devolve SOMENTE o JSON do checkpoint,
   validado por Zod e persistido (`inbound-turn.ts:573-607`, `3976-4003`, `parseCheckpointText`
   em `1355-1382`). Escolhido porque a chamada de fechamento SEMPRE acontece (uma tool dependeria
   de o modelo lembrar de chamá-la).
4. **Pós-checkpoint** — enfileira `operator_turn` (fire-and-forget) se o papel estiver ligado
   (`decidirSeEnfileiraOperador` em `1311-1319`, enqueue em `4030-4065`).

**`runAgentTurn`** (`inbound-turn.ts:1735-1790`) — entrada compartilhada por inbound/followup/
case-reply. Envolve `executarTurnoDoAgente` (`1811+`) no escort de orçamento
**`comHandoffSeOrcamentoAcabar`** (`672-745`): em `LlmBudgetExceededError`, lê o briefing do
checkpoint durável (sem LLM), avisa o lead (texto de código, sem tokens), roda `performHumanHandoff`
e **re-lança** (a fila decide o destino do job). Envolve o turno inteiro porque chamadas auxiliares
(`classifyStage`, `maybeCompact`) ocorrem ANTES da chamada do modelo e lançariam primeiro.

**`AGENT_TOOL_DEFS`** (`inbound-turn.ts:190-431`) — superfície estática de 13 tools (prefixo estável
de cache F2-17): `get_lead_context`, `send_message`, `update_lead_state`, `schedule_followup`,
`save_lead_note`, `get_lead_note`, `search_knowledge`, `request_human_handoff`,
`read_skill_reference`, `open_human_case`, `provide_case_update`, `send_template`. Schemas são
LARGOS (`.passthrough()`) para o SDK; a validação REAL é whitelist `.strict()` dentro de cada
`apply*` — campo extra/forjado vira **erro de ENSINO** ao modelo, nunca strip silencioso.

**Guarda de envio (`send_message.execute`, `inbound-turn.ts:2843-2962+`)** — `claimsCurrentInboundIsEmpty`
(guarda de falso-vazio, regex exigindo referência-a-mensagem + alegação-de-vazio dentro de 90
chars; pronome `ela` deliberadamente excluído por falso positivo medido); teto `seq >= maxSendsPerTurn`;
detecção de promessa fora da tabela; então a cadeia de guardrails `runBeforeSend`. `seq` só avança
em tentativa real de envio.

**Constantes de domínio:** `MAX_VETOS_DE_VOCABULARIO_INTERNO=2` (`:434`), `MAX_VETOS_DE_FALSO_VAZIO=2`
(`:449`), `DEFAULT_MAX_SENDS_PER_TURN=3` (`:462`), `ALLOWLIST_TTL_MS_PADRAO=21 dias` (`:1202`).
`JobSettledError` (`:466-469`) = job já resolvido pelo próprio run (ex.: veto is_blocked), worker
não re-tenta.

**Estruturas-chave:**
- `checkpointContentSchema` (`:525-545`): `{commitments:string[], objections:string[],
  next_action:string|null, rolling_summary:string, declaracao?:DeclaracaoDoTurno}`. `declaracao` é
  `.optional()` SEM default — `undefined` ("modelo não declarou") ≠ `{nada_a_declarar:true}`
  ("avaliou, nada a declarar").
- `LeadCheckpointRow` (`:554-568`): redeclara `declaracao: DeclaracaoDoTurno | null` (banco guarda
  null, modelo omite undefined).
- `inboundTurnPayloadSchema`: `conversation_id, contact_id, channel_session_id, inbound_message_id,
  crm_event_id` (todos uuid, passthrough). Organization/lead vêm da ROW do job (fonte confiável).
- `loadInboundBodyForJob` (`:506`): carrega a linha canônica do inbound pelo id exato
  (`corpoDaMensagem`, nunca `body` cru) para evitar corrida com eventos concorrentes.
- `checkpointDoJob` vs `latestCheckpoint` (`:1234-1300`): `latestCheckpoint` = mais recente por seq
  (abrir turno); `checkpointDoJob` filtra por `job_id` (processar — o Operador N lê a declaração N,
  não a N+1 que pode entrar no debounce de 8s).

### 1.2 Outros tipos de turno 🟢

- **`case-reply-turn.ts`** (164 l) — job `case_reply_turn` (spec 15 §4.3): a resposta de um HUMANO
  a um caso aberto re-injeta um turno ("follow-up ao contrário"). `REENTRY_ACTIONS =
  ['resolved','need_lead_info']`; `EXPECTED_STATUS_FOR_ACTION = {resolved:'resolved',
  need_lead_info:'awaiting_lead'}`; `resolved` é TERMINAL. Resolve a conversa a partir do CASO.
- **`followup-turn.ts`** (721 l) — job `followup_turn` (F3-03). Calcula o delta temporal no momento
  do disparo. `DAY_MS`/`HOUR_MS`. Três caminhos: (a) flow-driven `runFlowDrivenTurn`
  (send_message/classify/plan_timing); (b) template determinístico `runDeterministicReentry`
  (sem LLM, variante por hash); (c) `runAgentTurn` normal. `sendFixedOutbound` (`:486-620`) envia
  via cadeia de guardrails sem LLM. NÃO usa `sendInBubbles`/atraso humano.
- **`operator-turn.ts`** (678 l) — job `operator_turn` (spec 16 §3.2): o papel OPERADOR — muta o
  CRM, NUNCA fala com o lead (sem `send_message` no toolset — separação por AUSÊNCIA). `SYSTEM_DO_OPERADOR`
  (`:79-95`). Curto-circuito: `nada_a_declarar:true` → sem chamada de modelo; `declaracao===null`
  (ninguém avaliou) → RODA. `apuraDonoDaPromessa` (precedência: tool-neste-turno > retorno-vivo >
  sem-tools > não-agiu). `registrarDesfecho` grava `event_log` sempre; timeline `promise_unowned`/
  `promise_unfulfilled` só quando promessa sem dono.

### 1.3 Reply/Draft (resposta assistida) 🟢

- **`draft-reply.ts`** (Onda 5.1): `generateDraftReply` — rascunho sob demanda, sem envio; sem tools
  (`result.text` é o rascunho). Injeta `[MODO RASCUNHO]`. Usa `last_human_decision`.
- **`approved-reply.ts`**: `createApprovedReplyHandler` (job `approved_reply`) — envia rascunho
  aprovado por humano. `fn_reply_settle`, `reconcileAcceptedSend` (idempotente contra resposta HTTP
  perdida), `runBeforeSend`.
- **`reply-drafts.ts`**: `generateReplyDraft` — sugestão com contexto completo via `runAgentPreview`.
  `replyDraftSchema` (id, status, revision, generation_token uuid, agent_version_id, service_boundary).
  Usa `fn_reply_begin`. Persiste em `ai_reply_drafts`.
- **`sugestao-de-resposta.ts`**: regras puras de UI. `StatusDaSugestao` = generating|pending|approved|
  sending|sent|dismissed|stale|failed. `SEM_NADA_A_OFERECER = {dismissed, stale, sent}`.

### 1.4 Estado, classificação e roteamento 🟢

- **`lead-state.ts`** (236 l) — **Máquina de estados do funil (F2-10).** `LEAD_STAGES =
  ['new','contacted','qualifying','qualified','negotiating','won','lost']`. `LEAD_STAGE_TRANSITIONS`
  (grafo fixo `:37-45`): new→contacted|lost, contacted→qualifying|lost, qualifying→qualified|lost,
  qualified→negotiating|lost, negotiating→won|lost, won/lost terminais. `applyLeadStateUpdate`:
  guarda de chaves proibidas (`__proto__/constructor/prototype`), parse whitelist, validação de
  transição, upsert atômico via CTE + histórico. Idempotente no mesmo estágio. `next_action_seq`
  incrementa só quando `next_action` é reescrita (identidade da proposta para autorização humana).
- **`stage-classifier.ts`** (215 l) — classificador barato por turno (F3-11, SalesGPT): SUGERE
  estágio, sem escrita. `classifyStage` (purpose `stage_classifier`); falha do provider degrada
  para null EXCETO `LlmBudgetExceededError` (re-lança). `recordStageDivergenceCandidate` grava
  candidato golden-set em arquivo (PII vai a arquivo, nunca log).
- **`intent-classifier.ts`** (122 l) — classificador do Intent Router. `IntentVerdict{intentName,
  confidence}`. `parseIntentVerdict` NUNCA lança; intent tem de estar em `members` (anti-alucinação).
- **`resolve-turn-agent.ts`** (261 l) — decide QUAL agente atende: sticky → classificação →
  fallback → genérico. `outcome` = no_router|classified|sticky|reclassified|fallback|no_match|
  classifier_failed. 7 regras documentadas no cabeçalho.
- **`router-config.ts`** (110 l) — `loadActiveRouter` (≤1 ativo por channel_session). `classifierModel`
  default `claude-haiku-4-5`, `sticky` default true, `minConfidence` default 0.6.
- **`agent-config.ts`** (304 l) — `PublishedAgentConfig` (~40 campos): operationMode
  ('automatic'|'assisted'), systemPrompt, provider/model/credentialId, maxSteps, janelas de
  histórico, handoffKeywords, casesEnabled, toolIds, ragTopK (default 5)/ragSimilarityThreshold
  (default 0.40, calibrado migration 0097), operatorEnabled/Model/ToolIds, pipelineIds. Resolvido
  por ponteiro, sem cache de processo.

### 1.5 Handoff, casos e escalação 🟢

- **`human-handoff.ts`** (519 l) — handoff de primeira classe (F4-06). `HUMAN_HANDOFF_PATTERNS`
  (4 regex PT-BR conservadores sobre texto NFD). `isLeadInHandoff` (force_human OR
  bot_silenced_until>now). **`performHumanHandoff`** (`:145-320`): (a) `contacts.force_human=true`
  (irrevogável); (b) conversa ai_handling→pending + `bot_silenced_until='infinity'` + limpa
  active_ai_agent_id/intent; (c) cancela crons; (d) upsert item de inbox; (e) timeline
  `handoff_triggered`. `requestHumanHandoffInputSchema` ESPELHA `AGENT_TOOL_DEFS` (testado).
- **`human-cases.ts`** (427 l) — loop assíncrono IA↔humano (spec 15). `CaseEventKind` = opened|
  human_replied|lead_asked|lead_provided|lead_unresponsive|resolved|escalated|cancelled|
  agent_noted|alert_sent. Status abertos: awaiting_human|awaiting_lead; terminais: resolved/
  escalated/cancelled. Transições como CTEs de statement único condicionais
  (`WHERE status=precondição`, atômicas).
- **`aviso-de-escalacao.ts`** (255 l) — envio pelo engine do aviso "a IA está saindo". `SEQ_DO_AVISO=0`
  (nunca colide com seq de send_message). NUNCA lança; roda por `runBeforeSend` com
  `enforceSpinning:false` (único caller que desarma).

### 1.6 Gestão de contexto (tríade anti-context-rot) 🟢

- **`compaction.ts`** (281 l) — compactação + rolling summary + flush pré-compactação (F3-07).
  `CompactionKnobs{triggerMessages, model?, transcriptMaxTokens}`. `runFlush` grava lead_notes
  duráveis ANTES da compactação (best-effort). `maybeCompact` retorna null se < triggerMessages.
- **`prune-tool-results.ts`** (107 l) — poda determinística (sem LLM) de resultados de tool antigos
  (F3-10). `PruneToolResultsKnobs{windowTurns(≥1), minResultTokens}`. Rounds antigos viram stub
  `[resultado podado — tool=X …]`. Opera só no sufixo, nunca no prefixo cacheável.
- **`org-memory.ts`** (61 l) — memória da org (doc por ponteiro + entradas de aprendizado), no
  prefixo estável. `composeSystemPrompt` (ordem canônica: playbook → org memory → índice de skills).

### 1.7 Tools, memória e conhecimento 🟢

- **`schedule-followup.ts`** (184 l) — tool `schedule_followup` (F3-02). Guarda anti-empilhamento
  (1 follow-up vivo por lead). Mensagens de rejeição DECLARAM o horário atual (modelo não sabe "hoje").
- **`lead-notes.ts`** (204 l) — memória durável por lead (F3-05). Teto rígido no WRITE (orçamento de
  índice). `applySaveLeadNote` rejeita se `estimateIndexTokens > budgetTokens` (ensina consolidação
  via `supersedes`).
- **`search-knowledge.ts`** (169 l) — RAG no turno. `KnowledgeHit{chunk_id, knowledge_source_id,
  content, similarity, metadata}`. Consulta SEM threshold e filtra em JS. Erros ensinam o modelo a
  não inventar (no_knowledge_base, knowledge_sem_chave, knowledge_unavailable).
- **`skills.ts`** (316 l) — skills situacionais com matching determinístico de keywords (F3-09,
  Parlant). Disclosure progressivo: índice no prefixo estável, corpo carregado só em hard-match.
  `MAX_SKILL_BODY_LINES=200`.
- **`playbook.ts`** (164 l) — playbook em camadas versionado por ponteiro (F2-07).
  `PlaybookLayer`=platform|tenant|campaign. `MAX_PLAYBOOK_LAYER_LINES=200`.
- **`declaracao.ts`** (135 l) — declaração do turno, fronteira FALAR/OPERAR (spec 16 §5). Viaja na
  chamada de checkpoint (que sempre acontece). Vocabulário de negócio apenas.

### 1.8 Resiliência, entrega e knobs 🟢

- **`tool-breaker.ts`** (235 l) — **circuit breaker (F2-15).** Estado por-run no closure. 3 modos:
  exact_failure (mesma tool+args), same_tool_failure (mesma tool, args variados), idempotent_no_progress
  (tool read-only, mesmo resultado repetido). `READ_ONLY_TOOLS = ['get_lead_context','get_lead_note',
  'search_knowledge','read_skill_reference']`. Args/resultados NUNCA logados (só hash+contagem).
- **`split-message.ts`** (176 l) — divisão em bolhas (Onda 4). `splitIntoBubbles`: parágrafo→frase→
  palavra. `splitSentences` protege número BR ("R$ 10.990" — `.` entre dígitos não é fim de frase).
  `sendInBubbles<T>` para na 1ª bolha não-OK; `antesDaPrimeira` roda UMA vez (atraso humano).
- **`atraso-humano.ts`** (114 l) — simulação de atraso humano. `alvo = clamp(ATRASO_NOTAR_MS +
  MS_POR_CARACTERE×len, ATRASO_MINIMO_MS, ATRASO_MAXIMO_MS)`. Constantes: `ATRASO_NOTAR_MS=900`,
  `MS_POR_CARACTERE=22`, `ATRASO_MINIMO_MS=1200`, `ATRASO_MAXIMO_MS=7500`. Só inbound-turn usa.
- **`media-parts.ts`** (99 l) — partes de mídia nativas (Onda 3), com gate de capacidade.
  `NativeMediaPart={type:'file', data:Buffer, mediaType}` (AI SDK v7: imagem+pdf como `file` inline).
  Só o inbound MAIS RECENTE com mídia. Try/catch pula item (nunca aborta o turno).
- **`turn-knobs.ts`** (66 l) — `turnKnobsFromEnv` projeta todas as env vars nos knobs do turno.
- **`reentry-knobs.ts`** (97 l) — knobs de re-entrada por ponteiro (F5-10, alvo do flywheel).

### 1.9 Guardrails determinísticos (`guardrails/`) 🟢

**`before-send.ts`** — a costura determinística entre a decisão `send_message` do modelo e o canal.
Cadeia declarativa e versionada (`BEFORE_SEND_CHAIN_VERSION = 7`), avaliada nesta ordem:

| # | Gate | Veto/efeito |
|---|------|-------------|
| 1 | `stop` | veta se `ctx.optedOut` (is_blocked OU force_human) — `contato_bloqueado` |
| 2 | `lgpd` | `isAnonymized` → `lgpd_anonymized`; prospecting sem base legal → `lgpd_missing_legal_basis` |
| 3 | `pacing` | `decidePacing`; capability `banRisk`; throttle → `waitMs` (espera, não veto) |
| 4 | `messaging_window` | fora da janela de 24h e não-template → `messaging_window_closed` |
| 5 | `spinning` | `decideSpinning`; armado por default (anti-ban) |
| 6 | `promise` | `decidePromise` determinístico (preço/desconto/parcelas) |
| 7 | `semantic_promise` | classificação LLM assíncrona → `promise_semantic` |
| 8 | `case_promise` | promessa de humano sem caso aberto → `case_promise_without_case` |
| 9 | `internal_vocabulary` | `detectarVazamentoInterno` → `internal_vocabulary_leak` |
| 10 | `agenda_stall` | promessa de agenda sem tool chamada → `agenda_stall_sem_ferramenta` |
| 11 | `disclosure` | 1º outbound sem disclosure; modo `inject` (prepend) ou `veto` |

`evaluateBeforeSend` (puro) curto-circuita no 1º veto (gates restantes = 'skipped'), acumula
`throttleWaitMs`, aplica `amendBody`. `runBeforeSend` (stateful): paga o atraso humano ANTES de
tomar conexão (issue #654), pega `pg_advisory_xact_lock(hashtext(channelSessionId))` (serializa
read-then-act por NÚMERO), avalia, grava trace durável em `before_send_traces`, e em pass dorme
throttle e envia pelo adapter.

**Detectores de camada:**
- `vazamento-interno.ts` — detector determinístico de vazamento de vocabulário interno (snake_case,
  nomes de tool MCP, palavras de arquitetura, papéis/acesso, HTTP 403, UUID v4, SQLSTATE, stack traces).
- `human-promise.ts` — `detectHumanPromise` (8 regex PT-BR conservadores de promessa-de-humano).
- `sinal-de-urgencia.ts` — `detectUrgencySignal` (léxico); só prioriza alerta, nunca veta.
- `messaging-window.ts` — janela derivada de 24h (`WINDOW_MS=24h`); null = fechada (fail-closed).
- `ajustes-de-estilo-da-org.ts` — lista fechada de reescritas (`['sem_travessao_longo']`) de
  `org_guardrail_layers` (prefixo `estilo:`).
- `camadas-da-org.ts` — escolha três-estados (null/true/false) por org das camadas semânticas pagas
  (`['promessa_semantica','jailbreak']`); fail-open para default do ambiente.
- `promise/` — `engine.ts` (regex determinístico R$/%/parcelas + `decidePromise`), `table.ts`
  (tabela versionada por ponteiro), `semantic.ts` (classificador LLM binário, fail-open).
- `jailbreak/classifier.ts` — `classifyJailbreak` (purpose `jailbreak_detect`); ADVISORY, nunca veta
  inbound sozinho; `JAILBREAK_ESCALATION_LEVEL='high'`. `escalateJailbreakPromise` (high + promessa
  fora da tabela → agent_inbox_items).
- `disclosure/template.ts` — `DisclosureMode='inject'|'veto'`, versionado por ponteiro.
- `lgpd/legal-basis.ts` — `LegalBasis{basis:'consent'|'legitimate_interest'|null, ...}`,
  `LgpdInput{isAnonymized, isProspecting, legalBasis}`; origem importada sem prova = inválida.

### 1.10 Pacing (janela de envio + anti-ban) 🟢

- **`defaults.ts`** — `PACING_DEFAULTS`: throttle 1200ms, jitter 800ms, janela 7–22h, allowSunday
  true, tz America/Sao_Paulo, warmup [{0,20},{4,50},{8,100},{15,200},{31,null}].
- **`engine.ts`** (puro) — `decidePacing`: (1) janela de horário na tz do tenant → `outside_window`;
  (2) se !banRisk, allow; (3) warmup cap por idade do número, `effectiveCap=min(warmup, crmDailyLimit)`
  → `warmup_cap`/`daily_cap`; (4) throttle. Wall-clock via Intl (DST-safe).
- **`store.ts`** — seam Postgres (channel_knobs, pacing_ledger). `fusoDaJanela` (canal > org > default).
- **`ledger-supabase.ts`** — leitor CRM-side (Supabase/PostgREST) para o consumidor de event_log;
  mesma REGRA, leitor diferente; nunca lança (fail-open); não aplica janela de horário.
- **`aviso-de-janela.ts`** — `avisarJanelaFechada`/`resolverAvisoDeJanela` (agent_inbox_items).

### 1.11 Fila (`queue/`) 🟢

- **`queue.ts`** — `JobKind` (inbound_turn|followup_turn|watchdog|flywheel|case_reply_turn|
  operator_turn|transactional_delivery|approved_reply), `JobStatus` (pending|running|done|failed|dead).
  `enqueueJob` (idempotência evento→job: 23505 + sourceEventId → devolve linha existente
  `{deduped:true}`). `claimJobs` (dois estágios: `DISTINCT ON (coalesce(contact_id,id))` uma lane por
  vez + `FOR UPDATE SKIP LOCKED`; sob `pg_advisory_xact_lock(CLAIM_LOCK_KEY=727258)` para
  maxConcurrency). `completeJob` (guard status='running' AND locked_by AND locked_at=acquiredAt;
  rowCount≠1 → throw = efeito exactly-once). `failJob` (backoff exponencial `power(2,attempts-1)*10`
  cap 120s; 'dead' → inbox `job_dead`). `cancelJob` (terminal, sem retry). `reapExpiredJobs`
  (reaper de visibility-timeout).
- **`claim.ts`** — `claimOfJob(job)` (preserva microssegundo do claim).
- **`loop.ts`** — `rodarLoopDaFila`; relógio consultado só após rodada vazia; fail-open; nunca
  rejeita (shutdown gracioso).

### 1.12 Spinning (detecção de cópia em massa) 🟢

- **`engine.ts`** — `decideSpinning`: allowlist → sha256 exato + similaridade Jaccard ≥ threshold
  sobre janela; matchCount ≥ repetitionThreshold → veto `mass_identical`.
- **`defaults.ts`** — `SPINNING_DEFAULTS`: windowSize 20, similarity 0.8, repetition 2.
- **`store.ts`** — `outbound_copies` (normalizado + hash).

### 1.13 Edge (LLM, CRM, canal) 🟢

- **`edge/llm/orcamento.ts`** (puro) — `decidirOrcamento`: escapes ordenados (modo off, kill switch,
  purpose isento, teto≤0, teto<piso), depois limiar → seguir/avisar/`bloquear`. Nunca bloqueia sem
  aviso prévio. `PISO_DE_TETO_CENTS=100`, `PURPOSES_ISENTOS=['connection_test','jailbreak_detect',
  'promise_semantic']`, `LIMIAR_PADRAO_PCT=80`, `HANDOFF_REASON_ORCAMENTO='orcamento_de_ia'`.
  `SQL_ORCAMENTO` (lê ai_budgets + `fn_gasto_de_ia_do_mes`).
- **`edge/llm/run-model-call.ts`** — a ÚNICA costura de LLM. `runModelCall`: resolve config →
  binding purpose→provider/model → checa modelo habilitado → aplica orçamento (lança
  `LlmBudgetExceededError` em bloqueio) → `buildStablePrefix` (cache) → `generateText`
  (stopWhen stepCountIs(maxSteps)) → insere `llm_calls` (usage/cost/latency). `redigirMensagemDoProvedor`
  higieniza chaves (sk-/AIza/Bearer/api-key).
- **`edge/llm/providers.ts`** — `ProviderRegistry`; `createDefaultRegistry` (anthropic/openai/google/
  openrouter/deepseek, cada um com `allowlistedFetch`). Deps externas: `@ai-sdk/anthropic|openai|google`,
  `ai`, `ai/test`.
- **`edge/crm/send-message.ts`** — `sendTurnMessage` (assere políticas de operação/boundary/proactive/
  meeting/approvedReply, então `sendWithLedger`); ledger id É a idempotency_key. `applySendOutcome`
  (blocked → cancelJob + cancelPendingCrons; queued → rescheduleJob).
- **`edge/channel/waha-adapter.ts`** — `WahaChannelAdapter implements ChannelAdapter`; envia via
  `sendTurnMessage`; `sessionHealth` lê o espelho `channel_session_health` (NUNCA fala com WAHA
  direto — regra dura). `capabilities`→{freeformAnytime:true, serviceWindowHours:null}.
- **`edge/crm/get-lead-context.ts`** — `getLeadContext` (leituras org-scoped de contacts/conversations/
  messages/activities/demandas). `fitToBudget` (dropa antigas, depois corta corpo pela metade).
- **`edge/crm/move-lead-stage.ts`** — `mirrorLeadStageToCrm`; `MirrorReason` (not_configured|
  human_conflict|fora_do_escopo|perda_sem_motivo|crm_error|crm_unavailable).
- **`edge/crm/mcp-client.ts`** — pós-fusão o transporte MCP HTTP está morto; só carrega o client
  admin do Supabase (service role, bypassa RLS).
- **`edge/crm/mcp-tools.ts`** — `BLOCKED_TOOL_IDS = IDS_DO_HARNESS` (crm_send_whatsapp_message,
  crm_request_human_handoff nunca entram). `buildMcpTurnTools` (role ai_operator, scope pipelineIds).
- **`edge/egress.ts`** — `allowlistedFetch` (fail-closed, re-checa host no redirect `manual`).

### 1.14 Cron, db, health, obs, flywheel, raiz 🟢

- **`cron/scheduler.ts`** — `scheduleCronJob` (stagger), `fireOneDue` (SELECT FOR UPDATE SKIP LOCKED
  LIMIT 1 → savepoint → enqueueJob + reschedule no mesmo commit). O cron enfileira no job_queue —
  nunca reimplementa a fila.
- **`db/repository.ts`** — `insertInboxItem` (dedup via `where not exists` + `is not distinct from`),
  `InboxKind` (união grande espelhando o CHECK de agent_inbox_items), `InboxDedupe` (kind|kind_e_ref|
  kind_e_titulo|kind_ref_e_titulo).
- **`health/circuit.ts`** — circuito de saúde do número; `HEALTH_DEFAULTS` (janela 6h, block
  threshold 0.1, cooldown 1h). HOLD/UNHOLD por block_rate/response_rate.
- **`obs/logger.ts` + `metrics.ts`** — logger estruturado; `recordRunMetrics` (agrega llm_calls por
  run em `metrics`, inclui `run_cache_read_ratio`); `evaluateCacheHitAlert`; `metricsSnapshot`.
- **`flywheel/live.ts`** — `runFlywheelOnce` (juiz `flywheel_judge` sobre higiene de memória →
  destilador `flywheel_distiller` sobre veredictos 'no' → propostas com gate humano).
- **`channel-adapter.ts`** (raiz) — interface agnóstica `ChannelAdapter`; `ChannelSendResult`
  (sent|already_sent|queued|blocked|failed|unavailable). Tipos puros.
- **`surrogates.ts`** (raiz) — schemas Zod (contrato): união discriminada por `metric` (leadReplied,
  stageAdvanced, stopRequested, dropoff, timeToReply).
- **`env.ts`** — schema Zod `envSchema`, `loadEnv`. Ver seção de configuração abaixo.

### Metadados e configuração (env.ts) 🟢

Principais variáveis consumidas pelo worker (defaults entre parênteses):

- **Infra/DB:** `SUPABASE_DB_URL` (req), `NEXT_PUBLIC_SUPABASE_URL` (req), `SUPABASE_SERVICE_ROLE_KEY`
  (req), `DB_POOL_MAX`.
- **LLM:** `ANTHROPIC_API_KEY`/`OPENAI_API_KEY`/`OPENROUTER_API_KEY` (opcionais, "três irmãs" — todas
  declaradas ou o Zod remove no boot), `AGENT_DEFAULT_MODEL` ('claude-sonnet-4-5'), `LLM_CACHE_TTL`
  ('1h'), `DEEPSEEK_THINKING` ('provider').
- **Fila:** `QUEUE_MAX_CONCURRENCY` (8), `QUEUE_VISIBILITY_TIMEOUT_MS` (600000),
  `QUEUE_POLL_INTERVAL_MS` (2000), `QUEUE_CLAIM_RETRY_INTERVAL_MS` (250), `QUEUE_REAPER_INTERVAL_MS`
  (60000).
- **Orçamento IA:** `AI_BUDGET_ENFORCEMENT` (kill switch, `z.string` proposital).
- **Turno:** `AGENT_MAX_STEPS` (8), `MAX_SENDS_PER_TURN` (3), `INBOUND_DEBOUNCE_MS` (8000),
  `DISCLOSURE_MODE` ('inject').
- **Contexto/RAG:** `LEAD_CONTEXT_HISTORY_LIMIT` (20), `LEAD_CONTEXT_MAX_TOKENS` (1000),
  `RAG_TOP_K` (5), `RAG_SIMILARITY_THRESHOLD` (0.40), `RAG_MAX_TOKENS` (2000), `RAG_EMBEDDING_MODEL`
  ('text-embedding-3-small'), `LEAD_RECALL_*` (half-life 30d, mmr λ 0.7, top-k 5, bm25/vector 0.5).
- **Compactação/poda:** `COMPACTION_TRIGGER_MESSAGES` (40), `COMPACTION_TRANSCRIPT_MAX_TOKENS` (400),
  `PRUNE_TOOL_RESULTS_WINDOW_TURNS` (4), `PRUNE_TOOL_RESULTS_MIN_RESULT_TOKENS` (200).
- **Tool breaker:** `TOOL_BREAKER_EXACT_WARN` (2)/`_EXACT_BLOCK` (5)/`_SAME_TOOL_WARN` (3)/
  `_SAME_TOOL_HALT` (8)/`_NO_PROGRESS_WARN` (3)/`_NO_PROGRESS_BLOCK` (5).
- **Classificadores:** `STAGE_CLASSIFIER_MODEL`, `JAILBREAK_CLASSIFIER_MODEL`, `PROMISE_SEMANTIC_ENABLED`
  (true), `PROMISE_SEMANTIC_MODEL`, `FOLLOWUP_AI_MODEL`.
- **Health/watchdog/cron/flywheel:** `HEALTH_PORT` (8787), `NUMBER_HEALTH_INTERVAL_MS` (300000),
  `WATCHDOG_INTERVAL_MS` (60000), `CRON_TICK_INTERVAL_MS` (30000), `FLYWHEEL_INTERVAL_MS` (21600000,
  0=off), `SHUTDOWN_GRACE_MS` (30000).
- **Drains (event_log/CRM):** `CRM_DRAIN_*`, `EVENT_LOG_DRAIN_*`.
- **Egress:** `EGRESS_EXTRA_ALLOWED_HOSTS`, `AI_ALLOWLIST_TTL_DAYS` (21).

### Dependências 🟢
- **Externas:** `pg` (node-postgres), `zod`, `ai` + `@ai-sdk/anthropic|openai|google`,
  `@modelcontextprotocol/sdk`, `@supabase/supabase-js`.
- **Internas (lib/):** `lib/ai/*` (case-copy, elegibilidade), `lib/channels/*` (capabilities,
  meta/template-binding, meta/render-template), `lib/atendimento/fronteira-server`, `lib/leads/*`
  (handoff-stage-move, checkpoint-diff, agent-activity, active-lead, score-writer),
  `lib/escalacao/*` (disponibilidade, briefing-da-passagem), `lib/prospecting/context`,
  `lib/supabase/admin`, `lib/tempo/agora`, `lib/followup/*`.

### Complexidade 🟢
- `agent/inbound-turn.ts` + ritual do turno: **alta** (4352 linhas, o coração do sistema).
- Guardrails (before-send + camadas): **alta** (cadeia versionada de 11 gates, LGPD, jailbreak).
- Fila + idempotência + claim/lease: **alta** (exactly-once, advisory locks, backoff).
- Pacing/spinning/health: **média** (regras puras + seam Postgres).
- Reply/draft, cron, obs, flywheel: **média**.

### Discrepâncias/lacunas 🔴🟡
- 🔴 **`PORT-NOTES.md` desatualizado:** afirma Zod v3 e `ai ^6`, mas o código usa `z.enum(...).transform`,
  importa `MockLanguageModelV3`/`ai/test` e o AGENTS.md declara Zod 4. O runtime usa shape de usage
  ai@7 (`inputTokenDetails.cacheReadTokens`). Validar a versão exata contra `package.json`.
- 🟡 `session-watchdog.ts` (`enforceHolds`, `sessionHealthMetrics`) é o espelho F2-14 consumido por
  circuit.ts/waha-adapter.ts/metrics.ts — referenciado, não lido nesta passagem.
- 🟡 Assimetria de defaults dos flags opcionais de `GateContext`: `internalVocabularyEnforced` e
  `agenda` default NO-OP; `spinningEnforced` default ARMADO; `messagingWindow` ausente = fechado.
- 🟡 Miolo de `executarTurnoDoAgente` (~1865-3970: montagem de contexto de abertura, camada de
  jailbreak, fiação de tools MCP, corpos de execute de cada tool) lido parcialmente — o ritual,
  a guarda de send_message, o escort de orçamento e a chamada de checkpoint estão confirmados.

---

## Unidade 2 — IA de suporte: `lib/ai/` + `lib/mcp/`

### Propósito 🟢
Duas metades complementares em torno do núcleo (`lib/agent-engine/`):

- **`lib/ai/`** — a *plataforma de IA* do produto: resolução de modelo/credencial por tenant
  (Vercel AI Gateway com fallback de provider), catálogo de modelos e cálculo de custo, orçamento
  mensal por organização, pipeline de RAG (ingestão → chunking → embedding → busca top-K com
  citações), configuração declarativa de agentes (schema Zod único front+back), guardrails de
  camada de IA, handoff bot→humano, memória versionada da org, dispatcher legado (`@deprecated`) e
  agregação de uso/telemetria.
- **`lib/mcp/`** — o *servidor MCP* que expõe o CRM inteiro como tools para o modelo. Autenticação
  por Bearer `dsk_...` contra `api_tokens`, escopos convencionais, RBAC por `requiresRole`,
  auditoria com redação de PII e a semântica de "recusa para o modelo" e "UUID de aterro".

> ⚠️ `lib/ai/README.md` é um **placeholder obsoleto** (fala em `sentiment.ts`, `retrieve.ts`,
> `text-embedding-3-large`, Sonnet 4.6) — não reflete a implementação atual. Trate o código como
> fonte de verdade; o README foi ignorado nesta análise.

### 2.1 Gateway e resolução de modelo (`gateway.ts`, `gateway-binding.ts`) 🟢

**`gateway.ts`** — wrapper do Vercel AI Gateway com fallback de provider.
- Constantes: `DEFAULT_BOT_MODEL = "anthropic/claude-sonnet-5"` (`:36`), `DEFAULT_CLASSIFIER_MODEL
  = "anthropic/claude-haiku-4-5"` (`:37`), `DEFAULT_EMBEDDING_MODEL = "openai/text-embedding-3-small"`
  (`:38`), `OPENROUTER_BASE_URL` (`:23`, default `https://openrouter.ai/api/v1`).
- **`resolveLanguageModel(model): LanguageModel | null`** (`:74`) — algoritmo de fallback do mais
  específico ao mais genérico (`:75-97`): (1) gateway Vercel devolve a própria string (SDK roteia por
  `AI_GATEWAY_API_KEY`); (2) `OPENROUTER_API_KEY` → `createOpenAI` na base OpenRouter; (3) prefixo
  `anthropic/` + `ANTHROPIC_API_KEY` → `createAnthropic` com id sem prefixo; (4) prefixo `openai/` +
  `OPENAI_API_KEY`; (5) `null`.
- **`gatewayHeaders({organizationId})`** (`:108`) — **regra de privacidade**: injeta
  `X-AI-Gateway-Tenant-Id` e `X-AI-Gateway-Zero-Retention: "1"` (ZDR = opt-out de treino,
  privacy-by-default por tenant).
- `isAiGatewayConfigured()` (`:40`), `isEmbeddingProviderConfigured()` (`:100`), `gatewayConfig()`
  (`:122`).

**`gateway-binding.ts`** — ponte "painel de provedores → pilha antiga" para os pontos
`sentiment_classify`, `bot_respond` e ensaio.
- **`resolverModeloDoPonto(purpose, organizationId, padrao): Promise<ModeloResolvido | null>`**
  (`:60`). `ModeloResolvido {model, modelId, origem: "binding"|"credencial_da_organizacao"|"padrao"}`
  (`:31-36`).
- **Ordem de resolução (regra de negócio):** binding habilitado em `ai_purpose_bindings` (usa
  credencial cifrada dele) → credencial ativa+validada do provider da org → chave da instalação
  (`padraoDaInstalacao`). Provider desconhecido ou chave inutilizável **cai no padrão com
  `logger.warn`**, nunca fallback silencioso.
- `idParaOProvider(provider, id)` (`:190`) — o prefixo canônico é ROTA: OpenRouter recebe id inteiro,
  outros recebem id sem prefixo, id de outro provider → `null` (freio PR #151: não cruza chave de org
  com modelo alheio). `instanciar(provider, apiKey, modelId, baseUrl)` (`:353`) — switch
  `anthropic|openai|google|openrouter|deepseek`, default `null`.

### 2.2 Catálogo de modelos, custo, credenciais, validadores 🟢

- **`classifier-models.ts`** — `listClassifierModels(db, orgId, platformKeys)` (`:42`) filtra por
  credencial disponível (BYOK ativa+validada OU chave de plataforma via env), não pelo catálogo
  inteiro; só modelos de `ai_models` com `deprecated_at is null`; erro na consulta de credenciais
  **lança** (não degrada para lista vazia).
- **`cost.ts`** — custo em **centavos com `Math.ceil`** (nunca subfatura). Cache `_pricingCache`
  TTL `PRICING_TTL_MS = 5min` (`:23`); `loadPricing()` lê `ai_pricing where superseded_at is null`.
  `computeCost({model, promptTokens?, completionTokens?, embeddingTokens?})` (`:118`), fórmula
  `(tokens * rate)/1_000_000` por tipo. `precoDoCatalogo(modelo)` (`:87`) é fallback via `ai_models`
  (match exato preferido; devolve `null` se ambos os preços são 0 — não inventa "de graça").
- **`credentials.ts`** — `loadCredential(id, organizationId)` (`:56`); valida org, `is_active`,
  `validated_at`; decifra via `aes_gcm`; plaintext só no retorno. `CredentialUnavailableError.reason
  ∈ not_found|inactive|not_validated|wrong_org|decrypt_failed` (`:23`).
- **`provider-validators.ts`** — ping síncrono de validação, `TIMEOUT_MS = 5000` (`:39`), sem retry.
  `validateProviderKey(provider, apiKey)` (`:245`). **Regra crítica**: OpenRouter valida contra
  `/api/v1/key` (exige credencial), NÃO `/api/v1/models` (público — gravava `validated_at` em chave
  falsa). Distingue `auth_failed_401` × `provider_status_<n>` × `network_error`.
- **`pontos/provedores.ts`** — lista canônica que substituiu os CHECKs de banco (migration 0127):
  `PROVEDORES` (`:56`) = 5 provedores (`anthropic`, `openai`, `google`, `openrouter`, `deepseek`),
  cada um com `{id, rotulo, quandoUsar, aceitaEndpointProprio, catalogoSincronizavel, ...}`. Exporta
  `IDS_DE_PROVEDOR` (para `z.enum`), `PROVEDOR_POR_ID`, `ehProvedorSuportado`.

### 2.3 Orçamento e uso (`budget/check.ts`, `usage/aggregate.ts`) 🟢

- **`getBudgetStatus(orgId): Promise<BudgetStatus>`** (`budget/check.ts:170`) — **nunca lança,
  degrada para default**. Gasto vem da RPC `fn_gasto_de_ia_do_mes` (régua única, a mesma do gate);
  se a RPC falha, cai para a coluna materializada `current_month_consumed_cents` **com log**.
  `blocked_now` = existe `agent_inbox_items kind='budget_exceeded' status='open'` (lê a decisão, não
  recalcula). `gasto_incompleto` = há `llm_calls.cost_cents is null` no mês (furo de medição).
  `monthly_limit_cents: 0` = sem teto.
- **`aggregateUsage(rows, dailyInbounds, dailyHandoffs, range)`** (`usage/aggregate.ts:70`) —
  agregador puro do dashboard: totais (custo, tokens, invocações, p50/p95 latência, handoff_rate),
  séries e `by_kind`. `percentile(sorted, p)` = `ceil((p/100)*len)-1`; `handoff_rate =
  handoffs/inbounds` a 4 casas.

### 2.4 RAG — ingestão, chunking, embedding, busca, citações 🟢

- **`embeddings/chave.ts`** — modelo FIXO `MODELO_DE_EMBEDDING = "openai/text-embedding-3-small"`,
  `DIMENSOES_DO_EMBEDDING = 1536` (`:60-61`). **`resolverChaveDeEmbedding(orgId, ponto)`** (`:97`):
  escada binding → credencial OpenAI ativa+validada (desempate pela mais antiga) → gateway →
  `OPENAI_API_KEY` → `null`. O binding governa a chave, não o modelo.
- **`embed.ts`** — `embedText(content, {organizationId, ponto?, chave?, model?})` (`:64`); com
  gateway passa a string do modelo, sem gateway usa `createOpenAI(...).textEmbeddingModel`;
  **assere `embedding.length === 1536`** — dimensão divergente lança (recall quebraria em silêncio).
  `SemChaveDeEmbeddingError` code `"embedding_sem_chave"`.
- **`rag/chunker.ts`** — `chunkText(text, {maxChars=1500, overlapChars=200})` (`:33`): split por
  parágrafo (`\n\n+`) → sub-split de parágrafo grande por sentença → overlap dos últimos N chars do
  chunk anterior. `computeContentHash(content)` = SHA-256 hex (`:96`).
- **`rag/tipos-de-fonte.ts`** — `TIPOS_DE_FONTE` (`:71`, substituiu CHECK na migration 0181): `faq`,
  `documento`, `conversas`, `catalogo`, cada um com `ComoSePreenche ∈ texto_colado|arquivo_ou_texto|
  automatico`. `canonizarTipoDeFonte` traduz legados (`policy→documento`, `conversation(s)→conversas`,
  `catalog/nuvemshop_catalog→catalogo`).
- **`rag/version.ts`** — ciclo de vida de versão de índice **por fonte** (0181):
  `createKnowledgeVersion` (status `building`, `version_number = max+1`) → `markVersionReady` /
  `markVersionFailed` → `activateVersion` (**desativa a anterior ANTES**, índice único
  `ai_kbv_uma_ativa_por_fonte`, depois aponta `ai_knowledge_sources.active_kb_version_id`).
- **`rag/debounce.ts`** — `acquireDebounce(key, ttlSec)` / `releaseDebounce(key)` (SET NX EX,
  `TIMEOUT_MS = 2000`). **Falha ABERTA** (catch → `return true`): perder debounce custa uma
  indexação extra; perder o evento custaria o material inteiro.
- **`rag/format-product.ts`** — `formatProductForRag(product: NuvemshopProduct)` (`:39`): strip HTML +
  template rotulado PT-BR (Produto/Descrição/Preço/Variantes/Categorias/SKU/Link).
- **`knowledge/busca.ts`** — **`buscarConhecimento(supabase, p, deps?)`** (`:70`): embeda a pergunta
  (`ponto embedding_consultar`), RPC `fn_buscar_trechos_das_fontes` com `p_threshold = PISO = -1` e
  filtra pelo `limiar` **em memória** para expor `melhorSimilaridade` (o melhor candidato reprovado —
  distingue "não há nada" de "há algo perto mas fraco"). `resolverAcervoDoAgente` (`:129`) usa a
  versão publicada com fallback legado.
- **`citations/types.ts`** — `Citation` (com `source_type`, `score` 0..1). `extractCitations` /
  `isAiGeneratedMessage` (testa `ai_generated === true`).

### 2.5 Agentes, system prompt, guardrails, memória 🟢

- **`guardrails-schema.ts`** — **fonte Zod única (backend + frontend)**. `AGENT_MODELS` (`:15`) =
  `claude-sonnet-4-6`, `claude-haiku-4-5`, `claude-opus-4-7` (⚠️ **diverge** dos `DEFAULT_*` de
  `gateway.ts`, que usam `-5`). `guardrailKindEnum` (`:26`) = 5 kinds (Spec 05 §8.1):
  `regex_output_block`, `rag_must_hit`, `regex_input_block`, `window_check`, `contact_flag`;
  `guardrailsSchema` = array `.max(50)`. `AGENT_CONFIG_DEFAULTS` (`:161`): `temperature 0.4`,
  `max_tokens 1024`, `context_message_window 20`, `rag_top_k 5`, `rag_similarity_threshold 0.4`,
  `confidence_threshold 0.6`, `voice "marin"`, `voice_speed 0.85`, `voice_model "gpt-realtime"`.
- **`render-system-prompt.ts`** — `renderSystemPrompt(template, ctx)` (`:70`): substitui `{{...}}`
  (placeholders desconhecidos sobrevivem, para debug); vocabulário default `{lead:"cliente",
  deal:"pedido", won:"concluído", lost:"cancelado"}`; anexa bloco RAG ao fim se o template não tem
  `{{retrieved_chunks}}`.
- **`agents.ts`** — agente de **VOZ** (`ai_agents.channel='voice'`, 0347):
  `getActiveVoiceAgent(orgId)` (`:73`); a chave OpenAI abre sessão Realtime por WebSocket cru,
  **fora do Gateway**.
- **`memoria-da-org.ts`** — `publicarMemoriaDaOrg(admin, orgId, userId, content)` (`:26`): versionado
  (`org_memory_versions`, `version_number = max+1`) + ponteiro (`org_memory_pointers` upsert); nunca
  sobrescreve, distingue erro `"versao"` de `"ponteiro"`.
- **`guardrails/lista-de-conferencia.ts`** — **apresentação** da cadeia de saída (runtime real em
  `agent-engine/guardrails/before-send.ts`): `CONFERENCIAS_DE_SAIDA` (`:66`) = 10 conferências na
  ordem de execução (`stop`, `lgpd`, `pacing`, `messaging_window`, `spinning`, `promise`,
  `semantic_promise`, `case_promise`, `internal_vocabulary`, `agenda_stall`, `disclosure`). **Regra
  "9 não se desligam"**: só `semantic_promise` tem escolha (custo +1 consulta/mensagem enviada).
  `CONFERENCIA_DE_ENTRADA` = `jailbreak_detect` (+1 consulta/mensagem recebida).
- **`log-invocation.ts`** — `logInvocation(row)` (`:63`) fire-and-forget via `queueMicrotask`,
  escreve em **`llm_calls`** (0130 aposentou `ai_invocations`). `InvocationKind ∈ bot_respond|
  sentiment_classify|triage_classify|embedding_generate`. `providerDoModelo` / `codigoDoErro`
  reusam a mesma régua do motor.
- **`apply-proposal.ts`** — `applyProposal(admin, {orgId, agentId, proposalId, userId})` (`:38`),
  gate humano = clique. `org_memory_entry` → insere entry; `playbook_bullet` → copia a versão
  publicada, append de bullet datado ("## Aprendizado do flywheel"), cria versão `draft`, publica por
  ponteiro. `ApplyProposalErrorCode` (7 valores).
- **`pacing-knobs.ts`** — `pacingKnobsUpdateSchema` (`.strict()`): `daily_message_limit` em
  `DAILY_LIMIT_BOUNDS {min:1, max:10000}` (0 rejeitado), `timezone` IANA. Warm-up:
  `diaDeclaradoJaComecou` usa `MAIOR_ADIANTAMENTO_MS = 14h` (só recusa dia que não começou em lugar
  nenhum). Delega para `agent-engine/pacing/{defaults,engine,store}`.

### 2.6 Servidor MCP — núcleo (`lib/mcp/`) 🟢

- **`server.ts`** — `createMcpServer(auth, requestId)` (`:37`), `SERVER_NAME="deskcomm-crm"`.
  **Pipeline por tool:** (1) `higienizarUuidsDeAterro(args)`; (2) monta `McpContext`; (3)
  `ensureScope` + `ensureRole`; (4) `tool.handler(args, ctx)`; (5) `motivoDoVazio` → `success` da
  auditoria; (6) `auditMcpToolCall`; (7) devolve `content[]` + `structuredContent`. Usa
  `createAdminClient()` (service-role) — org sempre de `ctx`, nunca do arg.
- **`types.ts`** — `McpContext {organizationId, role, actor, apiTokenId, requestId, supabase,
  meetingBooking?}` (`:15`). `McpToolDefinition {name, description, inputSchema, category:
  read|write|handoff, requiresRole, requiresScope: mcp:read|mcp:write, motivoDoVazio?, handler}`
  (`:32`). **Regra `motivoDoVazio`** (issue #484): "não achei" ≠ sucesso — desce para
  `api_audit_log.metadata.motivo`.
- **`auth.ts`** — Bearer `dsk_<prefix>_<secret>` contra `api_tokens`.
  `validateBearerToken(authHeader)` (`:196`); `resolveApiToken(plaintext)` (`:130`) = hash SHA256 →
  lookup `token_hash` → checa `revoked_at`/`expires_at`; `last_used_at` fire-and-forget.
  **Scopes convencionais (sem migration):** `role:<r>` (default `agent`), `actor:ai_agent`,
  `agent_run:<uuid>`, `mcp:read`, `mcp:write`. `deriveActor` (`:64`) → `ai_agent` ou `api_token`,
  **nunca `user`** (evita quebrar FKs `_by_user_id` e furar o gate `pre_go_live`). `ensureRole` via
  `ROLE_RANK`; erros MCP -32001/401, -32002/403, -32603/500.
- **`audit.ts`** — `auditMcpToolCall(input)` (`:60`) fire-and-forget, `action='mcp.tool_called'`.
  Redação: `ARGS_REDACT_KEYS = {authorization, api_key, token, password, cpf}`, strings >500 chars
  truncadas; `actorUserId: null`, `resourceId: null` (nome da tool em `metadata.tool_name`).
- **`recusa-para-o-modelo.ts`** — `recusaDeCapacidadeParaOModelo(toolName)` (`:35`): traduz recusa por
  papel em texto para o modelo sem vazar "agent"/"role"/"permissão"; distingue `apenasHumano`
  (deliberado) de restrição acidental. Não afrouxa o `ensureRole`.
- **`uuid-de-aterro.ts`** — `ehUuidDeAterro(valor)` (`:96`, detecta NIL/MAX e mascarados),
  `higienizarUuidsDeAterro(shape, args)` (`:120`): **apaga a chave** (não escreve `null`) só quando o
  schema aceita o campo ausente; campo obrigatório mantém o valor para a recusa nomeada.

### 2.7 Tools MCP por domínio (nome / categoria / role) 🟢

Catálogo estático client-safe em `tools/catalogo/` (9 domínios agregados em `index.ts`); ~47
handlers em `tools/index.ts` com sanity-check 1:1 catálogo↔handler. Padrão universal:
`requiresScope = mcp:read|mcp:write`, org de `ctx`, RBAC por `requiresRole`. Ordem de papéis
(`ROLE_RANK`): `viewer < agent < ai_operator < manager < admin`.

| Domínio (arquivo) | Tools (categoria · role) | Regras-chave |
|---|---|---|
| `contacts.ts` | `crm_search_contacts` (r·agent), `crm_get_contact` (r·agent), `crm_propose_contact_field` (w·agent) | CPF nunca em plaintext; propor **não grava**, cria proposta p/ humano confirmar |
| `conversations.ts` | `crm_list_conversations`, `crm_get_conversation`, `crm_get_conversation_history` (r·agent) | enriquece `assignee_kind`, `queue_position`; histórico ≤100 |
| `leads.ts` | `crm_list_leads`, `crm_get_lead` (r·agent); `crm_create_lead`, `crm_update_lead`, `crm_move_lead_stage` (w·agent) | mover estágio só no MESMO pipeline; source default "ai_agent"; agente vira dono (0070) |
| `messages.ts` | `crm_send_whatsapp_message` (w·agent) | idempotência `idempotency_keys` TTL 24h (`deduplicated:true`) |
| `start-conversation.ts` | `crm_start_conversation_and_send` (w·**manager**, apenasHumano) | cold-start: `channel_session_id` obrigatório; idempotência cobre o par abrir+enviar |
| `governance.ts` | `crm_assign_conversation`, `crm_manage_tags` (w·agent); `crm_get_queue_status` (r·agent) | assign com optimistic lock + idempotente; tags normalizadas (≤40 chars, ≤20) |
| `escalacao.ts` | `crm_list_available_attendants`, `crm_list_human_cases`, `crm_get_human_case` (r·agent); `crm_add_case_note`, `crm_close_human_case`, `crm_resume_ai_attendance` (w·agent) | `resume_ai_attendance` **só pessoa** (`actor.type==="ai_agent"` → recusa); agente vê fila inteira |
| `handoff.ts` | `crm_request_human_handoff` (**handoff**·agent) | roteamento G5: alvo elegível → round-robin → fila; efeitos: pending, bot_silenced infinity, event_log, broadcast |
| `comercio.ts` | `crm_list_contact_orders` (r·agent), `crm_search_products` (r·agent, `motivoDoVazio`) | varredura paginada (`TAMANHO_DA_PAGINA=1000`, `PAGINAS_MAXIMAS=10`); só afirma "loja não tem" após varredura completa |
| `evolucao.ts` | `crm_search_knowledge`, `crm_list_knowledge_sources`, `crm_list_improvement_proposals`, `crm_get_org_memory` (r·agent); `crm_save_org_memory` (w·**ai_operator**) | `LIMIAR_PADRAO=0.4` (0097); IA lê propostas mas não se auto-aprova |
| `privacidade.ts` | `crm_list_privacy_requests` (r·agent) | só leitura de `lgpd_requests` (IA não anonimiza — irreversível) |
| `operacao.ts` | leituras (r·agent): stages, tags, templates, webhooks, automations, team; escritas (w·**manager**): create/update/archive stage, webhook source, toggles | archive nunca apaga (FK RESTRICT); agente não cria regra/muda papel/apaga |
| `agendamento.ts` | `crm_list_event_types`, `crm_find_free_slots`, `crm_list_appointments` (r·agent); `crm_book_appointment`, `crm_find_and_book_appointment`, `crm_reschedule/cancel/confirm`, `crm_set_appointment_outcome` (w·**ai_operator**) | "marcado ≠ confirmado" (`aguarda_confirmacao`); recusa de negócio = RESPOSTA, nunca exceção |
| `retencao.ts` | `crm_list_followups`, `crm_list_at_risk_leads` (r·agent); `crm_schedule_followup`, `crm_cancel_followup`, `crm_enroll_followup_flow` (w·**ai_operator**), `crm_close_demand`, `crm_propose_reactivation` (w·agent) | prazo relativo (`in_hours`) convertido no servidor (modelo não sabe a data); um retorno vivo por cliente |
| `_users.ts` | helper `resolveUserNames` | expõe só `full_name` (LGPD), dedupe sem N+1 |

### Estruturas de dados / entidades (destaque) 🟢
`ModeloResolvido`, `ClassifierModelOption`, `LoadedCredential`, `ProvedorSuportado`, `BudgetStatus`,
`UsagePayload`, `ChaveDeEmbedding`, `TrechoEncontrado`/`ResultadoDaBusca`, `Citation`,
`agentConfigSchema`/`guardrailItemSchema` (Zod), `McpContext`, `McpToolDefinition`, `McpAuthResult`.
Detalhes de campos no `data-dictionary.md`.

### Dependências 🟢
- **`lib/ai` →** `lib/agent-engine/*` (pacing, edge/llm: orcamento/run-model-call/providers/
  credentials, agent/human-cases, db/request-pool), `lib/crypto/aes_gcm`, `lib/supabase/admin`,
  `lib/env`, `lib/logger`, `@upstash/redis`, `@ai-sdk/{anthropic,openai,google}`, `ai`.
- **`lib/mcp` →** handlers REST `app/api/v1/*/_handler` (contacts, conversations, messages, leads,
  agenda), `lib/schemas/*`, `lib/routing/*`, `lib/escalacao/*`, `lib/followup/*`, `lib/leads/*`,
  `lib/operacao/*`, `lib/agenda/*`, `lib/catalogo/busca`, `lib/messaging/*`, `lib/ai/knowledge/busca`,
  `lib/ai/handoff/orchestrator`, `lib/audit`, `@modelcontextprotocol/sdk`.

### Complexidade 🟢
- RAG (chunker + versionamento + busca com piso/limiar): **média-alta**.
- Servidor MCP + ~47 tools com RBAC/idempotência/redação: **alta** (superfície de ataque grande).
- Gateway/binding + validadores de provider + custo/orçamento: **média**.

### Discrepâncias / lacunas 🔴🟡
- 🔴 **`lib/ai/README.md` obsoleto** — descreve arquivos e modelos que não existem mais.
- 🟡 **Dois catálogos de string de modelo divergentes:** `AGENT_MODELS` (`-4-6/-4-5/-4-7` em
  `guardrails-schema.ts`) × `DEFAULT_*` (`-5` em `gateway.ts`). Confirmar qual é canônico.
- 🟡 **`dispatcher/index.ts` é `@deprecated`** — o runtime real é `lib/agent-engine`; o dispatcher
  persiste só para orgs em modo externo (Vendaval). Não confundir com o caminho quente.
- 🟡 Furo de medição de custo documentado em `budget/check.ts` (`gasto_incompleto`): `ai_pricing` casa
  por prefixo e devolve `null` fora dele → gasto medido pode ser menor que o real.
- 🟡 Diretórios lidos só por listagem/nome (não em profundidade nesta passagem): `elegibilidade/`,
  `pontos/` (além de provedores), `replies/`, `runtime/` (parcial), `skills/`, `evolution/`,
  `anonymize/`, `catalogo/`, `prompts/`.

---

## Unidade 3 — Canais e mensageria: `lib/channels/`, `lib/waha/`, `lib/messaging/`, `lib/inbox/`, `lib/atendimento/`, `lib/notifications/`, `lib/email/`

### Propósito 🟢
A camada **agnóstica de canal** que leva mensagem entre o CRM e os provedores (WhatsApp via
WAHA/QR, Meta Cloud API oficial, Zernio/BSP) e as superfícies internas de mensageria (inbox,
fronteira de atendimento, notificações, e-mail). A lei que rege a unidade é o **invariante de
restrição de canal** (`docs/doctrine/restricao-de-canal.md`): **nenhuma feature fora de
`lib/channels/` pode nomear um provedor** — features perguntam *o que o canal permite*
(capabilities), nunca *quem ele é*. O gate é `pnpm lint:channels`.

### 3.1 Abstração de canal (`lib/channels/types.ts`, `capabilities.ts`, `index.ts`) 🟢

- **`ChannelProvider`** (`types.ts:13`) = `waha | meta_cloud | zernio | zernio_social | wacalls`;
  `ProviderDeMensagem = Exclude<..., "wacalls">` (`:31`) — `wacalls` (voz) mora em
  `channel_sessions` mas NÃO transporta mensagem; a distinção no TIPO força cada site "qual canal?"
  a decidir.
- **`ChannelCapabilities`** (`types.ts:33`): `freeformOutsideWindow`, `requiresTemplates`,
  `canManageTemplates`, `banRisk`, `minIntervalMs`, `voiceNote ("server-convert"|"opus-only")`,
  `groups ("full"|"limited"|"none")`, `costPerMessage`.
- **`ChannelAdapter`** (`types.ts:169`) — contrato do provedor: `provider`, `resolveRecipient`,
  `isConfigured`, `send(envelope): Promise<{externalId}>`, `codes`, e opcionais
  `fetchProfilePictureUrl`, `echoExternalIds`, `resolvePhoneForIdentity`, `templates`,
  `signalTyping`, `checkHealth`, `fetchInboundMedia`, `sendTemplate`. **Regra**: o adapter é
  *tradutor de formato puro* — nenhuma lógica de janela 24h / cap / horário (isso vive na cadeia
  `before_send`).
- **`OutboundEnvelope`** (`types.ts:114`) estende `ChannelTenantScope {organizationId}` (obrigatório,
  de fonte confiável, **nunca do body** — fix da issue #236). Traz `beforeSend?()` (re-valida a
  origem depois do preparo async), `sessionRef`, `to`, `kind`, `body?`, `media?`, `contact?`,
  `providerConversationId?` (thread, p/ Zernio), `replyToExternalId?` (o `wamid` citado).
- **`CHANNEL_CAPABILITIES`** (`capabilities.ts:26`) — matriz por provedor. Destaques de regra:
  - **waha**: freeform `true`, requiresTemplates `false`, `banRisk true`, groups `full`. "Falo quando
    quiser, mas o WhatsApp me bane se eu abusar."
  - **meta_cloud**: freeform `false`, requiresTemplates `true`, `minIntervalMs 6000`, `banRisk false`,
    voiceNote `opus-only`, `costPerMessage true`. **Regra**: enviar template NÃO reabre a janela 24h
    (só a resposta do cliente reabre); a API devolve 200+wamid e a Meta recusa a entrega por webhook
    **erro 131047**.
  - **zernio** e **zernio_social** com perfis próprios (`social` = freeform false, sem templates,
    groups none).
  - `DEFAULT_CHANNEL_PROVIDER = "waha"` (escolha conservadora — banRisk armado).
- **`capabilitiesOf(provider)`** (`capabilities.ts:189`) — **fail-closed, lança
  `unknown_channel_provider`**. `transportaMensagem(provider)` (`:151`) fail-closed. Exaustividade
  garantida em tempo de compilação por `ProviderNaoClassificado extends never` (`:178`).
- `index.ts` — registro `ADAPTERS` + `getAdapter(provider)` fail-closed.

### 3.2 Efeitos pós-entrada (`pos-entrada.ts`) — a ORDEM é a regra 🟢

`aplicarEfeitosPosEntrada(admin, entrada)` (`pos-entrada.ts:150`). A sequência é lei (guardada por
testes): guarda "número interno" → (1) **opt-out** (`aplicarOptOut` grava `is_blocked=true,
blocked_reason='stop_keyword'`) → (4) `guardarOrigemDaPagina` (antes da demanda, para a ficha copiar
a origem) → (2) `abrirDemanda` (`garantirLeadDaConversa`, idempotente, recusa contato bloqueado —
por isso opt-out é o passo 1) → (2b) `avaliarCampanha` (autoriza contato para IA quando
`metadata.ai_gate === "allowlist"`) → `acelerarPipelineDeEventos` → (3) `pedirDespachoDoAgente`
(emite `ai_agent.dispatch_requested`). **Regra**: nenhum efeito pode lançar (a mensagem já está
persistida; um 500 dispararia tempestade de reentrega do provedor) — opt-out falha loga `error`, o
resto `warn`. Antes esses efeitos eram inline em `lib/waha/ingest.ts` e só rodavam no WAHA (medido:
806 despachos no QR, 0 no oficial) — movidos para trás do seam.

### 3.3 Janela 24h, saúde, estado, canal mudo 🟢

- **`janela.ts`** — `estadoDaJanela(provider, lastInboundAt, agora): EstadoDaJanela`
  (`sem_restricao | aberta{restanteMs} | fechada{fechadaHaMs}`, `:58`). Deriva a cada leitura (sem
  coluna de expiração — evita segunda verdade rançosa). `LIMIAR_URGENTE_MS = 2h`.
- **`health.ts`** — `STATUS_SAUDAVEL = "WORKING"`; `STATUS_QUE_AVISAM = [SCAN_QR_CODE, FAILED,
  STOPPED]` (STARTING excluído = boot normal). `avisoDaConexao(saude, apelido)` (`:112`) mapeia
  observação → alerta (`warn`/`critical`, kinds `qr_rescan`/`channel_number_alert`).
  `sincronizarSaudeDaConexao(...)` (`:255`) dedup por episódio via
  `channel_session_health.escalated_status`; **regra**: só quem observou pode fechar (episódio aberto
  por push não é fechado pela varredura).
- **`estado.ts`** — `ESTADOS_DO_CANAL = [STARTING, SCAN_QR_CODE, WORKING, STOPPED, FAILED]`; só
  `WORKING` é `utilizavel:true`. `nomeDoCanal` mostra display_name→phone→"Número sem nome", nunca o
  nome interno da sessão.
- **`canal-mudo.ts`** (canal sem número, doc 11 dec. B) — `DIAS_ATE_AVISAR = 3`;
  `avaliarCanal(canal, agora)` → `{acao: avisar|resolver|aguardar}`; modo de teste =
  `metadata.ai_gate === "allowlist"`.

### 3.4 Conexão, nome de sessão, pairing, inbound seam 🟢

- **`nome-da-sessao.ts`** — `TETO_NOME_DE_SESSAO_WAHA = 54`, padrão `/^[a-zA-Z0-9_-]+$/`;
  `nomeDaSessaoNovo = org_<8>_<32>` (migration 0232). `podeRenomearSessaoDoWaha` só quando
  `phone_number is null && status !== 'WORKING'` (renomear sessão pareada órfã o diretório de
  credencial WAHA).
- **`pairing-code.ts`** — `requestChannelPairingCode(...)` (`:29`) rate-limited
  (`checkRateLimit('pairing-code:...', 1, 30)`), exige status remoto `SCAN_QR_CODE` (rejeita
  `WORKING`); nunca vaza telefone/código/credencial.
- **`inbound.ts`** — `handleInboundWebhook(admin, input)` (`:99`) para Zernio/social; **a assinatura
  é verificada AQUI, não na rota** (o esquema é específico por canal): `verifyZernioSignature` em
  `x-zernio-signature`, fail-closed (`MIN_SECRET_LEN = 16`). Erros: `unauthorized |
  provider_mismatch | invalid_json | contrato_violado`.

### 3.5 Adaptadores (`adapters/waha.ts`, `meta-cloud.ts`, `zernio.ts`) 🟢

- **waha.ts** — delega inteiro para `lib/waha/*`. `isConfigured` = `getWahaClient()!==null`;
  `checkHealth` (404→STOPPED, 401/403→credencial recusada); `send` em 3 caminhos (vcard / mídia /
  texto com `replyTo`). Erro carrega STATUS via `statusHttpDoErroWaha` (`^waha_..._(\d{3})`).
- **meta-cloud.ts** — sessionRef = phone_number_id (vai na URL). `isConfigured()` **sempre true**
  (issue #674: credencial em `channel_sessions.meta_token_encrypted`, migration 0118; `send` lança
  `meta_not_configured` sem creds). `fetchInboundMedia` com **allowlist de host**
  (`.fbsbx.com`/`.fbcdn.net`, só `https:`) — guarda SSRF. Áudio = `voice:true` (opus-only).
- **zernio.ts** — endereça por `providerConversationId` (thread do webhook), não derivado do contato;
  `send` lança `zernio_no_conversation` sem thread. `sendTemplate` abre conversa (reabre janela 24h),
  params numerados por chave `{{n}}`. SSRF guardado por `assertSafeOutboundUrl`.

### 3.6 WAHA client + ingest (`lib/waha/`) — o coração da entrada 🟢

- **`webhook-auth.ts`** — `authenticateWahaWebhook(input)` (`:64`). `MIN_SECRET_LEN = 16`. Três
  regras EM ORDEM: (1) assinatura presente e errada → rejeita `bad_signature` sempre; (2) exigida
  (`WAHA_WEBHOOK_REQUIRE_SIGNATURE === "true"`) e ausente → `signature_required`; (3) ausente e não
  exigida → `{ok, signatureVerified:false}` e o chamador loga essa verdade. **Era fail-OPEN** (buraco
  de forja provado com curl) — agora fail-closed. O default `false` existe porque o WAHA Core não
  assina; a defesa de rede (Caddy não publicando a rota global) é a camada que não depende disso.
- **`ingest.ts`** — pipeline "mensagem → conversa+mensagem". `dispatchWahaEvent(...)` roteia
  `message`/`message.any` → `handleInbound`/`handleOutboundFromUserPhone` (por `fromMe`);
  `message.ack`→`handleAck`; `message.edited`/`message.revoked`; `session.status`→`handleSessionStatus`.
  - `parseChatId(chatId): ChatIdentity` — `@g.us`→group, `@lid`→lid, `@c.us`/`@s.whatsapp.net`→phone,
    else→unknown (deixa trace, não silêncio).
  - **Guarda ReDoS** `semSufixoDeChat(chatId)` — substitui `replace(/@.*$/,"")` (CodeQL
    js/polynomial-redos). Usa `lastIndexOf` nos 4 terminadores de linha + `indexOf("@", ...)`. O regex
    guardado era O(n²) sobre `payload.from` controlado por atacante; linear agora (1MB em 0.7ms).
    Coberto por `ingest-redos.test.ts`.
  - **Timestamp tolerante** `dataDoTimestamp(ts, agora)` — infere unidade pela magnitude
    (≥1e16 ns, ≥1e11 ms, else s); nunca lança `RangeError` (derrubava o webhook). Coberto por
    `timestamp-tolerante.test.ts`.
  - Upserts atômicos via RPC `fn_upsert_wa_contact` / `fn_upsert_wa_conversation` (fecham a corrida
    check-then-act do NOWEB que emite `message` E `message.any`). Identidade canônica: migration 0027
    (`wa_identity`), 0122 (`wa_lid`).
  - `handleInbound` — INSERT message (`sent_via:"external_device"`, status `delivered`); idempotência
    `23505` sobre `unique(organization_id, external_id)`; `markConversation`; `audit("message.received")`;
    `aplicarEfeitosPosEntrada`. **Regra**: NÃO emite `message.received` — o trigger
    `trg_messages_emit_event` emite (era emissão dupla, medido 805 msgs com 2 eventos).
  - `handleOutboundFromUserPhone` (operador respondeu do celular) — `ehEcoDeEnvioNosso` (`:76`)
    prova eco vs digitação humana: nossa linha outbound, mesma conversa, sem `external_id`, mesmo
    corpo, dentro de `JANELA_DO_ECO_MS = 60_000`. Se não é eco → `pausarIaPorAtendimentoManual`.
    **Regra**: a decisão de gravar é *tolerante* (dupe > mensagem perdida), a de silenciar o bot é
    *estrita* (não silencia na dúvida) — direções opostas de propósito.
  - `verifyHmacSha512(rawBody, signatureHeader, secret)` — SHA-512 hex + `timingSafeEqual`.
- **`envelope.ts`** — `wahaPayloadSchema = z.looseObject({...})` (`:70`), todos os campos `.nullish()`.
  **Regra**: loose em tudo — um `.strict()` transformaria "mensagem entra no CRM" em "mensagem
  descartada". Duas etapas: `lerRoteamentoWaha` (mínimo p/ resolver tenant + arquivar bruto) e
  `conferirContratoWaha` (contrato cheio).
- **`client.ts`** — `WahaClient`. `TETO_PADRAO_MS = 15_000`, `TETO_DE_MIDIA_MS = 30_000`.
  `CONVERSAS_IGNORADAS = {status, broadcast, channels, groups}` empurrado ao `config.ignore` da
  sessão (medido: 376 de 395 MB descartado). **Regra**: `WahaSessionError` carrega STATUS, nunca o
  BODY do WAHA (pode conter PII — banido no `lib/logger.ts`). `startSession` faz GET-before-PUT
  (`convergirConfigDaSessao`) porque o PUT do WAHA substitui a config INTEIRA (inclui webhooks).
  Engine NOWEB. `getWahaClient()` = null quando não configurado ou placeholder
  `dev_plaintext_change_me`.
- **`message-id.ts`** — `parseWahaMessageId`, `bareWaMessageId` (slice após último `_`, ack NOWEB vem
  `{fromMe}_{chatId}_{bareId}`), `wahaEchoExternalIds`, `chatIdFromWaMessageId`.
- **`send.ts`** — `resolveWahaChatId(input)` (`:60`): **a ordem é a regra** — group→groupChatId;
  `waLid`→`{lid}@lid`; `waIdentity "lid:"`→`@lid`; phone→`{digits}@c.us`. `@lid` ANTES de phone
  (0122) — trocar o canal de uma conversa viva é o pior defeito.

### 3.7 Mensageria compartilhada, inbox, fronteira 🟢

- **`lib/messaging/`** — `open-shared-contact-conversation.ts`, `contact-card.ts`, `presenca.ts`,
  `media/` (deriva/transcodifica/valida/transcreve mídia; `fetchWahaMedia` com timeout 30s e base
  fixa contra SSRF). 🟡 Lido por referência cruzada, não linha a linha.
- **`lib/inbox/comando-da-conversa.ts`** — **máquina de estados do comando da conversa**.
  `Comando = humano | automatico | ninguem | aguardando | encerrada`. `INFINITO = "infinity"`,
  `MENOS_INFINITO = "-infinity"`, `STATUS_ENCERRADOS = {closed, archived, resolved}`.
  `silencioVigente(valor, agora)` — valor ilegível = **tratado como SILENCIADO** (fail-closed na
  ação). `comandoDaConversa(fatos, agora)`: atribuído→humano; encerrada→encerrada;
  (silêncio|force_human|blocked)→aguardando; `automaticoDaOrg===false`→ninguem; else automatico.
  Precedência do motivo: bloqueado (`contato_descadastrado`) ANTES de `contato_travado` ANTES do
  silêncio da conversa. **Dois gates DELIBERADAMENTE fora**: janela 24h (é capability) e status
  fechado. `ORDEM_DA_ESPERA` ordena por `awaiting_since` (o inbound mais antigo sem resposta, não
  `last_inbound_at` que reseta a cada mensagem — issue #990/#994).
- **`lib/atendimento/` (fronteira)** — a **fronteira IA↔humano**. `ServiceBoundary {organization_id,
  contact_id, conversation_id, service_revision, demanda_id, demanda_revision}`.
  `assertCurrentServiceBoundary(expected, current)` lança `StaleServiceBoundaryError` em qualquer
  divergência (org/contato/conversa/`service_revision`, status terminal, `demanda_fechada_em`,
  `demandaTrocou`). `demandaTrocou` retorna false quando `expected.demanda_id===null` (abrir a
  primeira demanda não é novo serviço — 0222 só bump `service_revision` ao TROCAR de demanda).
  `fronteira-server.ts` usa `AsyncLocalStorage` para o contexto de execução e `guardServiceTools`
  (tools read-only bypassam; as demais recebem `guardServiceEffect` antes do `execute`).

### 3.8 Notificações e e-mail 🟢

- **`lib/notifications/`** — `NOTIFY_KINDS` (message_inbound, lead_assigned/won/lost, mention,
  call_inbound, alerts_toggle). `emit.ts`: `BODY_MAX = 140`, checa permissão + alerts habilitados
  (salvo `force`), toca som se `visibilityState==="visible"`. `deliver.ts`: `entregarAviso` traduz e
  entrega por `in_app` (toast) e/ou `push` conforme `canalLigado(category, channel)`. `prefs.ts`:
  `NotifyPrefs`, localStorage `"notify.prefs.v1"`, external store via `useSyncExternalStore`.
  `mentions.ts`: token `/@[\p{L}\p{N}._-]+/gu`. `web_push.ts`: `enviarPushDaOrg(...)` — **404/410
  deleta a subscription morta**; VAPID via `vapid.ts` (`VAPID_PUBLIC_KEY`/`VAPID_PRIVATE_KEY`).
- **`lib/email/roteador.ts`** — decisão SMTP↔Resend por **CONFIGURAÇÃO, nunca por falha** (fallback
  em falha faria double-send em greylisting 4xx e esconderia erro). `EmailDeliveryError` NÃO é
  achatado para `send_failed` (distingue `dominio_nao_verificado`, `sender_rejected`,
  `rate_limited`...). `transporteDeEmail()` = SMTP se `isSmtpConfigured` senão Resend. `config.ts`:
  `getSmtpConfig()` prefere `platform_smtp_settings` (id=1, senha cifrada) sobre `env.SMTP_*`.

### Dependências 🟢
- `lib/waha/ingest.ts` → `lib/channels/{health,marcar-conversa,pos-entrada,phone-variants}`,
  `lib/escalacao/{atendimento-manual,numero-interno-de-aviso}`, `lib/leads/atribuicao-de-anuncio`,
  `lib/plataformas-de-anuncio/google/atribuicao`, `lib/audit`, `lib/types/messaging`.
- `lib/channels/adapters/*` → `lib/waha/*`, `lib/channels/{meta,zernio}/*`,
  `lib/messaging/media/waha-source`, `lib/automation/{outbound-ip,outbound-url}` (SSRF).
- `lib/channels/pos-entrada.ts` → `lib/leads/*`, `lib/opt-out/deteccao`,
  `lib/ai/elegibilidade/{autorizacao,campanha}`.
- `lib/channels/janela.ts` → `lib/agent-engine/guardrails/messaging-window`.
- `lib/atendimento/fronteira-server.ts` → `lib/ai/{agents/operation,replies/delivery}`,
  `lib/agenda/*`, `lib/agent-engine/queue/*`.

### Complexidade 🟢
- Ingest WAHA (parse de identidade, ReDoS/timestamp guards, upsert atômico, eco vs humano): **alta**.
- Abstração de canal + matriz de capabilities + 3 adaptadores: **alta** (superfície multi-provedor).
- Máquina de comando da conversa + fronteira de serviço (CAS/revision): **alta**.
- Notificações/web-push, e-mail (roteador): **média**.

### Discrepâncias / lacunas 🔴🟡
- 🟡 Subdiretórios lidos por referência/nome, não linha a linha: `lib/channels/{meta,social,zernio}/`,
  `lib/messaging/media/*`, e vários arquivos menores de `inbox/`, `atendimento/`, `notifications/`,
  `email/templates/`.
- 🟡 O default `WAHA_WEBHOOK_REQUIRE_SIGNATURE=false` é uma escolha de compatibilidade (WAHA Core não
  assina) que depende de defesa de rede (Caddy). Confirmar a configuração de rede do self-host.
- 🟡 `meta_cloud`/`zernio` reportam `isConfigured()===true` sempre (credencial na sessão, não em env)
  — o `send` é quem lança `*_not_configured`. Divergência deliberada do `main` documentada no código.

---

## Unidade 4 — Voz / Telefonia: `lib/voice/`, `lib/voip/`, `lib/wacalls/`

### Propósito 🟢
O produto oferece **dois canais de voz independentes**, que gravam na **mesma tabela `voice_calls`**
(unificada na migration 0348, discriminada pela coluna `provider`), mas usam pilhas técnicas
completamente diferentes:

1. **`wacalls` — chamada de voz por WhatsApp** (`lib/wacalls/` + `lib/voice/`). Um segundo aparelho
   é vinculado ao número de WhatsApp da organização via um serviço externo (**WaCalls**, binário Go
   com `whatsmeow`). O áudio trafega por **WebRTC** direto entre o navegador do atendente e o serviço
   WaCalls (o CRM só faz relay do SDP). É atendimento humano; a IA não fala.
2. **`sip` — telefonia PSTN/SIP via Asterisk** (`lib/voip/` + `workers/voice-agent/`). Chamadas
   entram/saem por um **trunk SIP** por organização; o áudio vai por **AudioSocket (TCP puro)** entre
   o Asterisk e um worker Node persistente que faz a ponte com a **OpenAI Realtime** (agente de voz
   por IA). É atendimento por IA.

O eixo transversal é `lib/voice/` (opt-in de duas perguntas, guarda de rota, número discável,
desparear) que serve **apenas o canal WaCalls**. O `lib/voip/` serve **apenas o canal SIP**.

> 🟡 **INFERIDO:** o canal WaCalls (`provider='wacalls'`) é sempre atendimento humano (WebRTC no
> navegador); o canal SIP (`provider='sip'`) é sempre atendimento por IA (worker + OpenAI Realtime).
> A separação é clara no código, mas a rota WebRTC humano→navegador do lado SIP "não está
> implementada neste esqueleto" (comentário em `calls/route.ts`), então o SIP hoje é IA-only na
> prática.

---

### 4.1 Opt-in de voz WhatsApp: a regra das duas perguntas (`voice/opt-in.ts`) 🟢

Função pura, testável sem banco e sem env. A chamada de voz WhatsApp existe **apenas** onde a
instalação a oferece **E** a organização a pediu — dois eixos distintos, combinados por `&&` (não a
precedência `??` usada nos guardrails da org):

```
chamadaDeVozLigada(escolhaDaOrg, instalacaoOferece) = instalacaoOferece && (escolhaDaOrg ?? false)
```

- **`instalacaoOferece`** vem da presença de `WACALLS_API_BASE_URL` (capacidade — decisão de quem
  administra a VPS, vale para todas as orgs do processo). `instalacaoOfereceVoz(baseUrl)` faz
  `(baseUrl ?? "").trim() !== ""` (o `.trim()` impede que espaço sobrando num `.env` ligue a feature).
- **`escolhaDaOrg: boolean | null`** vem de `org_voice_calls.enabled` (consentimento — decisão do
  admin da org). **`null` = nunca escolheu = desligado** (ausência de linha é "off"; ver migration
  0234, contraste deliberado com a 0142).

**`estadoDaVoz`** devolve três respostas, não uma: `{ ligada, instalacaoOferece, escolhaDaOrg, motivo }`
onde `motivo ∈ {"ligada", "instalacao_nao_oferece", "organizacao_nao_ligou"}`. A ordem importa: sem
serviço na instalação, a escolha da org é irrelevante e o motivo é `instalacao_nao_oferece` (senão a
tela mandaria o admin clicar um botão inútil).

**Regra de negócio 🟢:** inverter `&&` para `??` entregaria a capacidade a **todas** as organizações
da instalação de uma vez — exatamente o que o dono do produto proibiu.

### 4.2 Guarda de rota de voz (`voice/guarda.ts`) 🟢

Porta única que **toda rota que USA voz WhatsApp atravessa** (parear, iniciar/aceitar/rejeitar/
encerrar, trocar SDP, ler histórico, ponte de eventos). Uma função, não um `if` por rota — repetição
de gate é como um se perde.

- **`lerEscolhaDaOrg`** lê `org_voice_calls.enabled` + `risco_aceito_em`; `throw` em erro de leitura
  (distingue "não escolheu" de "não consegui perguntar" no log).
- **`exigirVozLigada`** devolve `null` (pode seguir) ou uma `Response`. **Três códigos distintos**,
  porque pedem ações diferentes:
  - `422 voice_desligada_na_organizacao` — a org não ligou (admin resolve na tela).
  - `503 voice_indisponivel_na_instalacao` — instalação não oferece (só quem administra a VPS resolve).
  - `503 voice_estado_indeterminado` — a leitura não voltou (falha **fechada na ação, com código honesto**).
- **`instalacaoOferece` é parâmetro opcional** para não haver duas fontes do mesmo fato: as rotas já
  resolveram `getWacallsClient()` antes; reler o env aqui é como duas respostas divergem em teste.
- **Falha fechada** (ao contrário de `lerCamadasDaOrg` nos guardrails, que falha aberta): aqui o custo
  de errar é vincular um aparelho a um número **sem consentimento confirmado**.

**As rotas de DESLIGAR não passam por esta guarda** de propósito (`DELETE /voice/sessions`, `PUT
/voice/opt-in enabled:false`): exigir a feature ligada para desligá-la prenderia o aparelho vinculado
se a flag caísse. "A porta de saída nunca depende do interruptor."

### 4.3 Cliente REST do WaCalls (`wacalls/client.ts`) 🟢

Classe `WacallsClient(baseUrl, apiToken)` — fala com o upstream **autenticado** (spec
`docs/specs/18-spec-voice-calls-wacalls.md §4.1`). Todo request leva `Authorization: Bearer <token>`;
`/healthz` é a única rota aberta e não passa por aqui. Sem `WACALLS_API_TOKEN` **o cliente não
funciona** (o upstream só aceita Bearer OU cookie de login, e um processo server-to-server não tem
cookie).

**Métodos (mapeamento 1:1 com o upstream):**
| Método TS | Rota upstream | Nota |
|---|---|---|
| `createSession(name)` | `POST /api/sessions` | **já inicia o pareamento por dentro** (`Manager.Create → startPairing`); o QR sai na `/api/events` antes da resposta HTTP |
| `listSessions()` | `GET /api/sessions` | |
| `deleteSession(id)` | `DELETE /api/sessions/{sid}` | sozinho não desparia (ver 4.4) |
| `logoutSession(id)` | `POST /api/sessions/{sid}/logout` | derruba o vínculo com o WhatsApp |
| `startCall(sid, clientId, phone)` | `POST /api/sessions/{sid}/calls` | `X-Client-Id` = dono (sempre `user.id`); `record` NUNCA `true` (LGPD, spec §1.2) |
| `exchangeWebrtc(sid, callId, sdpOffer)` | `POST .../calls/{id}/webrtc` | relay puro do SDP; converte `sdp_answer`→`sdpAnswer` |
| `acceptCall / rejectCall / endCall` | `POST .../accept`, `.../reject`, `DELETE .../{id}` | |
| `history(sid, {limit, cursor})` | `GET .../history` | envelope `{calls, nextCursor}` (keyset); versão antiga lia `{rows}` |

**Regra crítica 🟢 — nunca chamar `/pair` separado.** Descoberta medida em produção
(`wacalls/client.ts` cabeçalho, `wacallsSemConexao`): `POST /api/sessions/{sid}/pair`
(`replaceClient` no upstream) troca o cliente `whatsmeow` mas **não refaz `s.calls`** → quem pareia
pelo QR é o cliente novo, quem disca é o velho (morto), e `startCall` responde
`500 "websocket not connected"` por horas. Por isso o repo NUNCA chama `/pair`: `POST /api/sessions`
já pareia, e re-parear passa por **apagar e criar** a sessão. `wacallsSemConexao(err)` detecta a
string; `wacallsFriendlyError(err)` traduz `wacalls_401/403`, `operator already on a call`,
`not paired`, `no session <id>`/`no such session` para português.

**`getWacallsClient(): WacallsClient | null`** — exige as duas chaves. Sem `WACALLS_API_BASE_URL`
loga `debug` e devolve `null` (indisponível); com URL mas sem `WACALLS_API_TOKEN` loga `warn` (config
pela metade, intenção declarada) e devolve `null`.

### 4.4 Desparear de verdade (`voice/desparear.ts`) 🟢

O ato que faltava para "desligar" não ser um rótulo. `logoutSession` e `deleteSession` existiam desde
o dia 1 e **nenhuma rota os chamava** — só havia o botão "parear".

**Ordem obrigatória: `logout` → `delete` → banco** (mesmo molde de
`app/api/v1/channel-sessions/[id]/route.ts`). Invertido, a conta some antes do vínculo cair e fica um
aparelho vinculado que ninguém consegue endereçar.

- **Banco só muda depois que o WaCalls confirmou.** Se o WaCalls falhar, a função **lança** e a linha
  NÃO é arquivada (arquivar assim mesmo faria a tela mentir "desconectado" com o aparelho vinculado).
- **Exceção `wacalls_404`** (`tolerarSessaoInexistente`): o WaCalls não conhece a sessão (volume
  perdido, reinstalação, `Restore` descartando no boot). Sem sessão lá não há aparelho vinculado; sem
  a tolerância a org ficava presa (pareamento responde 409, desparear respondia 502 para sempre).
- **Arquiva (`archived_at`), não apaga:** `wacalls/session.ts` e a rota de status filtram
  `.is("archived_at", null)`; a linha sobrevive como âncora das FKs do histórico. Zera
  `wacalls_session_id` e `wacalls_paired_at`.
- **Idempotente:** desparear o nada (`!linha`) devolve `{ desapareado: false }` — sucesso, não erro.

### 4.5 Número discável — a grafia certa do celular (`voice/numero-discavel.ts`) 🟢

**Algoritmo não-trivial (medido na VPS 2026-09-15).** O WhatsApp registra muito celular brasileiro
**sem o nono dígito** (`553198966398`), mas o CRM guarda **com** (`+5531998966398`,
`lib/channels/phone-variants.ts`). Discar o número do cadastro cru monta um destino que não existe →
painel em "Chamando…" e ligação expirada sem tocar.

`resolverNumeroDiscavel` pergunta ao **diretório do canal de mensagens** qual grafia existe:
1. Busca sessões `channel_sessions` `status='WORKING'`, não arquivadas, `provider ∈ PROVIDERS_DE_MENSAGEM`, `limit 10`.
2. Sem sessão → devolve o número do cadastro (`fonte: "cadastro"`).
3. Para cada sessão, se o adapter tem `resolveRegisteredPhone` e `isConfigured()`, pergunta;
   primeira resposta não-nula vence (`fonte: "whatsapp"`).
4. **Prazo `PRAZO_DA_CONSULTA_MS = 4000`** via `Promise.race`: cada consulta tem teto de 15s no
   cliente e são até duas grafias em série; sem o race a rota passava dos 30s (`MUTATION_TIMEOUT_MS`)
   e a ligação saía ~31s depois, uma por clique. A consulta que estoura segue em background e é
   descartada (é leitura, sem efeito a desfazer).
5. **Falha ABERTA:** sem sessão, adapter que não sabe, identidade opaca, consulta que falha/estoura →
   disca o número do cadastro (comportamento anterior). Recusar trocaria "às vezes não toca" por
   "nunca liga".

### 4.6 Nome da sessão WaCalls (`wacalls/nome-da-sessao.ts`) 🟢

`nomeDaSessaoDeVoz(orgId) = "org_" + organizationId` — **uuid inteiro**, não os 8 primeiros chars.
É a chave pela qual o relay do pareamento reconhece a própria sessão **antes de o banco saber o id**
(o QR sai na `/api/events` antes da resposta HTTP com o id; o broker emite `session-list`/`session-qr`
com `name` no instante certo). Dois tenants com o mesmo prefixo de 32 bits receberiam o QR um do
outro. Sessões antigas (`org_XXXXXXXX`) seguem reconhecidas pelo id gravado, nunca pelo nome.

### 4.7 Resolver sessão WaCalls (`wacalls/session.ts`) e chamada+dono (`wacalls/calls.ts`) 🟢

- **`resolveWacallsSession(supabase, orgId)`** — helper das rotas `voice/*`. Lê a sessão pareada
  (`provider='wacalls'`, não arquivada). **Nunca aceita id vindo do body**; sempre relido do banco,
  escopado pela org da sessão autenticada.
- **`resolveVoiceCall(supabase, orgId, voiceCallId)`** — resolve `voice_calls` + a sessão WaCalls dona
  num round-trip (join `channel_sessions!inner`). O `id` do path é sempre o **nosso** uuid, nunca o
  `wacalls_call_id` upstream (esse não vaza pro frontend).
- **`podeEncerrar(call, userId)` — regra de autorização 🟢:** "só quem está na linha desliga".
  - Se há `ownerUserId` (alguém na linha) → só essa pessoa.
  - Se ninguém assumiu e a chamada saiu do CRM (`createdBy`) → quem discou.
  - Nenhum dos dois → não há áudio a cortar; o caminho é `/reject`, não desligar.
  - **Bug histórico corrigido:** escopar só por organização deixava qualquer `agent` derrubar a
    ligação de qualquer colega num número compartilhado.

### 4.8 Conversão de áudio WebRTC (`wacalls/pcm.ts`) 🟢

Ponte de áudio do navegador (DataChannel WebRTC):
- `float32ToInt16LE(Float32Array): ArrayBuffer` — microfone → PCM 16-bit LE. Clamp `[-1,1]`, `NaN→0`,
  assimetria de escala (`*32768` para negativos, `*32767` para positivos).
- `int16LEToFloat32(ArrayBuffer): Float32Array` — PCM recebido → playback (`/32768`).

### 4.9 Ponte de eventos WaCalls → Postgres (`wacalls/events-bridge.ts`) 🟢 — o coração do canal

Processo do **worker** (não trigger — "trigger nunca faz HTTP" é o inverso: HTTP alimentando o banco).
Mantém **1 conexão SSE persistente** contra `${WACALLS_API_BASE_URL}/api/events`, com reconexão por
**backoff exponencial** (1s → teto `maxBackoffMs`, reset a cada conexão boa). Spec §4.2/§6.

**`runVoiceCallsBridgeLoop(pool, cfg, log, signal)`** — loop de vida; sai só no abort (shutdown).
**`pumpSse`** — lê a stream linha a linha, filtra `data:`, chama `despacharEventoWacalls` por evento
(erro em um evento não derruba a stream). **`despacharEventoWacalls`** (exportada para os invariantes)
faz `JSON.parse` e despacha por `type`:

| Evento SSE | Handler | Efeito |
|---|---|---|
| `call-list` | (inline) | **snapshot na reconexão** — única fonte com `direction` declarado; re-processa cada chamada ativa pelo upsert de `call-status`; pula `status='ended'` |
| `incoming` | (inline) | insere `voice_calls` `direction='inbound' status='ringing'`; defesa para `call-status` perdido; corrige sentido inferido errado |
| `auth-state` | `handleAuthState` | pareamento/desvínculo |
| `call-status` | `handleCallStatus` | upsert canônico das duas direções |
| `call-ended` | `handleCallEnded` | fecha a chamada + efeitos |
| `session-list` / `session-qr` / `incoming-claimed` / `call-quality` | — | pura UI, sem escrita |

**`handleAuthState` — algoritmo de guarda de escrita 🟢:**
- `paired:false` **só com `state='logged_out'`** desvincula (zera `wacalls_paired_at`, `status='STOPPED'`);
  guarda `wacalls_paired_at is not null` torna o QR vencido de sessão nunca pareada um no-op. Durante o
  pareamento o upstream emite `paired:false state:'qr'` a cada código — isso não desfaz nada.
- `paired:true`: a guarda que funciona pergunta **o que muda** — `antes is null` (primeiro pareamento)
  ou `status ≠ 'WORKING'` (voltou ao ar). Heartbeat de sessão já WORKING casa **zero linhas**. (A guarda
  antiga `is distinct from now()` reescrevia a cada minuto e logava "pareada" repetidamente.)

**`handleCallStatus` — inferência de sentido 🟢 (a lógica mais delicada):**
- **O sentido NÃO vem no evento `call-status`** (o broker manda `type, sessionId, id, owner, status,
  peer, startedAt, peerName, peerPhotoUrl` — sem `direction`). Só `call-list` (snapshot) e a REST
  carregam `direction`.
- Inferência: `declarada = direction ∈ {inbound,outbound}`; se declarada usa; senão `dono ? 'outbound' : 'inbound'`
  (quem disca pelo CRM nasce com `owner` via `X-Client-Id`; recebida toca sem dono).
  - **Limite conhecido:** ligação feita FORA do CRM (tela web do WaCalls) nasce sem dono → lida como
    recebida até o snapshot corrigir.
- **INSERT vs conflito:** no INSERT o sentido inferido vale; no `on conflict ... do update` só o
  **declarado** (`$9`) reescreve `direction` (`case when $9 then excluded.direction else voice_calls.direction`)
  — assim `connected` com dono numa recebida não a vira; a reconexão via snapshot corrige o inferido.
- **`contact_id` por DUAS grafias** (`CONTATO_DO_PEER`): peer vem em dígitos puros (`5511...`),
  `contacts.phone_number` é E.164 com `+`; casa `phone_number = any(phoneLookupVariants) OR wa_lid = peer`,
  `is_merged_into is null`, desempate `order by (phone_number = '+' || peer) desc`.
- **`answered_at`** carimbado no INSERT **e** no conflito quando `status='connected'` (ligação de saída
  já nasce conectando; a ponte pode reconectar e ver `connected` primeiro; sem isso nunca contaria como
  atendida). `owner_user_id`/`contact_id` usam `coalesce` (evento posterior sem owner não apaga).
- **Efeito colateral:** `status='connected'` + contato → `calarIaDuranteALigacao`.

**`handleCallEnded` — fechamento e efeitos 🟢:**
1. `update voice_calls status='ended'`, `end_reason`, `ended_at`, `owner_user_id = coalesce(...)`,
   `duration_ms = answered_at ? endedAt - answered_at : null`.
2. Se tem contato → `devolverAVozDaIa` (antes de tudo: falha aqui = aviso não nasce, não conversa muda).
3. **Chamada perdida = RECEBIDA e nunca atendida** (`!atendida && recebida`) → `agent_inbox_items`
   `kind='voice_call_missed' severity='warn'`, título "Chamada perdida de <peer>", corpo
   `motivoDaChamadaEmPortugues`, `ref_kind='contact'` (o botão de ligar mora na ficha do contato).
   Ligação FEITA não atendida NÃO vira perdida.
4. `event_log` `voice_call.ended` com `status='done'` explícito (evento de registro, não comando; sem
   handler em `event-log/drain.ts`; nascer `pending` deixaria linha pendurada — sem poda).
5. `emitAgentActivityForContact` com **três tipos de atividade** distintos:
   - `voice_call` (atendida) — entra na lista positiva de `fn_update_last_activity_at` (quebra silêncio).
   - `voice_call_missed` (recebida não atendida) — **fora** da lista positiva (silêncio, não interação).
   - `voice_call_unanswered` (feita sem resposta) — tipo próprio, também fora da lista (evita escrever
     "Chamada perdida" no negócio de quem discou). `usuarioId = owner_user_id` (senão a linha do tempo
     diria "Sistema" onde havia pessoa).

**Silenciar/devolver a voz da IA 🟢:**
- `calarIaDuranteALigacao` — `bot_silenced_until = now() + interval '2 hours'` na conversa 1:1 mais
  recente do contato, `last_handoff_reason = 'Ligação de voz em andamento'`. **Teto de 2h é anti-morte,
  não estimativa** (`'infinity'` seria "certo" mas se o worker cair entre `connected` e `call-ended` a
  conversa ficaria muda para sempre). **NUNCA encurta** (`bot_silenced_until is null or < teto`): um
  handoff humano durável (`infinity`) vence.
- `devolverAVozDaIa` — só desfaz o que ESTA ponte fez (`where last_handoff_reason = 'Ligação de voz em
  andamento'`), senão desligar o telefone devolveria à IA uma conversa que um atendente assumiu.
- **NÃO mexe** em `assignee_kind` (CHECK exige `assigned_to_user_id`, e quem atende do aparelho pode
  não ser usuário do CRM) nem em `contacts.force_human` (aquilo tranca o contato inteiro).
- `donoValido(owner)` — só passa string com forma de UUID (o `owner` é texto livre do WaCalls; gravar
  não-uuid numa coluna com FK para `auth.users` derrubaria a escrita).

### 4.10 Motivo da chamada em português (`wacalls/motivo-da-chamada.ts`) 🟢

`motivoDaChamadaEmPortugues(motivo)` traduz o vocabulário `EndCallReason` do upstream
(`user_ended`, `declined`/`rejected`, `timeout`, `busy`, `cancelled`/`canceled`, `failed`,
`do_not_disturb`, `unknown`) para frases de gente. `voice_calls.end_reason` guarda o valor **cru** de
propósito (vocabulário de terceiro, sem CHECK). **Motivo desconhecido não é escondido:** devolve
"terminou por um motivo que ainda não sabemos traduzir (<token>)" — a única pista para quem investigar.

### 4.11 Canal SIP — cliente ARI (`voip/ariClient.ts`) 🟢

Cliente fino da **Asterisk REST Interface (ARI)**. Auth `Basic` (`ARI_USERNAME:ARI_PASSWORD`). Env:
`ARI_URL`, `ARI_WS_URL`, `ARI_USERNAME`, `ARI_PASSWORD`, `ARI_APP` (def `voice-agent`).

- **`originateCall(params)` — chamada de saída 🟢.** Entrega **direto pro dialplan** (`context:
  'voice-agent-out', extension:'s'`), **NÃO pra Stasis app**. Decisão medida ao vivo: bridge ARI
  "mixing" com canal `externalMedia` **nunca relaya** o áudio injetado de volta pro trunk (só
  silêncio) — limitação conhecida do `chan_rtp`/`externalMedia`. O dialplan chama `AudioSocket(uuid,
  voice-agent:9092)` (TCP puro).
  - `dialNumber = toNumber.replace(/^\+/, "")` — PJSIP rejeita `+` na URI SIP.
  - `endpoint = "<tech>/<dialNumber>@<endpointName>"` (formato `PJSIP/<destino>@<endpoint>`; Asterisk
    resolve host/porta pelo AOR).
  - **`channelId` gerado ANTES** e passado no create + como `AUDIOSOCKET_UUID` → fecha a race com o
    StasisStart (que chega por WebSocket em processo separado, podendo preceder o retorno do POST).
- Controle: `answerChannel`, `hangupChannel(id, reason)`, `setChannelVariable`, `continueDialplan(id,
  context, exten, priority)` (sai da Stasis rumo ao AudioSocket na entrada).
- **`connectAriEvents(onEvent): WebSocket`** — WS `?app=<ARI_APP>&api_key=<user:pass>&subscribeAll=true`;
  parseia eventos Stasis. `AriEvent = { type, channel?, cause_txt? }` (`cause_txt` só em ChannelDestroyed,
  texto ITU Q.850). Reconexão delegada ao supervisor do worker.

### 4.12 Canal SIP — trunk e caller (`voip/guardar-trunk.ts`, `voip/resolve-caller.ts`) 🟢

- **`guardarTrunk(p)` — persiste o trunk SIP (um por org, upsert em `organization_id`).** Cifra a senha
  **AES-GCM** (`encryptKey`), grava só `password_last4` em claro, audita (`voip_trunk.created`/`updated`).
  `endpoint_name` é **derivado** (`nomeDoEndpoint(orgId) = "org-<uuid>-trunk-endpoint"`), nunca digitado
  — tem que bater com `[org-<uuid>-trunk-endpoint]` em `asterisk/pjsip.conf` (aplicado manualmente,
  migration 0349). **Senha obrigatória só na criação** (`!existente && !password` → recusa
  `senha_obrigatoria_na_criacao`); numa atualização, omitida mantém a cifrada. Motivos de falha:
  `senha_obrigatoria_na_criacao | cifragem | banco`.
- **`resolveOrCreateCallerContact(admin, orgId, rawPhone)`** — identifica quem ligou pelo número.
  `canonicalPhoneBR`; reusa `encontrarContatoPorTelefone` (mesma busca anti-duplicação do WhatsApp).
  Contato novo nasce `source:'voip'` (nunca `fn_upsert_wa_contact`, que exige identidade de canal que
  chamada não tem). **Corrida tratada:** insert que colide em `uniq_contacts_org_phone` (`23505`)
  re-busca quem venceu.

### 4.13 Canal SIP — μ-law (`voip/ulaw.ts`) 🟢

G.711 μ-law ↔ PCM16 linear, **bit-exato** (tabela bias+clip ITU-T G.711, `BIAS=0x84`, `CLIP=32635`).
Necessário no caminho AudioSocket: Asterisk manda SLIN (PCM16 8kHz), a OpenAI Realtime está
configurada para `audio/pcmu` (μ-law), a conversão acontece no CRM. `linearToUlawSample`,
`ulawToLinearSample` (tabela `ULAW_DECODE_TABLE` de 256 entradas pré-computada), `pcm16ToUlaw`,
`ulawToPcm16`.

### 4.14 Canal SIP — vocabulário (`voip/call-vocabulary.ts`) 🟢

Fonte da verdade em TS para o invariante `tests/invariants/vocabulario-banco-x-typescript.test.ts`
(compara contra os CHECK das migrations 0347/0337):
- `CallDirection = outbound | inbound` (1:1 com o banco).
- `CallStatus = ringing | in_progress | completed | no_answer | busy | failed | canceled` — vocabulário
  **da API** (derivado de status+end_reason em `mapStatusParaApi`), **não** uma coluna.
- `CallHandledBy = human | ai | ai_then_human`; `PhoneNumberRoutingMode = ai | human | ai_then_human`;
  `AiAgentChannel = whatsapp | voice`.
- ⚠️ `voice_calls.status` (`starting/ringing/connected/ended`) é vocabulário **do binário WaCalls
  upstream**, reaproveitado pelo SIP, **fora** do invariante de vocabulário por isso.

### 4.15 Superfície HTTP e worker do canal SIP (`app/api/v1/calls/route.ts`, `workers/voice-agent/`) 🟢

- **`POST /api/v1/calls`** — cria `voice_calls provider='sip' status='ringing'` e origina via ARI.
  Guarda de efeito `requireSupportWrite()` **antes** do RBAC (acompanhamento só-leitura não disca);
  `requireRole("manager")`. Trunk vem de `voip_trunk_settings` ativo, fallback `VOIP_TRUNK_ENDPOINT`
  (compat); sem trunk → `422 trunk_not_configured`. `channelId = randomUUID()` gravado como
  `asterisk_channel_id` **antes** do originate (fecha a race). Falha do originate → marca `ended`/`failed`
  e devolve `502 originate_failed`. `mode:'human'` grava `owner_user_id` e `handled_by='human'`.
- **`GET /api/v1/calls`** — lista chamadas `provider='sip'` (WaCalls tem tela própria em
  `voice/calls/*`). `requireRole("manager")`. `mapStatusParaApi(status, endReason)` traduz o vocabulário
  compartilhado para o rico (`connected→in_progress`; `ended` + `end_reason`: `timeout→no_answer`,
  `busy→busy`, `failed→failed`, `cancelled→canceled`, senão `completed`). Filtro de status é
  pós-mapeamento (o vocabulário rico não existe no banco). `duration_seconds = round(duration_ms/1000)`.
- **`workers/voice-agent/index.ts`** — processo persistente (`Dockerfile.voice-agent`, tsx). Toda mídia
  por AudioSocket. **Dois fluxos convergem no AudioSocket:**
  - **Saída:** `originateCall` já entrega ao dialplan; nunca passa pela Stasis.
  - **Entrada (`handleStasisStart`):** resolve org pelo número discado (`resolveInboundNumber`), cria
    contato/lead, gera **UUID separado** para o AudioSocket (o `channel.id` nativo do Asterisk
    `<epoch>.<seq>` não é UUID e o `app_audiosocket.c` rejeita), `setChannelVariable AUDIOSOCKET_UUID`,
    `continueDialplan('from-trunk-audiosocket','s',1)`.
  - **`resolveInboundNumber`:** extensão `"s"` (trunk DID único) resolve **só com exatamente 1 número
    ativo** em `phone_numbers` (senão recusa por ambiguidade); caso contrário `rpc
    fn_resolve_inbound_number`.
  - **AudioSocket TCP (porta `AUDIOSOCKET_PORT`, def 9092):** primeiro frame tipo `0x01` = UUID (16
    bytes); acha `voice_calls` por `asterisk_channel_id`; `getActiveVoiceAgent`; RAG por
    `resolverAcervoDoAgente`/`buscarConhecimento` (disparado em paralelo com o WS OpenAI, não em série,
    para a IA não demorar a "notar" a fala); sobe `AudioSocketCallBridge` (OpenAI Realtime). `connected`
    grava `answered_at`, `handled_by='ai'`. `finalizeAudioSocketCall` grava `ended`, `duration_ms`
    (calculado na mão — não é generated column), `transcript` (array `{speaker,text,ts}`).
  - **`handleChannelDestroyed`:** fecha chamada de SAÍDA presa em `ringing` (nunca atendida) →
    `ended`/`timeout` (só toca linha ainda `ringing`, para não sobrescrever o `finalizeAudioSocketCall`).
  - ⚠️ **Sem publicação em `event_log`** para chamadas SIP (nenhum handler consumiria; sumarização/
    sentimento fora do escopo do esqueleto).

### Estruturas de dados principais (unidade 4) 🟢
Detalhadas em `data-dictionary.md`. Resumo: `voice_calls` (tabela unificada wacalls+sip), tabelas de
config `org_voice_calls` (opt-in) e `voip_trunk_settings` (trunk SIP cifrado), e os DTOs de transporte
(`WacallsSessionInfo`, `WacallsCallRecord`, `VoiceCallWithSession`, `NumeroDiscavel`, `OriginateParams`,
`AriEvent`/`AriChannel`).

### Dependências (unidade 4) 🟢
- `lib/voice/*` → `lib/voice/opt-in`, `lib/channels/*` (`getAdapter`, `resolveSessionRef`,
  `PROVIDERS_DE_MENSAGEM`, `phone-variants`), `lib/api/wrappers` (`fail`), `lib/env`, `lib/wacalls/client`.
- `lib/wacalls/events-bridge.ts` → `lib/channels/phone-variants` (`phoneLookupVariants`),
  `lib/leads/agent-activity` (`emitAgentActivityForContact`), `lib/wacalls/motivo-da-chamada`,
  `lib/agent-engine/obs/logger`, `pg`.
- `lib/voip/*` → `lib/audit`, `lib/crypto/aes_gcm`, `lib/supabase/admin`, `lib/channels/{contato-por-telefone,phone-variants}`, `ws`.
- `workers/voice-agent/index.ts` → `lib/voip/ariClient`, `lib/voip/resolve-caller`, `lib/supabase/admin`,
  `lib/ai/agents` (`getActiveVoiceAgent`), `lib/ai/knowledge/busca`, `lib/leads/nascimento-do-lead`,
  `./audioSocketBridge`.
- `app/api/v1/calls/route.ts` → `lib/voip/ariClient`, `lib/impersonate/support`, `lib/auth/require-role`,
  `lib/schemas/calls`, `lib/audit`, `lib/supabase/server`.

### Complexidade 🟢
- Ponte de eventos WaCalls (SSE persistente, inferência de sentido, upsert idempotente das duas
  direções, silenciar/devolver IA, três tipos de atividade): **alta**.
- Worker de voz SIP (Stasis de passagem + AudioSocket TCP + race de channelId + OpenAI Realtime): **alta**.
- Opt-in de duas perguntas + guarda de rota (três códigos honestos): **média** (a lógica é pequena, a
  doutrina por trás é densa).
- Conversões de áudio (`pcm.ts`, `ulaw.ts`): **baixa** (algoritmos de referência, bem contidos).

### Discrepâncias / lacunas 🔴🟡
- 🟡 `workers/voice-agent/audioSocketBridge.ts` (a ponte de áudio com a OpenAI Realtime) lido apenas por
  referência de nome/comentários, não linha a linha — o Data Master / uma escavação futura deve
  detalhá-lo se a spec de voz SIP for aprofundada.
- 🔴 `asterisk/pjsip.conf` / `extensions.conf` (contextos `voice-agent-out`, `from-trunk`,
  `from-trunk-audiosocket`) e a `rpc fn_resolve_inbound_number` são referenciados mas vivem fora de
  `lib/`; a topologia Asterisk completa é lacuna para validação humana (aplicada manualmente por VPS,
  migration 0349).
- 🟡 A ponte WebRTC humano→navegador do canal SIP não está implementada ("esqueleto", comentário em
  `calls/route.ts` e `workers/voice-agent/index.ts`); `mode:'human'` grava dono mas não abre áudio.
- 🟡 Limite conhecido da ponte WaCalls: ligação que TERMINOU com a ponte SSE caída não vem no snapshot
  `call-list` (registro `ended`), e a linha segue aberta até o teto de silêncio de 2h (issue aberto
  citado no código).
- 🟡 `voice_calls.status` é vocabulário de terceiro (WaCalls upstream) sem CHECK próprio nosso — o
  contrato rico da API é derivado, não persistido.

---

## Unidade 5 — CRM e Funil: `lib/leads/`, `lib/pipelines/`, `lib/kanban/`, `lib/contacts/`, `lib/tags/`, `lib/conversoes/`, `lib/prospecting/`

### Propósito 🟢
O núcleo do **"sistema vivo"** (doutrina anti-morte): nenhuma demanda aberta pode ficar sem próximo
passo nem sem desfecho. Cobre o ciclo de vida do negócio (nascimento → funil → score/risco →
encerramento → reativação), o quadro Kanban, os contatos (dedup, rótulo, CPF), etiquetas, o
reporte de conversão às plataformas de anúncio, e a esteira de prospecção fria. A regra de ouro é
que **quase toda lógica é pura e testável sem banco**; as escritas ficam na borda (route handlers e
workers), e o vocabulário do funil (`lead/deal/won/lost`) é renomeável por org (multi-nicho).

Distinção conceitual central 🟢: no harness da IA "lead" = **contato** (a pessoa); no CRM "lead" =
**negócio** (`crm_leads`). Uma pessoa pode ter vários negócios. `active-lead.ts` faz a ponte.

---

### 5.1 Scoring e Risk Radar (`lib/leads/`) 🟢 — o coração do sistema vivo

**Fórmula de score (`score-formula.ts`), não modelo.** Escolhida para que o `reason` seja derivado
do cálculo — "número sem porquê é impossível por construção". Pesos:
- `BASE = 30`; `POR_COMPROMISSO = +12` (teto 3); `POR_OBJECAO = -8` (teto 3); `POR_CAMPO_BANT = +5`
  (teto 4); `PESO_RISCO` por bucket: `em_dia +10 / em_voo 0 / em_risco -10 / critico -20`.
- `MINIMO_DE_SINAIS = 2` (substantivos = compromissos+objeções+BANT; **recência não conta** — lead
  novo nasce sem score de propósito). `score = clamp(0,100, soma+BASE)`.
- **Guarda de lastro citável:** score exige `checkpointId` e ao menos um fator com âncora
  (`{kind:'checkpoint',id}`); **BANT não ganha âncora** (vem de `lead_state`, âncora falsa é pior que
  ausente). Sem lastro → `score=null`, `semSinal ∈ {sem_lastro_citavel, sem_conteudo, negocio_fechado}`.
- Negócio fechado (`status≠open`) → `null` (não tem probabilidade de fechar). `FORMULA_V=1` versiona.
- Frase: no máx 3 parcelas nomeadas + "+N outros"; se houve clamp, acrescenta "limitado a X".

**Histerese de faixa (`kanban/score-band.ts`):** `frio|morno|quente`, `LIMIAR_QUENTE=70`,
`LIMIAR_MORNO=40`, zona morta `BANDA=5`. `resolveBand` sobe/desce **um degrau por vez** (histerese por
fronteira; degrau-a-degrau eliminou defeitos que o CHECK do banco pegava). `ai_probability_band_since`
só se move quando a faixa muda (`is distinct from`).

**Classificador puro de risco (`risk-radar.ts`).** `RiskBucket = critico|em_risco|em_voo|em_dia`.
Janela **por estágio** (`crm_stages.expected_duration_hours`; fallback `RISK_COLD_HOURS=24`,
`criticalHours = coldHours*3`, `RISK_CRITICAL_HOURS=72`). "Sem resposta há 2 dias" é normal num
contrato e abandono num agendamento — por isso a janela não é global. Precedência de `classifyRisk`:
1. `agenda.adiar` → `em_voo`; 2. `agenda.presenca_vencida` → `critico`; 3. fresca (`< coldHours`) →
`em_dia`; 4. `inFlight` (follow-up agendado) → `em_voo`; 5. `≥ criticalHours` → `critico`; 6. senão
`em_risco`. `onRadar = bucket≠em_dia`. `compareRisk` ordena por `BUCKET_RANK` e, dentro, mais frio
primeiro.

**"Desde quando neste estado" (`risk-since.ts`):** `since` = instante do **cruzamento do limiar**, não
`last_activity_at` (negócio com janela 72h e 100h de silêncio está em risco há 28h). **Nunca no
futuro** (clampa em `agora` — os atalhos de agenda atribuem bucket sem cruzar limiar, e futuro
violaria `check (since <= detected_at)` derrubando o worker).

**Worker de score (`score-writer.ts`):** lê os sinais de UM lead (join `crm_leads`+`crm_stages`+
`lead_state`+lateral do `lead_checkpoints` mais recente+band anterior), classifica recência com
`classifyRisk`, chama `calculaScore`. Score `null` → **apaga** `crm_lead_scores` (null é melhor que
número velho). Nunca por trigger.

**Radar montado (`radar-de-risco.ts`):** lista acionável de demandas abertas que esfriaram.
`SCAN_CAP=500`, `IDS_POR_CONSULTA=100` (lotes p/ não estourar querystring do PostgREST). Funil
**arquivado** excluído antes do SCAN_CAP. Dono agente resolvido sem filtrar `is_active` (exibir dono é
obrigatório mesmo com agente desligado). Compartilhado com a IA em `lib/mcp/tools/retencao.ts`.

**Seed do risco (`risk-seed.ts`):** grava o estado de todo negócio aberto uma vez (backfill), sem
timeline falsa, com `detected_at` do `now()` **do banco** (dois relógios na mesma decisão foi defeito
real). `crm_lead_risk_states` (`bucket, since, cold_hours, detected_at`), upsert `onConflict lead_id`.
Negócio sem relógio (sem `last_activity_at` e sem `created_at`) fica de fora e é **contado**.

**Worker do risco (`risk-worker.ts`):** faz "esfriando" acontecer sem ninguém abrir tela; **só escreve
quando o bucket muda** (a tabela está na publicação realtime; refresh faria o board piscar).
`narra(de,para)` decide a linha de timeline: entrar em `em_risco` (de `em_dia`/null) → `lead_cooled`
"sem resposta além do prazo"; `critico` → `lead_cooled` "parado há muito tempo"; voltar a `em_dia` →
`lead_reactivated`; entrar em `em_voo` → nada (quem agendou já registrou). Ao esfriar, cria proposta de
reativação no mesmo tick. Não toca `crm_leads` (atividades fora da lista positiva de
`fn_update_last_activity_at`).

### 5.2 Máquina de estados do funil (`lib/leads/` + `lib/pipelines/`) 🟢

**Regras de edição de funil (`pipelines/pipeline-editing.ts`), puras.** `slugDeFunil` só na criação
(considera arquivados — índice único não parcial). `validarArquivamento` recusa **na ordem do que o
usuário consegue resolver**: único funil ativo → funil padrão → fonte de webhook (`ON DELETE CASCADE`,
apagaria a captação em silêncio) → automação ativa que aponta pra ele (`actions` jsonb sem FK).
`podeExcluirDeVez` herda toda recusa do arquivamento + recusa se tem negócios. `updatesDeMarcaExclusiva`
(default/client pipeline): **libera o anterior ANTES** de marcar o novo (índices únicos imediatos, não
deferíveis, senão `23505`). `ETAPAS_INICIAIS` = Novo/Em andamento/Ganho/Perdido (funil sem etapa de
ganho é incapaz de fechar — `/win` responde `pipeline_no_won_stage`). `regrasQueApontamPara` é
defensivo porque `actions` não tem schema.

**Regras de edição de etapa (`stage-editing.ts`), puras.** `chaveDeNome` normaliza NFD sem acento
("Pos venda" colide com "Pós-venda"). Slug não muda ao renomear (`SLUG_MIN=2`, `SLUG_MAX=40`).
`validarMarcacao`: mutex ganho/perda, desmarcar sem substituta é proibido. `updatesDeMarcacao`: desmarca
a antiga primeiro (índice único imediato); `agent_stage_hint` acompanha (CHECK de coerência).
`validarArquivamento`: recusa única/ganho/perda; `precisaDestino` quando há negócios.

**Operações com I/O (`stage-operations.ts`):** validar → liberar antiga → marcar nova → mover negócios
→ arquivar, tudo em **sequência** (nunca paralelo). `conflitoDoBanco` traduz `23505`/`23514` em pt-BR
(409). Arquivada não se edita (furaria o índice parcial `uniq_crm_stages_pipeline_won`).

**Mapa passo-do-agente → etapa (`agent-mapping.ts`):** `crm_stages.agent_stage_hint`.
`PASSOS_QUE_PRECISAM_DE_ETAPA = [new, contacted, qualifying, qualified, negotiating]` (won/lost têm
coluna própria). `validarMapeamento` recusa etapa de outro funil, etapa usada por 2 passos, incoerência
won/lost. `diffParaUpdates`: UNSETs primeiro (índice imediato).

**Três movimentadores de card (padrão comum: trava otimista `.eq("stage_id", origem).select("id")` →
0 linhas = `conflito_humano`; negócio não-open → `lead_fechado`; emite `stage_changed` +
`emit_event lead.stage_changed entity_kind='crm_lead'`):**
- `agent-stage-sync.ts` — o agente move o card; destino de perda chama `decideMotivoDaPerda` **sem**
  `motivoAtual` (o agente não herda a licença do motivo). Erro do SELECT é lido (supabase-js não lança
  em rede) → `indisponivel`.
- `handoff-stage-move.ts` — opt-in por `slug='chamar-humano'`; sem a etapa → `sem_etapa_de_handoff`.
- `appointment-stage-move.ts` — espelha `calendar_appointments.status`; **só `pending` e `confirmed`
  avançam** (cancelar/faltar um compromisso não é o negócio morrendo).

### 5.3 Ciclo de vida (`lib/leads/`) 🟢

- **Nascimento (`nascimento-do-lead.ts`):** conversa vira lead, **idempotente por contato** (um lead por
  DEMANDA, não por mensagem). Funil de entrada: `is_client_pipeline` se cliente (coluna `first_service_at`
  + `lerClientePelaAgenda`, nunca a tag), senão `is_default`. INSERT via RPC `fn_nascer_lead_da_conversa`
  (advisory lock por org+contato — 3 mensagens juntas criaram 3 cards em produção). Toda falha cai no
  funil de entrada (não nascer = a pessoa sumir). Emite `lead_created`.
- **Lead ativo (`active-lead.ts`):** `resolveActiveLeadForContact` — 0 abertos → `no_open_lead`; 1 →
  rota; vários → prefere `defaultPipelineId`, ordena por atividade; empate no topo → `ambiguous_open_leads`
  (não escolhe — mover card errado é o único bug visível ao cliente). `assertMesmaOrg` lança em >1 org.
- **Encerramento (`encerramento.ts`):** won/lost, `motivo` obrigatório em lost, idempotente. Status/closed_at
  vêm do trigger `fn_crm_lead_close_on_stage` (não escreve à mão). Rede de segurança traduz `23514` → 422.
- **Motivo da perda (`motivo-da-perda.ts`):** ponto único (issue #917). `moved_to_another_pipeline` é
  canônico e **excluído das métricas** (0266); nunca ofertado na tela (seria 1 clique para tirar perda real
  da métrica). `decideMotivoDaPerda`: quem arrasta dentro da coluna de perda não gera perda nova.
- **Reativação (`reactivation.ts`):** proposta nasce ao esfriar, `expiresAt = now + coldHours`, nasce sem
  rascunho (LLM indisponível não pode virar silêncio), vence sozinha (`venceReativacoes`). Vencimento **não
  mexe no bucket** (bucket mede silêncio do cliente; a inação foi do time).
- **Clone para funil (`clonar-para-funil.ts`):** `pipeline_id` é imutável (P-01); mover cross-pipeline =
  clonar + encerrar origem. Herda `custom_fields` inteiro; não herda `external_id`/status/closed_at.
- **Correção humana (`correcao-humana.ts`):** só o último movimento importa e precisa ser da IA;
  devolução vs redirecionamento; `ondeAIaMaisErra` agrega por etapa.

### 5.4 Atividade e Timeline (`lib/leads/`) 🟢

- **Vocabulário fechado (`activity-vocabulary.ts`):** fonte única (~45 `ActivityType`), gate é o compilador
  (`Record<ActivityType,string>` exaustivo), **sem CHECK no banco** (clone com tipo legado não quebra
  `update.sh`). Rótulos multi-nicho ("Agendamento", não "Consulta"). `voice_call_missed`≠`voice_call_unanswered`
  porque `fn_update_last_activity_at` decide por tipo. `NOME_DO_CAMPO` traduz colunas — nomes, nunca valores (PII).
- **Emissor (`activity-emitter.ts`):** grava UMA linha em `crm_lead_activities` (fire-and-forget).
  `ActivityActorKind = user|ai|system|rule|contact`. Ator `ai` sem lastro → grava como `system` mantendo
  `actor_agent_id` (não afirma autoria sem prova). `evidence` chaves `run_ids/trace_ids/llm_call_ids` → cada
  uma UMA tabela.
- **Timeline (`timeline-query.ts`, `timeline-grouping.ts`):** portão de exaustividade da coluna em compile-time;
  `eixoDoDossie` inclui atividade de conversa sem lead, exclui negócio irmão; imports tardios de `env`.
  Agrupamento: `JANELA_MS=60000`, `LIMITE_DE_BLOCOS=12`, `NUNCA_COLAPSA` = decisões humanas; se >12 blocos
  agrupa por dia (parte dos itens, não dos blocos).
- **Falha de escrita (`activity-write-failure.ts`):** atividade que É a decisão → falha alto (500); rastro de
  mutação já ocorrida → falha baixo mas conta (`crm.activity_write_failed`).

### 5.5 Classificação inicial (`lib/leads/`) 🟢

`classificacao-inicial.ts` (pura, ingestão do webhook Respondi): três saídas `desqualificado |
revisao_humana | classificado(A|B|C|D|nao_avaliado)`. Desqualificação = 2 motivos técnicos/legais
(`contato_invalido`, `sem_consentimento` — ausência da pergunta NÃO desqualifica). Revisão = 3 sinais
(`conflito_de_identidade`, `spam_suspeito`, `incoerencia_investimento`) que não bloqueiam envio.
`percentual = clamp(0,100,(score/100)*100)`; bandas A≥70/B≥40/C senão. **D por duas vias** (frase exata
"ainda não posso investir" OU percentual 0; não há mais corte numérico de orçamento — decisão de produto).
`config-classificacao-inicial.ts`: `maxScoreConhecido=100` (🟡 INFERIDO do produto: Respondi tratado como
escala 0-100, baseado em 2 amostras 40/55).

### 5.6 Kanban (`lib/kanban/`) 🟢

- **Indexação fracionária (`fractional-indexing.ts`):** `midpoint(prev,next)` com `STEP=1000`; `NaN` quando
  iguais (dispara rebalance global); `fractionalPrecision` detecta necessidade de rebalance (>20 níveis, P-05).
- **Estado do card (`card-state.ts`):** `CardInput` deliberadamente não é `Lead` (o card responde 4 perguntas:
  quanto vale · vai fechar · o que fazer agora · quem toca). `resolveCardState` tem **precedência estrita** da
  faixa ③ (§5 do contrato): `awaiting` (nextAction) > `reactivation` (proposta viva) > `cooling` > `meter`
  (score) > `idle`. Um card só mostra o que exige decisão AGORA. `probability` `null` ≠ `0` (`typeof ===
  "number"`, nunca truthiness). Faixa é persistida, nunca recalculada (respeita histerese). `coolingLabel`:
  dias quando >48h.
- **Dono (`owner.ts`):** `resolveLeadOwner` (0070) humano OU agente OU ninguém; agente vem anexado ao lead sem
  filtro de is_active; nome null → rótulo genérico, nunca "Sem responsável" (mentiria dizendo órfão).
- **Local echo (`local-echo.ts`):** o que EU mudei não pulsa na minha tela. Marca segue o **ciclo de vida da
  mutação** (`marcarEcoLocal`/`liberarEcoLocal`), contador (ações se sobrepõem), `FOLGA_MS=1000` após assentar,
  `FALLBACK_MS=4000` rede de segurança. `ehEcoLocal` não consome a marca (defeito 12.c).
- **Filtros (`filters.ts`):** deep-linkável; `AGENT_OWNER_PREFIX="agent:"` (humano=uuid, agente=`agent:<uuid>`);
  `unassigned` = sem dono nenhum; tag casa as 3 caixas (negócio/contato/conversa) via `cardTemMarcador`;
  `overdueOnly` = open + `expected_close_date < hoje`.
- **Vocabulário (`vocabulary.ts`):** defaults `Lead/Negócio/Ganho/Perdido`, sobrepostos por `PipelineVocabulary`.

### 5.7 Contatos (`lib/contacts/`) 🟢

- **Duplicados (`duplicados.ts`):** **union-find** sobre "é a mesma pessoa" (critérios se encadeiam: A~B por
  telefone, B~C por email → os 3). Motivos: `telefone` (grafias do 9º dígito), `email`, `telefone_em_conflito`
  (o que `fn_upsert_wa_contact` parkou em `source_metadata`). Anonimizado sai (L-04 irreversível).
  `principalSugerido` = atividade mais recente, empate no mais antigo. É sugestão, nunca aplicada sozinha.
- **Rótulo (`rotulo-do-contato.ts`):** fonte única de "como se chama esta pessoa na tela" (antes 6 cópias, 4
  finais diferentes). `nomeDoContato`: `name` (editável pela pessoa) > `display_name` (ingestão) — quem FALA
  (prompt, lembrete) não cai no telefone. `ehIdentificadorTecnico` recusa `@lid`/`Contato 5431`/dígitos longos
  (conservador: recusar nome legítimo apaga identidade real). `rotuloDoContato` acrescenta telefone com 9º dígito.
- **CPF (`cpf.ts`):** `hashCpf` = sha256(11 dígitos) para lookup/dedup sem plaintext. `encryptCpfSql` via RPC
  `encrypt_cpf` (🟡 LACUNA: cifra at-rest ainda não provisionada — degrada para hash only com warn).
- **Cliente pela agenda (`cliente-pela-agenda.ts`):** lê o interruptor `settings` no servidor; **falha lê como
  desligada** (errar para o funil padrão é o comportamento de antes da regra; lançar impediria o lead de nascer).

### 5.8 Etiquetas (`lib/tags/cor-da-etiqueta.ts`) 🟢

Paleta **fixa** de 8 tons, escolhida por busca contra as réguas de `lib/branding/contraste.ts` (pior separação
OKLab 0,1195; sob dicromacia 0,0576; contraste do texto 4,75 — todos acima do piso). Cor livre é a única forma
de uma etiqueta virar ilegível por decisão humana. Texto do chip escolhido pelo sistema (`melhorFrenteSobre`),
nunca por quem escolhe a cor — cor nunca é único portador de significado. `chaveDaEtiqueta` = `lower(trim)` (a
mesma do banco). Lê as duas formas gravadas (string legada e objeto `{tag,cor}`). Forma validada na borda/banco/
leitura; pertinência à paleta não é validada (instalação pode querer outro tom sem migration).

### 5.9 Conversões — reporte de venda ao anúncio (`lib/conversoes/`) 🟢

`envio.handler.ts` é um **handler de evento**, não uma chamada em `encerramento.ts` (invariante 1: `lib/leads/`
não sabe que conversões existem; a Meta fora do ar nunca bloqueia uma venda). Escuta **duas portas**: `lead.won`
(botão Ganhar/IA) e `lead.stage_changed` (arrasto no kanban + bulk). O `payload.status` é **dica**, o banco é
verdade (re-lê `crm_leads`; o `/move` manda status, o `/bulk` não — confiar no payload falharia calado numa
porta). Só grava no livro-razão (`conversion_ledger` via `registro-de-envio`) quando **há atribuição de anúncio**
(venda orgânica não é conversão não-reportada). `Purchase` exige valor+moeda (`value_cents` nullable → pendência
mais comum, razão de a tela existir; mandar 0 ensinaria o otimizador que a venda não vale nada). Idempotente
(`jaFoiEnviada`, `eventoId = "<leadId>:Purchase"`). Erro de leitura/transitório → `retry`; recusa → `error`.

### 5.10 Prospecção fria (`lib/prospecting/`) 🟢 — a única linha que fala PRIMEIRO

`worker.ts` (`sendNextCandidate`/`tickProspecting`): esteira que aborda empresas de pesquisa pública.
- **Ritmo mais lento que a esteira de resposta (`ritmo-da-esteira-fria.ts`):** `FATOR_DA_ESTEIRA_FRIA=4` (a
  doutrina fixa campanha ~4× mais lenta que resposta). `tetoDiarioDaEsteiraFria = warmupCapFor / 4` (deriva dos
  degraus da casa, não copia — quem ajusta os knobs ajusta os dois juntos), com piso 1 e **falha fechada** (knobs
  incompletos → teto 1, nunca "sem teto"; o número que morre é o do cliente). `proximoEnvioDaEsteiraFria` acrescenta
  **jitter que só atrasa** (cadência exata é assinatura de robô).
- **Guardas antes do efeito externo:** janela de envio, pacing (`decidePacing`), teto do warm-up frio, `daily_limit`
  da campanha, teto global 50/24h, intervalo mínimo. Pré-go-live do canal, `serviceBoundary`, autorização do
  contato para IA. Reserva a cota do canal (`recordSend`) ANTES do envio (timeout não pode liberar cota).
- **Abordagem gerada** por `gerarAbordagemDeFormulario` com `origemDaAbordagem="prospeccao_fria"` (prompt que proíbe
  afirmar preenchimento). **Rodapé de saída** (`comSaida`) montado aqui no idioma da org (senão a saída é "Denunciar
  spam", que queima o número do cliente). Audita `prospecting.approach_sent` só quando houve efeito (sem PII).
- **Falha por escopo:** `ProspectingError.escopo==='candidato'` marca o item e segue; qualquer outra pausa a campanha
  (antes qualquer exceção pausava tudo — oposto do "item ruim marca o item").

### Estruturas de dados principais (unidade 5) 🟢
Detalhadas em `data-dictionary.md`. Tabelas centrais: `crm_leads`, `crm_stages`, `crm_pipelines`,
`crm_lead_scores`, `crm_lead_risk_states`, `crm_lead_reactivations`, `crm_lead_activities`, `lead_checkpoints`,
`lead_state`, `demandas`, `contacts`, `prospecting_campaigns`, `prospecting_candidates`, `conversion_ledger`.

### Dependências (unidade 5) 🟢
- `lib/leads/*` (scoring/risco) → `lib/agenda/protecao-followup`, `lib/kanban/score-band`, `lib/contacts/rotulo-do-contato`, `lib/auth/types`.
- `lib/leads/*` (lifecycle) → `lib/api/*`, `lib/audit`, `lib/i18n/*`, `lib/schemas/*`, `lib/atendimento/{origem,fronteira}`, `lib/agent-engine/agent/lead-state`, `lib/contacts/cliente-pela-agenda`.
- `lib/pipelines/pipeline-editing.ts` → `lib/leads/stage-editing`, `lib/kanban/fractional-indexing`.
- `lib/kanban/*` → `lib/types/leads`, `lib/kanban/{score-band,owner,marcadores-do-card}`, `lib/leads/risk-radar` (janela).
- `lib/contacts/*` → `lib/channels/phone-variants`, `lib/branding/*` (via tags), `lib/schemas/settings`, `lib/logger`.
- `lib/tags/cor-da-etiqueta.ts` → `lib/branding/{contraste,rampa}`.
- `lib/conversoes/*` → `lib/event-log/dispatcher`, `lib/plataformas-de-anuncio/*`, `lib/supabase/admin`.
- `lib/prospecting/*` → `lib/agent-engine/{pacing,agent/abordagem-de-formulario,edge}`, `lib/ai/elegibilidade/*`, `lib/atendimento/*`, `app/api/v1/messages/_handler`, `lib/audit`, `lib/env`.

### Complexidade 🟢
- Scoring/risco (fórmula + histerese + classificador por estágio + 3 workers): **alta**.
- Máquina do funil (edição de funil/etapa + 3 movimentadores com trava otimista + mapa do agente): **alta**.
- Prospecção fria (pacing derivado + guardas em cadeia + falha por escopo): **alta**.
- Kanban (card-state com precedência estrita, local-echo por ciclo de mutação): **média-alta**.
- Contatos (union-find de dedup, rótulo, CPF): **média**.
- Conversões (handler de duas portas idempotente): **média**.

### Discrepâncias / lacunas 🔴🟡
- 🔴 Cifra de CPF at-rest (`encrypt_cpf` RPC) **não provisionada** — hoje só `cpf_hash`; degrada com warn.
- 🟡 `config-classificacao-inicial.ts::maxScoreConhecido=100` é INFERIDO (Respondi como escala 0-100; 2 amostras).
- 🟡 Vários arquivos menores lidos por referência: `lib/prospecting/{provider,store,context,agent-*,schema}.ts`,
  `lib/kanban/{marcadores-do-card,selecao,dados-do-contato,score-band}` (score-band via sub-agente), `lib/contacts/
  {cliente,csv,tag-normalizada,proposta-de-dado}`, `lib/leads/{planilha,links-de-contato,origem-do-*,campos-*,
  next-action,owner-patch,escopo-de-funil}` (cobertos pelo sub-agente, não linha a linha por mim).
- 🟡 As constraints/triggers citadas (`fn_nascer_lead_da_conversa`, `fn_crm_lead_close_on_stage`,
  `fn_update_last_activity_at`, `fn_mesclar_contatos`, índices únicos parciais) vivem em `supabase/baseline.sql` —
  a validação completa é tarefa do Data Master.

---

## Unidade 6 — Agenda e Financeiro: `lib/agenda/`, `lib/financeiro/`, `lib/catalogo/`

### Propósito 🟢
Três domínios que fecham o laço comercial do CRM: **quando** o atendimento acontece (agenda), **o que**
se vende (catálogo de produtos) e **quanto** entra e sai (financeiro/comanda). A agenda é de longe o
maior e mais denso: um motor de horários livres 100% puro, uma sincronização de duas vias com o Google
Calendar (reconciliação com merge de três pontas) e um vocabulário espelhado no banco por migration. O
financeiro cobre a comanda (conta do atendimento) e o catálogo financeiro (contas, formas de pagamento,
planos de conta, regras de comissão, recorrências). O catálogo de produtos cobre importação por planilha
e busca difusa "token a token" que separa palavra (aproximada) de número (exato).

Princípio transversal 🟢: **quase toda regra é pura e testável sem banco nem relógio** — `agora` é sempre
parâmetro injetado, nunca `new Date()` interno. Duas armadilhas já pagas justificam isso (CI vermelho de
madrugada quando a janela anti-banimento usava `new Date()` cru; teste dependente de hora que mente nos
dois sentidos). O que toca banco fica na borda; o vocabulário do domínio (`tipos.ts`) é a fonte dos CHECK.

---

### 6.1 Vocabulário da agenda (`agenda/tipos.ts`) 🟢 — a fonte da verdade dos CHECK

Cada `const X = [...] as const` é a fonte de um CHECK correspondente (migration 0176/0177), e o extrator
de vocabulário (`tests/invariants/agenda-vocabulario.test.ts`) só reconhece as formas `type X = "a"|"b"`
e `const X = [...] as const` — por isso a lista de códigos é crua e o rótulo pt-br mora num `Record`
exaustivo à parte, cuja completude o compilador cobra.

- `CATEGORIAS_DE_AGENDAMENTO` (10 valores: consulta, procedimento, retorno, visita, vistoria, reuniao,
  call, orcamento, demonstracao, outro) — espelha os nichos do onboarding.
- `LOCAIS_DE_AGENDAMENTO` (in_person, phone, whatsapp, video_link, google_meet) e
  `CAMPO_EXIGIDO_PELO_LOCAL` (in_person→endereço, video_link→url, resto→nada) — decide o que o formulário
  exige.
- `SITUACOES_DO_AGENDAMENTO` (pending, confirmed, cancelled, completed, no_show).
- **`SITUACAO_SEGURA_O_LEAD`** (Record→bool): pending/confirmed seguram o lead; o resto não. É a resposta
  a "este lead tem consulta marcada?" que o motor de follow-up e o Radar de Risco fazem antes de cobrar
  alguém — servida pelo índice parcial `calendar_appointments_org_vivos_idx`. `SITUACOES_VIVAS` deriva
  dela (o `Record` exaustivo **obriga a decisão** quando um estado novo entra, não só a cobertura).
- `AUTORES_DO_AGENDAMENTO` (user, ai, system, contact, sync) — segue `crm_lead_activities.actor_kind`,
  com `sync` a mais (linha nascida de evento externo).
- `ORIGENS_DO_AGENDAMENTO` (ui, mcp, google_sync, public_page).
- **`SITUACOES_DA_CONEXAO`** (connecting, healthy, token_expired, scope_missing, disconnected,
  rate_limited, error) e **`CONEXAO_CONTA_COMO_OCUPACAO`** — a regra numa linha: **BLOQUEIA, A MENOS QUE
  UM HUMANO TENHA MANDADO PARAR** (DECISÃO 3.2). Só `disconnected` (o único estado decidido por gente) e
  `connecting` (ainda não leu nada) não contam como ocupação; `token_expired`/`scope_missing`/`error`/
  `rate_limited` continuam bloqueando, porque o compromisso segue existindo no Google — o que parou foi a
  atualização, não a existência. `CONEXOES_QUE_NAO_CONTAM` deriva dela.
- `PROVEDORES_DE_AGENDA = ["google_calendar"]` e `PROVEDOR_GOOGLE = "google_calendar"` (valor canônico
  para quem CONSULTA; um bug usou o literal `"google"`, que o CHECK proíbe, e deixou a conexão do Google
  invisível na v1.9.0).
- `ENTIDADE_DO_AGENDAMENTO = "calendar_appointment"` (entity_kind no event_log; singular da tabela).
- `NOME_GENERICO_DO_TIPO = "Agendamento"` (fallback, nunca padrão — condição de automação não casa de
  propósito).
- `ATIVIDADES_DA_AGENDA` (appointment_scheduled/rescheduled/cancelled/completed/no_show) com `satisfies
  readonly ActivityType[]` — a amarra que impede duplicar o vocabulário de `lib/leads/activity-vocabulary`.

**Algoritmo de cor de trilha** 🟢: `trilhaPadraoDoMembro(userId)` = hash estável (`soma*31+charCode mod
1_000_003`, depois `mod 8`) derivado do id (não da posição na lista — senão a equipe troca de cor quando
alguém entra). `trilhasDaEquipe(userIds)` parte do hash e resolve **apenas colisões** avançando para a
próxima trilha livre, ordenando por id para estabilidade (com >8 pessoas a repetição é inevitável — a
resposta é mais trilhas na paleta, não mais lógica).

### 6.2 Motor de horários livres (`agenda/horarios-livres.ts` + `fuso.ts`) 🟢 — puro, sem banco, sem relógio

`horariosLivres(entrada)` responde "dado a jornada publicada, o que já está marcado e as regras do tipo,
quais horários oferecer entre X e Y?". Sequência: **(1)** a grade nasce no início da janela publicada e
NÃO SE MOVE, de `intervaloMin` em `intervaloMin` (`?? duracaoMin`); **(2)** faixas sobrepostas são UNIDAS
antes de tudo (`unirFaixas`, `<=` funde contíguas); **(3)** menos os ocupados; **(4)** menos o que começa
antes de `agora+avisoMinimo`; **(5)** menos o que passa de `agora+janelaDias`.

⚠️ **A lição que custou uma versão inteira (DECISÃO 12):** o bloqueio por exceção é um OCUPADO, não uma
janela menor. A versão anterior SUBTRAÍA o bloqueio da janela, e subtrair **reparte** — uma reunião
10:30-11:30 fazia a tarde inteira deslizar (12:30, 13:30, 14:30) e o 17:00-18:00 livre sumia; a MESMA
reunião como compromisso não fazia nada disso. Fazer os dois virarem o mesmo mecanismo (remoção de
instantes) é a correção, porque "não tem como divergir". O buffer (`slotInflado`, `bufferAntes`/`Depois`)
infla o SLOT, não o compromisso (diverge quando os buffers são diferentes).

⚠️ `windows` vazio significa **coisas opostas** nos dois usos da mesma coluna: `isWithinSchedule`
(roteamento, produção) lê vazio como 24/7 aceita; a agenda lê vazio como zero horário (agenda 24/7
ofereceria consulta às 3h). Por isso `janelasDoDia` existe em vez de reusar `isWithinSchedule`.
Limitação declarada e aceita: **nada cruza a meia-noite**. Dedup final por instante protege da hora de
horário de verão que desliza para cima do slot seguinte (Santiago/Assunção têm DST; Brasil não).

`fuso.ts` concentra toda conversão de fuso via `Intl.DateTimeFormat` cacheado. `instanteDe(parede, fuso)`
usa **duas passagens** (o offset depende do instante buscado); na hora inexistente do DST devolve o
`Math.max` dos dois candidatos (DECISÃO 15: `Math.min` cairia no dia anterior em fusos de offset positivo
como Beirute/Teerã). Nunca devolve Invalid Date.

### 6.3 Coleta do banco e recusa honesta (`agenda/consulta.ts`, 835 linhas) 🟢

O adaptador entre o motor puro e o Supabase, **compartilhado pela rota REST e pelas tools MCP** (o client
é injetado: rota passa client de sessão com RLS, MCP passa admin que bypassa RLS → **toda query filtra
`organization_id` explicitamente**). A ocupação do Google do dono é lida por **RPC, não join**, porque a
RLS de `calendar_connections` esconde conexões de `agent`/`viewer` (issue #879).

- `horariosLivresDaOrg(...)` — lê `calendar_event_types` (por id OU slug: rota usa uuid, MCP usa slug),
  recusa `tipo_desconhecido`/`tipo_desativado`; resolve o dono (`ownerUserId ?? default_owner_user_id`,
  recusa `sem_responsavel`); lê `attendant_availability.schedule` (recusa `jornada_mal_configurada`);
  lê exceções de `calendar_availability_exceptions` filtradas pelo **dia LOCAL** (issue #878 — corte por
  dia UTC perdia linhas em offset negativo). Constante `MAXIMO_DE_DIAS = 62`.
- `coletaOQueOcupa(...)` — `calendar_appointments` por dono + sobreposição ESTRITA (`starts_at < ate`,
  `ends_at > de`) + `neq id` opcional (issue #1084: o compromisso remarcado ocupa a ORIGEM, não o
  destino); Google via RPC `fn_agenda_ocupacao_google_do_dono`; classificação delegada a `ocupadosDoDono`.
- `listaAgendamentos(...)` — exige um recorte (`sem_alvo`); o lead é **polimórfico** (DECISÃO 6): não há
  `lead_id` na tabela, o vínculo vem de `crm_lead_links` (target_kind='appointment'); distingue "não é
  lead" (`alvo_nao_e_lead`, issue #509) de vínculo legitimamente vazio. Sem recorte de tempo, piso = AGORA
  (senão limit+ordem-asc devolveria os mais antigos e derrubaria o recém-marcado).

**Recusa em duas vozes (DECISÃO 20)** 🟢: `motivoParaOperador` (nomeia o campo/pessoa) vs
`motivoParaCliente` (não vaza nome de campo, diz o que fazer). Sinais de defasagem no resultado:
`fontesDefasadas`, `agendaExternaNuncaLida`, `googleCoberturaParcial`, `publicouHorarios`, `fusoSuposto`.

### 6.4 O que ocupa vs o que só parece (`agenda/ocupados.ts`, `ocupacao-externa.ts`, `jornada.ts`) 🟢

Princípio: **oferecer de menos se recupera; marcar em dobro não** → na dúvida, OCUPA. `LIBERAM_O_HORARIO
= {cancelled, no_show}` — todo o resto (inclusive status desconhecido e `pending`) ocupa. `OCUPA_NO_GOOGLE`
(Record exaustivo): confirmed/tentative ocupam, cancelled não; `transparent` nunca ocupa.
`ocupadosDoDono` usa `CONEXAO_CONTA_COMO_OCUPACAO[situacao] ?? true` e junta toda situação não-`healthy`
em `fontesDefasadas`. `agendaExternaNuncaLida` distingue "sem Google" de "Google nunca sincronizou"
(todas as conexões vivas com `last_sync_at === null`).

`lerJornadaDoBanco` (`jornada.ts`) é a fronteira entre o jsonb do banco e o motor puro: null/undefined =
"ainda não configurada" (≠ "mal configurada"); `fusoSuposto` capturado ANTES do parse (o schema tem
default `America/Sao_Paulo` que apagaria a distinção); fail-closed na AÇÃO, aberto na INFORMAÇÃO.

`lerOcupacaoExterna` (`ocupacao-externa.ts`) faz uma RPC por dono **em paralelo** (um dono falhando não
derruba os outros), fatia à janela visível (corrige issue #525: a tela mostrava livre o que o motor
recusava). `donosDaAgenda` usa admin client (RLS de `user_organizations` só devolve a própria adesão a
viewer/agent) e lê membros com `revoked_at is null`.

### 6.5 Sincronização com Google Calendar (`agenda/google/*`, ~3.700 linhas) 🟢 — a metade mais cara

Duas vias sobre a API Calendar v3 + OAuth2, com **dependência externa isolada** e regras densas.

**OAuth e configuração** 🟢: `config.ts` resolve o app OAuth da instalação DB-first (`platform_google_oauth`
id=1, cifrado) com fallback `.env` (`GOOGLE_CALENDAR_CLIENT_ID/SECRET`) — DECISÃO 3.1: sem as duas vars a
agenda funciona inteira, só o botão "Conectar Google" some. `oauth.ts` é puro (sem rede/env/relógio):
`ESCOPOS_OBRIGATORIOS = [calendar.events, calendar.readonly]` (NÃO userinfo), `access_type=offline`,
`prompt=consent select_account`; quatro armadilhas medidas documentadas (refresh_token só no primeiro
consent; renovação omite refresh_token e scope → `fundirTokens` preserva; expires_in é relativo). `estado.ts`
assina o `state` (HMAC-SHA256, carrega a PESSOA org.user.nonce.expira); `vinculo.ts` é o cookie de vínculo
navegador↔state (SameSite=Strict), prova mesmo navegador em 10min mas não mesma pessoa autenticada (lacuna
documentada, correção real = PKCE). `token.ts` faz as duas chamadas de rede (nunca lançam; leem o corpo
mesmo no erro — invalid_grant vem como HTTP 400).

**Tradução de campos (`evento.ts`, 546 linhas, o mais arriscado)** 🟢: assimetria proposital —
`paraEventoDoGoogle` LANÇA (nossos dados), `doEventoDoGoogle` NUNCA lança (dados de terceiro). `STATUS_PARA_GOOGLE`:
completed/no_show ficam `confirmed` (ocuparam o slot). Sempre escreve `dateTime`+`timeZone` (nunca `date`),
`transparency:"opaque"`, `reminders:{useDefault:true}` (o lembrete do cliente vai por WhatsApp).
⚠️ `id` e `iCalUID` NÃO são enviados juntos (HTTP 400 em produção 2026-09-01). Evento all-day usa
`end.date` EXCLUSIVO (não subtrai um dia). `SUFIXO_ICAL_UID = "deskcomm.app"` é FIXO, nunca marca própria
resolvida (é identidade técnica, fora do DOM).

**Classificação de erro (`erros.ts`, tabela de desfechos)** 🟢: `classificarErroDoGoogle(erro, operacao)`
→ `DesfechoDoGoogle` (reautenticar, recuar, sem_permissao, evento_sumiu, calendario_sumiu, ressincronizar,
ja_esta_feito, transitorio, permanente). Lógica central: app-errado→permanente; invalid_grant→reautenticar;
401→reautenticar; 429→recuar; 403→(cota? recuar : sem_permissao); **404/410 tratados por operação** — o
410 tem duplo sentido (DELETE=`ja_esta_feito` vs sync incremental=`ressincronizar`, syncToken morreu = risco
de apagar tudo). `estadoDaConexaoApos(desfecho)` mapeia para `SituacaoDaConexao` (só grava os cinco estados
decididos pelo sistema; transitorio/ressincronizar/evento_sumiu → null, não toca a conexão).

**Reconciliação (`sync-model.ts` + `sync-executor.ts`)** 🟢: contrato de **merge de três pontas** —
outbound guardado como **hashes SHA256, não PII** (`groups = [title, description, location, guest]`).
`compare(base, local, remote)` classifica em `accept_remote | publish | converged | conflict`; sem base,
converged-ou-conflict(legacy); shared conflita se os dois lados moveram diferente; outbound conflita se um
grupo sujo mudou remotamente para um terceiro valor. `reconcileAppointment(...)` é a máquina de estados por
compromisso (RPC `fn_google_appointment` com ações claim/commit/meet/idle/prepare/renew/error/release),
sempre com `release` no finally; trata cancelado-antes-de-publicar, identidade ausente, revalidação por
GET-exato (não list), conflito de série (recorrência), e injeta `conferenceData.createRequest` para Meet
quando `meeting_state==="pending"`. `calendar-executor.ts` faz sync incremental (`fn_google_calendar`:
claim/renew/item/page/error/reset/release; 410→reset) e refresh de catálogo (`fn_google_catalog`).
`escrita.ts` dá identidade estável ao evento (`idDeEventoDoGoogle`, charset [a-v0-9]) e reconhece o próprio
eco (`ehEventoNosso`).

**Meet (`meet.ts`, `meet-delivery.ts`, `motivo-do-meet.ts`)** 🟢: `meetVideoUrl` valida estritamente
(https, host exato `meet.google.com`, sem porta/query/hash); `observeMeeting` acompanha o `createRequest`.
A entrega do link passa por gate de autorização (`fn_meet_delivery_policy`, fronteira de serviço + comando
humano/gate de IA). `motivoDoMeet` traduz recusa do banco para a tela **sem nunca dar 5xx a recusa
conhecida** (5xx faria o cliente repetir 3× e travar ~20s), casando por nome primeiro e SQLSTATE depois.

**Lembretes (`agenda/lembretes.ts`)** 🟢: empacota/desempacota degraus (`LEMBRETE_MIN=15min`,
`LEMBRETE_MAX=10080min`=7d, `TETO_DE_LEMBRETES_EXTRAS=20`); dedupe por minuto, ordena desc, primeiro é o
principal; texto próprio vence, principal cai para `reminder_body`, extra NÃO herda o principal.

**Proteção de follow-up (`agenda/protecao-followup.ts`, `efeito.ts`)** 🟢: `protecaoDaAgenda` decide se o
follow-up deve ADIAR quando há compromisso vivo (motivos: agendado, em_atendimento, presenca_pendente,
presenca_vencida) com horizonte `fim + unknown_protection_minutes`. `efeito.ts` usa `AsyncLocalStorage`
para impor fronteira de serviço (`assertAgendaEffect*` verifica correnteza via RPCs e lança
`StaleServiceBoundaryError`/`AgendaDeferredError`).

### 6.6 Financeiro (`lib/financeiro/comanda.ts`, `catalogo.ts`) 🟢

**Comanda (`comanda.ts`)** — o schema (migration 0240) garante os invariantes (nada é apagado, saldo
derivado, lançamento pago imutável, comissão congelada na linha); sobram duas contas puras que precisam
acontecer ANTES do insert:
- **`percentualDaComissao(regras, alvo)`** — precedência **pessoa+serviço > pessoa > serviço > zero**
  (regra mais específica vence a geral); empate no mesmo nível resolve pelo MAIOR percentual (errar a favor
  de quem trabalhou não precisa ser explicado depois). Comissão errada é dinheiro errado no bolso de quem
  trabalhou — não se descobre por teste de tela.
- **`totalDoItem({quantidade, precoUnitarioCents, descontoCents})`** — piso em zero (desconto maior que o
  item vira item de graça, não erro; desconto da COMANDA não entra aqui, só o do ITEM — abatimento de caixa
  não reduz o combinado com quem atendeu).
- Schemas Zod: `itemSchema`, `abrirComandaSchema` (idempotente por `appointment_id` — faturar conclui o
  compromisso via `fn_finalizar_comanda`), `alterarComandaSchema` (cancelar é mudança de STATUS, nunca
  delete), `finalizarSchema`, `estornarSchema`.

**Catálogo financeiro (`financeiro/catalogo.ts`)** — `ENTIDADES_DO_CATALOGO` mapeia 5 entidades para
tabelas: contas→`financial_accounts`, formas_de_pagamento→`payment_methods`, planos_de_conta→
`account_plans`, regras_de_comissao→`commission_rules`, recorrencias→`recurring_entries`. **Três schemas
explícitos, não um CRUD genérico** (a divergência é o que importa: conta tem saldo inicial+moeda, forma
aponta pra conta, plano tem direção). `TIPOS_DE_CONTA = [cash, bank, other]`; `DIRECOES = [in, out]`.
`opening_balance_cents` assinado (conta pode nascer negativa); `planoDeContaSchema.direction` sem default
de propósito (obriga a escolher — no sistema de origem 17 linhas eram todas `debito`); `recorrenciaSchema.
day_of_month` aceita até 31 (o cron resolve a queda para o último dia existente).

### 6.7 Catálogo de produtos (`lib/catalogo/busca.ts`, `planilha.ts`, `moeda-da-org.ts`) 🟢

**Busca difusa (`busca.ts`)** — a regra que decide tudo: **PALAVRA é difusa, NÚMERO é exato**. Não é
`ilike '%termo%'` (cliente escreve "ifone 15", "15 pro max 256"), nem trigrama da frase inteira (a métrica
pune tamanho). `tokenizar` separa: palavra → prefixo/Levenshtein com corte; número → casamento exato com
sufixo de unidade curto (`256` casa "256gb", NÃO casa "1256" nem "153ml"). O número **FILTRA, não
ranqueia**: um número dito que o produto não tem ELIMINA o produto (`pontuar` devolve `null`, ≠ nota zero) —
é o que impede o 128GB de aparecer para quem pediu 256GB (nenhum limiar separa 128 de 256, e é aí que o
preço erra). Notas medidas: inicial vale mais que distância ("ifone"→"iphone" na frente de "fone"); prefixo
exige ≥3 letras; distância 1 para ≥4 letras, distância 2 só para ≥5. **`buscarComRelaxamento`**: quando o
filtro numérico esvazia o resultado, refaz sem os números que o catálogo não conhece e devolve MARCADO como
relaxado (`ignorados`), para o agente CONFIRMAR em vez de responder preço errado ("quero 2 iphone 15" acha
relaxando o "2"). Empate é preservado (a ambiguidade é real, quem escolhe é a pessoa).

**Importação por planilha (`planilha.ts`)** — reusa `parseCsv` (RFC 4180, detecta `;` do Excel pt) para
não criar segunda verdade sobre CSV; só o mapeamento é do domínio. `COLUNAS` mapeia sinônimos pt/es por
campo. `codigoDoProduto` dá identidade estável (reimportar ATUALIZA em vez de duplicar): corte em 60 chars
com assinatura FNV-1a de 32 bits quando estoura (corte seco dava o MESMO código a "…Cor Manhã" e "…Cor
Noite"). Recusa o ambíguo com o valor cru na mensagem (preço ilegível vira linha recusada, nunca chute —
chute aqui é preço errado dito ao cliente). Coluna de estoque AUSENTE ≠ estoque zero (`controla_estoque`).

**Moeda da organização (`moeda-da-org.ts`)** — `moedaDaOrganizacao(supabase, orgId)` é UMA função (não
select inline) porque dois caminhos de escrita (cadastro + import) precisam responder igual (senão o
catálogo teria duas moedas na mesma org, que a coluna existe para impedir). Fallback `BRL` (= o default da
coluna, comportamento de antes da feature) mas com RASTRO em console+Sentry (silêncio faria uma org em MXN
gravar em BRL até o cliente reclamar). NUNCA aceita a moeda de quem chamou (corpo não decide unidade, como
não decide escopo).

### Estruturas de dados principais (unidade 6) 🟢
Detalhadas em `data-dictionary.md`. Resumo: tabelas `calendar_appointments`, `calendar_event_types`,
`calendar_availability_exceptions`, `attendant_availability`, `calendar_connections`,
`calendar_external_events`, `platform_google_oauth`, `crm_lead_links` (vínculo polimórfico), e as
financeiras `financial_accounts`, `payment_methods`, `account_plans`, `commission_rules`,
`recurring_entries`, `sale_orders`/`sale_items` (comanda, migration 0240), `catalog_products`. DTOs puros:
`Slot`, `Ocupado`, `JornadaDaAgenda`, `ExcecaoDeData`, `TipoDeAgendamento` (motor); `TipoDeAtendimento`,
`AgendamentoListado`, `ResultadoDaConsulta` (consulta); `TokenDoGoogle`, `EventoDoGoogle`,
`CorpoDeEventoDoGoogle`, `Projection`/`Base`/`Comparison` (sync); `RegraDeComissao`, `LinhaImportada`,
`ProdutoBuscavel`, `Achado`/`BuscaRelaxada`.

### Dependências (unidade 6) 🟢
- `lib/agenda/*` → `@/lib/schemas/routing` (`availabilityScheduleSchema`, `ScheduleWindow`),
  `@/lib/schemas/settings` (`agendaSettingsSchema`), `@/lib/routing/eligibility` (`localMoment`),
  `@/lib/tempo/fusos` (`fusoValido`), `@/lib/leads/activity-vocabulary` (`ActivityType`),
  `@/lib/atendimento/fronteira` (`ServiceBoundary`), `@/lib/ai/elegibilidade/gate`, `@/lib/money`,
  `@/lib/i18n/*`, `date-fns`.
- `lib/agenda/google/*` → API Google Calendar v3 + OAuth2 (externo), `@/lib/webhooks/secrets`
  (decrypt/encrypt), `crypto` (HMAC), `zod`, `pg`/Supabase admin.
- `lib/financeiro/*` → `zod` (schemas puros; a rota faz a escrita e as RPCs `fn_finalizar_comanda`).
- `lib/catalogo/*` → `@/lib/contacts/csv` (`parseCsv`), `@/lib/schemas/produtos` (`precoParaCentavos`),
  `@/lib/money` (`MOEDA_PADRAO`), `@sentry/nextjs`.

### Complexidade 🟢
- Sincronização Google (OAuth + merge de três pontas + máquina de estados por compromisso + tabela de
  desfechos de erro + Meet): **muito alta**.
- Motor de horários livres puro + fuso com DST de duas passagens: **alta**.
- Coleta do banco `consulta.ts` (client injetado, recusa em duas vozes, vínculo polimórfico): **alta**.
- Busca difusa token-a-token com relaxamento: **média-alta**.
- Importação por planilha (identidade estável, recusa do ambíguo): **média**.
- Comanda (precedência de comissão, total do item): **média** (lógica pequena, doutrina densa).
- Catálogo financeiro (schemas explícitos): **baixa-média**.

### Discrepâncias / lacunas 🔴🟡
- 🔴 As RPCs security-definer (`fn_google_appointment`, `fn_google_calendar`, `fn_google_catalog`,
  `fn_agenda_ocupacao_google_do_dono`, `fn_agenda_conexoes_google_do_dono`, `fn_google_coverage`,
  `fn_meet_action`, `fn_meet_delivery_policy`, `fn_finalizar_comanda`, `fn_degraus_de_lembrete_validos`)
  são chamadas mas vivem em `supabase/baseline.sql` — a lógica interna delas é lacuna para o Data Master.
- 🔴 A migration 0240 (invariantes da comanda: nada apagado, saldo derivado, lançamento pago imutável,
  comissão congelada) é referenciada em prosa; os CHECK/triggers exatos ficam para o Data Master.
- 🟡 O TTL de autorização de entrega do Meet lê uma env var via `ttlDaAutorizacaoMs(process.env)` cujo nome
  exato não aparece nestes arquivos (INFERIDO).
- 🟡 `AUDIOSOCKET`-style: alguns componentes de tela (`components/agenda/paleta.ts`, `corDaTrilha()`) e o
  cron mensal de recorrências são citados mas vivem fora de `lib/agenda|financeiro|catalogo`.
- 🟡 A ponte com `crm_lead_activities` para agendamento de CONTATO sem lead (`emitAgentActivityForContact`,
  `lead_id NOT NULL`) é referenciada de `lib/leads/` — cobertura completa é da unidade 5.

---

## Unidade 7 — Automação e roteamento: `lib/automation/`, `lib/routing/`, `lib/followup/`, `lib/escalacao/`

### Propósito 🟢
Os quatro subsistemas que decidem **o que acontece sozinho** e **para quem vai**, todos disparados
por eventos do `event_log` ou por crons, e todos multi-tenant (service role que **bypassa RLS**, então
cada consulta filtra `organization_id` de fonte confiável — a linha do evento, nunca o body):

- **`lib/automation/`** — motor de regras "quando isto acontecer, faça aquilo". Consome eventos-gatilho,
  avalia condições em AND, executa ações (add-tag, mover lead, webhook, WhatsApp/IA, iniciar follow-up).
- **`lib/routing/`** — roteamento de conversa para atendente humano (rodízio real, elegibilidade
  tz-aware, fila). Também alimenta a fila do handoff.
- **`lib/followup/`** — motor de cadência/sequência: um **grafo de nós** (trigger/wait/condition/
  ai_classify/match_reply/repeat/action/end) percorrido no tempo por um worker de claim, com estado
  em event sourcing.
- **`lib/escalacao/`** — passagem do agente de IA para o humano e a volta: registro imutável da
  passagem, briefing, aviso ao cliente, aviso ao suporte (WhatsApp da equipe), disponibilidade,
  retomada e devolução automática.

Padrão transversal das quatro: **regra pura + adaptador(es) fino(s) de I/O**. A decisão vive numa
função testável sem DB; a leitura/escrita é reescrita por backend (`supabase-js` nas rotas Next,
`pg.Pool` no worker 24/7). A `escalacao` chega a ter dois adaptadores porque **dois motores de IA**
(`agent-engine` em `pg`, `ai/handoff` em `supabase-js`) passam conversa a humano por encanamentos
diferentes.

### 7.1 Motor de regras — engine e condições (`automation/engine.ts`, `conditions.ts`) 🟢

**`runAutomationForEvent(admin, row): Promise<HandlerResult>`** (`engine.ts`) — registrado no dispatcher
do `event_log` via `engine.handler.ts` (consumer key `automation-rules`). Fluxo:

1. **Anti-loop (profundidade 1):** evento com `metadata.caused_by_rule` OU `metadata.request_id`
   prefixado `"rule:"` retorna `skipped:caused_by_rule`. Cadeia regra→regra fica para v2.
2. **Guard de `entity_kind`:** `EXPECTED_ENTITY_KIND[row.event_type]` (de `ENTIDADE_ESPERADA_POR_GATILHO`,
   `lib/schemas/webhooks`). O trigger legado emite `lead.*` com `entity_kind='lead'` e os handlers
   novos com `crm_lead` — sem este filtro a regra rodaria 2× por mudança de lead.
3. **Seleção:** `automation_rules` ativas do tenant com `trigger_event = row.event_type`, `is_active`,
   ordenadas por `created_at`.
4. **Evento dirigido:** `regraDoEvento(row.payload)` (`gatilho-de-data-do-funil.ts`) — se o payload traz
   `rule_id` (só o gatilho `lead.date_field_due` da varredura `cron/lead-date-field-due` traz), a
   seleção se restringe àquela regra. Duas regras de data com `dias` diferentes não disparam juntas.
5. **Contexto:** `buildContext(admin, row)` hidrata `{ event, lead?, contact?, appointment? }` a partir
   do `entity_kind` — sempre filtrando `organization_id` do evento. Contato de compromisso vem do
   **COMPROMISSO** (linha de agora), não do payload (linha de quando o evento nasceu).
6. **Aplicáveis:** `evaluateConditions(rule.conditions, context)`.
7. **Pré-checagem de postpone (all-or-nothing):** ANTES de executar qualquer ação, roda
   `executor.postponeUntil` de cada ação; se alguma devolve um ISO, grava `automation_rule_runs`
   `status='adiado'` (`registrarAdiamento`, fire-and-forget) e retorna `{ status:'retry', retry_at }`.
   Reexecução parcial no retry seria pior que atraso.
8. **Execução:** cada ação por `getAction(type).execute(...)`; ação desconhecida → `failed:unknown_action`;
   exceção → `failed` com a mensagem.
9. **Agregador honesto** — o status do run é DERIVADO: `naoEnviadas = failed + skipped`;
   `adiados = postponed`. `naoEnviadas>0 ? (todas ? "failed" : "partial") : adiados>0 ? "adiado" : "success"`.
   Falha/skip vencem adiamento (a ordem importa: "algo quebrou" > "está a caminho"). `skipped` entra
   junto de `failed` porque "não saiu e a fila não resolve" (bug medido: envio sem contato/consentimento
   aparecia "Sucesso" verde). `audit()` só em não-`success`. `run_count` por read-modify-write (contador
   informativo, não invariante).

**`conditions.ts`** — filtros simples em **AND**: `ConditionOp = eq | neq | contains`; `RuleCondition
{ field, op, value }`. `resolveField(context, "a.b.c")` navega por path. Regras não-triviais:
campo ausente/null → condição falsa (nunca erro), exceto `neq` (ausente satisfaz `neq`). `contains`
tem dois significados por tipo (issue #956, decisão do dono 16/09): em **lista** é pertinência da tag
inteira sem caixa (`"Google"` pega `google`, NÃO pega `google ads` — atualização não pode fazer uma
regra alcançar quem não alcançava); em **texto** é `includes` sem caixa. Coerção via `String()`.

### 7.2 Catálogo de ações (`automation/actions/*`) 🟢

Registro por side-effect: `register-all.ts` importa os 7 executores; cada um chama `registerAction()`
no `Map` de `actions/index.ts`. Contrato `ActionExecutor { type, postponeUntil?, execute }`;
`ActionResultDetail { type, status: success|failed|skipped|postponed, error?, detail? }` (`types.ts`).

| Ação | O quê | Notas de regra |
|------|-------|----------------|
| `add_tag` | merge idempotente de tags no lead (ou contato) | emite `lead.tag_added`/`contact.tag_added` com `caused_by_rule` (a ação carrega o anti-loop) |
| `assign_owner` | seta `owner_user_id`+`assigned_at` do lead | valida membership ativa (role > viewer, G3-04); 3 respostas: membro / `user_not_in_org` / `membro_indeterminado` (infra caiu ≠ não é membro) |
| `create_or_move_lead` | cria ou move o negócio | reusa handlers de `/api/v1/leads`; recusa mover entre funis; negócio aberto em OUTRO funil é **transferido** (clone + encerra origem `moved_to_another_pipeline`, issue #992); `publicaNoContexto` injeta a linha inteira para as ações seguintes |
| `call_webhook` | POST `{event, occurred_at, data}` | HMAC-sha256 opcional (`secret_enc` cifrado tem precedência); anti-SSRF `assertSafeOutboundUrl` + `assertDestinoResolvidoSeguro` (resolve o IP); `redirect:"manual"` (3xx = falha, nunca seguido); retry 3× (1s/5s), timeout 10s; projeta só `LEAD_PUBLIC_FIELDS`/`CONTACT_PUBLIC_FIELDS` (nunca a row inteira) |
| `send_whatsapp_message` | template `{{campo}}` via `sendMessageHandler` | guardas de contato + `postponeUntil` (janela/cap); desfecho pelo ESTADO da mensagem |
| `send_ai_message` | texto escrito por agente publicado | irmã da anterior (mesmas guardas importadas); gate pré-go-live antes da chamada paga; `autorizarContatoParaIA`; carimbo `textoEscritoPelaIA:true` |
| `start_message_flow` | inscreve o contato num follow-up | reusa `enrollFollowupFlow`; conflito de inscrição viva (23505) → `failed:live_enrollment_exists` |

**Guardas compartilhadas de envio (`guarda-do-contato.ts`)** — `checarGuardasDeContato(ctx)` corre 4 na
ordem mais-barato-falha-primeiro: existe contato → não bloqueado → tem telefone → **consentimento**.
O gate de consentimento é FIXO (não uma `condition` declarável, para não ter exceção por regra mal
configurada) e lê `consent.marketing.declined_at` — **a recusa registrada**, NÃO a ausência de
`granted_at`. Motivo medido: o único escritor de `granted_at` é o mapeador do Respondi; bloquear por
ausência pararia TODA automação de WhatsApp para lead de webhook/importação/inbound, sem tela para
consertar. `MotivoDeBloqueio = no_contact | contact_blocked | no_phone | consent_declined`.

**Desfecho do envio (`desfecho-do-envio.ts`)** — `sendMessageHandler` NÃO lança em falha: marca a linha
`failed`/`queued` e devolve normal. `desfechoDoEnvio(tipo, msg)` traduz o estado REAL: `sent|delivered|read`
→ `success`; `queued` → `postponed` (com `queued_reason`, ainda a caminho); resto/`failed` → `failed`
(desconhecido nunca vira sucesso — falha aberta na informação). `reportarEnvio` anexa a conversa e, só em
`failed`, abre aviso na Central (`kind='message_send_stuck'`) com **janela de silêncio de 15 min**
(agrupa a rajada de 200 leads de formulário em 1–2 avisos; resolvido nunca silencia; busca que falha
avisa mesmo assim).

### 7.3 Janela de envio e throttle (`automation/janela-do-canal.ts`, `throttle.ts`) 🟢

**Uma régua só de janela, no fuso do tenant.** Existia uma segunda régua local (`withinSendWindow` 7h–22h
com `new Date().getHours()` — hora do processo, UTC no contêiner: virava 4h–19h de Brasília e represava
envios). Foi REMOVIDA. Agora `janela-do-canal.ts` lê `channel_knobs` (mesma tabela e defaults do pacing
do agente) e chama a regra pura de `agent-engine/pacing/engine` (`janelaDeEnvioAberta`,
`proximaAberturaDaJanela`). `knobsDoCanal` cai nos `PACING_DEFAULTS` (7–22h) no fuso da org quando não há
linha; falha de leitura → default nomeado no log (falha ABERTA, não cala o envio).
`adiarAteAJanelaAbrir(...)` devolve `null` (pode enviar) ou o ISO da próxima abertura (vira `retry_at`).

**`throttle.ts`** — `checkDailyLimit` lê `channel_sessions.daily_message_limit` (def 300) vs
`channel_session_warmup.messages_sent` do dia. ⚠️ **Ramo do cap inalcançável hoje:** `channel_session_warmup`
não tem escritor (quem conta é `pacing_ledger`), então `sent` é sempre 0 — documentado de propósito, com
o defeito de `setHours` (relógio do processo) marcado para quem for reanimá-lo. `espacarEnvio(sessionId)`
= espaçamento 1200ms + jitter (0–800ms) por sessão, estado de módulo COMPARTILHADO com a ação de IA (duas
cópias do contador dariam rajada pelo mesmo número = padrão de ban).

### 7.4 Roteamento — decisão pura (`routing/decide.ts`, `eligibility.ts`) 🟢

**`decideRouting(input): RoutingAction`** (`decide.ts`) — toda a lógica de branch do worker, sem DB nem
relógio implícito (`now` injetado). `RoutingAction = assign{userId} | skip{reason} | requeue{nextAttemptAt,attempts}`.
Regras (spec 13 §5 / acceptance AT-03): `alreadyAssigned` → `skip:already_assigned` (idempotência);
`mode==='manual'` → `skip:manual_mode`; `mode!=='round_robin'` → `skip:unsupported_mode` (`load` é
inalcançável hoje, tratado defensivamente); senão `selectRoundRobin`; sem elegível → `requeue` com
backoff da config (não hardcoded; `slow` a partir de `max_retries` usa ≥900s).

**`selectRoundRobin(eligibles)`** — rodízio REAL (não random): quem recebeu atribuição há mais tempo (ou
nunca, `lastAssignedAt=null=-1`) vem primeiro; desempate determinístico por `userId`. Deriva o "último
atribuído" de `conversation_assignment_events` — sem coluna de estado.

**`eligibility.ts` (puro, tz-aware via `Intl`)** — a regra de "quem pode atender agora":
- `isAttendantEligible(input, now)` = `isAvailable ∧ currentLoad < capacity ∧ isWithinSchedule`.
- `estaDePlantao(input, now)` = `isAvailable ∧ isWithinSchedule` (a mesma conta sem capacidade; a tela
  e o roteador têm de dizer o mesmo sobre a pessoa).
- `isWithinSchedule(schedule, now)` — `windows` vazio ⇒ 24/7 (janelas RESTRINGEM, não habilitam);
  `localMoment(now, tz)` calcula dow+HH:MM no fuso via `Intl.DateTimeFormat` (respeita DST).
- ⚠️ **Não há mais auto-offline por presença.** Havia `HEARTBEAT_TIMEOUT_MINUTES=15` + cron que
  gravava `is_available=false` — mas o heartbeat **nunca teve emissor** (varredura 2026-09-11), então
  derrubava todo atendente ~15 min após o plantão e nada religava. A causa era de modelagem
  (`is_available` misturava decisão durável com presença efêmera). Agora `is_available` é só a
  decisão e o plantão é CALCULADO a cada leitura — "religa sozinho" sem cron (anti-pattern 5 ao contrário).
- `OPEN_LOAD_STATUSES = [open, pending, claimed, ai_handling]` — fonte única de "carga" (router e painel).

### 7.5 Roteamento — worker e elegíveis (`routing/worker.ts`, `eligibles.ts`, `queue.ts`) 🟢

**`runRoutingWorker(opts)`** (`worker.ts`) — cron TS que drena `conversation.routing_requested` do
`event_log` (trigger AFTER INSERT em conversations, migration 0040; **trigger NUNCA faz HTTP**). Mecânica
de claim = CAS `pending→processing` + `consumed_by` + `next_attempt_at`/`attempts` (mesma do
agent-dispatcher); recupera `processing` abandonado > 5 min. `processEvent` valida payload/UUID, lê a
conversa (só `open|pending|claimed|ai_handling`), resolve `organizations.settings.routing` por Zod
(`routingConfigSchema`, default manual), carrega elegíveis por canal, chama `decideRouting`, e executa:
`assign` via RPC `fn_channel_routing_claim` (trata `already_assigned`/`conversation_changed`→lost race;
`candidate_revoked`/`not_allowed`/`capacity_changed`→requeue) + **`adotarLeadsDoContato`** (o rodízio
distribui CONVERSA; sem isto o LEAD não muda de dono e no modo de visibilidade `own` a fila sumiria —
issue #144; adota só lead `open`, do contato, sem dono nenhum; best-effort). `requeue` no esgotamento
abre `fn_routing_unassigned_notice`.

**`loadEligibleAttendants(supabase, orgId, now, scope)`** (`eligibles.ts`) — UM algoritmo para 3 clientes
(worker cron, handoff v2, `crm_get_queue_status`). `scope=conversation_channel` respeita
`channel_routing_policies`/`_responsibles` (policy vazia = restrição explícita → `[]`; sem policy =
`legacy_unconfigured`). Junta membros ativos (`agent|manager|admin`), `attendant_availability`
(`is_available=true`), carga (conversas abertas por dono) e última atribuição, e filtra por
`isAttendantEligible`.

**`queue.ts`** — snapshot da fila (`crm_get_queue_status`): fila = sem dono ∧ `status='open'` (definição
canônica do badge); `avg_wait_seconds` medido de `awaiting_since` (não `last_inbound_at`, que o cliente
insistente reiniciava — issue #990); `getQueuePositions`/`getQueuePosition` na MESMA ordenação
(`ORDEM_DA_ESPERA`) que o inbox.

**`channel-policies.ts`** — `loadChannelRoutingSettings` projeta a config de roteamento por canal para a
tela (`legacy_unconfigured | restricted | restricted_empty`).

### 7.6 Follow-up — o grafo e seu schema (`followup/graph-schema.ts`) 🟢

O follow-up é um **grafo dirigido** validado por Zod. `NodeType = trigger | wait | condition | ai_classify
| match_reply | repeat | action | end`. Cada nó = `{ id, type, label(1..60), position{x,y}, config }`.
Configs (union discriminada):
- **`wait`** — `fixed{ duration_ms: 5min..90d, immune_to_reply? }` | `smart{ min_ms, max_ms, guidance? }`
  (`min<=max`). `immune_to_reply` só em `fixed`: a espera não é cortada nem cancelada por mensagem do
  cliente (cadência longa de 28 dias); em runtime vira status `dormente`.
- **`ai_classify`** — `classes[1..8]`, `branches?` (v2 estável), `grace_timeout_ms ≥15min`,
  `target: last_reply|summary`. Refine: `branches[].label` espelha `classes` na ordem.
- **`match_reply`** — `branches[1..8]{ id, label, op: eq|contains, pattern }`, `grace_timeout_ms ≥15min`,
  `save_to?`, `if_exists?: skip|overwrite|confirm` (roteia por texto, sem LLM).
- **`repeat`** — `max_count 1..20` (repete o caminho `body` N vezes, N vindo da resposta do contato).
- **`condition`** — `combinator: and|or`, `branching?: combined|per_check`, `checks[1..10]
  { id?, label?, field: lead_stage|tag|steps_taken|last_outcome, op: eq|neq|gte|lte|contains, value }`.
- **`action`** — `text{body}` | `ai_message{prompt_hint, fallback_template_id?}` | `template{template_id}`.
- **`end`** — `outcome: converted|exhausted|custom`, `note?`.

`flowGraphSchema` = `nodes[2..60]`, `edges[..120]` + **`superRefine` de integridade** (sem id de
nó/aresta repetido; toda aresta aponta para nó existente — o grafo corrompido do #586 parava aqui, não
adiante no disparo). `flowEdgeSchema { id, source, target, priority(def 0), condition }` com
`condition ∈ always | class_match{value} | cond_result{boolean} | branch{branch_id}`. Branches
reservados (contract-owned): `else`(FALLBACK), `no_reply`, `true`, `false`, `body`, `done`.

**Dialeto v1/v2 e o único leitor (`nodeBranches`)** — v1 casa aresta por rótulo/`class_match`; v2 por
`branch_id` estável (rename não desconecta a aresta). `nodeBranches(node)` é o ÚNICO ponto que lê o
dialeto (canvas, painel de aresta, validação de publish, roteamento). ⚠️ Migrar um nó v1→v2 exige
reescrever as arestas no MESMO instante: `classEdgeMatch` (roteamento) não tem a cortesia do
`branchIdForCondition` (canvas) — tela certa com roteamento errado, e nada acusa. Por isso o
`ClassifyForm` emite v1 de propósito.

### 7.7 Follow-up — decisões puras por nó (`followup/node-handlers.ts`) 🟢

**`processNode(input): NodeResult`** — switch puro por tipo de nó. `NodeResult = advance{next_node_id,
next_eval_at, reason?, repeat?} | wait{next_eval_at, wake_status?} | enqueue_turn{purpose, wake_status,
fixed_body?} | recheck{next_eval_at} | dead{reason} | complete{outcome, cancel_reason?} | fail{error}`.

**Reconstrução de estado por eventos (o mecanismo central).** Todo o estado vive em
`followup_enrollment_events`. Um nó `wait`/`ai_classify`/`match_reply`/`action` é entrado **duas vezes**
com o mesmo `current_node_id` enquanto `steps_taken` sobe de 1 em 1. "Já comecei esta espera?" =
`resolveWaitPhase(events, nodeId, steps)` = existe evento com chave `${nodeId}:${steps-1}`. 1ª entrada:
arma o timer / enfileira o turno. 2ª entrada (após `next_eval_at`): avança.

**`selectEdge(edges, from, match)`** — maior `priority` primeiro, match exato antes de `always` (fallback).
**`classEdgeMatch(node, classe)`** — unifica v1 (rótulo) e v2 (`branch_id`) para que o roteamento por
timeout (`no_reply`) e por classificação real concordem.

**`evaluateCheck`** (condition) — regra crítica: **desconhecido (`null`) NÃO satisfaz negação (`neq`)** —
senão um lead sem classificação tomaria todo ramo "não foi X" em silêncio. `tag` é multivalorado
(eq/contains = pertinência). `last_outcome` = a classe do último `ai_classify` (lida dos eventos por
`ultimoDesfechoDe`) — antes era `null` fixo no engine, controle decorativo.

**Timers e dead-man (constantes):** `BACKOFF_MS = [30s,60s,5m,15m,1h]` (por `attempts-1`, clamp);
`ACTION_RECHECK_MS=5min`, `ACTION_RECHECK_MAX_MS=1h`, `atrasoDoRecheck` = 5min dobrando até 1h;
`MAX_ACTION_RECHECKS=14` (dead-man da ação, conta só rechecks **ociosos desde o último `action_deferred`**
via `rechecksOciososDaAcao` — cobre ~11h, corrige o bug de a janela anti-ban 22h matar o envio como
`action_turn_never_completed`); `MAX_PLAN_RECHECKS=3` (dead-man do plano → **segue sem plano**, cada
espera cai no `max_ms`, nunca morre). `EVENTO_ACAO_ADIADA="action_deferred"` é a prova de vida do envio
estacionado. `MAX_STEPS=80`.

**Envio at-most-once (nó `action`):** enfileira o turno EXATAMENTE 1× por ocupância (`!actionEnqueued &&
!actionCompleted`); recheck com turno em voo = `recheck` (fica no nó, não reenfileira — 2º `job_id`
furaria o dedup do send sink → mensagem dupla). Confirmado (`actionCompleted`) → avança (sara corrida com
recheck). Esgotou o dead-man → `dead`.

### 7.8 Follow-up — worker, ponte de turno, reatividade (`followup/engine.ts`, `turn-bridge.ts`, `reactivity.ts`) 🟢

**`runFollowupTick(deps, opts)`** (`engine.ts`) — cron. `claimDueEnrollments` (RPC
`fn_claim_due_followup_enrollments`, lease 120s, limite 20) e `processEnrollment` por inscrição.
Claim falhando NUNCA lança (DB fora do ar não pode matar todo follow-up) mas marca `claim_falhou=true`
(`claimed:0` não pode esconder "o claim quebrou sempre"). `AdminClient` é interface estreita (não
`SupabaseClient`) para rodar contra o harness bare-Postgres e produção; dois adaptadores:
`createSupabaseAdminClient` (rotas) e `createPgAdminClient` (`turn-bridge.ts`, worker 24/7).

**`processEnrollment`** — assere as fronteiras (service boundary + agenda), guarda `MAX_STEPS`, carrega o
grafo pinado + nó atual + `LeadFacts` + eventos, computa fase/wokeEarly/contadores, chama `processNode`,
depois `applyResult`. `applyResult` grava evento idempotente (`${node}:${steps}`), detecta replay
(`inserted===false`) e a corrida `action_sent` vs `action_recheck` (aplica só o avanço, sem inventar
evento), monta o `EnrollmentPatch` por `result.kind`, enfileira o job de turno em `enqueue_turn`, abre o
aviso de no-show-esgotado ANTES de `completed`, e persiste a resposta em `match_reply`+`save_to`.
Falha de handler → `applyHandlerFailure` (backoff, `dead` ao estourar `max_attempts`). `markDead` grava
o item de Central ANTES de `status='dead'` (crash entre as escritas → tick futuro re-executa; duplicata
visível > perda silenciosa).

**`completeTurnForEnrollment(...)`** (`turn-bridge.ts`) — costura o fim de um turno do agente de volta na
inscrição. Guardas: inscrição existe, ainda no `nodeId`, e status numa **allowlist positiva**
(`active|waiting_reply` só — deliberadamente positiva para que todo estado NOVO pause por omissão; o
guard já teve de ganhar `paused_handoff`/`paused_manual` por regressão). Trata `skipped` (cancela),
`deferred` (estaciona em janela-abre + carência, grava `action_deferred` sem gastar a chave do passo),
`sent` (action→avança por `always`; match_reply→fica), `classified` (ai_classify→`classEdgeMatch`),
`planned` (trigger→`montarTimingPlan` + grava `timing_plan`).

**`reactivity.ts`** — reações dirigidas por evento, decididas por **status** (nunca carrega o grafo).
`applyReactivityEvent`: `message.received`→`reactToInbound`; `ai.handoff_triggered`→`reactToHandoffOpen`;
`ai.handoff_resolved`→`reactToHandoffClose`. Inbound: opt-out/STOP (contato `is_blocked`) **cancela TUDO
vivo** (`opted_out`, inclusive `dormente`); senão `cancel_on_reply` cancela `waiting_reply`+`active`
(`replied`) ou acorda (`inbound_woke`, `next_eval_at=agora`). A espera imune sobrevive por **não estar**
no conjunto que reage (status `dormente` ∉ {waiting_reply, active}), não por um `if`. Handoff pausa/retoma
por CONTATO (não por conversa): `handoff_policy allow|cancel|pause(→paused_handoff)`, retomável só pelo
close (`RESUME_GRACE_MS`).

### 7.9 Follow-up — inscrição, publicação, gatilhos, gate, varredura (`followup/*`) 🟢

- **`enroll.ts`** — `enrollFollowupFlow` valida ponteiro `active` + `active_version_id` (`flow_not_active`
  422), parseia o grafo, resolve o agente que pina (valida `agentId` da org ou `resolveAgentForAutomaticTrigger`),
  abre a fronteira e insere em `followup_enrollments` no trigger. `23505` → `conflict` 409 (**1 por lead**).
  Manual não passa pelo gate de agente.
- **`agent-followup-gate.ts`** — `isPointerEnabledForAutomaticTrigger` / `resolveAgentForAutomaticTrigger`:
  gatilho automático (silence/stage_change/case_opened/appointment_no_show, `GATILHOS_QUE_EXIGEM_AGENTE`)
  só inscreve se algum agente **publicado** tem `followup.enabled=true` e inclui o ponteiro; pick
  determinístico = menor uuid. `manual`/`webhook` isentos.
- **`silence-sweep.ts`** — `runSilenceSweep`: varre ponteiros de silêncio ativos, memoiza o agente por
  (org,ponteiro), corta contatos sem inbound desde `now - threshold` (filtrados por `segments`), inscreve
  1 por contato. `23505` → `skipped_existing`.
- **`gatilho-etapa.ts` / `gatilho-caso.ts` / `gatilho-presenca.handler.ts`** — gatilhos de mudança de
  etapa / caso aberto / presença; todos passam pelo gate de agente.
- **`publish.ts` / `validate-publish.ts`** — `publishFollowupFlowVersion` (RPC `fn_publish_followup_flow_version`,
  insert+ativa atômico). `validateFlowForPublish`: exatamente 1 `trigger`; todo nó alcançável do trigger;
  todo nó alcança um `end`; detecção de ciclo sem "espera suficiente"; `too_many_steps` (>`MAX_STEPS`);
  toda branch com aresta; checks de condição válidos.
- **`timing-plan.ts`** — `coletarEsperasAdaptativas`, `clampEspera`, `montarTimingPlan` (clampa cada
  proposta do modelo contra o GRAFO PINADO), `esperaPlanejadaDe` (leitor tolerante do jsonb, all-or-nothing,
  nunca lança). `no-show-recuperacao-esgotada.ts` — aviso de Central só em complete+`exhausted`+`appointment_id`.

### 7.10 Escalação — a passagem como fato (`escalacao/passagem.ts`, `briefing-da-passagem.ts`) 🟢

**`passagem.ts`** é o vocabulário + a escrita, compartilhada pelos dois motores. Tuplas (espelhadas contra
os CHECK do banco por `vocabulario-banco-x-typescript.test.ts`): `MOTORES_DA_PASSAGEM` (engine|crm),
`ORIGENS_DA_PASSAGEM` (13 origens de código), `MOTIVOS_DA_PASSAGEM` (9 razões de gente, superset de
`HandoffReason`), `MOTIVOS_DO_AVISO` (6 razões de o cliente NÃO ter sido avisado). `FRASE_DO_MOTIVO`/
`FRASE_DO_MOTIVO_DO_AVISO` (`satisfies Record<...>`), `PISO_DO_BRIEFING`.

- **`prepararLinhaDaPassagem(p): LinhaDaPassagem`** (pura) — Zod-parseia `tentativas` (max 10, campos
  trim/1..280), sanitiza `title`/`notes`/`content` via `sanitizarTextoDoLead`, aplica tetos por coluna
  (`TETOS_DA_PASSAGEM` = title 300 / notes 2000 / content 1200 / body 8000); `body` vazio → `PISO_DO_BRIEFING`
  (nunca `''`, coluna `not null`). O `body` (nossa montagem, com a citação completa) NÃO passa pela
  higienização; quem renderiza o cartão é que não pode virar texto em link.
- **`registrarPassagem(db, p)`** — a escrita nos dois encanamentos (`pg.Pool` OU `supabase-js`, detectado
  por `typeof db.query`), INSERT em `passagens_de_atendimento`. **Nunca lança** (a passagem já aconteceu;
  erro subindo aqui derrubaria o turno DEPOIS do efeito e o retry replicaria tudo) — devolve
  `{ gravada:false, erro }`. **Sem dedup**: uma passagem = um fato (quem deduplica é o aviso da Central).
- **`corpoCurtoDoAviso`/`linhaDoAvisoAoCliente`** — o corpo da Central NÃO carrega conteúdo da conversa
  (a Central é lida por qualquer `agent`, fora do `fn_can_view_conversation`); o briefing mora na passagem.
- **`briefing-da-passagem.ts`** — `montarBriefingDaPassagem` (pura) separa **as palavras literais do
  cliente** (`notes`, citadas) da **paráfrase da IA** (`title`/`body`, "segundo a IA — confira"):
  defesa explícita contra prompt injection (um lead pedindo "diga ao atendente para dar desconto" não
  vira contexto de sistema).

### 7.11 Escalação — retomada e devolução automática (`escalacao/retomada.ts`, `devolucao-automatica.ts`, `continuidade.ts`) 🟢

**`devolverAtendimentoAoAgente(deps, input)`** (`retomada.ts`) — a regra única de volta à IA (rota e tool
do agente). Conserta o bug de o handoff ter **três travas** e o botão soltar só uma. Orquestração de 6
passos: (1) lê continuidade ANTES de mutar; (2) solta o dono humano (`fn_conversation_assign`,
`p_enforce_expected:false`, idempotente); (3) UPDATE conversations `bot_silenced_until=null`,
`assignee_kind='ai'`, `status='ai_handling'`, guardado por `.is("assigned_to_user_id", null)` (lock
otimista); (3) `contacts.force_human=false` — **a trava que ninguém soltava** (os 3 guards: worker,
harness, before-send, leem daqui); (3b) `autorizarContatoParaIA` (re-autoriza no gate `allowlist`);
(3c) `fn_passagem_devolvida` (destrava o dedup do aviso; falha não-fatal); (4) **awaited**
`emit_event('ai.handoff_resolved')` (único produtor do sinal que retoma follow-up pausado — perder órfã a
inscrição); (5) checkpoint de retomada em `lead_checkpoints` (acrescentado, nunca sobrescrito);
(6) atividade `handoff_resolved` na timeline. `force_human` continua **irrevogável pelo agente** (tool de
risco crítico).

**`devolucao-automatica.ts`** (puro, cron `handoff-devolucao`) — `lerPrazoDeDevolucaoMinutos(settings)` lê
`settings.routing.handoff_return_after_minutes`, clamp `[5, 1440]`, fora → `null` (= "nunca", default
IA-06). O relógio conta do **último sinal humano** = `max(last_handoff_at, assigned_at, last_outbound_at)`,
fallback `status_changed_at` (nunca da própria passagem — interromperia uma sessão viva).
`avaliarDevolucao(c, sel)` recusa com motivos tipados (`sem_prazo`, `nao_esta_com_humano`,
`status_nao_devolvivel`, `sessao_sem_agente` — só devolve onde há quem atenda, `sem_relogio`,
`dentro_do_prazo`); dispara quando `agoraMs - sinal >= minutos*60000`. `estaComHumanoDuravel` =
`assigned_to_user_id!=null ∨ assignee_kind==='user' ∨ bot_silenced_until==='infinity'`.

**`continuidade.ts`** — `lerContinuidadeHumana` (read-only) junta `agent_cases`, `agent_case_events`
(`human_replied`) e `conversation_notes` (cada um cap 10); `montarResumo` produz a prosa que o agente lê
ao retomar.

**`atendimento-manual.ts`** — `pausarIaPorAtendimentoManual`: o dono respondeu pelo celular (caminho
`fromMe` da ingestão, fora do composer). Silencia o bot por `PRAZO_DO_SILENCIO_MS=60min` a contar da
ÚLTIMA fala humana, **extend-only** (nunca encurta; `Infinity`=handoff formal vence). NÃO toca
`force_human`/`ai_authorized_at`/`assignee_kind`/`status`. Fire-and-forget.

### 7.12 Escalação — aviso ao suporte (`escalacao/aviso-ao-suporte.ts` + handler + estado) 🟢

**`aplicaAvisoDeCaso(deps, row)`** — pipeline de ~16 passos consumindo `ai.case_opened`/`ai.case_closed`,
ordenado do mais barato ao mais caro (toda instalação que nunca ligou o aviso sai no passo 2 com 2 leituras
e zero rede). Passos: case_closed cancela pendentes → 0 dreno-em-request adia → 1 payload → 2 config
(`sem_configuracao`) → 3 teto de idade do evento (`IDADE_MAXIMA_DO_EVENTO_MS=30min`) → 4 caso ainda aberto
(`ORIGENS_ACEITAS`, `STATUS_ABERTOS`) → 5 **reivindica ANTES da rede** (INSERT único → `23505`=outro processo;
`JANELA_DE_REIVINDICACAO_MS=2min`) → 6 validade da entrega (`VALIDADE_DA_ENTREGA_MS=24h`) → 7 titular
anonimizado → 8 canal (arquivado/aceita-livre/status, `TETO_DE_TENTATIVAS_DO_CANAL=6`) → 9 URL pública →
10 transporte configurado (`TETO_DE_TENTATIVAS_DE_ENVIO=3`) → 11 pacing (espaçamento + cap diário de
warm-up, **sem janela de horário** — destinatário interno) → 12 destino → 13 texto → 14 envia → 15 marca
`enviado` com `corpo_hash` sha256 (texto NUNCA guardado) → `depoisDoEnvio`. Do passo 15 em diante tudo
falha **ABERTO** (a mensagem já está no celular). Falha definitiva por `condena` (status `falhou`/`cancelado`
+ item de Central `aviso_de_caso_nao_entregue` + audit). **Nunca retorna `error`** — só `ok|skipped|retry`
(o único `error` é o `catch` do handler, para defeito de programa). `TETO_ABSOLUTO_DE_TENTATIVAS=10`.
`mascara(telefone)` = só os 4 últimos dígitos.

**`estado-do-aviso.ts`** (puro, tela de config) — `avisosDaTela(f)` classifica 11 estados separando
`bloqueia:true` (trava o switch: `sem_conexao`, `so_canal_oficial`, `sem_endereco_publico`,
`conexao_removida`) de alertas (`conexao_fora_do_ar`, `conexao_atende_clientes`, `casos_desligados`,
`agente_assistido`, `atendimento_externo`, `aquecimento`, `descarte_acontecendo`). Ordem é contrato
(bloqueios primeiro). `agentesPublicados=null` → sem alerta ("não deu para medir" ≠ false).
`podeLigarOAviso` = URL pública ∧ telefone E.164 válido ∧ canal aceita mensagem livre.
`sanitizar-texto-do-lead.ts` — tira URLs, corridas de dígitos ≥8 (CPF/telefone), markdown do WhatsApp,
controles; devolve `null` (nunca `''`) quando nada sobrou.

**`numero-interno-de-aviso.ts`** — o corte que impede o número do suporte de virar lead: `ehNumeroInternoDeAviso`
roda ANTES de `upsertContact` em todo webhook; ignora `ligado` (configurado = interno mesmo pausado) e
falha **aberta** (não descartar mensagem de cliente de verdade). Cache memoizado 30s; contador visível
(`mensagens_ignoradas`) porque descarte silencioso parece bug.

### Estruturas de dados / entidades (destaque) 🟢
`RuleCondition`, `ActionResultDetail`, `ActionExecutor`, `ActionCtx` (automation); `RoutingCandidate`,
`RoutingAction`, `DecideRoutingInput`, `AttendantEligibilityInput`, `QueueStatus` (routing);
`FlowNode`/`FlowEdge`/`FlowGraph` (Zod), `NodeResult`, `EnrollmentRow`, `LeadFacts`, `EnrollmentEventRef`,
`EnrollmentPatch`, `TimingPlan` (followup); `PassagemNova`/`LinhaDaPassagem`, `DesfechoDoAvisoAoCliente`,
`ContinuidadeHumana`, `ConversaEmHandoff`/`Selecao`/`MotivoDeNaoDevolver`, `AvisoDb`/`TransporteDoAviso`/
`DesfechoDoAviso`, `FatosDaTelaDeAviso`/`AvisoDaTela` (escalacao). Detalhes de campos no `data-dictionary.md`.

### Dependências (unidade 7) 🟢
- **`lib/automation` →** `lib/event-log/dispatcher`, `lib/audit`, `lib/schemas/webhooks`, `lib/atendimento/*`
  (fronteira, origem-automacao), `lib/agenda/*` (efeito, protecao-followup), `lib/agent-engine/pacing/*`
  (janela — regra pura), `lib/channels/*` (frases-de-falha, capabilities), `lib/leads/*` (handlers,
  clonar-para-funil, encerramento), `lib/webhooks/secrets`, `app/api/v1/{messages,leads}/_handler`,
  `lib/followup/enroll`, `zod`, `node:crypto`.
- **`lib/routing` →** `lib/schemas/routing`, `lib/supabase/admin`, `lib/auth/types` (ROLE_RANK),
  `lib/users/nome-do-atendente`, `lib/channels/capabilities`, `lib/inbox/comando-da-conversa`,
  `lib/ai/agents/org-tem-automatico`, `Intl` (fuso), RPCs `fn_channel_routing_claim`/`fn_routing_unassigned_notice`.
- **`lib/followup` →** `lib/atendimento/{fronteira,origem}`, `lib/agenda/{efeito,protecao-followup}`,
  `lib/channels/contato-por-telefone`, `lib/audit`, `lib/logger`, `lib/agent-engine/queue/claim`, `zod`,
  `@supabase/supabase-js` + `pg` (dois adaptadores). RPCs `fn_claim_due_followup_enrollments`,
  `fn_publish_followup_flow_version`, `fn_followup_claim_current`/`fn_followup_job_current`.
- **`lib/escalacao` →** `lib/api/handlers/types`, `lib/audit`, `lib/leads/{activity-emitter,active-lead}`,
  `lib/ai/elegibilidade/{autorizacao,gate}`, `lib/agent-engine/{agent/declaracao,queue,pacing}`,
  `lib/routing/eligibility` (`isAttendantEligible`), `lib/auth/types` (ROLE_RANK), `lib/channels/*`,
  `lib/i18n/*`, `lib/branding/saida`, `lib/event-log/dispatcher`, `lib/automation/janela-do-canal`,
  `node:crypto`. RPCs `fn_conversation_assign`, `fn_passagem_devolvida`, `emit_event`,
  `fn_contar_mensagem_ignorada`, `fn_registrar_jid_do_aviso`.

### Complexidade 🟢
- Follow-up (grafo event-sourced, dois backends, dialeto v1/v2, dead-man/backoff, timing adaptativo):
  **muito alta** — `node-handlers.ts` e `engine.ts` são os pontos mais densos.
- Escalação (dois runtimes, pipeline de 16 passos com claim/retry/expire, retomada de 6 efeitos, PII):
  **muito alta** — `aviso-ao-suporte.ts` (~773 l) e `retomada.ts`.
- Motor de regras (anti-loop, postpone all-or-nothing, agregador honesto, `contains` por tipo): **alta**.
- Roteamento (decisão pura + worker de claim + rodízio real + adoção de lead): **alta**.
- Ações de automação (guardas compartilhadas, anti-SSRF, desfecho derivado): **média-alta**.

### Discrepâncias / lacunas 🔴🟡
- 🔴 RPCs security-definer chamadas mas com corpo em `supabase/baseline.sql` (lacuna do Data Master):
  `fn_claim_due_followup_enrollments`, `fn_publish_followup_flow_version`, `fn_followup_claim_current`,
  `fn_followup_job_current`, `fn_channel_routing_claim`, `fn_routing_unassigned_notice`,
  `fn_conversation_assign`, `fn_passagem_devolvida`, `fn_contar_mensagem_ignorada`, `fn_registrar_jid_do_aviso`.
- 🔴 CHECK/triggers de vocabulário (migrations 0292/0293/0347/0337 etc.) e os invariantes da comanda
  ficam para o Data Master; aqui só o lado TS.
- 🟡 **Ramo do cap diário de `throttle.ts` inalcançável** — `channel_session_warmup` não tem escritor em
  produção (quem conta é `pacing_ledger`), então `sent` é sempre 0 e o cap nunca dispara. Documentado no
  próprio código, com o defeito de `setHours` (relógio do processo) para quem reanimar.
- 🟡 **`load` como modo de roteamento é inalcançável** — `routingConfigSchema` só permite `manual|round_robin`;
  tratado defensivamente como no-op.
- 🟡 Migrar um nó de follow-up v1→v2 é operação atômica de canvas (reescrever arestas junto); fora disso,
  tela correta com roteamento errado e nada acusa (por isso `ClassifyForm` emite v1).
- 🟡 A predicado `isAttendantEligible` é a fonte única de elegibilidade, mas `disponibilidade.ts`
  (pg, engine-side) e `atendentes.ts` (supabase-js, API-side) são dois LEITORES — divergência entre eles
  seria silenciosa (mitigada por importarem a mesma função pura).

---

## Unidade 8 — Auth, tenancy e RBAC: `lib/auth/`, `lib/tenants/`, `lib/team/`, `lib/users/`, `lib/impersonate/`, `proxy.ts`

### Propósito 🟢
A camada que responde três perguntas de toda requisição: **quem é** (autenticação), **em que empresa
está** (tenancy), e **o que pode fazer** (autorização). Tudo multi-tenant com RLS desde o dia 1; os
helpers server-side resolvem a identidade do JWT validado ANTES de tocar qualquer tabela tenant-aware,
e quando usam service role (que bypassa RLS) filtram `user_id`/`organization_id` de fonte confiável
(JWT, cookie validado, path token, segredo de instalação) — **nunca do body**.

- **`lib/auth/`** — sessão (`server.ts`), guard canônico de RBAC (`require-role.ts`), guard de
  plataforma (`requirePlatformAdmin.ts`), política de MFA (`politica-mfa.ts`), política de cadastro
  aberto/só-convite (`politica-de-cadastro.ts`), token de convite HMAC stateless (`invite-token.ts`,
  `issue-invite.ts`, `aplicar-convite.ts`, `convite-no-signup.ts`), provisionamento de tenant
  (`provision.ts`), rate limit da superfície de auth (`rate-limit.ts`), códigos de recuperação MFA
  (`recovery-codes.ts`), auth de cron (`cron-auth.ts`), anti open-redirect (`safe-next.ts`), schemas
  Zod (`schemas.ts`), allowlist de borda (`public-paths.ts`), detecção de vínculo revogado
  (`vinculo-revogado.ts`).
- **`lib/tenants/`** — emissão/rotação da API key `dsk_...` de uma organização para integração externa.
- **`lib/team/`** — o convite como REGISTRO (`team_invites`): emissão, reenvio, status derivado.
- **`lib/users/`** — resolução do nome de exibição do atendente (com fallback declarado quando não há
  service role).
- **`lib/impersonate/`** — acompanhamento administrativo (platform admin agindo como tenant): cookie
  HMAC (server + edge) e contexto de suporte resolvido por RPC.
- **`proxy.ts`** — middleware de borda do Next 16 (renomeado de `middleware.ts`): injeta `X-Request-Id`
  e `x-pathname`, valida o JWT, bloqueia `/admin/*` para não-platform-admin, verifica o cookie de
  impersonation em `/app/*`.

Padrão transversal: **a decisão de acesso mora em `requireRole()` (API) e nos guards de layout
(`requireAuth`/`requirePlatformAdmin`), o role efetivo vem SEMPRE do banco (`fn_user_role_in_org`,
`fn_is_platform_admin`) — a MESMA função `SECURITY DEFINER` que as policies RLS usam** (fonte única de
verdade), nunca do snapshot em memória do cookie. Segredos são comparados em tempo constante
(`timingSafeEqual`); tokens só existem como hash SHA256 no banco, plaintext mostrado uma vez.

### 8.1 Sessão e organização ativa (`auth/server.ts`) 🟢

**`loadAuthUser(): Promise<AuthUser | null>`** — memoizada por request (`cache` do React). Fluxo:

1. `createClient()` (client de sessão, cookie) → `supabase.auth.getUser()` — **valida o JWT no
   servidor, nunca `getSession()`** (regra de segurança da doutrina; `getSession` confia no cookie sem
   revalidar).
2. **`ehSessaoAusente(error)`** distingue o estado NORMAL (`AuthSessionMissingError`, sem cookie:
   visitante deslogado, crawler) de uma falha transitória (rede, GoTrue fora do ar, token ilegível). O
   erro transitório é logado (`name`/`code`/`status`/`message`); o ausente é silêncio. Compara por
   `error.name` e **não** `instanceof` porque há duas cópias de `@supabase/auth-js` na árvore de
   `node_modules` (2.111.0 e 2.112.1) e `instanceof` só acertaria a cópia importada pelo teste.
3. Se `user` nulo → retorna `null` (falha FECHADA na ação: redireciona para login).
4. Em paralelo (`Promise.all`, elimina round-trip): `platform_admins` (ativo = `revoked_at` null) e
   `user_organizations` com dois embeds do MESMO `organizations` (nome/idioma/fuso pela relação padrão;
   `interface_settings` da empresa pelo alias `interface_da_empresa`), filtrado por `user_id` e
   `revoked_at` null, **ordenado por `accepted_at` nulls-first e depois `organization_id`**.
5. ⚠️ **FALHA ALTO, não baixo:** se qualquer das duas queries erra, **lança**
   `auth_permissions_unavailable` em vez de degradar para "usuário sem organização". Incidente medido
   em 2026-07-30: um restart do Docker deixou o PostgREST devolvendo `name resolution failed`, e o
   descarte silencioso do erro fez TODOS os cards de admin sumirem — um defeito de infra que parecia
   decisão de autorização, custou seis diagnósticos errados.
6. `readSupportContext` (acompanhamento), metadados do usuário (`full_name`, `avatar_url`, `locale`,
   `timezone`), e resolução de idioma via `normalizarIdioma(locale ?? support.locale ?? locale da org
   ativa)`.

**`escolherMembroAtivo(memberships, cookieOrg)`** — a organização ativa: se o cookie `active_org`
aponta para um membership existente, é ele; senão `memberships[0]`. Extraída porque DUAS coisas
dependem da mesma resposta e não podem divergir (`resolveActiveOrg` decide o escopo dos dados; a
resolução de idioma decide a língua da tela) — divergir mostraria dados de uma empresa com a interface
no idioma de outra. O `[0]` só é uma ESCOLHA porque a query ordena; sem o `ORDER BY` seria sorteio.

**`resolveActiveOrg(authUser): ActiveOrg | null`** — memoizada. Se há `support` (impersonation): status
≠ `active` → `redirect("/support-ended")`; o role é `admin` (full) ou `viewer` (readonly). Senão,
`escolherMembroAtivo`.

**`requireAuth()`** — garante usuário em Server Components de `/app/*`; redireciona `/login` se null.

### 8.2 MFA como política de SESSÃO, não só de cadastro (`auth/server.ts`, `politica-mfa.ts`) 🟢

Duas perguntas que **não são a mesma** (o achado central do módulo de MFA):

- **Preciso CADASTRAR?** É POLÍTICA. `exigeCadastroDeMfa({role, isPlatformAdmin, plataformaExige,
  empresaExige})` (`politica-mfa.ts`) — **as duas origens SOMAM, nunca se anulam**: platform admin com
  `platform_admins.mfa_required=true` é obrigado mesmo numa empresa que não exige; admin de tenant é
  obrigado se `organizations.settings.security.mfa_required=true` mesmo que a plataforma não exija. O
  default de ambos é **NÃO exigir** (`empresaExigeMfa` trata ausência como `false` — regra 6 de
  packaging: default preserva comportamento anterior). ⚠️ Isto deixou de ser a constante
  `isPlatformAdmin || role==="admin"`: como o `install.sh` cria o dono como platform admin, TODA
  instalação self-host forçava TOTP na primeira tela (um "sétimo passo" que a barra de progresso nunca
  anunciou). `platform_admins.mfa_required` já existia no schema, aparecia na tela de admin com badge, e
  **nunca era lido** — era controle decorativo até esta mudança.
- **Preciso PROVAR agora?** É SESSÃO. `mfaEmDivida()` (`server.ts`) é `true` só quando as TRÊS valem: o
  usuário JÁ cadastrou fator (`isMfaEnrolled`), e a sessão é `aal1` (`sessionAal` ≠ `aal2`). ⚠️ NÃO
  pergunta a política — quem TEM fator prova SEMPRE, senão o cadastro opcional viraria buraco (fator
  ignorado na sessão = o mesmo que não ter).

`requiresMfa(role, isPlatformAdmin, userId, orgId)` — carrega as duas leituras (`platform_admins`,
`organizations.settings`) e delega a `exigeCadastroDeMfa`. `sessionAal()` lê
`getAuthenticatorAssuranceLevel()` e estreita o tipo aberto do SDK para `aal1|aal2|null`.

### 8.3 Guard canônico de RBAC (`auth/require-role.ts`) 🟢

**`requireRole(min: Role, opts): Promise<RoleCheck>`** — o helper ÚNICO de autorização por role nas
rotas `/api/v1` (spec 13 §4, invariante G2-01). Reimplementar a comparação de rank direto na rota é
anti-padrão proibido ("matriz advisória"; gate `pnpm lint:role-rank`). Fluxo:

1. `loadAuthUser()` → 401 `unauthenticated` se null.
2. Support não-`active` → 403 `forbidden`.
3. Org resolvida: `opts.organizationId` (autorização sobre a org do RECURSO, ex. LGPD anonymize —
   admin na org do CONTATO, resolvida de fonte confiável) ou `resolveActiveOrg`. Ausente → 403
   `forbidden_tenant`.
4. `allowPlatformAdmin && is_platform_admin && !support` → **bypass** do rank do tenant (role
   transversal).
5. **Role efetivo do BANCO:** `rpc("fn_user_role_in_org", {p_org})` — não do snapshot do cookie; falha
   fechada se o membership foi revogado. Erro da RPC → 500 `internal_error`.
6. **Gate de MFA (posição deliberada):** DEPOIS do rank e ANTES do sucesso — `rank >= min && mfaEmDivida()`
   → audit `authz.denied` (reason `mfa_required`) + 403 `mfa_required`. Fica depois do rank para que
   quem não tem papel leve 403 por FALTA de papel, sem a resposta revelar o estado de MFA de quem nem
   chegaria lá. ⚠️ Existe porque o gate de MFA vivia em `app/app/layout.tsx`, e layout **não roda em
   rota de API**: uma sessão `aal1` de admin com TOTP cadastrado chamava direto as ~33 rotas gateadas
   por `requireRole("admin")` (criar token, convidar, LGPD anonymize, publicar agente).
7. `rank < min` → audit `authz.denied` (fire-and-forget: falha de audit alerta, não bloqueia) + 403
   `forbidden_role`.
8. Sucesso: `{ ok:true, user, org: {...org, role: effectiveRole} }`.

**Papéis (`auth/types.ts`):** `Role = viewer(1) | agent(2) | ai_operator(3) | manager(4) | admin(5)`.
⚠️ `ai_operator` é o papel do **AGENTE PUBLICADO**, existe SÓ no escopo do token efêmero — **nunca** em
`user_organizations` (cujo CHECK só tem os quatro papéis humanos: `PAPEIS_HUMANOS`). Senta entre `agent`
e `manager` porque descreve a faixa que faltava: capacidades que um atendente humano não tem (configurar
operação, régua de retorno) mas que o agente precisa (invariante 4 da doutrina — nenhuma demanda sem
próximo passo). `fn_role_at_least` no banco não conhece o papel, e está certo: o agente não é usuário,
a RLS segue intacta. **`roleAtLeast(role, min)`** é helper de leitura de rank (campo informativo,
regra de escopo sobre role já resolvido) — **NÃO** é gate de rota, não decide 401/403 sozinho.
**`VisibilityMode = all | own_and_unassigned | own`** (G4-01): escopo de conversa por atendente, só
restringe `agent`; a RLS (`fn_can_view_conversation`) é quem garante, não o campo.

### 8.4 Guard de plataforma (`auth/requirePlatformAdmin.ts`) e borda (`proxy.ts`) 🟢

**`requirePlatformAdmin(): PlatformAdminContext`** — guard server-side do sub-produto `/admin/*`. Valida
JWT (`getUser`), confirma linha ativa em `platform_admins` (senão `redirect("/admin/forbidden")`), e se
`mfa_required` exige AAL2 (senão `redirect("/login/mfa?next=/admin")`). É a checagem AUTORITATIVA
(roda no layout `/admin`, runtime Node, redirects baratos, acesso a AAL).

**`proxy.ts`** (matcher: tudo exceto assets/internals do Next):
1. Injeta `x-request-id` (correlação com audit/wrappers) e `x-pathname`.
2. `isPublicPath(pathname)` → passa sem auth.
3. `createServerClient` (cookie `sb-deskcomm-auth`, `SameSite=strict`, `httpOnly`, `secure` condicional)
   → `getUser()`. Null: `/api/*` responde JSON `401 unauthenticated` (nunca redirect HTML para
   consumidor JSON); UI redireciona `/login?next=...`.
4. Em `/app/*`: verifica o cookie de impersonation (HMAC + expiry no Edge, **sem DB**); inválido →
   deleta o cookie de apresentação (o banco continua autoritativo).
5. Em `/admin/*` (exceto `/admin/forbidden`, que evita loop): `rpc("fn_is_platform_admin")`; erro ou
   false → `redirect("/admin/forbidden")` (gate ANTECIPADO; o autoritativo é `requirePlatformAdmin`).

**`public-paths.ts`** — allowlist ordenada (primeira que casa vence). "Público" aqui significa **"o
proxy não decide"**, não "sem autenticação": muitas rotas listadas têm guard próprio DENTRO delas
(webhooks HMAC + path token; cron/system/provision por Bearer; `/api/mcp`; `/api/v1/contacts$`,
`/messages$` etc. por auth-dual cookie-OU-Bearer). Padrões âncorados com `$` de propósito, para que um
sub-path futuro não nasça público de carona. Casos notáveis: callbacks OAuth (Google Agenda/Ads,
Nuvemshop) — o navegador volta de OUTRO site e o cookie `SameSite=strict` não viaja, a identidade vem
do `state` assinado (HMAC de `INTERNAL_SECRET`) com nonce de uso único; moldes de e-mail do GoTrue
(processo de terceiro sem sessão); `/icon` (o matcher só dispensa caminho COM extensão). ⚠️ **Adicionar
path aqui remove a checagem de auth de borda — só com guard próprio dentro da rota** (`isPublicPath` é
usada em `proxy.ts`).

### 8.5 Convite: token HMAC stateless + registro em `team_invites` 🟢

**`invite-token.ts`** — token auto-contido `<body>.<sig>`, base64url, assinado com HMAC-SHA256. Payload
`{invite_id, email, organization_id, role, exp, iat?, invited_by?, interface_settings?}`, validado por
Zod no verify. Verificação usa `timingSafeEqual` (após checar igualdade de comprimento), confere
expiração (`exp*1000 < Date.now()`). TTL 24h. Secret: `INVITE_TOKEN_SECRET → INTERNAL_SECRET →
"dev-fallback"` (o fallback é inalcançável em produção — `INTERNAL_SECRET` é obrigatório e derruba o
boot). **Não requer linha no banco para emitir.**

**`issue-invite.ts`** (`issueInvite`) — assina o token, valida `interfaceTemDestino(role)` (ao menos uma
área permitida), monta o `acceptUrl`, dispara o e-mail via `sendEmail` (roteador com dois transportes),
e audita `member.invited` com `email_dispatched`/`email_error`/`email_via`. Falha de e-mail NÃO desfaz
a organização nem o link — o link devolvido é a superfície de recuperação, e o **motivo** é devolvido
(`send_failed` etc.) porque três causas levam a três consertos diferentes e o booleano sozinho os
achatava.

**`team/convites.ts`** — o convite como REGISTRO (`team_invites`, migration 0238): o que a tela de
Equipe lista, o que diz se o e-mail saiu, o que torna a REVOGAÇÃO possível (cancelar um token stateless
exige algo contra o que verificar). O `id` da linha É o `invite_id` no token; reconvidar um e-mail
pendente RENOVA a mesma linha (índice único parcial: no máximo um pendente por (email, org)).
`emitirConvite` faz upsert (update se pendente, incrementando `resend_count`; insert senão);
`reenviarConvite` re-assina e renova, preservando `invited_by`, e devolve `null` se a linha deixou de
estar em aberto entre a leitura e a escrita. `linkDeAceite` reconstrói o link re-assinando (o token não
é guardado). **`convite-status.ts`** — status DERIVADO, nunca coluna (doutrina DIRC: Calcular):
`statusConvite` = `revogado` (revoked_at) > `aceito` (accepted_at) > `expirado` (expires_at ≤ now) >
`pendente`.

**`aplicar-convite.ts`** (`aplicarConvite`) — o ATO de virar membro. DOIS chamadores legítimos:
`app/actions/team/acceptInvite.ts` (quem já tinha conta clicou no botão) e `app/auth/confirm/route.ts`
(quem confirmou o e-mail por convite). A validade do token é decidida por quem chama; aqui trata a linha
`team_invites`: **checa revogação** (convite cancelado na tela mantém assinatura/validade boas no token
— só a linha diz que morreu) e **grava `accepted_at`** (senão o convite fica "Pendente para sempre"). O
vínculo em si vai por `rpc("fn_accept_team_invite")` — org/papel/convidador SÓ do token assinado, o
usuário de quem chamou, **nada do body**; `42501` (recusa da própria função) → `invalid_or_expired`, não
500. Grava o cookie `active_org` (senão a pessoa entra sem org escolhida).

**`convite-no-signup.ts`** (`decidirConviteDoSignup`) — função **PURA** (propriedade de segurança,
testável sem banco). Decide, no signup, se a pessoa está abrindo a própria empresa ou foi CONVIDADA.
⚠️ `user_metadata.invite_token` é gravável pelo próprio usuário (anon key, do navegador): **nada de lá é
autoridade**. A regra é assinatura HMAC do token + comparação do `payload.email` com o e-mail que o
provedor de auth confirmou. **FALHA FECHADA:** token inválido/expirado → `recusar` (nunca
`provisionar`); e-mail divergente → `recusar`. Sem esta comparação, alguém colaria um token de convite
alheio num signup próprio e entraria na organização da vítima.

### 8.6 Provisionamento de tenant (`auth/provision.ts`) 🟢

**`ensureTenantForUser(user, options)`** — provisiona o tenant de um signup self-service confirmado:
cria a organização (`status='active'`, `onboarded_at` null → cai no onboarding) e a membership `admin`.
Idempotente (se já há membership viva, não faz nada). Service role intencional (o usuário ainda não
pertence a org nenhuma, RLS bloquearia os INSERTs; a fonte confiável é o JWT já validado por `verifyOtp`
no caller). `slugify` gera candidato (unique citext); colisão de slug (`23505`) → até 3 tentativas com
sufixo aleatório. Audita `tenant.created_by_signup` ou `tenant.created_by_recovery`.

**`provisionExternalTenant(input)`** — provisionamento por sistema externo via
`POST /api/v1/tenants/provision` (rota que só existe quando o dono define `TENANT_PROVISIONING_SECRET`).
O dono nasce JÁ ATIVO, senha aleatória, sem convite (quem opera usa o sistema de fora, que fala pela API
key). **Idempotente por (integração, id externo)**, inclusive sob corrida e sob falha transitória no
meio. Peças críticas:

- **`slugDoProvisionamento`** — slug determinístico por hash SHA256 do `externalId` (não `slugify`, que
  corta em 32 chars e podia colidir dois ids diferentes num "replay").
- **Marcador** (`{integration, external_id}`) em `organizations.settings.provisioning` E no
  `app_metadata` do dono — prova que a org/conta nasceu DESTE provisionamento. Slug igual sem marcador
  igual → `ProvisionConflictError` (409): é outra empresa criada à mão, devolver a chave dela seria
  entregar dados de terceiro.
- **`reencontrarECompletar`** — só conclui por replay sobre estado COMPLETO, completando o que faltar
  (`garantirAdminDaOrganizacao`). ⚠️ Antes devolvia `replay:true` sobre org sem admin (o INSERT do
  vínculo morreu por timeout) e respondia 200 "tudo certo" sobre empresa que ninguém acessava — para
  sempre, porque o próprio caminho de recuperação carimbava sucesso.
- **`vinculoVivo`** — a ÚNICA régua de "vínculo está vivo?" (usada para detectar conta órfã e replay
  completo). Erro NÃO vira `null` (falha alto).
- **`garantirAdminDaOrganizacao`** — completa o vínculo que uma tentativa não gravou; ⚠️ `23505` sai em
  SILÊNCIO (não vira update): `user_organizations` tem `unique(user_id, organization_id)`
  (baseline.sql:2441), então linha revogada ou dono rebaixado para viewer devolve `23505` e nada muda —
  um sistema de FORA não ressuscita acesso que o admin da empresa removeu. Sucesso deixa rastro
  (`tenant.provisioning_completed`).
- **`ensureExternalOwnerUser`** — cria o dono ou REAPROVEITA a conta que uma tentativa anterior DESTE
  mesmo provisionamento abandonou. ⚠️ Reaproveita só com DUAS provas: marcador em `app_metadata` (escrito
  SÓ por service role — `updateUser` do próprio usuário só toca `user_metadata`, sem o campo) E nenhum
  vínculo vivo. `EmailJaTemContaError` (decisão do dono, doc 40 item 8): e-mail já com conta REAL →
  recusar, não reaproveitar. `donoOrfaoDesteProvisionamento` varre o diretório paginado porque
  `listUsers` não filtra por e-mail (só no galho raro, após o GoTrue dizer que o e-mail existe); erro do
  GoTrue/Postgres NÃO vira "não é órfã" (falha alto). Todas as ações auditam com `actorUserId: null` (a
  máquina fez, não o dono — creditar humano por ação de máquina inflaria a autoria).

### 8.7 Política de cadastro aberto/só-convite (`auth/politica-de-cadastro.ts`) 🟢

`ModoDeCadastro = "aberto" | "so_convite"`. Existe porque quem hospeda a própria instalação e vende
tenant precisa fechar `/signup` pelo PRODUTO (regra de nginx no proxy reverso barrava junto o
`/signup?invite=…`, que deveria passar). Fonte: **banco acima do `.env`** — `platform_settings.signup_mode`
manda; o `.env` (`SIGNUP_MODE`) é SEMENTE e PISO. Default `aberto` (regra 6 de packaging).

⚠️ **A leitura é PEGAJOSA (ponto de segurança do módulo):** `modoDeCadastro()` **nunca lança** (é lida na
porta de entrada — `/signup` e confirmar e-mail; um throw viraria 500). Se o banco não responde, **vale
o último valor lido com sucesso**; só quando nunca houve leitura boa é que o piso do `.env` entra. Assim
uma instalação fechada continua fechada durante um soluço do banco (não reabre sozinha), e uma que nunca
aplicou a migration 0253 (`42P01`) continua aberta (não fecha para quem nunca pediu). Memo e "último
conhecido" moram em `globalThis` (não `let` de módulo) porque o Next instancia o módulo duas vezes no
mesmo processo (runtime de `route.js` vs `page.js`) — quem GRAVA é uma server action e um dos LEITORES é
`app/auth/confirm/route.ts`. TTL 30s; `invalidarModoDeCadastro` sobe uma "geração" para impedir que uma
leitura em voo reinstale o valor pré-escrita (lost-update). `padraoDaInstalacao` cai em `aberto` para
valor inválido do `.env` (um `SIGNUP_MODE=aberot` digitado errado não pode FECHAR a porta).

### 8.8 Rate limit da superfície de auth (`auth/rate-limit.ts`) 🟢

Aplica o `checkRateLimit`/`peekRateLimit` de `lib/ai/dispatcher/rate-limit.ts` (janela FIXA: `INCR` +
`EXPIRE`; sem Upstash cai para memória do processo). Cobre login, signup, recuperação, aceite de convite
e recuperação de org (issue #64: antes só webhook de captação e dispatcher de IA tinham limite).

- **`authRateLimited(action, identifier, limits)`** — duas contagens: por **IP** (isola quem varre
  muitas contas de um lugar só) e por **identificador hasheado** (barra o ataque distribuído contra UMA
  conta). O identificador entra SEMPRE por `opaque()` (SHA256, 32 chars — chave de Redis é lugar de dado
  opaco, não de e-mail). ⚠️ **Sem IP identificável, o limite por IP NÃO entra** (decisão de segurança):
  o kit self-host expõe o app sem proxy, então `x-forwarded-for` não existe na instalação padrão — a
  versão anterior jogava todos num balde global (`opaque("sem-ip")`), e 60 requisições anônimas trancavam
  o login da instalação INTEIRA (DoS de custo zero). `clientIp()` devolve `null` (não string sentinela)
  quando não sabe.
- **`AUTH_LIMITS`** — `login {ip: loginIpLimit()=60, id:5, win:300}`, `signup {ip:20, win:3600}`,
  `reset {ip:30, id:3, win:3600}`, `invite_accept {ip:60, win:3600}`, `org_recovery {ip:5, id:3,
  win:3600}` (o mais apertado: cada acerto CRIA uma organização). O teto por IP do login é o ÚNICO
  configurável (`AUTH_RATE_LIMIT_LOGIN_IP`), por defeito de AMBIENTE: no CI todo teste sai do mesmo IP
  (um runner) e 28 specs estouravam 60/5min. Afrouxar ISTO é seguro porque o limite que barra brute
  force é o `id` (5 falhas na mesma conta), que NÃO é configurável.
- **`contaBloqueadaPorFalhas` / `registrarFalhaDeLogin`** — bloqueio por FALHA para o login: consulta
  ANTES do provedor (barra antes de acontecer), incrementa DEPOIS, só quando a senha errou (quem acerta
  não gasta o próprio orçamento de bloqueio). Vale inclusive contra ataque distribuído por muitos IPs.

### 8.9 Códigos de recuperação, auth de cron e anti open-redirect 🟢

- **`recovery-codes.ts`** — 10 códigos de 8 chars de alfabeto sem ambiguidade (sem 0/O, 1/I/L),
  `randomBytes` com **rejection sampling** (`MAX_VALID = floor(256/31)*31`) para evitar viés de módulo.
  Guardados como `sha256(code)` bytea em `user_recovery_codes`.
- **`cron-auth.ts`** — `autorizaCron(req)`: aceita `Authorization: Bearer <secret>` ou
  `x-cron-secret`, compara em tempo constante (`timingSafeStringEqual`, que hasheia SHA256 antes de
  `timingSafeEqual` para comprimento fixo) contra `INTERNAL_CRON_SECRET` e `INTERNAL_SECRET`.
  **Fail-closed:** sem token ou sem secret configurado → `false`.
- **`safe-next.ts`** — `safeNext(next, fallback)` contra open-redirect (CWE-601; relatório de segurança
  da comunidade — `next` chegava cru ao `redirect()` de telas públicas). **Allowlist, não denylist:** só
  passa caminho absoluto-na-raiz (`/`), rejeitando `//host` e `/\host` (protocol-relative), qualquer `\`
  (normalizado para `/` por vários agentes) e controles/espaço (`\u0000-\u0020`, `\u007f`).

### 8.10 API key de organização e nome do atendente (`tenants/api-key.ts`, `users/`) 🟢

**`rotateIntegrationApiKey(input)`** (`tenants/api-key.ts`) — emite/reemite a API key `dsk_...` de uma
org para integração externa (`dsk_<prefix>_<secret>`, SHA256 em `token_hash`, plaintext nunca
persistido). Como só o hash fica no banco, chamada repetida REVOGA a chave anterior dessa integração e
emite outra (nunca duas vivas para o mesmo par org/integração; a busca da anterior usa `.contains` com
JSON string porque o `.contains` do postgrest-js serializa array como literal Postgres que o `@>` de
jsonb não aceita). Falha da busca/revogação → **falha FECHADA** (throw → 500). Escopos:
`["mcp:read", "mcp:write", "role:agent", <integrationScope>]` — sem eles a auth passa mas o despacho
devolve 403 `Token missing required scope`. **Papel `agent` (rank 2) deliberado** (menor privilégio,
cobre 46 das 63 tools; as 17 de `ai_operator`/`manager` exigem chave pela tela, decisão de humano).
**Sem `actor:ai_agent` de propósito**: a ausência faz `deriveActor` devolver `api_token` (o parceiro é
integração, não IA) — com o escopo posto, a timeline atribuía à IA o que o parceiro fez, `crm_resume_agent`
ficava barrada e `run_id` gravava id de token inexistente em `ai_agent_runs`. Auditas `token.created`/
`token.revoked` com `actorUserId: null`.

**`users/nome-do-atendente.ts`** (`nomesDosAtendentes`) — resolve `full_name` via
`admin.auth.admin.getUserById` (só service role — o client do request leva 403 `not_admin` no endpoint
admin do GoTrue, e o supabase-js **não lança** nesse caso, devolve `{user:null, error}`; seria um badge
sem nome sem erro nenhum). Sem service role, devolve **Map VAZIO** (não Map de nulls) com log — `null`
DECLARADO, para nunca confundir "não consegui ler" com "esse atendente não tem nome". LGPD: expõe SÓ
`full_name`. **`users/com-nome-do-atendente.ts`** — desde a migration 0202 `fn_conversation_assign`
grava `assigned_to_user_name` desnormalizado na linha; este helper só repassa a coluna e cai em
`nomesDosAtendentes` APENAS para linhas inconsistentes (id sem nome, dado pré-migration), nunca a página
inteira. Custo do lookup: uma requisição HTTP ao GoTrue por id único (~60ms/1, ~1,2s/50).

### 8.11 Acompanhamento administrativo / impersonation (`impersonate/`) 🟢

Platform admin agindo COMO um tenant (EPIC-11, S-11.07). O cookie é **aditivo** à sessão — NÃO troca o
usuário Supabase autenticado; o código server-side lê o cookie para saber que o admin está agindo como
tenant e propaga `organization_id`/autoria no audit.

- **`cookie.ts`** (server, `node:crypto`) — `signImpersonateCookie`/`verifyImpersonateCookie`: envelope
  `<base64url(payload)>.<base64url(hmac_sha256)>`, HttpOnly + Secure + `SameSite=Lax`, TTL 1h.
  `verify` usa `timingSafeEqual` e **checa expiração mesmo com HMAC válido** (nunca confia em cookie
  vencido). `signImpersonateCookie` LANÇA se `IMPERSONATE_COOKIE_SECRET` < 32 chars (caller pré-checa via
  `isImpersonateSecretReady` e responde 503).
- **`cookie-edge.ts`** (Edge, Web Crypto SubtleCrypto — `node:crypto` não existe no Edge do middleware)
  — só VERIFICA (assinatura mora no server). Sem `timingSafeEqual`: `constantTimeEqual` manual por XOR
  sobre bytes de mesmo comprimento. Usado pelo `proxy.ts` em `/app/*`.
- **`support.ts`** — `readSupportContext(db)` resolve o contexto por `rpc("fn_support_context")` (auth.uid
  + auth.session_id, nunca cookie); `SupportContext` validado por Zod (`access_mode: full|support_readonly`,
  `status: active|expired|revoked`). `requireSupportWrite` é guarda de EFEITO antes de clients service
  role (não substitui RBAC/MFA); erro de leitura → 503 (falha fechada). `supportCallbackWriteAllowed`
  valida a identidade pelo `state` HMAC já verificado (nunca do body).

### Escala de confiança e lacunas — Unidade 8
- 🟢 Todo o fluxo TS de auth/RBAC/tenancy foi lido diretamente do código (helpers, guards, tokens,
  provisionamento, impersonation, rate limit).
- 🔴 As funções `SECURITY DEFINER` do banco que sustentam este módulo — `fn_user_role_in_org`,
  `fn_is_platform_admin`, `fn_role_at_least`, `fn_accept_team_invite`, `fn_support_context`,
  `fn_support_callback_write_allowed` — e as policies RLS (`user_orgs_select`, `orgs_select`,
  `conversations_select`, `fn_can_view_conversation`) ficam para o **Data Master**; aqui só o lado que
  as INVOCA.
- 🟡 **Secret HMAC de convite com fallback `"dev-fallback"`** — inalcançável em produção (`INTERNAL_SECRET`
  é obrigatório e derruba o boot), documentado no próprio código.
- 🟡 **Ramo do cap de warm-up e o balde global de rate limit** já foram tratados como inalcançáveis nas
  unidades anteriores; aqui o rate limit de auth herda a mesma implementação de `dispatcher/rate-limit`.
- 🟡 **Fallback de `com-nome-do-atendente.ts`** — só dispara para linhas atribuídas ANTES da migration
  0202 cujo backfill não alcançou; se nunca disparar em produção, é candidato a código morto (declarado
  no próprio código).
- 🔴 **Rate limit ausente em crons e MCP** — `authRateLimited`/`checkRateLimit` cobrem login/signup/
  reset/convite/webhook/dispatcher; crons e MCP seguem sem (limitação conhecida do repo, a validar com
  o threat-model).

---

## Unidade 9 — Compliance: `lib/lgpd/`, `lib/legal/`, `lib/opt-out/`, `lib/retencao/`, `lib/audit/`

### Propósito 🟢
A camada que faz o produto cumprir a lei e deixar rastro: **direitos do titular** (LGPD Art. 18 —
acesso e anonimização/exclusão), **trilha de auditoria** append-only, **política de retenção** (poda e
expurgo com piso), **detecção de descadastro** (opt-out) e o **perfil legal por país** (documento, lei
citada, calendário do prazo, padrões de PII). Tudo multi-tenant: quase toda escrita usa o admin client
(service role que **bypassa RLS**), então cada query filtra `organization_id` **à mão**, de fonte
confiável — nunca do body.

- **`lib/audit/`** — `audit()` fire-and-forget append-only; `AUDIT_ACTIONS` (vocabulário canônico,
  ~330 códigos, fonte única do filtro do painel); `isServiceRoleConfigured`, `hashEmail`.
- **`lib/lgpd/`** — SLA em dias úteis, cascata de anonimização idempotente com retomada, coletor do
  export de acesso, renderizador de PDF, assinatura PAdES (stub), fila de redação de storage, alarme de
  SLA, entrega por e-mail, máscara de PII.
- **`lib/legal/`** — perfil do país da organização (`perfil-do-pais.ts`) e responsável legal da
  instalação (`operador.ts`).
- **`lib/opt-out/`** — detecção de pedido de descadastro por intenção (não por palavra solta), pt-BR e
  espanhol.
- **`lib/retencao/`** — política pura de retenção (dias por tabela, com piso).

Padrão transversal: **falha na direção segura + sucesso nunca declarado sobre trabalho não feito**. A
anonimização falha ABERTA (aborta antes de deixar PII órfão); o audit falha FECHADA (não bloqueia a
mutação, mas grita no Sentry); a retenção nunca vira apagador de rastro recente (piso no SQL); o
opt-out inequívoco (que bloqueia) exige o objeto de comunicação, o ambíguo (que só escala) não bloqueia
sozinho.

### 9.1 Trilha de auditoria (`audit/index.ts`, `audit/actions.ts`) 🟢

**`audit(entry)`** — insert append-only em `api_audit_log`, **fire-and-forget**: falha NUNCA bloqueia a
mutação primária (por doutrina), mas é reportada por `reportAuditFailure` (console.error + Sentry) —
senão a trilha inteira poderia parar sem ninguém perceber (foi o que aconteceu: toda tool MCP falhava ao
auditar por "invalid input syntax for type uuid" e o único sinal era um console dentro do contêiner).
Prefere o admin client (bypassa RLS, funciona para eventos sem sessão como login falho); cai no client
de sessão quando não há service role (a policy `audit_log_insert_tenant_member` permite o membro inserir
suas próprias linhas). Resolve **contexto de suporte** (impersonation) para carimbar `acting_as_platform_admin`
e `support_*` no metadata, por dois caminhos: state OAuth assinado (`actorAuthSessionId`) ou sessão de
cookie (`readSupportContext`). **`auditForOrganizations(entry, ids)`** — o mesmo fato de plataforma em N
organizações num único insert de N linhas (remoção de extensão: cada org desligada precisa ver o porquê
em `/app/audit`).

**`isServiceRoleConfigured()`** — ⚠️ **NUNCA infere validade pelo comprimento.** O corte antigo
`length > 50` (que assumia JWT) rejeitava a chave curta `sb_secret_...` (~41 chars) real e funcional;
o efeito era login/convite/atribuição em massa falhando com a chave certa configurada. A pergunta real
é só "tem chave de verdade?": vazio ou `PLACEHOLDER` são os únicos "não"; erra para "tenho a chave"
(tentar e falhar alto > degradar em silêncio). **`hashEmail`** — sha256 do e-mail normalizado, para
correlacionar login falho no audit sem PII plaintext.

**`AUDIT_ACTIONS` (`audit/actions.ts`)** — o vocabulário canônico como **ARRAY** (o tipo `AuditAction`
é derivado dele). ⚠️ Até 2026-08-14 havia uma cópia em runtime mantida à mão no painel ("keep in sync
manually", sem gate) — medido: 209 códigos aqui, 89 lá, **120 emitidos e não-filtráveis na tela**. A
cópia foi apagada; o painel mapeia o array, então código novo aparece no filtro sem ninguém lembrar de
nada. ⚠️ **PROIBIDO importar qualquer coisa neste arquivo** (é importado por um `"use client"`; um
`import env` arrastaria a validação de env para o bundle do browser — `tests/unit/audit-lista-do-painel-e-derivada.test.tsx`
reprova). Regra: acrescentar código no fim, **nunca renomear** (cada código casa 1:1 com uma linha de
`api_audit_log.action`). Segredos **nunca** entram no metadata: `api_audit_log` é append-only por schema
(nenhum papel tem GRANT de UPDATE/DELETE, nem service role), então um segredo ali ficaria os 5 anos da
retenção.

### 9.2 SLA em dias úteis (`lgpd/sla.ts`, `lgpd/holidays-br.ts`) 🟢

**`computeDueAt(receivedAt, businessDays, holidays)`** — prazo LGPD em dias úteis brasileiros (L-04):
pula sábado/domingo (`getUTCDay` 0/6) e feriados nacionais; se o recebimento cai em dia não-útil, a
contagem começa no próximo dia útil. Tudo em UTC midnight (o set de feriados é `YYYY-MM-DD`).
`HOLIDAYS_BR_ISO` cobre 2026-2030 (fixos + móveis: carnaval, sexta-feira santa, Corpus Christi listados
à mão). O conjunto de feriados é **parametrizável** — é o que permite o prazo ser contado no calendário
do país da organização (issue #1033, ver 9.6).

### 9.3 Cascata de anonimização (`lgpd/redact-cascade.ts`, `lgpd/cascata.ts`) 🟢

**`cascadeRedactContact(args)`** (`redact-cascade.ts`) — invoca a RPC `SECURITY DEFINER`
`fn_lgpd_cascade_redact_contact` (anonimização completa numa transação Postgres: muta `contacts`
irreversível, conversas, mensagens, atividades, leads; tira PII de `orders.payload` preservando valores;
enfileira mídia em `storage_redaction_queue`; grava audit denso na TX). ⚠️ **A foto de perfil é
enfileirada ANTES da cascata**, e a ordem importa: a RPC zera o ponteiro do avatar; se o arquivo não
fosse enfileirado antes, ficaria órfão no bucket (a pessoa "anonimizada" com o rosto guardado — numa
auditoria LGPD, o mesmo que não ter anonimizado). Usa `upsert` (não `insert`) porque o caminho do
avatar é estável por contato (`{org}/avatars/{id}.jpg`) e reaproveitado; **falha FECHADA** se o
enfileiramento falhar (aborta antes de zerar o ponteiro).

**`cascata.ts`** — os passos 2-4 da cascata (leads, atividades, régua de recuperação), num lugar só
porque **duas bocas escrevem a mesma redação** (a rota `POST /api/v1/lgpd/anonymize` e o cron de
retenção) e duplicar a regra faria títulos redigidos de dois jeitos (anti-pattern nº 2). ⚠️
**Idempotência não é firula:** o passo 2 monta `title.slice(0,20) + " (anonimizado)"`; rodar de novo
sobre título já redigido produziria "Orçamento telhado (an (anonimizado)" e comeria o resto a cada
rodada diária — por isso `jaRedigida()` guarda o sufixo. Os passos 3 e 4 SELECIONAM antes de escrever
(senão a varredura diária reescreveria dado já certo e a auditoria registraria "efeito" todo dia). O
**passo 4** (issue #701) cancela a régua de recuperação viva (`STATUS_DA_REGUA_VIVA`: active,
waiting_reply, dormente, paused_*) — sem ele, a régua esgotava DEPOIS da redação e ressuscitava o
vínculo que a LGPD mandou cortar (mensagens chegando a quem pediu para ser esquecido). `ResultadoDaRedacao.tabelas`
registra o que foi REALMENTE tocado (antes gravava as três tabelas como literal, afirmando ter redigido
o que não redigiu).

**`varrerRedacoesIncompletas(db, teto)`** — o cron que torna a correção alcançável sem clique (a tela
troca o botão por um parágrafo quando o contato já está anonimizado, então a retomada era inalcançável;
e um direito com prazo não pode depender de alguém lembrar de clicar). Parte de `contacts.is_anonymized=true`,
detecta resíduo em bloco (`CONTATOS_POR_BLOCO=100`, duas consultas por bloco em vez de uma por contato).
⚠️ **`MAX_CONTATOS_EXAMINADOS=5000` (leitura) e `MAX_CONTATOS_POR_VARREDURA=200` (conserto) são números
DIFERENTES de propósito:** um só produzia STARVATION silenciosa — com `limit(200)` sem ordenação, toda
rodada olhava os mesmos 200 primeiros e nunca alcançava o contato 201 (resíduo pendente indefinidamente,
num prazo legal, com a trilha dizendo "correu bem"). A detecção NÃO filtra org (só decide quais visitar);
a escrita (`completarRedacaoDoContato`) filtra a org da linha de `contacts`.

### 9.4 Export de acesso (`lgpd/export-collector.ts`, `pdf-renderer.tsx`, `pades-signer.ts`, `email-delivery.ts`) 🟢

**`collectExportData(args)`** (`export-collector.ts`) — agrega TODO dado pessoal que o CRM guarda sobre
um contato (Art. 18 II — direito de acesso). PII **nunca** vai para o log (só ids e contagens). ⚠️
**Invariante central: o que se APAGA a pedido do titular é o que se ENTREGA a pedido dele.** O gate
`tests/unit/lgpd-exporta-o-que-redige.test.ts` **deriva** a lista de blocos das duas pontas (cascata de
redação × coletor) em vez de escrevê-la à mão — foi assim que `calendar_appointments`,
`webhook_lead_captures`, `sales`, `crm_tasks`, `voice_calls`, casos/eventos/chat, passagens etc.
entraram (várias depois do fato, achadas pelo gate). Campos como `case_chat_messages` e `passagens` são
**obrigatórios** (não opcionais) no `ExportPayload` de propósito: campo obrigatório faz um caminho de
export novo não COMPILAR se esquecer — a única sincronia que não depende de memória. Colunas de operação
(datas, status, valores) que a cascata PRESERVA ainda entram: o Art. 18 II é sobre o que a organização
sabe A RESPEITO do titular. Telefone de funcionário entra MASCARADO (`AvisoDeCasoEntrega.destino_mascarado`);
corpo de aviso não entra (só `corpo_hash` é guardado). `lerControlador` nunca lança: campos vazios e
rodapé com traço — abortar seria estourar o SLA de D+7; preencher com o nome do produto escreveria a
entidade errada num documento jurídico.

**`pdf-renderer.tsx`** — template PT-BR do Art. 18 II via `@react-pdf/renderer`. ⚠️ **NÃO leva marca —
decisão, não esquecimento:** o rodapé imprime o CONTROLADOR (`organizations.legal_name`) e o Encarregado,
nunca a marca do revendedor (que é OPERADOR, não controlador — nomeá-lo inverteria papéis num documento
jurídico). Consequência boa: a armadilha do `@react-pdf` (`var(--x)`/`oklch()` renderizam PDF válido
descartando a cor em silêncio) não alcança o documento, porque ele não recebe cor de marca.

**`signPdfPades(buffer)`** (`pades-signer.ts`) — assinatura PAdES. 🟡 **Stub:** enquanto `LGPD_SIGNING_KEY`
não estiver provisionada (P12 cert pendente), degrada para não-assinado (`signed_pades: false`,
`warning: "pades_key_missing"`) e computa SHA-256 para integridade; **mesmo com a key set** degrada para
não-assinado, para não produzir documento falsamente marcado como assinado (TODO: `node-signpdf` +
`@signpdf/signer-p12`). **`email-delivery.ts`** — entrega o export pelo transporte da instalação
(`lib/email/roteador.ts`); ⚠️ **nunca loga o e-mail em plaintext** (só sha256 no log/audit, L-08). Este
e-mail nomeia quem OPEROU (marca resolvida); o PDF anexo nomeia quem RESPONDE (controlador) — dois papéis
diferentes de propósito. **`mask.ts`** — máscara de PII para previews: `maskEmail` (`a***@dominio.com`),
`maskPhone` (`(**) ****-1234`); CPF **nunca** exposto (omitido inteiro).

### 9.5 Fila de redação de storage e alarme de SLA (`lgpd/storage-redaction-queue.ts`, `sla-alarm.ts`) 🟢

**`drainStorageRedactionQueue(opts)`** — drena `storage_redaction_queue` (mídia enfileirada pela cascata)
removendo os objetos do Supabase Storage. Idempotente: claim por transição `pending → processing`
(`processed_at` + `attempts++`) e finaliza em `deleted | failed | skipped`; "not found" conta como
`skipped` (objeto já foi); re-run ignora linhas terminais; `MAX_ATTEMPTS=3`, lote 50.

**`triggerSlaAlarm(args)`** (`sla-alarm.ts`) — disparado pelo cron `lgpd-sla-watcher` nos limiares D+5
(`data_request_d5`) / D+10 (`redact_d10`). Privacidade (L-08): **zero PII no Sentry** (só ids, contagens,
limiar); e-mail do DPO da coluna do banco ou env, **nunca logado em plaintext**; dedup **fire-once-per-24h**
via `request_payload.last_alarm_at`. O e-mail usa a marca resolvida (`marca.accent`), mas ⚠️ `#dc2626`
(vermelho de alerta) **fica fixo** — é semântica de ALERTA, não marca (um atraso em verde-sálvia deixaria
de comunicar urgência). Nome da org e marca entram no HTML **escapados** (`escapeHtml`) porque vêm de
campos que uma pessoa digita.

### 9.6 Perfil legal do país e responsável da instalação (`legal/perfil-do-pais.ts`, `legal/operador.ts`) 🟢

**`perfil-do-pais.ts`** — o perfil do país da ORGANIZAÇÃO (não da instalação: duas orgs no mesmo banco
podem estar em países diferentes, e cada uma vê o documento/lei/prazo do país dela; `organizations.country`
ISO-3166 alpha-2, `null` = Brasil). Uma função (`perfilDaOrganizacao`), não `select` inline por rota,
para não divergir (a divergência aqui afirmaria a lei de um país com o prazo de outro num documento
entregue ao titular). ⚠️ **Régua para um país ENTRAR (issue #1033): "país entra com a citação revisada,
ou não entra".** `lei.revisada === false` faz o documento NÃO citar lei nenhuma — **sem fallback para a
LGPD** (afirmar a lei brasileira para um titular em Angola é a citação errada, pior que nenhuma).
`paisesOferecidos()` (o seletor) só inclui país com citação revisada; o registro pode conhecer mais do
que oferece. **Separação documento × forma:** país com checksum público (Brasil, `isValidCpf` mod-11)
confere dígito; país sem checksum valida FORMA, e a `regra`/`mensagemInvalido` dizem isso. `padroesDePii`
carrega `naoCobre` (declaração obrigatória: padrão largo demais redige nº de pedido, estreito demais deixa
PII chegar ao modelo — os dois defeitos silenciosos). `perfilDaOrganizacao` **degrada para o Brasil** com
RASTRO (console.error + Sentry) quando a leitura falha ou o país não tem perfil.

**`operador.ts`** — quem responde legalmente por ESTA instalação (MIT self-host: quem instala opera e
responde; os mantenedores não são parte de contrato nenhum). Os documentos legais nomeiam o OPERADOR,
não o "serviço DeskcommCRM" (seria literalmente falso em toda instalação de terceiro). `resolverOperador`
usa o client de SESSÃO, **nunca service role**: `/legal/*` é rota pública, e um admin client resolveria
"alguma" org e publicaria razão social/CNPJ/DPO de um tenant numa URL sem auth. Sem sessão → fallback com
o documento íntegro ("o operador desta instalação"). ⚠️ **`urlDePoliticaSegura`** — guarda de saída:
`z.string().url()` do zod ACEITA `javascript:alert(1)`; como `/legal/*` é público, o valor cru num
`<a href>` deixaria um admin de tenant alcançar anônimos. A checagra mora na SAÍDA (não no schema) para
valer inclusive para o que já está gravado (só `https:`/`http:`).

### 9.7 Detecção de opt-out (`opt-out/deteccao.ts`) 🟢

A regra de "esta mensagem é pedido de descadastro?", num lugar só. ⚠️ Existe porque a mesma pergunta era
respondida por duas regras, e a que decidia o bloqueio (`STOP_RX` na ingestão) caçava a PALAVRA em
qualquer posição — medido numa clínica: "tem como parar a dor?", "posso sair antes das 15h?" bloqueavam
o paciente, enquanto "não quero mais receber nada" (opt-out de verdade) não bloqueava. **A regra: verbo
de cessação + OBJETO DE COMUNICAÇÃO.** "parar" só vira opt-out quando o objeto é a MENSAGEM ("parar de me
mandar", "sair da lista") ou a palavra ISOLADA (mensagem inteira = a palavra; `PALAVRAS_DE_OPT_OUT`).
Nunca a palavra solta no meio da frase. Lookahead `OBJETOS_NAO_COMUNICATIVOS` exclui "parar de mandar o
pedido/boleto" (cliente que quer continuar sendo atendido). Cobre pt-BR e espanhol (a plantilla espanhola
pede "Respondé BAJA", daí `baja`/`salir`).

**Dois níveis:** `ehPedidoDeOptOut` (INEQUÍVOCO) autoriza gravar `is_blocked` (estado que só uma pessoa
desfaz); `ehOptOutProvavel` soma os AMBÍGUOS ("me deixa em paz", "chega") e é o sinal conservador do
runtime — parar de responder e escalar ao humano, que confirma o bloqueio (deixar o ambíguo bloquear
sozinho inverteria a política).

### 9.8 Política de retenção (`retencao/politica.ts`) 🟢

Regra PURA (sem banco, sem env) para a poda da fila e o expurgo da auditoria (issue #261).
**`interpretarRetencao(bruto, {chave, padrao, piso})`** — interpreta a env var: ausente/vazio → padrão
sem aviso; não-numérico/zero/negativo → padrão COM aviso; abaixo do piso → piso COM aviso. ⚠️ **Não é
`z.coerce.number()` em `lib/env.ts`** porque `lib/env.ts` lança no import (no Next, na primeira requisição
— com healthcheck TCP puro, contêiner `healthy` com 100% em 500); `JOB_QUEUE_RETENTION_DAYS=noventa`
digitado às 2h derrubaria o produto. Usa `Number()`, não `parseInt` ("90dias" viraria 90 em silêncio).
⚠️ **O piso de verdade mora no SQL** (`greatest(..., piso)` dentro de `fn_podar_fila_de_jobs`,
`fn_expurgar_auditoria_vencida` etc.), para valer para qualquer chamador (inclusive `psql` na mão); a
cópia aqui serve para o operador ver no log que o valor dele foi elevado. **Exceção declarada:** a poda
de `webhook_lead_captures` é um DELETE do admin client (não uma `security definer`), então o piso dela
termina no TypeScript — a frase não promete proteção que não existe. Constantes: fila 90d/piso 7d;
auditoria 1825d (L-10, 5 anos)/piso 90d; captação 365d/piso 30d; espelho de agenda 90d/piso 7d; conversa
de caso 365d/piso 90d; passagem 1825d/piso 90d (+ só apaga `reconhecido_em IS NOT NULL`); aviso de caso
180d/piso 30d.

### Escala de confiança e lacunas — Unidade 9
- 🟢 Todo o lado TS do compliance foi lido diretamente (audit, LGPD SLA/cascata/export/fila/alarme,
  perfil de país, operador, opt-out, retenção).
- 🔴 As funções `SECURITY DEFINER` do banco — `fn_lgpd_cascade_redact_contact`,
  `fn_expurgar_auditoria_vencida`, `fn_podar_fila_de_jobs`, `fn_redigir_captacoes_do_contato_anonimizado`,
  `jsonb_set_last_alarm_at` e triggers de redação (`trg_redigir_tarefas_ao_anonimizar` etc.) — e as
  policies RLS de `api_audit_log` ficam para o **Data Master**; aqui só o lado que as invoca.
- 🟡 **PAdES é stub** — `signPdfPades` degrada para não-assinado mesmo com `LGPD_SIGNING_KEY` set (P12
  cert pendente); `signed_pades:false` + `warning:"pades_key_missing"` deixam o gap na trilha.
- 🟡 **Só o perfil do Brasil está publicado** — `PERFIS_DO_PAIS` conhece só `BR`; o mecanismo multi-país
  existe (issue #1033) mas nenhum outro país tem citação revisada, então `paisesOferecidos()` devolve só
  o Brasil.
- 🟡 **Piso de retenção da captação vive só no TS** — a poda de `webhook_lead_captures` é DELETE do admin
  client, sem função SQL onde ancorar o piso (declarado no próprio código).
- 🔴 Contagem exata de `AUDIT_ACTIONS` (~330 códigos) e o conjunto de tabelas na cascata de redação a
  confirmar com o Data Master; aqui a lista foi lida do array, não contada do banco.


---

## Unidade 10 — Plataforma e operação

> Módulos: `lib/settings/`, `lib/onboarding/`, `lib/instalacao/`, `lib/branding/`, `lib/navigation/`, `lib/system/`, `lib/operacao/`
> Analisado pelo Arqueólogo em 2026-09-22. Escala: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

Esta unidade reúne a camada que faz o produto ser **self-host e revendável**: como a marca é resolvida sem tocar no `.env`, como a instalação lê configuração do banco acima do ambiente, como o onboarding propõe um funil, como a navegação sai de um registro único, como o update se narra na tela e como as superfícies de operação são compartilhadas entre a tela e o agente de IA. Um fio atravessa tudo: **banco acima do `.env`, `.env` é semente + piso de rollback**, e **nunca lançar no caminho de render**.

### 10.1 `lib/settings/` — reset de dados operacionais (zona de perigo)

**Propósito** 🟢 — Apaga os dados de atendimento de UMA organização (Configurações › Organização). O PR original (#556) delegava a uma RPC inexistente `fn_apagar_dados_operacionais_da_organizacao`; a reimplementação move a exclusão para o app para não criar uma `security definer` cujo único seletor de linha é a org (vetor de adulteração, doutrina da migration 0167).

**Função principal** — `apagarDadosOperacionaisDaOrg(client, organizationId): Promise<ResultadoDoApagamento>` (`lib/settings/apagar-dados-operacionais.ts:98-121`).

**Entidades** 🟢
- `Raiz` { `tabela: TabelaOperacional`, `porque: string` } — `:35-38`
- `TabelaOperacional` = `messages | conversations | calendar_appointments | orders | crm_leads | contacts` — `:40-46`
- `ContagensApagadas` = `Record<TabelaOperacional, number>` — `:65`
- `ResultadoDoApagamento` = `{ok:true, counts}` | `{ok:false, falha, counts}` — `:72-74`

**Regras de negócio**
- 🟢 **A ORDEM de exclusão é carga, não estilo**: três FKs para `contacts` são `ON DELETE RESTRICT` (`messages.contact_id`, `conversations.contact_id`, `calendar_appointments.contact_id`); os filhos precisam sair antes de `contacts` ou o Postgres devolve `23503` e o reset morre pela metade. `contacts` é sempre o último. (`:16-24`, lista em `:56-63`, `contacts` em `:62`)
- 🟢 **Segurança multi-tenant**: todo DELETE carrega `.eq("organization_id", organizationId)` e o client é service-role (bypassa RLS), então o filtro é o ÚNICO separador de tenant; `organizationId` vem da sessão (`resolveActiveOrg`), nunca do body. (`:92-97`, `:112-116`)
- 🟢 **NÃO é atômico** (PostgREST não tem transação multi-chamada). A ordem foi escolhida para que uma falha no meio deixe o banco íntegro (filhos antes dos pais) e o reexecutar retome; contagens parciais voltam junto com a falha em vez de se perderem. (`:98-121`, retorno na primeira falha em `:117-118`)
- 🟢 **Sobreviventes por design** (NÃO apagados): users, invites, org+settings, pipelines/stages, agentes de IA + credenciais, canais de WhatsApp, API tokens, `api_audit_log` (append-only), `lgpd_requests` (FK SET NULL). (`:26-31`)

**Dependências**: só o tipo `SupabaseClient`. Nenhum outro módulo lib.

### 10.2 `lib/branding/` — motor de marca própria (white-label)

**Propósito** 🟢 — Resolve a marca do revendedor/tenant (nome, logo, cor de destaque) **do banco acima do `.env`**, deriva uma rampa de 11 tons + tokens por tema com contraste WCAG e correção de dicromacia, serializa para CSS e **nunca lança** (roda em `app/layout.tsx` envolvendo toda tela). Três camadas: organização (topo) → instalação/banco → `.env` (piso/rollback).

#### `rampa.ts` — núcleo de cor (sRGB↔OKLab, sem lib de cor)
- Exports-chave: `ehHexValido` (`:49`), `hexParaRgb` (`:56-72`, **lança** em hex malformado, deliberado `:52-55`), `hexParaOklch`/`oklchParaHex`, `deltaEOklab` (`:186-190`), `rampaDeSemente(semente): Rampa` (`:255-263`).
- Constantes: `GRAUS = [50..950]` (`:41`), `ESCADA_L` (11 lightnesses OKLab de globals.css, `:217-219`), `CURVA_C` (croma relativo ao stop 600, `CURVA_C[6]===1`, `:225-227`), `K=6` (âncora da semente = stop 600, `:239`).
- Algoritmos não-triviais 🟢:
  - Conversão sRGB↔OKLab (Björn Ottosson) com zero dependência de lib de cor.
  - `oklchParaHex`: clamp de gamut por **redução de croma** via busca binária de 40 passos, NÃO clipping de canal (clipping desloca matiz). (`:159-183`; slack de 1e-4 em `:155`)
  - `lightnessDoStop`: reescala a escada L em DOIS segmentos lineares para cruzar a lightness da semente exatamente no índice K sem mover as pontas. (`:249-253`)
  - `rampaDeSemente`: deriva 11 stops; o stop K recebe a semente normalizada LITERAL (não o round-trip OKLab) para evitar descasamento de ±1/255 entre o color picker e a UI. (`:255-263`, override em `:261`)
  - Semente ancorada por PAPEL (stop 600), não pela lightness mais próxima — decisão que impede o navy `#0f172a` de virar periwinkle. (`:229-239`)

#### `contraste.ts` — legibilidade/separação em três eixos
- Exports: `luminanciaRelativa`/`razaoDeContraste` (WCAG), `melhorFrenteSobre` (frente preto/branco calculada), `simularDicromacia`, `deltaESimulado` (pior caso entre dicromacias), `extrairRegua(css)`, `escolherAccent(rampa, tema, alcance=10)`, `reconciliarSemanticas`, `derivarMarca(semente, regua): Marca` (entrada, `:836-866`).
- Constantes: `PISOS = {texto:4.5, componente:3.0}` (WCAG 1.4.3/1.4.11), matrizes `MACHADO` (deuteranopia/protanopia, Machado 2009, aplicadas no rgb LINEAR), `LIMIAR_ACROMATICO=0.01` (portão de croma; navy medido C=0.0398 dá 40× de margem), `ROTACAO_MAXIMA=60`, `PASSO_DE_ROTACAO=5`.
- Algoritmos 🟢:
  - Parser de CSS `varrerRegras`: recursão por profundidade de chaves descendo em at-rules (@layer/@media) para enxergar `:focus-visible`/`::selection` dentro de `@layer base`.
  - `escolherAccent`: desloca a rampa INTEIRA junto por um offset de stop; ordem de candidatos prefere 0, depois a direção que se afasta da superfície; fallback "menos ruim" ranqueia por menos falhas e maior folga relativa, nunca lança.
  - `reconciliarSemanticas`: rotaciona NOSSAS semânticas (success/warning/error/info) para longe do accent do cliente sob dicromacia; quando nenhuma rotação ≤60° resolve, emite `redundancia_nao_cromatica_necessaria` em vez de falhar.
  - `derivarMarca`: semente acromática (C<LIMIAR) mantém a rampa do produto, o hex da marca vai para `--color-brand`, emite `marca_acromatica`. (`:836-866`)

#### `resolve.ts` — resolvedor de camadas, NUNCA lança
- Exports: `resolverMarca(camadas, regua): MarcaResolvida`, `camadaDaInstalacao`, `camadaDoAmbiente`, `camadaDaOrganizacao`.
- Regras 🟢:
  - **Nunca lança** — chamado em `app/layout.tsx`; uma exceção = 500 em toda tela incl. login. Recusas voltam como `MotivoDaMarca` (dado, não log perdido). (`:1-24`)
  - **Precedência por CAMPO** via `primeiroDefinido` — campo ausente NÃO apaga a camada de baixo; pilha org→instalação→env, primeira cor não-vazia vence, cor inválida em camada superior é anotada e a busca continua descendo. (`:79-93`, `:392-431`)
  - `format` divergente → cai para o padrão do produto (`formato_desconhecido`); `algo` divergente → anota e CONTINUA (o valor guardado é ENTRADA, este build re-deriva) — é o que mantém um rollback pintado. (`:307-333`)
  - Cache de derivação: `WeakMap<Regua, Map<seed, Marca>>`, evicção FIFO com `TETO_DE_SEMENTES=64`. (`:205-249`)

#### `schema.ts` — envelope
- `esquemaDaCorDaMarca` (zod) usa `.catchall(z.unknown())` NÃO `.strict()` (código velho sobre schema novo não pode lançar no render); envelope carrega eixos de versão `format`/`algo`; `FORMATO_ATUAL=1`, `ALGORITMO_ATUAL=1`, `PAPEIS_DA_SEMENTE=["accent","marca"]`. **Guarda ENTRADA nunca SAÍDA** (nunca armazena os 11 stops derivados) para que correções de contraste alcancem instalações existentes.

#### `instalacao.ts` — I/O de `platform_branding`
- Regras 🟢:
  - Memo de processo em `globalThis` (NÃO `let` de módulo) com `TTL_MS=30_000` + contador de geração — o Next instancia o módulo DUAS vezes por processo (runtime route.js vs page.js/ssr); um memo `let` zerado pela rota de escrita nunca era lido pela página de render (logo atrasava 19.6s). Geração lida antes do await, checada depois, para evitar lost-update em voo; `"erro"` fica FORA do memo.
  - Semeadura controlada por `seeded_from_env`: false → nunca re-semeia (protege a escolha humana de apagar a marca).
  - `CODIGOS_DE_RECUSA` define o que conta como fallback real e exclui `accent_deslocado`/`semantica_deslocada`/`cor_ausente`/`papel_nao_pinta` para o alarme `fallback_at` não ficar sempre ligado.

#### `logo.ts` / `logo-arquivo.ts` — storage
- 🟢 `BUCKET_DE_LOGOS="brand-logos"` (primeiro bucket PÚBLICO, exceção documentada), `TAMANHO_MAXIMO_DO_LOGO=512KB`, `PREFIXO_DA_INSTALACAO="platform/"`.
- 🟢 Guarda PATH nunca URL (a URL é determinística de path+host; guardar amarra a marca ao host Supabase de hoje).
- 🟢 `baseDoStorage()` lê `NEXT_PUBLIC_SUPABASE_URL` por chave MONTADA EM RUNTIME (`["NEXT","PUBLIC","SUPABASE","URL"].join("_")`) para derrotar a substituição estática do placeholder do Dockerfile pelo Next.
- 🟢 `podeApagar` (logo-arquivo.ts): o prefixo é reafirmado no DELETE, senão um admin de tenant poderia apagar o logo da instalação inteira (service-role bypassa RLS).
- 🟢 Tipo de arquivo decidido por ASSINATURA DE BYTE (`farejarTipo`: PNG `89 50 4E 47`, JPEG `FF D8 FF`), nunca por `file.type`; SVG banido com código próprio `logo_svg_recusado`.

#### `css.ts` — serialização, fail-closed
- `cssDaMarca(cor, escopo)`. Allowlist é de FORMA DE VALOR não de nome de token (`FORMAS_DE_VALOR`); rede de segurança pós-montagem rejeita `<` e `;}` (`SEQUENCIAS_SUSPEITAS`). Dupla especificidade `:root:root` (0,2,0) para vencer globals.css sem depender da ordem de link. Emite os 11 stops JÁ DESLOCADOS por `t.deslocamento` para o bloco concordar com `escolherAccent`.

#### `saida.ts` — saídas sem DOM (email/PDF/MFA)
- `marcaDaSaida(organizationId | null)`, `emailDeSuporte()`. **TEMA CLARO SEMPRE** (email não tem tema; Gmail derruba media queries). **Nunca lança** — o chamador é email de LGPD com SLA legal de D+7; degrada para o padrão do produto e loga uma vez. Classe A (org≠null: org→install→env→default), Classe B (null: install→env→default), Classe C (PDF, não chama). `emailDeSuporte` lê via `valorDaInstalacao("SUPPORT_EMAIL")` (banco acima do env, migration 0341).

#### Demais arquivos
- `contexto.tsx`: `MarcaDaInstalacaoProvider`/`useMarcaDaInstalacao()` — servidor resolve uma vez e passa a marca por PROP para evitar hydration mismatch (React #418); componentes client usam o hook, nunca `branding()`.
- `barra-do-navegador.ts`: `coresDaBarraDoNavegador(regua)` — lê `--color-bg`; a marca nunca move a barra; nunca lança (roda no viewport).
- `icone.ts`: `letraDoIcone(nome)` — primeira LETRA OU DÍGITO (`\p{L}|\p{N}` /u), maiúscula; null → sem letra, nunca cai na inicial do produto (white-label).
- `desenho.ts`: geometria da marca (SIMBOLO, LOGOTIPO, CORES_DA_MARCA) — não é SVG em public/ (vazaria a marca do produto).
- `regua-do-produto.ts`: const GERADA `REGUA_DO_PRODUTO` (extrairRegua sobre globals.css congelado no build; imagem standalone não tem globals.css).
- `linguagem.ts`: `TRADUCOES: Record<CodigoDaMarca, Traducao>` (união dos três tipos de código → código novo falha no typecheck aqui); jargão banido (rampa/stop/OKLCH/ΔE/token/WCAG) é asserção de teste.

**Grafo interno da marca**: rampa ← schema/contraste/css; contraste ← resolve/regua-do-produto/saida/barra; resolve ← organizacao/saida/instalacao; logo ← resolve/logo-arquivo. Importa também `@/lib/env`, `@/lib/logger`, `@/lib/supabase/admin`, `@/lib/instalacao/config`.

### 10.3 `lib/instalacao/` — configuração da instalação (banco acima do `.env`)

**Propósito** 🟢 — Mesma doutrina da marca (migration 0155 generalizou): o banco está ACIMA do `.env`, o `.env` é semente + piso de rollback. Nada lança; falhas degradam para o `.env` com log.

- `config-resolve.ts` (precedência PURA): `resolver(linha, doAmbiente): ValorResolvido` — banco vence se a linha existir (mesmo `valor:null` = branco intencional); senão env; senão ausente. `Fonte` = `banco | ambiente | ausente`. `texto()` trata whitespace como ausente.
- `config.ts` (I/O de `platform_config`, segredos cifrados): `valorDaInstalacao(chave)`, `estadoParaTela(chave, ehSegredo)` (nunca devolve o valor do segredo; só last4), `gravarPelaTela(...)` **fail-CLOSED** (sem chave de cifra → não grava, motivo `sem_chave_de_cifra`), `voltarAoAmbiente(chave)`. AES-GCM via `lib/crypto/aes_gcm`. `upsert` com `onConflict:"chave"` nunca `update`. SEM memo aqui (credenciais lidas no uso; o worker é segundo processo que nunca veria invalidação).
- `catalogo.ts` (catálogo declarativo, PT-BR): `CATALOGO_DA_INSTALACAO`. `controle:"edita"` SÓ para chaves que o código de produção lê via `valorDaInstalacao` (campo que aceita e ignora é pior que ausente). Primeira onda editável: RESEND_API_KEY, RESEND_FROM_EMAIL, SUPPORT_EMAIL, LGPD_DPO_EMAIL. Chaves de canal vêm de `CHAVES_DE_CANAL_DA_INSTALACAO` consumidas como DADO (doutrina de restrição de canal proíbe nomear provedor aqui).
- `comportamento.ts` (kill-switches de runtime, leitura STICKY): `ComportamentoDaInstalacao` { `orcamento_de_ia: "on"|"avisar"|"off"`, `exigir_assinatura_no_webhook: boolean`, `divulgacao_de_pagamento: "inject"|"veto"`, `promessa_semantica: boolean }`. Cada campo ruim cai para SEU próprio piso, não para o padrão do produto (lixo editado à mão não desliga a proteção de gasto). STICKY: último valor lido com sucesso vence. Sem `@/lib/env` e sem client de DB (importável por Next e worker).
- `comportamento-servidor.ts` / `comportamento-sql.ts`: leitor/escritor Next (`platform_settings` id=1) vs leitor do worker por pool `pg` (tipo estrutural `{query}` para não puxar `pg` no bundle).
- `ambiente.ts` (snapshot do env, sem DB): `lerAmbiente(source)`. Google NÃO tem chave de plataforma por design. `NOME_PLACEHOLDER_DA_INSTALACAO="Minha Empresa"`, `nomeAindaEhPlaceholder(org)`.
- `prova-de-credito.ts` (a chave de IA tem SALDO?): `provarSaldo(provider, apiKey, modelo, ...)` faz uma geração mínima REAL (`max_tokens:1`; openai usa `max_completion_tokens`) porque `GET /v1/models` devolve 200 numa conta com saldo zero. NÃO usa `runModelCall` (que escreve `llm_calls` e é orçamento-gated). `TIMEOUT_MS=8000`. Provedor desconhecido → fail-closed.
- `retrato.ts` (retrato da instalação): `lerRetratoDaInstalacao(deps)`. Precedência de credencial: credencial da org validada → "org"; senão chave de provedor no env → "instalacao"; senão "nenhuma". `prontaParaPublicar` = chave presente E modelo padrão curado existe.

### 10.4 `lib/navigation/` — registro único de navegação + gate de apresentação

**Propósito** 🟢 — `NAV_CATALOG` é a ÚNICA lista de destinos do app de tenant; sidebar, hubs e ⌘K são projeções puras. Configuração de interface é APRESENTAÇÃO, nunca autorização.

- `catalogo.ts`: `NavGroupId` = atendimento|crm|ia|canais|analise|organizacao. `NAV_GROUPS` (hub só onde >4 telas), `GRUPO_NO_RODAPE="organizacao"`. Entradas de `NAV_CATALOG`: href, label, description (nunca vazio; buscável no ⌘K), icon (chave string), group, section?, `minRole?` (padrão viewer), `sidebar?` (padrão só-hub), healthDot?. Muitas entradas sem `sidebar:true` de propósito porque o e2e `navegacao.spec.ts` exige o menu inteiro caber em 900px.
- `interface.ts` (gate de apresentação + interseção): `interfaceSettingsSchema` (zod .strict, preset completa|simplificada). `PORTAS_ESSENCIAIS` (profile, security, team, settings/tenant — não removíveis; settings/tenant hospeda a escolha, então escondê-la não pode trancar a org fora). `canSee(d, platform, role)` (compara ROLE_RANK — a ÚNICA função de autorização). `lerInterface(raw)` tolerante (inválido → COMPLETA = fail-OPEN). `combinarInterfaces(daEmpresa, doVinculo)` (migration 0367): INTERSEÇÃO nos dois eixos (org estreita o universo, vínculo estreita dentro; nenhum pode alargar); interseção vazia → SO_O_ESSENCIAL.
- `registry.ts` (projeções + ligação de ícone): liga chaves de ícone string a componentes Phosphor. `sidebarGroups(...)`, `hubSections(...)`, `searchable(...)`. Ponto único de decisão de permissão que substituiu 7 `usePermission()` sequenciais. Depende de `lib/auth/types` (Role, ROLE_RANK).

### 10.5 `lib/system/` — changelog + máquina de estado do update

- `changelog.ts` (parser Keep-a-Changelog, puro): `extractChangelogSection(raw, version)`, `extractChangelogRange(raw, alvo, instalada)`, `markdownParaTextoSimples`. `CHANGELOG_MAX_BYTES=64_000`. `body` = seção MENOS o bloco de atenção (evita exibição dupla). Seleção de range é POSICIONAL não semver (o arquivo é mais-novo-primeiro; um comparador tropeçaria em tag de fork `v1.1.1-jmpo.1`); 4 casos explícitos. `markdownParaTextoSimples` respeita a regra CommonMark de delimitador fora de palavra para `fn_user_org_ids()` manter os underscores.
- `update-run.ts` (estado da rodada de update na UI): `RunStatus` = dispatched|success|failed|failed_rolled_back; `RunStep` = backup|codigo|banco (espelha CHECK da migration 0089); `RUN_STALE_AFTER_MS=15min`.
  - `canTransition(from,to)` — só dispatched→terminal; terminal é imutável.
  - `rollbackFoiSuperado(...)` — um rollback só é superado quando o host reporta uma versão que a rodada NÃO descreve; a VERSÃO reportada decide, tempo sozinho não basta.
  - `sucessoJaInstalado(...)` — sucesso + to_version, dentro da janela stale, antes do heartbeat de 5min escrever `current_version`, para a UI não re-oferecer "Atualizar agora"; tem corte de validade (para de afirmar após `RUN_STALE_AFTER_MS`).
  - `rollbackDesmentidoPeloApp(run, versaoEmExecucao)` — para failed/failed_rolled_back, se o `APP_VERSION` do app rodando == run.to_version, a imagem nova está de pé (caso reinstalar-mesma-versão-e-funcionou). Lê `APP_VERSION` assado na imagem, nunca `APP_IMAGE`.
  - `RodadaDoBanco` {disputa, retentativas, passada} + narração PT-BR; devolve null em estados impossíveis.

### 10.6 `lib/operacao/` — superfícies de operação (tela + agente de IA compartilham)

**Comum** 🟢 — todas recebem `DepsDaOperacao` { supabase, organizationId, actor, requestId } (`entradas-automaticas.ts:31-36`). Toda query é org-scoped; capacidades de escrita que tocam o mundo externo são classificadas `critico` (nunca concedidas por pacote). Erros via `ApiError`. Escritas carimbam autoria via `autoriaDaMudanca(deps.actor)` + `audit(...)`.

- `autoria.ts`: `autoriaDaMudanca(actor, agora)` { `last_change_actor_kind: EspecieDeAutor`, `last_change_at` }, `autorNaTela(kind)` (ai→"alterado pelo assistente", system→"...automaticamente", user/null→NENHUM selo). Guarda ESPÉCIE não id do agente (deriveActor devolve id de run/token, não de agente; uma FK daria 23503 no caminho MCP). Mudança humana não emite selo (7/8 selos "você" afogavam o 1 que importava).
- `entradas-automaticas.ts` (`webhook_sources`): `listarEntradasAutomaticas`, `criarEntradaAutomatica`, `definirEntradaAtiva`, `recebimentosDaEntrada`. Entidade `FonteVisivel` (secret nunca sai → `has_secret`). Validação de TENANCY de pipeline+stage é obrigatória aqui (`destinoValido`) porque a tool de IA usa service-role bypassando RLS; checa também que o stage pertence àquele pipeline e não está arquivado (422s). `path_token` = 24 bytes aleatórios base64url. Colunas explícitas, nunca `*`.
- `regras-automaticas.ts` (`automation_rules`): `listarRegrasAutomaticas`, `definirRegraAtiva`, `execucoesDasRegras`. Superfície mais perigosa — regra dispara para sempre sem vigilância; toggle é `critico`. Agente só pode alternar (humano escreveu/revisou a regra), NUNCA criar/editar/apagar. `actions` nunca sai com `config` cru (segura secret_enc + URL de webhook). `execucoesDasRegras` devolve só detalhes de ações FALHAS.
- `marcadores-e-time.ts` (tags + time, só-leitura): `listarMarcadores(deps, {limite=60})` — funde vocabulário oficial (`organizations.settings.canonical_conversation_tags`) + tags em uso; listar (não um segundo escritor) é a cura para tag drift; contagem em JS (tags é text[]). `listarTime(deps)` { user_id, papel, convite_pendente, desde } — SEM email/nome/último-acesso (vai para o contexto do LLM, poderia ser ecoado a um cliente).
- `modelos-de-mensagem.ts` (`message_templates`): `listarModelosDeMensagem(deps)` (compartilhado = owner_user_id null), `preencherModeloDeMensagem(...)`. PREENCHER ≠ ENVIAR (enviar é `crm_send_whatsapp_message`, critico); devolve TEXTO só. Agente não cria template. LACUNAS devolvidas explicitamente (`lacunasDe` renderiza cada `{{marcacao}}` isolada; vazia→lacuna) porque `renderTemplate` substitui ausente por "" gerando "Olá , tudo bem?".

### 10.7 `lib/onboarding/` — passos do wizard + sugestão de funil

- `passos.ts` (definição única de passo): `PASSOS` (welcome, connect-whatsapp, connect-nuvemshop, setup-ai, funil, testar, invite-team), `passosVisiveis(ctx)`, `proximoPasso(state, ctx)`. Um passo decide a própria existência (`existe`) — nuvemshop só quando `lojaLigada`, então nunca aparece como fantasma/pulado. `funil` vem DEPOIS de `setup-ai` (a sugestão usa a chave/modelo recém-confirmados).
- `proposta-de-funil.ts` (regras da proposta de quadro, sem DB/IA): `normalizarProposta`, `validarProposta`, `etapasParaGravar`. `MIN_ETAPAS=4`, `MAX_ETAPAS=8`. Uma etapa é NOME+PASSO inseparável (medido: 312 etapas/43 funis, só 4 com `agent_stage_hint` → toda instalação nascia muda). `is_won`/`is_lost` DERIVADOS do passo. Rejeita: sem nome, <MIN, sem etapa won (senão /win 422 `pipeline_no_won_stage`), sem etapa lost. `position = (i+1)*1000`. RECUSAR é o desfecho certo (o chamador cai num pacote curado), nunca auto-completar.
- `pacotes-de-funil.ts` (quadros curados por tipo de negócio): `PACOTES` (clinica, imobiliaria, servicos, curso, loja, generico), `PACOTE_PADRAO` (busca por id, lança no load se ausente — nunca `PACOTES[0]`). Existem como PLANO B (IA indisponível) E como a RÉGUA contra a qual a sugestão de IA é comparada. Toda etapa carrega o passo `LeadStage`.
- `sugerir-funil.ts` (sugestão de IA, geração injetável): `escolherPacotePorTexto`, `sugerirFunil(ctx, gerar)`. Usa o MESMO cérebro/modelo/chave do agente publicado. `extrairJson` fatia do primeiro `{` ao último `}` (modelos embrulham em ```json/prosa). Pipeline: escolherPacotePorTexto → gerar (único ponto de rede, injetado) → extrairJson → normalizarProposta → validarProposta; QUALQUER falha → pacote com `porque` (nunca quadro vazio). `PISTAS` (mapa de regex por tipo; empate quebrado pela ordem de `PISTAS`).
- `o-que-mais-existe.ts` (descoberta pós-wizard): `oQueMaisExiste()` — 7 destinos curados, cada um com comoChamar/porQue/comoFunciona[] escritos à mão, mas `label`+`descricao` puxados de `NAV_DESTINATIONS` (uma fonte). Destino removido do registro → não vira card órfão (flatMap pula).

### Padrões transversais desta unidade
- 🟢 **"Banco acima do `.env`, `.env` é semente + piso de rollback"** — em branding (instalacao.ts), instalacao (config-resolve.ts, comportamento.ts) e saida.ts, porque `agent.sh` reverte só a imagem, não o schema.
- 🟢 **"Nunca lançar no caminho de render/layout; degradar + carregar um motivo"** — resolve.ts, instalacao.ts, saida.ts, contexto.tsx, barra-do-navegador.ts, comportamento.ts, config.ts.
- 🟢 **Memo de processo em `globalThis`** (não `let` de módulo) porque o Next instancia módulos duas vezes por processo (route vs page), com contador de geração contra lost-update.
- 🟢 **Service-role bypassa RLS → filtro explícito de `organization_id` é o ÚNICO guarda de tenant** — apagar-dados-operacionais.ts, entradas-automaticas.ts, saida.ts, I/O de instalacao.
- 🟢 **Diagnósticos emitem FORMA nunca IDENTIDADE** (sem hex do cliente/PII em logs/motivos) — contraste.ts, resolve.ts, instalacao.ts.
- 🟢 **Fonte única para matar drift** — registro de nav (catalogo.ts), passos de onboarding (passos.ts), régua-do-produto, pacotes vs sugestão de funil.

### Escala de confiança e lacunas — Unidade 10
- 🟢 Todo o lado TypeScript desta unidade foi lido diretamente (settings, branding completo, instalacao, navigation, system, operacao, onboarding).
- 🔴 As tabelas/CHECKs/RLS que estas funções assumem (`platform_branding`, `platform_config`, `platform_settings`, `webhook_sources`, `automation_rules`, `message_templates`, colunas `last_change_actor_kind`/`last_change_at`, CHECK `crm_stages_hint_coerente_com_won_lost`, CHECK da migration 0089) ficam para o **Data Master**; aqui só o lado que as invoca.
- 🟡 A conta exata de destinos em `NAV_CATALOG` e de chaves em `CATALOGO_DA_INSTALACAO` foi lida do array, não contada do disco em runtime; o Redator/Arquiteto deve reconferir com `git grep`/`wc -l` antes de citar número.
- 🟡 `provarSaldo` (prova-de-crédito) e `sugerirFunil` fazem chamada de rede real a provedores; o comportamento exato de cada provedor sob erro (rate limit, saldo zero) foi inferido do mapeamento em `classificarResposta`, não observado ao vivo.

---

## Unidade 11 — Integrações externas: `lib/external-db/`, `lib/nuvemshop/`, `lib/plataformas-de-anuncio/`, `lib/extensions/`

### Propósito 🟢
Quatro subsistemas independentes que ligam o CRM ao mundo de fora, cada um com sua própria fronteira de confiança. Padrão comum às quatro: `organization_id` SEMPRE no filtro (service-role bypassa RLS), segredo cifrado em repouso (AES-GCM) e decifrado just-in-time (nunca logado, nunca devolvido por rota), e falha-fechada (DNS que não resolve, cifra ausente, código de banco desconhecido → recusa segura).

### 11.1 `lib/external-db/` — conector read-only para um Postgres externo do cliente 🟢

Acesso só-leitura ao Postgres próprio da organização (grade na tela + tools do agente de IA). Camadas: `credenciais` (decifra) → `guardas` (rede) → `acesso` (orquestra) → `conexao` (pool + transação read-only) → `introspeccao` (catálogo ao vivo) → `leitura` (montador de SELECT seguro). `limites`/`schemas`/`types` são vocabulário compartilhado.

- **`limites.ts`** (só números, client-safe): `LIMITE_LINHAS={minimo:1,maximo:5000,padrao:200}`, `LIMITE_FILTROS={minimo:0,maximo:100,padrao:20}`, `LIMITE_RESPOSTA_BYTES={minimo:4096,maximo:1048576,padrao:30000}`, `LIMITE_PADRAO_DA_GRADE=50`. São o piso/teto ABSOLUTOS, espelhados no CHECK do banco e no Zod da rota. `respostaBytes` limita bytes só para o MODELO (a grade ignora).
- **`schemas.ts`** (Zod): `MODOS_TLS=["disable","prefer","require","verify-ca","verify-full"]` (espelha CHECK de `external_db_connections.ssl_mode`). `criarConexaoSchema` (`.strict`): label(1-80), host(1-255), port(int 1-65535, def 5432), database_name(1-128), username(1-128), password(1-2048), ssl_mode(def "require"), enabled(def true). `leituraQuerySchema`: limit(coerce 1..maximo, def 50), offset(≥0), order_by, order_desc enum(["true","false","1","0"]), colunas(csv ≤4000). **SEM parâmetro de filtro na querystring de propósito** — filtro carrega PII e querystring vaza para log de proxy.
- **`credenciais.ts`** — `cifrarSenha(senha)` devolve `{password_encrypted,password_iv,password_tag}` via AES-256-GCM (`@/lib/crypto/aes_gcm`). `carregarConexao(admin, organizationId, connectionId)` filtra `.eq("organization_id")` **E** `.eq("id")` (lição da #236: admin bypassa RLS). `MotivoSemConexao` = nao_encontrada|desativada|cifra_indisponivel|banco. Decifra que falha → `cifra_indisponivel` (falta `AI_CRED_AES_KEY` = problema de INSTALAÇÃO, distinto de "não existe"); erro do decrypt NÃO é logado. `limiteOuPadrao`: null/inválido/negativo → padrão, nunca "ilimitado".
- **`guardas.ts`** — guarda de egress (defesa SSRF). `validarHostDeBanco(host)`, `ipDeBancoProibido(ip)`. **⚠️ POLÍTICA DELIBERADAMENTE DIFERENTE DA DO WEBHOOK**: faixas RFC1918 (10/8, 172.16/12, 192.168/16) são PERMITIDAS (banco na LAN é caso real do dono da VPS, que cadastra como `admin`). BLOQUEADOS SEMPRE (`FAIXAS_PROIBIDAS`): 0/8, 100.64/10 (CGNAT), 127/8, 169.254/16 (link-local, inclui o metadata de nuvem 169.254.169.254), 192.0.0/24, 192.0.2/24, 198.18/15, TEST-NETs, 224/4 (multicast), 240/4. IPv6 normalizado a 16 bytes (`ipv6ParaBytes`) e `ipv6Proibido` barra ::, ::1, fe80::/10, fc00::/7, ff00::/8, 2001:db8::/32, NAT64 64:ff9b::/96 e IPv4-mapeado por recursão. Hostname resolvido por `dns.lookup(all:true)`; recusa se QUALQUER endereço cair em faixa proibida (rebinding devolve um público e um privado). DNS falho → fail-closed. Janela residual de DNS-rebinding declarada, não escondida.
- **`acesso.ts`** — `abrirAcesso(admin, organizationId, connectionId)`: `carregarConexao` → `validarHostDeBanco` (RE-VALIDADO a cada leitura, não só no cadastro, para fechar a janela de rebinding) → `obterPool`.
- **`conexao.ts`** — pool + read-only. Constantes: `MAX_POOLS=32`, `MAX_CONEXOES_POR_POOL=2`, `CONNECTION_TIMEOUT_MS=5000`, `STATEMENT_TIMEOUT_MS=10000`, `LOCK_TIMEOUT_MS=5000`, `IDLE_TX_TIMEOUT_MS=15000`. Pool chaveado por `id\0versao\0host\0port\0database\0username\0sslMode` (editar credencial derruba pool velho — a senha antiga não mora em memória); LRU quando > MAX_POOLS. `sslPara(modo)`: disable→false; prefer/require→`{rejectUnauthorized:false}` (cifra sem verificar cadeia); verify-*→`{rejectUnauthorized:true}` (fail-closed). **Regra central `consultar()`**: abre `begin read only` + `set local statement_timeout/lock_timeout/idle_in_transaction_session_timeout`. Só-leitura imposto pelo POSTGRES, não por análise de string — qualquer INSERT/UPDATE/DDL é recusado. `SET LOCAL` por transação (não `options` do pool) porque o PgBouncer do Supabase rejeita startup options.
- **`leitura.ts`** (SEGURANÇA-CRÍTICA) — montador de SELECT seguro. `quotarIdentificador` (`"`→`""`), `escaparLike` (escapa `\ % _`), `exigirColuna` (lança `coluna_inexistente:<c>` se fora do conjunto do catálogo). `clausulaDeFiltro`: switch fechado; valores viram `$n`. **Regra C-008**: `contem`/`comeca_com` são TOLERANTES A ESPAÇO E CAIXA (`replace(lower(cast(col as text)),' ','') like … escape '\'`) porque o modelo manda "cb250"/"CB250" e a base tem "CB 250 F Twister" — ILIKE estrito devolvia zero e a IA concluía "não temos". `montarConsulta`: valida TODA coluna (projeção/filtro/ordem) contra o conjunto do catálogo; teto = `min(LIMITE_LINHAS.maximo, limiteMax válido senão absoluto)`, limiteMax inválido (NaN/negativo) cai no absoluto, nunca abre. Identificadores quotados+validados, valores parametrizados — nenhum caminho por onde texto do usuário vira SQL.

### 11.2 `lib/nuvemshop/` — OAuth + cliente REST da Tiendanube/Nuvemshop 🟢

- **`config.ts`**: `NUVEMSHOP_AUTH_BASE="https://www.tiendanube.com"`, `NUVEMSHOP_API_BASE="https://api.tiendanube.com/v1"`, `APP_USER_AGENT`. `getConfig()` lê env `NUVEMSHOP_APP_ID/CLIENT_ID/CLIENT_SECRET` e devolve `null` se faltar (env opcional no build; chamadores expõem `not_configured`). `SUBSCRIBED_EVENTS`: order/created|updated|paid|cancelled, product/created|updated|deleted, app/uninstalled. `eventToSlug` (`/`→`-`), `slugToEvent`.
- **`oauth.ts`** (SEGURANÇA-CRÍTICA): header do webhook é `x-linkedstore-hmac-sha256` (hex, chave = client_secret). **Regra: `user_id` da resposta de token É o storeId; tokens não expiram.** `buildAuthorizeUrl({appId,state})` sempre inclui `state` como defesa CSRF (mesmo Nuvemshop não exigindo). `exchangeCodeForToken(code,cfg)`: POST JSON `grant_type:"authorization_code"`, `cache:"no-store"`; erros network_error/token_exchange_failed/invalid_token_response; sucesso exige `access_token` E `user_id`. `verifyHmac(rawBody, signatureHex, clientSecret)`: HMAC-SHA256 sobre rawBody (**não re-stringificar**), decodifica hex, checa comprimento, `crypto.timingSafeEqual`; false em qualquer erro de parse.
- **`state.ts`** (CSRF, SEGURANÇA-CRÍTICA): formato `base64url(orgId.nonce.expMs[.userId.authSessionId]) + "." + hex(HMAC-SHA256)`. `TTL_MS=10min`. `key()` usa `INTERNAL_SECRET` se ≥16 chars, senão fallback aleatório por processo (só dev). `issueState`/`verifyState` com length-guard + `timingSafeEqual`, payload de 3 ou 5 segmentos, `expMs` finito, checagem de expiração.
- **`api-client.ts`**: `NuvemshopApiClient` — `headers` usa **`Authentication: bearer <token>`** (minúsculo "bearer", conforme spec Nuvemshop, NÃO `Authorization: Bearer`). `request<T>` `cache:"no-store"`; mapeia status→code (401 unauthorized, 403 forbidden, 404 not_found, 429 rate_limited, ≥500 upstream_error); wrappers getStore/listWebhooks/createWebhook/deleteWebhook.

### 11.3 `lib/plataformas-de-anuncio/` — integrações Meta/Google Ads 🟢

Dois eixos INDEPENDENTES: **conversões** (ESCREVE na conta de anúncio do cliente) e **leitura/insights** (LÊ métricas). Deliberadamente separados de `lib/channels/` (reportar conversão não tem física de canal — sem janela/template/ban/destinatário). Despacho por registry.

- **`types.ts`**: `PlataformaDeAnuncio="meta_ads"|"google_ads"`; `NomeDoEvento="Purchase"` (Lead é Fase 2). `ConversaoOffline`: organizationId, leadId, evento, `eventoId=<leadId>:<evento>` (dedup determinístico — segunda cópia descartada pela plataforma), ocorridoEm (=`closed_at` do lead, nunca `now()`), cliqueDeOrigem (`ad_source_id` do contato), telefone (E.164 sem `+`, EM CLARO — hash é do transporte), valorCentavos, moeda. `ResultadoDeEnvio` = `ok`|`transitorio`(reagenda sem contar tentativa)|`permanente`(precisa mexer na config, vira linha de erro). **Regra: nunca fundir os dois tipos de falha.** `CredencialDeConversao`: datasetId, accessToken, testEventCode, `google?:{refreshToken,customerId,loginCustomerId(null=sem MCC),conversionActionId}`. Eixo de leitura (0214): `CredencialDeLeitura`, `ContaDeAnuncio{id,nome,moeda,status(account_status cru)}`, `FalhaDeLeitura`=token_invalido|permissao_insuficiente|limite_de_chamadas|campo_invalido|transitorio.
- **`credenciais.ts`** (SEGURANÇA-CRÍTICA) — `lerCredencial(admin, organizationId, plataforma)` de `ad_platform_connections` (RLS + ZERO policies, migration 0213 → EXIGE admin client). Filtra `.eq("organization_id")` + `.eq("platform")` (lição #236, índice único `(organization_id,platform)`). Completude POR PLATAFORMA: `google_ads` exige `google_refresh_token_encrypted`+`google_customer_id`+`google_conversion_action_id`, decifra o refresh token, devolve `accessToken:""` de propósito (Google deriva na hora); Meta exige `dataset_id`+`access_token_encrypted`. Decifra falha → `cifra_indisponivel`.
- **`credenciais-de-leitura.ts`** — `lerCredencialDeLeitura` de `ad_insights_connections`; `MotivoSemLeitura`=sem_conexao|cifra_indisponivel (um a menos que conversões — sem interruptor `enabled`; desconectar = apagar linha). `existeConexaoDeLeitura` checa presença SEM decifrar (mantém token fora de Server Component que renderiza HTML).
- **`registry.ts`** — `TRANSPORTES: Record<PlataformaDeAnuncio, TransporteDeConversao|null>` = {meta_ads:transporteMeta, google_ads:transporteGoogle}. Entrada `null` deixa uma plataforma entrar no vocabulário antes de ter transporte sem virar `undefined`; o chamador loga `plataforma_sem_transporte` (registrado, não omitido).
- **`hierarquia-do-contato.ts`** — `resolverHierarquiaDoContato`, cache-first (`ad_hierarchy_cache`, RLS zero policies 0380). `IDADE_MAXIMA_MS=7 dias`. Nunca lança; devolve cache VELHO quando a plataforma recusa re-leitura (nome velho > erro); resolvido preguiçosamente ao abrir a ficha, NÃO na ingestão (rede em hot path → tempestade de reentrada + custo de cota).
- **`google/conversions.ts`** (SEGURANÇA-CRÍTICA) — access token NÃO é armazenado: derivado a cada envio via `renovarToken(app, google.refreshToken)`. `formatarDataDeConversao` → `yyyy-MM-dd HH:mm:ss+00:00` UTC. Corpo: `conversions[{gclid, conversionAction:customers/<id>/conversionActions/<actionId>, conversionValue=centavos/100, orderId=eventoId(dedup)}]`, `partialFailure:false`. Headers `Authorization: Bearer`, `developer-token`, `login-customer-id` opcional. `classificaErro`: ≥500 ou RESOURCE_EXHAUSTED/UNAVAILABLE/429 → transitorio; senão permanente. NÃO valida idade do evento (deixa o Google recusar).
- **`meta/conversions.ts`** (SEGURANÇA-CRÍTICA) — `IDADE_MAXIMA_MS=7 dias` (evento mais velho → permanente com contagem de dias). `hash(v)`=SHA-256 hex de `trim().toLowerCase()` (PII: telefone como `ph:[hash(...)]`). `action_source:"business_messaging"` + `messaging_channel:"whatsapp"`. `event_time=floor(ms/1000)` (segundos; ms daria ano ~55000 com 200 silencioso). URL `graph.facebook.com/<ver>/<datasetId>/events`, token só em header. `classifica4xx`: 613/80004 → transitorio (throttle); resto permanente (190/102/463 = token).

### 11.4 `lib/extensions/` — sistema de extensões declarativas (modelo de capacidade) 🟢

**Encarnação da doutrina**: extensões são pacotes declarativos que pedem CAPACIDADES NOMEADAS de uma lista fechada; o host traduz o nome num destino que é CONSTANTE DE CÓDIGO — o pacote nunca fornece/monta/influencia endereço, nunca importa código do núcleo, nunca recebe cliente Supabase. `data.mode` fixo em `"none"`.

- **`capacidades.ts`** (NÚCLEO, SEGURANÇA-CRÍTICA): `EXTENSION_PERMISSIONS` = navigation.tasks|inbox|kanban|contacts|agenda|radar. `EXTENSION_CAPABILITIES` = tasks.open|inbox.open|kanban.open|contacts.open|agenda.open|radar.open. `PORTA_DA_CAPACIDADE: Record<ExtensionCapability, {destino, permissao}>` — Record EXAUSTIVO de propósito: capacidade nova sem porta não COMPILA, em vez de virar `undefined` em runtime. Cada capacidade → destino LITERAL (`/app/tasks`…), nunca concatenado. **Regra (ADR-0003)**: só telas de TRABALHO qualificam; config/credencial/cobrança/webhook/admin excluídas — "uma extensão guia o trabalho, nunca leva alguém para onde ficam os segredos". `destinoDaCapacidade(cap)` → destino ou `null` (o chamador recusa, nunca um destino de reserva).
- **`manifest.ts`** — `EXTENSION_LIMITS`: packageBytes=64KB, catalogBytes=512KB, jsonDepth=12, jsonNodes=20000, objectProperties=32, catalogEntries=128, cards=4, blocksPerCard=8. `HOST_API_VERSION=2`. `ExtensionManifest`: format_version:1, profile:"declarative", publisher/name(slug), version(semver), license:"MIT", host_api{min,max}, permissions[], dependencies:`z.tuple([])` (vazio), data{mode:"none"}, display{title/summary LocalizedText pt-BR+es?, category, icon}, contributions.crm_cards[]. `CatalogEntry`: subset + sha256 + byte_length + metadados de loja OPCIONAIS (publisher_label/homepage/repository — moram no CATÁLOGO revisado, nunca no pacote — ADR-0003 D3). `checkCompatibility`: format_version, profile, `host_api.min ≤ 2 ≤ max`, permissões conhecidas, dependências vazias, capacidades conhecidas, e **regra de cobertura**: toda permissão exigida pela capacidade do card tem que estar declarada. `validateArtifact`: byte_length casa + ≤packageBytes; **SHA-256 tem que bater** (senão `extension_digest_mismatch`); `mirrorsCatalog` (manifesto == entrada do catálogo); compatibilidade.
- **`strict-json.ts`** (SEGURANÇA-CRÍTICA) — `parseStrictJson(bytes, limits)`. `FORBIDDEN_KEYS`=__proto__/prototype/constructor (anti prototype-pollution). Rejeita BOM, comentários, tokens desconhecidos, strings não-JSONB-safe (`\0`, pares surrogates inválidos); limites de profundidade/nós; chaves duplicadas.
- **`download.ts`** (SSRF-endurecido, SEGURANÇA-CRÍTICA) — `DOWNLOAD_TIMEOUT_MS=15000`. `validateCatalogOrigin`: exceção local só quando `http://127.0.0.1` E `policy.localCatalogOrigin===origin` E appUrl loopback; senão https obrigatório, recusa `.localhost`, recusa IP literal não-público. **DNS pinning (anti-rebinding)**: `resolvePublicAddresses` resolve tudo e recusa se algum não-público; `pinnedLookup` entrega ao socket SÓ os endereços pré-classificados — sem segunda resolução DNS. `readResponse`: status 200, `content-encoding` identity, content-length ≤ packageBytes E ≤ entry.byte_length.
- **`service.ts`** (NÚCLEO, `server-only`) — orquestrador. Schemas Zod das linhas (catalog/artifact/installation/binding/operation). `dbFailure`: mapeia P0001 + message via `SQL_ERRORS`; desconhecido → 503 `upstream_unavailable` (nunca vaza detalhe do banco). Persistência INTEIRAMENTE via RPCs `fn_extensions_*` (migrations 0271/0340) — o serviço NUNCA escreve tabelas direto; toda RPC devolve `applied_now` (idempotência). `installExtension`: RPC prepare → `validateCatalogSnapshot` → `checkCompatibility` → `downloadArtifact` → `validateArtifact` → RPC finish + audit. Entrada vem do RECIBO preparado, não do request.
- **`http.ts`** — fronteira de API. `ExtensionServiceError{code,message,status=409}`. `requireExtensionPlatform`: is_platform_admin && !support, `platform_admins.scope==="full"`, MFA (aal2). `operationKey`: `Idempotency-Key` tem que ser UUID (senão 422). `requireExtensionOrganization`: header `X-Expected-Organization-Id` tem que casar (precondição, nunca autoridade).
- **`vocabulario.ts`** — fonte única espelhada no banco. `EXTENSION_OPERATION_KINDS`=catalog_admission, install, update, revert, removal, configure, module_install. `EXTENSION_OPERATION_STATUSES`=preparing, completed, failed, cancelled. CHECK no banco (0271), verificado por invariante.
- **`erros-do-banco.ts`** — `SQL_ERRORS: Record<string,{message,status}>` (25+ códigos de `fn_extensions_*`): extension_forbidden(403), extension_idempotency_conflict(409), extension_removed(**410**), extension_active_limit(409), extension_module_unknown(404)… Código ausente → 503 genérico.
- **`versao.ts`** — `compararVersoes(a,b)` compara cada parte x.y.z como texto-de-dígito por comprimento+lexicográfico (evita colisão de `Number` >2^53). `ehTrocaParaVersaoMenor` separa "Atualizar" de "Trocar para menor".
- **`errors.ts`** — `EXTENSION_ERROR_CODES` (6 estáveis). `PUBLIC_MESSAGES` — mensagem pública nunca incorpora bytes/texto do pacote. `causaSegura(error)` extrai só `{cause_code (^[A-Za-z0-9_]{1,40}$), cause_status}` — texto remoto/do pacote nunca vaza para log/audit.

### Padrões transversais desta unidade
- 🟢 **Segredo cifrado em repouso (AES-GCM), decifrado JIT**: external-db (3 colunas via `@/lib/crypto/aes_gcm`), ad-platforms (`decryptWebhookSecret`). Nunca logado, nunca devolvido por rota.
- 🟢 **`organization_id` sempre no filtro com admin client** (bypassa RLS) — external-db, ad credenciais, hierarquia; todas citam a lição da #236.
- 🟢 **HMAC com `crypto.timingSafeEqual` + length-guard** — Nuvemshop `verifyHmac` e `state.ts`.
- 🟢 **Defesas SSRF/rebinding** — external-db `guardas.ts` (blocklist de faixa IP + resolve-tudo-recusa-qualquer) e extensions `download.ts` (DNS pinning).
- 🟢 **Bearer só em header, nunca querystring** — meta/google conversions, api-client.
- 🟢 **Anti prototype-pollution** no parse de JSON de extensão (`FORBIDDEN_KEYS`).

### Escala de confiança e lacunas — Unidade 11
- 🟢 Todo o lado TypeScript foi lido diretamente.
- 🔴 As tabelas/RLS/CHECKs que estas funções assumem (`external_db_connections`, `ad_platform_connections`, `ad_insights_connections`, `ad_hierarchy_cache`, `extension_catalogs`/`installations`/`operations`/`organization_extensions`, `idempotency_keys`) e os corpos das RPCs `fn_extensions_*` ficam para o **Data Master**.
- 🟡 O comportamento exato de cada provedor externo sob erro (rate limit, saldo zero, versão de API desativada) foi inferido do mapeamento em `classificaErro`/`classifica4xx`, não observado ao vivo.

---

## Unidade 12 — Eventos e tempo real: `lib/event-log/`, `lib/realtime/`, `lib/relogio/`, `lib/tempo/` + `workers/`

### Propósito 🟢
Duas filas duráveis paralelas no MESMO Postgres, drenadas por um worker 24/7 E (como rede de segurança) por tiques HTTP de cron/relógio; trigger de Postgres NUNCA faz HTTP:
- **`event_log`** — event sourcing leve com fan-out para handlers registrados (`lib/event-log/`).
- **`job_queue`** — fila de trabalho do agente (`lib/agent-engine/queue/`, já detalhada na Unidade 1); eventos `ai_agent.dispatch_requested` viram jobs por um drain à parte (`edge/crm/drain.ts`).
- **`cron_jobs`** — agendador persistente por contato (`lib/agent-engine/cron/`) que, ao disparar, ENFILEIRA um job (nunca reimplementa a fila).

### 12.1 `lib/event-log/` — escrita, drain e dispatch 🟢

- **`dispatcher.ts`** — registro de handlers. `EventRow`: id, organization_id, event_type, entity_kind, entity_id, payload, metadata, `consumed_by: string[]`, attempts, `created_at?` (OPCIONAL de propósito — 26 arquivos constroem EventRow; produção sempre o traz; quem depende dele para DESCARTAR precisa falhar ABERTO). `HandlerResult`: consumer_key, status `"ok"|"skipped"|"error"|"retry"`, `retry_at?` (obrigatório se status="retry"), detail?. `EventHandler`: key (estável, gravada em `consumed_by`), events[], `handle(row)`. Registro em módulo (`_handlers[]` + `_registeredKeys`); `registerHandler` hot-reload-friendly (sobrescreve por key). `dispatchEvent(row)` filtra handlers cujo `events.includes(type)` E key não está em `consumed_by`, roda cada um em try/catch (handler que lança vira `{status:"error"}`, não aborta o lote).
- **`drain.ts`** — o algoritmo central. `MAX_ATTEMPTS=5`; `PROCESSING_STALE_MS=10min`; `backoffAt(attempts)` = `2^attempts` MINUTOS (1,2,4,8… — expoente é attempts direto, difere do backoff da job_queue). `DrainSummary`: `pulados?[]`, scanned, done, retried, failed, dead. `drainEventLog(admin, {limit=50})`:
  1. `handledTypes` = eventos distintos dos handlers registrados; vazio → retorna (tipos drenados por cron dedicado, como `ai_agent.dispatch_requested`, ficam intocados).
  2. **Reaper de `processing` preso**: seleciona `status='processing'` E `updated_at < now()-PROCESSING_STALE_MS`; `attempts+1`, `dead` no limite; reclaim otimista `where status='processing'`. **Regra**: evento órfão CONTA como tentativa (conserta o laço do PDF envenenado que derrubou o worker 313× em 2026-09-15 porque crash não incrementava attempts). **Exceção**: `primeiraVolta = preso.attempts===0` → primeira volta reprocessa NO MESMO TIQUE (`next_attempt_at=null`); da segunda em diante paga `backoffAt`. `updated_at` confiável via trigger `trg_event_log_touch` (BEFORE UPDATE).
  3. Select principal: `status='pending'` E `(next_attempt_at is null OR <= now())` E `event_type in handledTypes`, `order by created_at asc limit`.
  4. Por linha: **claim otimista** `update status='processing' where id=? and status='pending'`; `if (!claimed) continue` (é isto que torna worker-loop + cron seguros em paralelo).
  5. `dispatchEvent(row)` → particiona ok/skipped, retry, errors. Ramo **retry**: volta a `pending`, NÃO conta attempt, `next_attempt_at = retry_at ?? backoffAt`. Ramo **error**: `attempts+1`, `dead` no limite, `avisarEventoMorto` se dead. Ramo **success/skipped**: `done`; `detail` do skip preservado em `last_error` e em `summary.pulados` (para o trace do CI mostrar qual pulo disparou).
- **`avisarEventoMorto`** (+ `aviso-de-evento-morto.ts`) — `agent_inbox_items` `kind='event_dead'`, `severity='critical'`. Dedupe: um aviso aberto por org por família, EXCETO exclui `IA_QUE_NAO_RESPONDEU.titulo` (duas famílias no mesmo kind). Fire-and-forget: falhar ao avisar não derruba o dreno. Motivo do nascimento: quatro `media.derive_requested` mortos numa VPS com o cliente ouvindo "não consigo ouvir áudio" e ninguém sabendo.
- **`register-handlers.ts`** — `ensureHandlersRegistered()` (idempotente) registra EM ORDEM (a ordem importa): followupReactivity, aiResponse, aiSentiment, aiHandoffFromSentiment, ragIndexer, lgpdExport, lgpdRedact, automationRules, followupGatilhoEtapa/Caso/Presenca, mediaPersist, mediaDerive, webPushInbound, avisoDeCasoAoSuporte (penúltimo — rede de terceiro), conversaoDeVenda (último — o mais externo).
- **`origem-do-dreno.ts`** — `AsyncLocalStorage<"worker"|"request">`. `comOrigemDeRequest(fn)` marca; `origemDoDreno()` default `"worker"` (fail-safe). Impede handlers de rede de terceiro de adicionar latência DENTRO do webhook do WhatsApp (que tem timeout + redelivery). ALS, não flag de módulo, porque requests concorrentes intercalam nos awaits.

### 12.2 `lib/event-log/drain-loop.ts` — o laço do worker 🟢
- **Imports dinâmicos de propósito** (`carregarDepsDoLaco`): a cadeia termina em `@/lib/env`, que lança no topo do módulo se faltar var; import estático mataria o WORKER INTEIRO. Carrega `createAdminClient`, `drainEventLog`, `ensureHandlersRegistered()` dentro de try/catch. Nunca lança; devolve `Deps|null`. Sucesso → readiness + `log.info(MARCA_LACO_CARREGADO='event-log drain: laço carregado')` — string de CONTRATO checada pelo gate de publicação (`scripts/sonda-do-laco-de-event-log.ts`, `publish-image.yml`). Falha → `log.error` (não warn — a #648 ficou muda 10 dias) + aviso 'degradado' na Central.
- `proximaEspera(resumo, knobs)`: `feitos = done+retried+failed+dead`; `intervalMs` se >0, senão `idleIntervalMs`. `scanned` FORA da conta de propósito (linha que outra instância reivindicou primeiro não pode manter o laço rápido girando à toa).
- `runEventLogDrainLoop`: carrega deps (early-return se null); `while(!signal.aborted)`: tick que EXPLODE mantém a espera ociosa (banco fora não pode virar tempestade de tentativas).

### 12.3 `workers/agent-worker/main.ts` — entry point do worker 24/7 🟢
- `Sentry.init` ANTES dos imports de negócio; DSN da comunidade → tracesSampleRate 0.
- `ehVetoPermanenteDeNegocio(err)`: lê `err.terminal===true` (único produtor: `LlmBudgetExceededError`) — distingue veto de negócio permanente (→cancelJob, warn) de incidente de sistema (→failJob, error+Sentry). Impede bloqueio de orçamento de gerar N×5 alertas críticos.
- `assertHarnessSchema`: recusa boot se faltar `job_queue, lead_checkpoints, agent_inbox_items, send_ledger` (migration 0050) — o boot NÃO aplica migrations.
- `createHealthzServer`: `/healthz` (contagem da fila por status, sessions, `event_log_drain: prontidaoDoLacoDeEventLog()` nos DOIS ramos 200 e 503 — #604/#648) e `/metrics`.
- `startWorker`: cria pool + `workerId=agent-engine-<hostname>-<pid>`; `seedPlatformPlaybook`; boot `reapExpiredJobs`; `carregarComportamentoPorPool` (comportamento da instalação ANTES dos laços). `reaperTimer` (releitura do comportamento + reap no ritmo `QUEUE_REAPER_INTERVAL_MS`), `holdsTimer` (`enforceHolds`). `loopsAbort = AbortController` compartilhado. Laços ligados condicionalmente por env: `runDrainLoop` (edge CRM), `runEventLogDrainLoop`, `runSessionWatchdogLoop` (só com WAHA), `runVoiceCallsBridgeLoop` (só com WACALLS), `runHealthLoop`, `runFlywheelLoop` (só se FLYWHEEL_INTERVAL_MS>0), `runCronLoop`, `workerLoop`.
- `runJob`: despacha por kind; `transactional_delivery`/`approved_reply` são donos do settle atômico (sem `completeJob` por fora); senão `withServiceJob` → `completeJob` + métricas. Erro: `AgendaDeferredError` → reschedule manual; terminal → warn + `cancelJob`; senão error + Sentry + `failJob`.
- **Shutdown gracioso**: SIGTERM/SIGINT → limpa timers, fecha server, `loopsAbort.abort()`, aguarda todos os laços; corrida entre drain em voo e `SHUTDOWN_GRACE_MS`; grace ganha → `process.exit(1)`.

### 12.4 `lib/agent-engine/cron/` — agendador `cron_jobs` 🟢
- **`schedule.ts`** (núcleo puro): `CronSpec` = `{kind:'at',at}|{kind:'every',intervalMs}|{kind:'cron',expr,tz}`. `parseCronExpr` (5 campos, `7`→`0` para dow; `*`, `/n`, faixas `a-b`, `a-b/n`, `a/n`, listas). `nextCronTime(expr, tz, afterMs)`: tz-aware via `Intl.DateTimeFormat({hourCycle:'h23', weekday:'short'})`, varredura minuto-a-minuto até `CRON_SCAN_HORIZON_MIN=366×24×60`; DST correto casando a representação de parede. `matches` implementa a regra Vixie de dom/dow (ambos restritos → OR, senão AND). **`staggerOffsetMs(leadId, windowMs)`**: FNV-1a 32-bit determinístico do contact_id mod windowMs (anti-thundering-herd, sem estado; `windowMs<=0` desliga). `computeNextRunAt`: 'every' colapsa runs perdidos (`while(next<=now) next+=interval`, sem stampede pós-downtime); 'cron' recalcula do agora (auto-corretivo); 'at' → null. `classifyFireError`: SQLSTATE classe 22/23 → 'permanent', senão 'transient'.
- **`scheduler.ts`** (camada DB): `CronJobRow` {kind, interval_ms(bigint como string), cron_expr, tz, job_kind:JobKind, next_run_at, enabled, attempts, max_attempts}. `scheduleCronJob` (defaults tz='UTC', job_kind='followup_turn', max_attempts=5, 1º next_run_at com stagger). `cancelPendingCronsForLead` (opt-out/handoff — nenhum follow-up dispara depois). `fireOneDue`: `select … where enabled and next_run_at<=now() for update skip locked` → `savepoint fire` → `enqueueJob` → reschedule (null → desabilita) → commit; erro de disparo → `rollback to savepoint fire` + `applyFailure` (permanente OU esgotou tentativas → desabilita + inbox 1×; senão backoff). `tickCron` dispara até `batchSize`, uma transação cada.

### 12.5 `lib/relogio/` — o "relógio" HTTP (rede de segurança para instalação sem contêiner scheduler) 🟢
- **`tarefas.ts`**: `TAREFAS_DO_RELOGIO` = event-log-drain, followup-flow-worker, routing-worker, recover-stuck-messages; `CAMINHO_DO_TICK='/api/v1/system/relogio/tick'`; `comandoCurlDoRelogio` (Bearer `$INTERNAL_SECRET`).
- **`executar.ts`**: `executarTickDoRelogio()` roda as 4 tarefas envolvidas por `uma(id, fn)` (try/catch → `ResultadoDeTarefa{id, ok, detalhe}`, falha `log.warn` não lança). `aplicarRespostasQueChegaram` fecha o buraco do "SIM que chegou antes do próximo next_eval_at" (lê `followup_enrollments` `waiting_reply`, última inbound entre gêmeos de telefone, aplica se `inboundEhDestaPergunta`). Depois `runFollowupTick`, `runSilenceSweep`, `enviarTextoFixoPendente`, `runRoutingWorker`, `recoverStuckMessages`. `clock: () => new Date()` injetado.

### 12.6 `lib/tempo/` + `lib/agenda/fuso.ts` — relógio e fuso 🟢
- **`fusos.ts`**: `FUSOS_OFERECIDOS` (lista IANA curada — América do Sul + Angola/Lisboa/UTC); `FUSO_PADRAO='America/Sao_Paulo'`; `fusoValido(tz)` pergunta ao `Intl` (não a uma lista); `fusoUtilizavel(...candidatos)` devolve o primeiro válido ou o padrão (falha ABERTA). Razão: `organizations.timezone` é `z.string()` sem refine nem CHECK, e `Intl.DateTimeFormat` LANÇA com fuso inválido (ex.: acento em `America/Asunción`) — degradar aberto em vez de tela branca.
- **`agora.ts`**: `renderAgora(agora, fuso)` monta o bloco `## Agora` do prompt (dia da semana da TABELA `DIAS_DA_SEMANA`, não do `Intl` que devolve inglês em ICU pequeno); `agora` é PARÂMETRO nunca `new Date()` interno (relógio injetável). `isoLocalComOffset(instante, fuso)` produz ISO com o OFFSET REAL daquele instante (conserta o bug UTC-vs-local: 15:45 local chegava ao modelo como 18:45+00 → agente dizia "loja fechada" no horário de atendimento). Todas falham abertas para `FUSO_PADRAO`, nunca lançam.
- **`lib/agenda/fuso.ts`** (a matemática de DST, dependência): `partesNoFuso` (formatador em cache por fuso, `hourCycle:'h23'`), `offsetEmMinutos` (offset NÃO é constante por fuso — DST), **`instanteDe(parede, fuso)`** — conversão parede→instante em duas passadas porque o offset depende do instante buscado. Bordas de DST explícitas: hora inexistente (spring-forward) devolve `Math.max` (resolução "compatible" — `Math.min` daria véspera em fuso de offset positivo como Beirute); hora ambígua (fall-back) devolve o primeiro.

### 12.7 `lib/realtime/` — Supabase Realtime 🟢
- `channels.ts` é o ÚNICO arquivo: `alertsPlatform()` → `createClient().channel("alerts-platform")` (canal broadcast de alertas cross-tenant, consumido por `useAlertsRealtime`). Helper de centralização de nome de canal. As assinaturas `postgres_changes` de inbox/kanban NÃO ficam aqui — moram em hooks de cliente (`hooks/`) e consomem as tabelas direto; só o canal broadcast é centralizado aqui.

### Knobs (`lib/agent-engine/env.ts`, Zod com defaults) 🟢
Fila: `QUEUE_MAX_CONCURRENCY=8`, `QUEUE_VISIBILITY_TIMEOUT_MS=600_000`, `QUEUE_POLL_INTERVAL_MS=2_000`, `QUEUE_REAPER_INTERVAL_MS=60_000`, `SHUTDOWN_GRACE_MS=30_000`, `HEALTH_PORT=8787`. Event-log: `EVENT_LOG_DRAIN_INTERVAL_MS=2_000`, `EVENT_LOG_DRAIN_IDLE_INTERVAL_MS=10_000`, `EVENT_LOG_DRAIN_BATCH_SIZE=50`. Cron: `CRON_TICK_INTERVAL_MS=30_000`, `CRON_RETRY_BASE_MS=30_000`, `CRON_BATCH_SIZE=100`. Constantes hardcoded do drain: `MAX_ATTEMPTS=5`, `PROCESSING_STALE_MS=600_000`.

### Escala de confiança e lacunas — Unidade 12
- 🟢 Todo o lado TypeScript foi lido diretamente (event-log, drain-loop, worker main, cron, relogio, tempo, realtime).
- 🔴 As tabelas `event_log`, `job_queue`, `cron_jobs`, `agent_inbox_items`, `followup_enrollments`, o trigger `trg_event_log_touch` e os corpos das RPCs ficam para o **Data Master**.
- 🟡 Os corpos individuais dos handlers de `workers/*.handler.ts` (só `media-persist` foi lido como amostra) e o `edge/crm/drain.ts` não foram lidos por inteiro; a ORDEM e as chaves vêm de `register-handlers.ts`.
- 🟡 O `voice-agent/` (ponte de áudio) é subsistema à parte, não parte de event_log/cron; fica para a análise de voz da Unidade 4.

---

## Unidade 13 — Infra transversal e relatórios: `lib/api/`, `lib/supabase/`, `lib/crypto/`, `lib/env.ts`, `lib/logger.ts`, `lib/i18n/`, `lib/schemas/`, `lib/query/`, `lib/http/`, `lib/net/`, `lib/reports/`, `lib/metrics/`

### Propósito 🟢
O substrato compartilhado sobre o qual toda rota, worker e tela se apoiam: formato canônico de resposta HTTP, catálogo de códigos de erro, os três clients Supabase, auth dual, idempotência, cripto AES-GCM, contrato de env validado por Zod, log estruturado, i18n e dois construtores de relatório/métrica.

### 13.1 Formato canônico de resposta — `lib/api/wrappers.ts` 🟢
Três funções, todas devolvendo `NextResponse`:
- **`ok<T>(data, opts)`** — sucesso `{ data }` ou `{ data, meta }` (`meta` só quando presente). Status default 200, restrito a `200|201|204`. **TODA resposta seta `X-Request-Id`** = `opts.requestId ?? randomUUID()` (contrato de correlação com o audit).
- **`fail(code, message, status, opts)`** — erro `{ error: { code, message, details? } }`; também seta `X-Request-Id`. **⚠️ Pegadinha de tipo**: `code: ApiErrorCode | (string & {})` — o segundo ramo aceita QUALQUER string, então `fail()` NÃO impõe o catálogo (é por isso que os blocos Agenda/Ads/Leads existem em errors.ts: inventar código no call site vira contrato de wire calado).
- **`noContent(requestId?)`** — 204 com `X-Request-Id`.
- Tipos: `ApiSuccess<T>={data, meta?}`, `ApiError={error:{code,message,details?}}`. `lib/api/types.ts` tem a classe `ApiError` (status, code, details, requestId, message) que é o objeto LANÇADO. `lib/api/recusa.ts`: `respostaDeRecusa(err, requestId)` — se `err instanceof ApiError` → `fail(...)`; senão RE-LANÇA (não-ApiError é defeito de programação e tem que chegar ao Sentry com stack).

### 13.2 Catálogo de erros — `lib/api/errors.ts` 🟢
`ApiErrorCodes` é `const` (chave===valor); `ApiErrorCode` = união dos valores. Agrupado por status HTTP nos comentários. Blocos: 400 (invalid_request, validation_failed, invalid_cursor); 401 (unauthenticated, token_expired, token_revoked, mfa_required, auth_in_query_forbidden…); 403 (forbidden, forbidden_role, forbidden_tenant, lgpd_anonymization_irreversible); 404 (not_found, pipeline_not_found); **Agenda** (8 códigos); 409 conflito (idempotency_conflict, idempotency_in_progress, state_conflict, contact_exists, duplicate_external_id, event_gone, next_action_changed, voice_already_paired…); 422 semântica; 415 (unsupported_media_type, logo_svg_recusado); 413 payload_too_large; 429 rate_limited; **Ads leitura 0214** (6); **External DB 0372** (4); **Voz spec 18** (4); **Negócios/funil #917/#922** (lead_stage_changed_concurrent, lost_reason_required/invalid, pipeline_immutable_use_clone…); **Aviso de caso 0292** (4 `aviso_*`); 500/upstream (internal_error, upstream_unavailable, unavailable, waha_error, wacalls_error, ai_provider_error, nuvemshop_error). **Regras**: código nunca é renomeado (versiona via `/api/v2/`); todo código não-genérico existe porque a TELA age diferente por código.

### 13.3 Os três clients Supabase — `lib/supabase/` 🟢
- **`server.ts`** — `createClient()` (async). Server Components/Route Handlers/Server Actions. Anon key + ponte de cookie para `next/headers`. Cookie: nome `sb-deskcomm-auth`, `sameSite:"strict"`, **`httpOnly:true`**, `secure:cookieSecure()`. **Manda sempre `getUser()`, nunca `getSession()`** (getUser revalida o JWT no backend; getSession confia no cookie local).
- **`browser.ts`** — `createClient()` singleton. `createBrowserClient` com anon key; URL/key de `window.__PUBLIC_ENV__` primeiro, `process.env.NEXT_PUBLIC_*` como fallback (uma imagem Docker serve qualquer projeto Supabase sem rebuild). Cookie `sameSite:"strict"` SEM httpOnly (lado browser). Máquina de auth do Realtime: como o cookie é httpOnly, o supabase-js não vê o token, então `realtime.accessToken` busca `/api/v1/auth/realtime-token`, cacheia com margem de 60s, coalesce fetches concorrentes, nunca memoiza falha.
- **`admin.ts`** — `createAdminClient()` singleton. Service-role → **BYPASSA RLS**. `autoRefreshToken:false, persistSession:false`. **Regra**: handlers que o usam DEVEM filtrar `organization_id` de fonte confiável (cookie/JWT/webhook secret/path token), NUNCA do body. Permitido: webhooks, cron/workers, onboarding/admin, health.
- **`cookie-secure.ts`** — `cookieSecure()` = `env.NEXT_PUBLIC_APP_URL.startsWith("https://")`. `Secure` derivado do PROTOCOLO da URL, não de `NODE_ENV` (self-host em HTTP puro com NODE_ENV=production descartaria o cookie — loop de login).

### 13.4 Auth dual — `lib/api/auth-dual.ts` 🟢
`resolveAuthDual(req, {requestId, resource, role, scope})` → `AuthDual` (sucesso: organizationId, actor, supabase, idioma?, `via:"session"|"token"`; falha: response). Bearer presente → `validateBearerToken` → `ensureScope` + `ensureRole` → `organizationId` DO TOKEN (nunca do cliente), `createAdminClient()`, `via:"token"`. Senão → `requireRole` (mesmo gate RBAC), `createClient()` (RLS), `via:"session"`. **Regra**: sozinho não basta — a rota Bearer TAMBÉM precisa de entrada em `lib/auth/public-paths.ts` senão o `proxy.ts` devolve 401 antes.

### 13.5 Idempotência — `lib/api/idempotency.ts` 🟢
`comIdempotencia<T>(entrada)`. `TTL_MS=24h`, `JANELA_DA_RESERVA_MS=60s`. Tabela `idempotency_keys`. `DesfechoIdempotente` = executou|replay|conflito|em_curso. Algoritmo (fecha a corrida, issue #778 migration 0321): (1) `hashDoCorpo` = SHA-256 de `JSON.stringify(ordenar(corpo))` (ordena chaves — ordem diferente não vira conflito falso). (2) lê linha viva (`expires_at > now`); achou → `classificar` (hash difere → conflito; `status_code===null` → em_curso; senão → replay). (3) sem linha → INSERT de RESERVA (`status_code:null`, expira em 60s); índice único decide o vencedor. (4) `23505` → alguém reservou primeiro; re-lê crua; vencida → reescreve como reserva nova com guard `.eq("expires_at", linha.expires_at)` (não rouba posse). (5) roda `executar()`; se LANÇA, LIBERA a reserva (vence agora) e propaga; se OK, grava recibo terminal (status + body + 24h) na MESMA linha filtrada por hash. `hashLido` normaliza `bytea` (string `\x…` do PostgREST ou Buffer do `pg`) para hex minúsculo; formato desconhecido → `null` (erra para conflito, nunca replay).

### 13.6 Cripto AES-GCM — `lib/crypto/aes_gcm.ts` 🟢
`KEY_LENGTH_BYTES=32`, `IV_LENGTH_BYTES=12`, `TAG_LENGTH_BYTES=16`. `getKey()` memoizado: lê `env.AI_CRED_AES_KEY`, lança se ausente/base64 malformado/comprimento≠32. `encryptKey(plaintext)` → `{ciphertext, iv, tag, last4}` — IV aleatório de 12 bytes, `aes-256-gcm`, tag de 16 bytes verificada, `last4 = plaintext.slice(-4)` (única parte exposta, via view `_safe`). `decryptKey({ciphertext,iv,tag})` reverte com `setAuthTag`. `bufToBytea` → `\x<hex>`; `byteaToBuffer` aceita Buffer/Uint8Array/`\xHEX`. **Regra**: plaintext NUNCA logado/persistido/devolvido.

### 13.7 Contrato de env — `lib/env.ts` 🟢
Zod validado no import (`schema.safeParse(process.env)`). Três tiers: `requiredAlways` (obrigatória em todo ambiente), `required` (obrigatória só em produção; em dev default ""), `diasDeRetencao` (nunca derruba o app; `0` NÃO desliga a poda). Leniência de build: se `NEXT_PHASE=phase-production-build` e o parse falha, semeia placeholders; no boot real, falha LANÇA. **requiredAlways**: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`. **required (só produção)**: `INTERNAL_SECRET`, `CPF_ENCRYPTION_KEY`, `WAHA_BYO_ENCRYPTION_KEY`, `AI_CRED_AES_KEY`, `SUPABASE_DB_URL`, `WAHA_API_BASE_URL`/`WAHA_API_KEY`/`WAHA_WEBHOOK_BASE_URL`, `UPSTASH_REDIS_REST_URL`/`UPSTASH_REDIS_REST_TOKEN`. **Regra**: knobs que uma pessoa digita (`AI_BUDGET_ENFORCEMENT`, `DISCLOSURE_MODE`, `SIGNUP_MODE`, `APP_ACCENT_HEX`, dias de retenção) usam `z.string()` e NUNCA `z.enum`/`z.coerce.number` — o schema lança no import (primeira request), o healthcheck é probe TCP, então enum deixaria o contêiner `healthy` com 100% das requests em 500. `CRON_SECRET` da Vercel, se presente, é copiado para `INTERNAL_CRON_SECRET`.

### 13.8 Logger — `lib/logger.ts` 🟢
Logger JSON estruturado sem dependência. `fmt(level, msg, ctx)` → uma linha JSON `{level, msg, ts, ...ctx}`. `info`/`warn`/`error`; `debug` só em development. **Regra**: nunca logue segredo, token, corpo de mensagem, CPF ou telefone.

### 13.9 i18n — `lib/i18n/` 🟢
- `registro.ts` — fonte única de todo idioma. `REGISTRO_DE_IDIOMAS`: pt-BR (completo), es (completo), zh-CN (em_construcao). `IdiomaVisivel` = só níveis não-construção (a visibilidade é o TIPO, não uma lista paralela).
- `idiomas.ts` — `Idioma`=`IdiomaVisivel["codigo"]`; `IDIOMA_PADRAO="pt-BR"`. `normalizarIdioma` (desconhecido → padrão). `parseAcceptLanguage` (ordena por q, primeira subtag servida). Puro (client component importa).
- `idiomaAnonimo.ts` — `idiomaDoVisitante` (cadeia server-only: pref salva → Accept-Language → pt-BR).
- `dicionario.ts` — `traduzir(texto, idioma)`: pt-BR devolve como está; senão `DICIONARIO[texto]?.[idioma] ?? texto`. **A CHAVE é o texto em português** — tradução ausente degrada para português legível, nunca chave crua.
- `datas.ts` — `localeDeData`/`tagDeIdioma` (date-fns aqui, não em registro, para client component não arrastar objetos).
- Cadeia de resolução (pref da pessoa → idioma da org → padrão) em `lib/auth/server.ts`, chega como `AuthUser.idioma`.

### 13.10 Relatórios e métricas 🟢
- **`lib/reports/atividades.ts`** — relatório de atividades ("o que aconteceu no período, e quem fez cada coisa — humano ou máquina?"). DTOs de `fn_activity_report` (0215). `origemDoAtor`: `user`→pessoas, `ai`→agentes, resto (system/rule/contact/null)→automatico (`contact` = cliente respondendo é "automatico" de propósito — a pergunta é o que o TIME fez). `rotuloDoDia("2026-09-03")`→"03/09" (sem `new Date`, string já no fuso do leitor). Vocabulário de `ACTIVITY_LABELS`.
- **`lib/metrics/atrito.ts`** — Índice de Atrito (spec 17). **Regra central**: toda medida de eficiência é publicada PAREADA com sua medida de dano (o tipo `Par` tem `eficiencia` + `danos[]` não-vazio, proibindo mostrar eficiência sozinha). `Medida`={chave, rotulo, valor:number|null, unidade}. Dado ausente é `null` NUNCA `0` — `razao(num,den)` devolve null se `den<=0` (tela mostra "—"; 0 leria como "0% de contorno" falso). `lerAbandonoHoras` lê jsonb defensivamente (`typeof` antes de `Number()` porque `Number(true)===1`); default 72h, limites 1..2160.

### 13.11 Outros transversais 🟢
- **`lib/api/client.ts`** — fetch do browser (`apiClient.{get,post,patch,...}`). Gera `X-Request-Id` por request, carimba `Idempotency-Key` em métodos mutantes. Timeouts: 10s leitura, 30s escrita. Retry: MAX_ATTEMPTS=3, só `{429,503}`, honra `Retry-After`. **Regra**: método mutante NUNCA é retentado em erro de rede/timeout (write timeout é "não sei", não "não aconteceu" — issue #783 triplicou cobrança de LLM). Em 403 `no_active_org`, um `reload()` guardado por `sessionStorage`.
- **`lib/query/client.ts`** — TanStack Query: staleTime 30s, gcTime 5min, retry só ApiError 429/503 (máx 2), mutação nunca retenta.
- **`lib/http/ip-do-cliente.ts`** — `ipDoCliente`: primeiro hop de `x-forwarded-for` ou `x-real-ip`, senão `null` (nunca sentinela). `ipDoClienteParaInet` rejeita o que `net.isIP()` recusa + `%`/`/` (Postgres `inet` daria 22P02 derrubando a captação). **Regra**: NÃO é prova de origem — nada autoriza/bloqueia por ele, só rate-limit e exibição.
- **`lib/net/alcance.ts`** — `FalhaDeAlcance` (endereco_nao_resolve|conexao_recusada|host_inalcancavel|tempo_esgotado|tls_recusado|indeterminada). `classificarFalhaDeAlcance` anda a cadeia `cause`/`errors` até profundidade 8 (undici embrulha erro de socket). Distingue "endereço não existe" (config) de "serviço fora" (servidor) — ações opostas.
- **`lib/schemas/_validate.ts`** — `validateRequest(schema, request)` (lança ApiError 400 `body_malformed`/422 `validation_error`). 24 schemas de entidade + `index.ts`.

### Escala de confiança e lacunas — Unidade 13
- 🟢 Todo o lado TypeScript foi lido diretamente.
- 🔴 As RPCs `fn_activity_report`, `fn_atrito_metrics`, a tabela `idempotency_keys`, a view `ai_provider_credentials_safe` e a policy `idempotency_tenant` ficam para o **Data Master**.
- 🟡 Os corpos individuais dos 24 schemas de entidade em `lib/schemas/` foram vistos pelo registro/validação, não campo a campo — o Redator deve reconferir campos ao escrever specs por entidade.

---

## Unidade 14 — Superfície HTTP: `app/api/**` (route handlers) e Server Actions `app/actions/`

### Propósito 🟢
A borda de entrada: os route handlers REST sob `app/api/` e as Server Actions sob `app/actions/`. O `proxy.ts` (middleware de borda do Next 16) autentica a sessão antes da rota; cada superfície não-cookie tem guard próprio.

### 14.1 Estrutura e contagem de rotas 🟢
`app/api/` tem exatamente três superfícies de topo: `internal/`, `mcp/`, `v1/`. Contagem (reconferir com `git ls-files 'app/api/**/route.ts' | wc -l`; medido nesta análise): ~339 route.ts em `app/api/**`, ~337 em `app/api/v1/**`, 1 em `app/api/internal/` (`agents/run`), 1 mcp (`app/api/mcp/route.ts`). `app/api/v1/` tem ~48 grupos: admin, ads, agenda, ai, anuncios, attendants, audit, auth, automation-rules, calls, channels, contacts, contact-tags, conversations, conversation-tags, cron, demandas, extensions, external-db, financeiro, health, integrations, leads, lead-captures, lgpd, marca, mcp, message-templates, messages, metrics, notifications, onboarding, phone-numbers, pipelines, plataformas-de-anuncio, products, prospecting, reports, settings, system, tags, tasks, team, tenants, voice, voip, webhook-sources, webhooks. Superfícies não-cookie: `cron/` (~30 rotas, Bearer `INTERNAL_CRON_SECRET`/`INTERNAL_SECRET`), `webhooks/` (~9, HMAC + path token), `internal/` (1, `x-internal-secret`), `mcp/` (1, Bearer `dsk_`), auth-dual (4 arquivos: `messages`, `conversations/[id]/media`, `conversations/open-with-contact`).

### 14.2 `proxy.ts` — middleware de borda 🟢
`proxy(request)` roda na Edge, em ordem:
1. **X-Request-Id**: lê `x-request-id` ou gera `crypto.randomUUID()`, seta na resposta (correlação com audit/wrappers).
2. **x-pathname**: seta na resposta E nos headers da request encaminhada (Server Component do onboarding lê).
3. **Detecção de superfície admin**: `host.startsWith("admin.") || pathname.startsWith("/admin")` (o ramo por host é NOOP documentado — self-host não mapeia subdomínio).
4. **Curto-circuito público**: `if (isPublicPath(pathname)) return response`.
5. **Validação de sessão server-side**: `createServerClient` (cookie `sb-deskcomm-auth`, `sameSite:"strict"`, `httpOnly`, `secure:cookieSecure()`) → `supabase.auth.getUser()`. Comentário explícito: NUNCA `getSession()` no backend.
6. **Não-autenticado**: rota `/api/` → JSON 401 `{error:{code:"unauthenticated"}}` com `x-request-id` (nunca redirect HTML para consumidor JSON); rota de UI → `redirect("/login?next=...")`.
7. **Cookie de impersonation** em `/app*`: `verifyImpersonateCookieEdge` (só HMAC + expiração na Edge, sem DB); falha → apaga o cookie de apresentação (a sessão de suporte no DB continua autoritativa).
8. **Gate platform_admin** em `/admin/*`: `rpc("fn_is_platform_admin")`; erro/false → `redirect("/admin/forbidden")` (early gate; a checagem autoritativa é `requirePlatformAdmin` no server).
9. **Matcher**: roda em tudo exceto assets estáticos/internos do Next.

### 14.3 Receita de route handler vs. os 6 passos 🟢
- **(a) CRUD cookie-authed — `app/api/v1/contact-tags/route.ts` (GET)**: `requestId = randomUUID()`; guard `requireRole("viewer", {requestId, resource:"contacts"})`; org de `authz.org.orgId`; query via `createClient()` (RLS) filtrada `.eq("organization_id", …)`; resposta `ok(tags, {requestId})`; erro → `logger.error` (causa crua no log) + `fail("internal_error", frase-de-produto, 500, {requestId})`. Sem passo 1 (GET sem input) nem 5 (só-leitura) — omitidos corretamente.
- **(b) Cron — `app/api/v1/cron/snooze-watcher/route.ts`**: GET e POST → `handle(req)`. Auth: Bearer, `accepted = [INTERNAL_CRON_SECRET, INTERNAL_SECRET].filter(Boolean)`, **fail-closed**. `createAdminClient()` (service role) → varre cross-tenant de propósito, audita `organizationId:null, bypassedRls:true`. Claim atômico (`.update(...).not("snooze_until","is",null)` é o lock — dois ticks não disparam duplo). Audit CONDICIONAL: só quando `reopened > 0` (varredura que não mudou nada não é mutação).
- **(c) Internal — `app/api/internal/agents/run/route.ts` (POST)**: `authorize(req)` aceita `x-internal-secret` OU `Authorization: Bearer <INTERNAL_SECRET>`, comparado com `timingSafeEq` constante; fail-closed. Zod `bodySchema.safeParse` → `fail("validation_failed", ..., 422, {details:{errors:...flatten()}})`; JSON malformado → `fail("invalid_request", 400)`. `runAgent()` em try/catch → `fail("internal_error", message, 500)` (nenhum throw cru escapa). `maxDuration=300`, `runtime="nodejs"`.
- **(d) MCP — `app/api/mcp/route.ts`**: GET/POST/DELETE → `handle`. `validateBearerToken` (`dsk_` contra `api_tokens`); `McpAuthError` → envelope **JSON-RPC 2.0** (`jsonRpcError(err.mcpCode, ...)`, códigos -32001/-32002/-32603), não o envelope REST. `createMcpServer(auth, requestId)` + transport por request; `X-Request-Id` ainda setado.
- **(e) Webhook (HMAC + path token) — `app/api/v1/webhooks/in/[token]/route.ts`**: org do PATH TOKEN, nunca do body (`webhook_sources` `.eq("path_token", token)`; 404 se token <8 chars ou fonte inativa). Rate-limit por token (`checkRateLimit` 60/min → 429 com `Retry-After`). HMAC via `verifyInboundSignature(rawBody, sig, secret)` com `crypto.timingSafeEqual`; assinatura inválida → audit + `fail("unauthenticated","invalid_signature",401)`; mas se o secret não decifra (`hmacSkipped`), PULA validação (disponibilidade > defesa opcional, precedente WAHA). Idempotência por `external_id` (`uniq_crm_leads_org_source_external`, fast-path + catch de 23505). `ok({lead_id}, {requestId})` ou 303 para form post.
- **(f) Cookie-OR-bearer — `resolveAuthDual`** (ver Unidade 13.4): org do TOKEN, admin client; ou sessão via `requireRole`, RLS client.

### 14.4 Server Actions — `app/actions/` 🟢
~51 arquivos `.ts` em admin/, auth/, integrations/, onboarding/, settings/, shell/, team/. Cada um `"use server"`. Padrão espelha a receita (Zod → guard → identidade de fonte confiável → mutação → audit) mas devolve OBJETO de resultado tipado (não `NextResponse`) e usa `revalidatePath`/`redirect`:
- **`settings/updateProfile.ts`**: Zod `profileSchema.safeParse` → `{ok:false, error:"validation_failed", details}`. Guard `loadAuthUser()` (usa `getUser()`) → `unauthenticated`. Lê `x-request-id`/`x-forwarded-for`/`user-agent` de `headers()`. Muta via `supabase.auth.updateUser`. `audit({action:"profile.updated", ...})`. `rpc("emit_event", ...)` best-effort (org-scoped, pula se sem org ou support-readonly). `revalidatePath`. Devolve `{ok:true} | {ok:false, error, details?}`.
- **`team/acceptInvite.ts`**: guard é TOKEN assinado (`verifyInviteToken` — assinatura + expiração), não papel. `getUser()` do cookie → `not_authenticated`. Bloqueia se em impersonation/suporte. EXIGE match de e-mail entre user do JWT e token → `email_mismatch`. Org/papel/convidador vêm SÓ do payload assinado; delega insert em `user_organizations` + audit `member.accepted` a `aplicarConvite`, depois `redirect("/app")`.

### 14.5 `lib/auth/public-paths.ts` — allowlist 🟢
`PUBLIC_PATHS: RegExp[]` + `isPublicPath(pathname)` = `.some(re => re.test(pathname))`. Precedência = ordem do array. `proxy.ts` consome para pular auth. Duas categorias: (1) genuinamente público (`/`, `/login`, `/signup`, `/auth/confirm`, páginas de erro, `/api/v1/health`, `_next/`, `/icon`, `/legal/(terms|privacy)`, `/email-templates/(confirmation|recovery)` [GoTrue], `/team/accept-invite/.+`); (2) **"o proxy não decide" — auth mora DENTRO da rota** (`/api/v1/webhooks/`, `/api/v1/cron/`, `/api/internal/`, `/api/mcp`, callbacks OAuth de agenda/plataformas-de-anuncio/nuvemshop — navegador volta de outro site onde o cookie `sameSite:strict` não viaja, identidade vem do `state` assinado; e as rotas auth-dual `/api/v1/contacts$`, `/api/v1/messages$`, `/api/v1/conversations/open-with-contact$`, `/api/v1/conversations/[^/]+/media$`). A maioria ancorada com `$` de propósito para sub-path futuro não nascer público de carona.

### 14.6 Convenções e superfície de erro 🟢
- **Envelope**: sucesso `{data, meta?}` (`meta` suporta cursor `{cursor, has_more, total}`); erro `{error:{code, message, details?}}`; TODA resposta seta `X-Request-Id`.
- **JSON snake_case** (`lead_id`, `has_more`, `run_id`); dinheiro em `_cents` + `currency`; datas ISO-8601 UTC; UUID v4.
- **Erro só via `fail(code, message, status)` na borda** — nenhum `throw` cru chega ao cliente: internal wrappa `runAgent` em try/catch; webhook captura `ApiError` e mapeia `fail(err.code, ...)`, re-lançando só o inesperado; erro de DB é logado com o requestId e o cliente recebe frase de produto. `console.log` banido (`no-console` warn); `logger` estruturado.
- Handler com service-role (`createAdminClient`) filtra `organization_id` manualmente — SEM gate automático, responsabilidade de quem escreve.

### Escala de confiança e lacunas — Unidade 14
- 🟢 `proxy.ts`, `public-paths.ts`, os wrappers e ~6 handlers representativos por superfície foram lidos diretamente, além de 2 Server Actions.
- 🟡 As ~337 rotas de `v1` NÃO foram lidas uma a uma (amostra representativa por superfície + contagens estruturais); o Redator/Arquiteto deve reconferir contagem com `git ls-files` antes de citar número, e mapear cada endpoint ao escrever a spec de superfície.
- 🔴 As tabelas/RLS que os handlers assumem (`webhook_sources`, `api_tokens`, `conversations`, `contacts`, `crm_leads`…) ficam para o **Data Master**; aqui só a borda que as invoca.
