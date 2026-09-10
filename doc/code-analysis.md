# Análise de Código — DeskcommCRM

> Gerado pelo **Arqueólogo** (Reversa) em 2026-09-10 · Nível: **Detalhado**
> Escala de confiança: 🟢 CONFIRMADO (lido no código) · 🟡 INFERIDO · 🔴 LACUNA
>
> Este documento consolida a análise técnica módulo a módulo. Regras de negócio,
> algoritmos e estruturas de dados são extraídos do código-fonte real (nunca inventados).
> O dicionário de dados completo está em `data-dictionary.md`; os fluxogramas em `flowcharts/`.

---

## Visão arquitetural em uma frase 🟢

DeskcommCRM é um monólito Next.js 16 (App Router) que processa mensagens de WhatsApp (via WAHA) para operar um CRM de vendas multi-tenant, com **dois runtimes de IA convivendo** (o `agent-engine` rico sobre fila Postgres, e workers legados sobre `event_log`), guardrails determinísticos de saída, e um barramento de eventos (`event_log`) que desacopla ingestão, automação, follow-up, LGPD e conversões.

**Dois planos de execução (crítico para entender o resto):**
- **Plano web** (rotas Next, Server Actions): usa `supabase-js`. Identidade do JWT, org de fonte confiável.
- **Plano worker** (`workers/agent-worker/main.ts`): usa `pg.Pool` puro. Fila durável `job_queue`, loops de drain/cron/watchdog/health/flywheel.
- A dualidade de clients (`createSupabase*Client` vs `createPg*Client`) é deliberada — testabilidade contra Postgres efêmero.

---

## Grupo 1 — Núcleo de IA e Agentes 🟢

### 1.1 `workers/agent-worker/main.ts` — processo 24/7

Processo long-running que hospeda a fila e os loops de background.

- `startWorker(env, handlers, log?)`: boot ritual → `assertHarnessSchema` (recusa subir sem tabelas `job_queue`, `lead_checkpoints`, `agent_inbox_items`, `send_ledger` — migration 0050) → `seedPlatformPlaybook` → `reapExpiredJobs` → healthz → loops → graceful shutdown (SIGTERM/SIGINT drena até `SHUTDOWN_GRACE_MS`).
- Handlers por `JobKind`: `approved_reply`, `transactional_delivery`, `inbound_turn`, `followup_turn`, `case_reply_turn`, `operator_turn`.
- `createHealthzServer`: `/healthz` (contagem de `job_queue` por status + saúde de sessões) e `/metrics`.
- **Distinção de erro crítica** (`ehVetoPermanenteDeNegocio`): erro com `terminal === true` (hoje só `LlmBudgetExceededError`) → `cancelJob` (não reagenda, não vira alerta crítico); falha real → `failJob` + Sentry. Sem isso, um bloqueio de orçamento gerava N×5 alertas críticos afogando o `budget_exceeded`.
- **Loops:** `runDrainLoop` (event_log→jobs), `runEventLogDrainLoop`, `runSessionWatchdogLoop` (WAHA×`channel_sessions`), `runHealthLoop`, `runFlywheelLoop` (OFF se `FLYWHEEL_INTERVAL_MS=0`), `runCronLoop`, `rodarLoopDaFila`.

### 1.2 `lib/agent-engine/agent/inbound-turn.ts` — loop do agente rico (F2-09)

Handler do job `inbound_turn`. Cada job é uma sessão FRESCA do motor LLM; todo estado do run vive no closure (isolamento entre leads por construção).

**Ritual imposto pelo runtime** (não pelo modelo):
1. Lê playbook (system, por ponteiro) + `lead_checkpoints` (compromissos/objeções/next_action + rolling summary) + `lead_state` (estágio) + últimas N mensagens (`get_lead_context`).
2. O modelo decide tools. **Enviar é SEMPRE tool call** (`send_message`); texto direto do modelo NUNCA vira mensagem (é descartado).
3. Fecha com 2ª chamada `purpose='checkpoint'` que devolve JSON validado por Zod.

**Superfície de tools** (`AGENT_TOOL_DEFS`): `get_lead_context`, `send_message`, `update_lead_state` (máquina de estados `new→contacted→qualifying→qualified→negotiating→won|lost`, BANT), `schedule_followup`, `save_lead_note`/`get_lead_note` (memória durável com `supersedes`), `search_knowledge` (RAG), `request_human_handoff`, `read_skill_reference`, `open_human_case`/`provide_case_update`, `send_template` (Meta, fora da janela 24h). Validação real é whitelist `.strict()` — campo forjado vira erro de ensino ao modelo, nunca exceção do SDK.

**Escolta de orçamento** (`comHandoffSeOrcamentoAcabar`): envolve o turno INTEIRO — qualquer chamada de modelo (inclusive indiretas: `classifyStage`, `maybeCompact`) que estoure orçamento devolve a conversa à fila humana antes de relançar.

Constantes: `MAX_VETOS_DE_VOCABULARIO_INTERNO=2`, `MAX_VETOS_DE_FALSO_VAZIO=2`, `DEFAULT_MAX_SENDS_PER_TURN=3`.

### 1.3 `lib/agent-engine/guardrails/before-send.ts` — cadeia de guardrails de saída (F2-13)

Seam determinístico entre a decisão do modelo (`send_message`) e o canal. Estilo "exit-2 do Claude Code": cada gate pode VETAR e a razão volta ao modelo como erro instrutivo. Roda sob `pg_advisory_xact_lock(hashtext(channel_session_id))` (serialização por número, INBOX-008).

**Ordem versionada** (`BEFORE_SEND_GATES` / `BEFORE_SEND_CHAIN_VERSION` v6, vigiada por `tests/unit/before-send-chain-shape.test.ts`):

| # | Gate | O que veta |
|---|------|------------|
| 1 | `stopGate` | opt-out / `is_blocked` / `force_human` — irrevogável (regra dura nº 2) |
| 2 | `lgpdGate` | anonimização (veta tudo); 1º toque de prospecção sem base legal |
| 3 | `pacingGate` | anti-ban (janela/warm-up/cap); throttle vira `waitMs`, não veto |
| 3.5 | `messagingWindowGate` | janela de 24h fechada em canal com hetero-restrição |
| 4 | `spinningGate` | template idêntico em massa (Jaccard ≥ 0.8) |
| 5 | `promiseGate` | preço/desconto/parcela fora da tabela versionada (determinístico) |
| 6 | `semanticPromiseGate` | promessa em texto livre ("faço de graça") — camada semântica |
| 6.5 | `casePromiseGate` | promessa-de-humano sem caso aberto (invariante sagrada) |
| 6.7 | `internalVocabularyGate` | vazamento de vocabulário interno ao cliente |
| 6.9 | `agendaStallGate` | "vou verificar horário"/"está confirmado" sem chamar a tool de agenda |
| 7/8 | `disclosureGate` | 1ª mensagem sem se apresentar como assistente virtual (pode emendar corpo) |

**Defaults assimétricos deliberados:** `internalVocabularyEnforced` ausente = DESARMADO (no caminho determinístico sem modelo, veto seria drop silencioso); `spinningEnforced` ausente = ARMADO (protege o número de ban); `messagingWindow` ausente = janela FECHADA (veto visível, não envio errado).

`GateVerdict` = `{pass:true, waitMs?, amendBody?, skipped?:'not_applicable'}` | `{pass:false, code, reason, nextAllowedAt?, detail?}`. `detail` leva números/rótulos ao trace, NUNCA o corpo (PII fora de log).

### 1.4 `lib/agent-engine/pacing/` — motor anti-ban (F2-11)

Decisão PURA (sem I/O): dado relógio, knobs, estado de envios e limite do CRM → `{allow, waitMs}` ou veto.
- Ordem: janela horária (tz do tenant, domingo por `allowSunday`) → caps diários (warm-up por idade + `crmDailyLimit`) → throttle+jitter.
- `banRisk=false` desarma SÓ o anti-ban; janela horária é cortesia e vale em todo canal (invariante 3 da doutrina de restrição de canal).
- `warmupCapFor` falha FECHADO (idade aquém do 1º degrau usa o mais conservador). Exportada para a UI usar a mesma regra.
- `PACING_DEFAULTS`: throttle 1200ms, jitter ≤800ms, janela 7h–22h, domingo ligado, tz `America/Sao_Paulo`, warm-up `[{0,20},{4,50},{8,100},{15,200},{31,null}]`. Fonte única (lint `scripts/lint-pacing.ts` proíbe literais fora daqui).

### 1.5 `lib/agent-engine/spinning/` — anti-template-idêntico (F2-12)

Veta copy repetida em massa (gatilho de ban confirmado). Duas frentes sobre copy normalizada: igualdade exata (sha256) + quase-idêntica via **Jaccard de tokens** ≥ `similarityThreshold`. Veta se `matchCount >= repetitionThreshold`. Allowlist (curtas utilitárias + regex de link de pagamento) isenta; regex inválido falha FECHADO. `SPINNING_DEFAULTS`: windowSize 20, similaridade 0.8, repetição 2 (a 3ª é vetada).

### 1.6 `lib/agent-engine/flywheel/live.ts` — judge + distiller (Fase 2C/4B)

Avalia turnos REAIS com LLM-juiz sobre a dimensão `memory_hygiene` e propõe deltas de playbook. **Gate humano inegociável**: propostas só viram comportamento quando o dono publica na tela.
- `runFlywheelOnce`: coleta turnos `inbound_turn` done → julga (veredito `yes`/`no`/`unknown`, ordem de opções alternada por hash do job_id = anti-viés) → grava `flywheel_judge_verdicts` (dedup unique). Se `no`: distiller → `flywheel_distiller_proposals` (`playbook_bullet` ou `org_memory_entry`). Agrega `aggregateFollowupOutcomes`.
- Modelo NÃO é fixado (resolve pela cadeia do painel de provedores — id literal dava 400 em OpenRouter).

### 1.7 Guardrails determinísticos (regex PT-BR, sem LLM)

- `promise/engine.ts`: `extractPromises` (preço R$/reais, desconto %, parcelas Nx com contexto de pagamento) → `decidePromise` veta contra tabela versionada da org. Conservador (só valor estruturado).
- `human-promise.ts`: `detectHumanPromise` — 8 padrões sobre `TARGET_WORDS` (equipe, time, setor, responsável, atendente...). `extraHumanNames` estende com nomes próprios do prompt do tenant (issue YADEA "Fernando"). Evita 2 armadilhas ("verificar no sistema", "equipe à disposição").
- `jailbreak/classifier.ts`: classificador ADVISÓRIO (não veta sozinho). Correlacionado a promessa fora de tabela → `escalateJailbreakPromise` (item em `agent_inbox_items`, dedup por episódio). `reason` NUNCA logado (pode ecoar PII).

### 1.8 Workers legados de IA (pré-engine, sobre `event_log`)

Atendem organizações **sem** versão de agente publicada no engine.
- `ai-response-worker.ts`: `processMessageReceived` — pipeline buildContext → checkGuards (IA-01..IA-08) → invokeBot (`claude-sonnet-4-6`) → postProcess → persistAndDispatch. Triagem síncrona sem LLM primeiro (G1 pede humano, G4 legal, G4 stage), depois veto de orçamento e guard de modelo. `RAG_THRESHOLD=0.4` (calibrado; 0.72 descartava paráfrases). G3 (baixa confiança) persiste rascunho mas não despacha + handoff.
- `ai-sentiment-worker.ts`: `processSentiment` — `claude-haiku-4-5` via `generateObject` com schema Zod (`sentiment_score` 0..1, `reasoning_short` descartado). Resolve QUAL agente atende a conversa (issue #486). Emite `ai.sentiment_alert`. Catch global nunca lança.
- `ai-handoff-from-sentiment.handler.ts`: consome `ai.sentiment_alert` → `triggerHandoff(reason:"low_sentiment")` (gate G2).
- `rag-indexer.ts`: `processRagIndexer` — indexa FAQ/documentos/catálogo em `ai_chunks` versionados; nunca ativa versão vazia; falta de chave → `retry` (não `skipped`) + item na Central.

### 1.9 `lib/mcp/` — servidor MCP interno (Spec 11)

Expõe operações do CRM como tools MCP para clientes externos autenticados (distinto do agente in-process). Auth Bearer `dsk_<prefix>_<secret>` → SHA256 contra `api_tokens.token_hash`. Cada handler: higieniza uuids de aterro → `ensureScope`/`ensureRole` → executa → audita → retorna `content[]`. Atributos (role, actor, run) codificados em `scopes` sem migration.

### 1.10 `lib/ai/handoff/` — handoff bot→humano (EPIC-06)

`triggerHandoff` (nunca lança) executa a transição dos 4 gates. `HandoffReason`: `requested_human`, `low_sentiment`, `low_confidence`, `critical_stage`, `legal_mention`, `refund_mention`, `orcamento_de_ia`. Efeitos: idempotência 5s → gate de elegibilidade → **avisa o lead** → `conversations.status='pending'` + `bot_silenced_until='infinity'` → activity → move card para etapa `chamar-humano` (opt-in) → `emit_event` → broadcast realtime → audit → item `agent_inbox_items` (dedup por episódio). Predicados puros: `checkG1`/`checkG4Legal`/`checkG3`/`checkG4Stage` (regex em `regex.ts`).

---

## Grupo 2 — Canais e Mensageria 🟢

### 2.1 `lib/waha/ingest.ts` — ingestão de webhook WAHA

Fonte única para parse de identidade WhatsApp, resolução de contato/conversa e persistência. Compartilhado por `/waha` (global) e `/waha/[token]` (per-tenant).
- `parseChatId`: `@c.us`/`@s.whatsapp.net`→phone E.164, `@lid`→lid, `@g.us`→group (skip), resto→unknown (emite `whatsapp.chat_id_not_recognized`).
- Resolução ATÔMICA via RPC (`fn_upsert_wa_contact`/`fn_upsert_wa_conversation`) porque NOWEB emite `message` E `message.any` (corrida). Idempotência por unique `(organization_id, external_id)`.
- `ehEcoDeEnvioNosso` (`JANELA_DO_ECO_MS=60_000`): distingue eco de envio próprio de digitação no celular (dúvida = silencia).
- `semSufixoDeChat`/`telefoneAlternativoDe`: corte manual anti-ReDoS (CodeQL js/polynomial-redos) — não regex, porque `from` vem de webhook sem assinatura obrigatória.
- `verifyHmacSha512`: `timingSafeEqual`, fail-closed.
- Mapas `WA_TYPE_MAP` e `NOWEB_MESSAGE_KEY_TYPE` traduzem tipo cru → `messages_type_check`.

### 2.2 `lib/waha/client.ts` / `send.ts`

Cliente REST do WAHA com teto de relógio (`TETO_PADRAO_MS=15s`, `TETO_DE_MIDIA_MS=30s`). `CONVERSAS_IGNORADAS` (status/broadcast/channels/groups) cortadas na FONTE (economia medida de ~376MB). Erro expõe só o status HTTP, nunca o corpo (PII). `resolveWahaChatId`: **`lid:` vem ANTES do telefone** (trocar o canal de conversa @lid viva quebraria o envio).

### 2.3 `lib/channels/pos-entrada.ts` — efeitos pós-ingestão

`aplicarEfeitosPosEntrada` roda 3 efeitos **em ordem que é regra de negócio**: (1) opt-out grava `is_blocked` ANTES do lead; (2) nascimento do lead relê contato e recusa bloqueado; (3) despacho do agente (`ai_agent.dispatch_requested`). Cada passo falha "para dentro" (log), nunca derruba a ingestão. Vigiado por `tests/unit/pos-entrada-*.test.ts`.

### 2.4 `lib/channels/janela.ts` / `lib/atendimento/fronteira.ts`

- `estadoDaJanela`: estado da janela de 24h do lado de quem atende (derivado a cada leitura, sem coluna de expiração). `sem_restricao | aberta | fechada`.
- `fronteira.ts`: trava otimista de atendimento (`ServiceBoundary`). `service_revision` só incrementa ao TROCAR de demanda (migration 0222); `StaleServiceBoundaryError` quando o atendimento capturado na origem não é mais o mesmo.

### 2.5 `lib/agent-engine/channel-adapter.ts` — contrato agnóstico (F2-25)

Interface `ChannelAdapter` (v1: WAHA-via-CRM). `ChannelSendResult`: `sent`/`already_sent`/`queued`/`blocked`/`failed`/`unavailable`. Idempotência = `(jobId, seq)`. `ChannelCapabilities` (freeformAnytime, serviceWindowHours) e `ChannelCost` (per-message vs flat).

---

## Grupo 3 — CRM e Vendas 🟢

### 3.1 `lib/leads/nascimento-do-lead.ts` — conversa vira lead

`garantirLeadDaConversa`: **um lead por DEMANDA, não por mensagem**. 1) recusa bloqueado; 2) recusa se já há lead `open`; 3) `funilDeEntrada` = `crm_pipelines.is_default` + etapa de menor `position` não-terminal; 4) insere `crm_leads` (título via `rotuloDoContato`, source `whatsapp`/rótulo de anúncio); 5) `emitLeadActivity` `lead_created`. `MotivoSemLead`: `ja_existe | contato_bloqueado | sem_funil_de_entrada | sem_etapa | erro`.

### 3.2 `lib/leads/risk-radar.ts` + `radar-de-risco.ts` — radar de risco

Máquina de classificação PURA (`classifyRisk`): buckets `critico | em_risco | em_voo | em_dia`. Janela de esfriamento vem do estágio (`expected_duration_hours`), não de constante global; `criticalHours = coldHours × 3`. `RISK_COLD_HOURS=24`, `RISK_CRITICAL_HOURS=72`. `radar-de-risco.ts` monta a lista (leads frios + donos + follow-ups em voo), reaproveitando `classifyRisk` para paridade humano↔IA. `SCAN_CAP=500`.

### 3.3 `lib/leads/score-formula.ts` + `lib/kanban/score-band.ts` — score

`calculaScore`: FÓRMULA, não LLM (razão derivada de cada parcela, lastro citável obrigatório). BASE 30, +12/compromisso (teto 3), −8/objeção (teto 3), +5/campo BANT (teto 4), ajuste de recência por bucket. Recusa se status≠open, <2 sinais substantivos, ou sem checkpoint. `score-band.ts`: converte score em faixa (`frio/morno/quente`) com **histerese** (banda morta 5pt, degrau a degrau) para o card não piscar. Limiares 70/40.

### 3.4 `lib/kanban/card-state.ts` + `fractional-indexing.ts`

`resolveCardState`: precedência estrita da "faixa ③" do card (`awaiting > reactivation > cooling > meter > idle`) — um estado por vez. `midpoint` (STEP=1000, NaN dispara rebalance) para posicionamento fracionário.

### 3.5 `lib/leads/stage-operations.ts` + `lib/pipelines/pipeline-editing.ts`

Operações de etapa/funil (regras puras + I/O sequencial). Validar ANTES de tocar o banco; marcação win/lost pode exigir 2 updates em sequência (índices únicos parciais imediatos); arquivar move negócios antes (ON DELETE RESTRICT); `conflitoDoBanco` traduz 23505/23514. Funil: recusa arquivar único/padrão/com webhook/com automação; excluir herda tudo + recusa com negócios.

### 3.6 `lib/leads/agent-stage-sync.ts` + `encerramento.ts` + `classificacao-inicial.ts`

- `sincronizaEstagioDoAgente`: agente move o card via `crm_stages.agent_stage_hint` (7 passos fixos → estágio nomeado do tenant). Trava otimista `.eq(stage_id).select()`. Resultado tipado distingue `movido | sem_mapeamento | ja_esta_la | sem_negocio | ambiguo | conflito_humano | fora_do_escopo | falha_de_escrita | indisponivel`.
- `encerraDemanda`: ganhar/perder movendo para estágio terminal (status/closed_at vêm do trigger). Idempotente. `lost` exige motivo.
- `classificarLeadInicial` (PURA): `desqualificado` (contato inválido/sem consentimento) | `revisao_humana` (conflito de identidade/spam/incoerência) | `classificado` (A/B/C/D/nao_avaliado). D vem do critério combinado do score ou frase exata, nunca de corte de R$ isolado.

### 3.7 `lib/conversoes/envio.handler.ts` — reportar venda

Handler de `event_log` (desacoplado do fechamento — invariante 1). Escuta `lead.won` E `lead.stage_changed` (payload é dica, banco é verdade — re-lê). Só reporta com atribuição de anúncio. Evento `Purchase`, dedup `<leadId>:Purchase`. Transporte por plataforma (google_ads sem transporte → skip).

---

## Grupo 4 — Automação e Follow-up 🟢

### 4.1 `lib/automation/engine.ts` — motor de regras

`runAutomationForEvent`: anti-loop (metadata `caused_by_rule` / `request_id` prefixo `rule:`) → guard de `entity_kind` → carrega `automation_rules` ativas por `trigger_event` → `buildContext` → `evaluateConditions` (ops `eq/neq/contains` em AND; campo ausente + `neq` = true) → pré-checagem de postpone (all-or-nothing) → executa ações. Agregação de desfecho HONESTA: `skipped`+`failed` contam juntos, falha vence adiamento (`success/partial/failed/adiado`). Guarda de contato (`checarGuardasDeContato`): `no_contact → contact_blocked → no_phone → consent_declined` (por `declined_at`, não ausência de `granted_at`).

### 4.2 `lib/followup/` — máquina de follow-up por grafo

Enrollment percorre nós de um fluxo publicado.
- `enrollFollowupFlow`: 1 por lead ativo (conflito 23505 → 409). `gatilho-etapa.ts`: enrollment por mudança de etapa.
- `runFollowupTick`: `claimDueEnrollments` (lease 120s) → processa isolado (falha de um não derruba o tick). `MAX_STEPS=80`.
- `processNode` (PURO): tipos de nó `trigger | wait (fixed/smart) | condition | ai_classify | match_reply | repeat | action | end`. Planejamento adaptativo de tempo no trigger; `wait` cortado por resposta (`wokeEarly`); `action` at-most-once send com dead-man `MAX_ACTION_RECHECKS=14` (era 5 — matava follow-up da noite pela janela anti-ban).
- `completeTurnForEnrollment`: **lista POSITIVA** de estados que avançam (só `active`/`waiting_reply`) — turno stale não sobrescreve intervenção humana.
- `reactivity.ts`: reage a `message.received` (inbound corta espera; opt-out cancela tudo `opted_out`), `ai.handoff_triggered` (policy `allow`/`cancel`/`pause`→`paused_handoff`), `ai.handoff_resolved` (retoma).
- `EnrollmentStatus`: `active | waiting_reply | paused_handoff | paused_manual | completed | cancelled | dead`. `EnrollmentOutcome`: `converted | replied | exhausted | opted_out | handoff`. `outcome-stats`: terminal = completed+cancelled (dead excluído); `conversion_rate = converted/terminal`.

### 4.3 `lib/escalacao/` + `lib/agenda/protecao-followup.ts`

- `quemPodeAssumirAgora` / `expectativaDeAtendimento`: quem pode assumir agora (roster agent+ × disponibilidade × carga); frase instrutiva ao modelo (3 casos, incl. instalação fresca sem ninguém).
- `pausarIaPorAtendimentoManual`: `bot_silenced_until = agora + 60min` quando humano responde por fora do CRM. Renova a cada fala; NUNCA encurta silêncio maior (`'infinity'` do handoff formal vence). Não toca `ai_authorized_at`/`force_human`/`assignee_kind`/`status`.
- `protecaoDaAgenda`: protege lead com compromisso agendado da cobrança do follow-up. `sem_compromisso | agendado | em_atendimento | presenca_pendente | presenca_vencida | leitura_indisponivel`.

---

## Grupo 5 — Auth, Multi-tenancy e RBAC 🟢

### 5.1 Modelo de papéis (`lib/auth/types.ts`)

`Role`: `viewer(1) | agent(2) | ai_operator(3) | manager(4) | admin(5)`. `ai_operator` é o papel do AGENTE PUBLICADO — só em token efêmero, nunca em `user_organizations` (RLS intacta). `PAPEIS_HUMANOS` exclui `ai_operator`.

### 5.2 Gate canônico (`lib/auth/require-role.ts`)

`requireRole(min, opts)`: `loadAuthUser` (JWT) → support ativo? → resolve org → platform admin bypass? → **role efetivo do BANCO** (`fn_user_role_in_org`, a mesma função das RLS) → **gate de MFA de sessão** (`mfaEmDivida`: papel exige + fator cadastrado + sessão aal1) → rank check. Anti-padrão proibido: comparar `ROLE_RANK` direto em rota.

### 5.3 Multi-tenancy (`lib/auth/server.ts`, `lib/supabase/`)

`organization_id` sempre de fonte confiável (cookie validado contra memberships / JWT / path / state HMAC), NUNCA do body. `admin.ts` bypassa RLS (89/169 handlers o usam → filtro manual obrigatório). `loadAuthUser` **falha ALTO** (lança em erro de permissão em vez de degradar para "sem organização"). Ordenação `accepted_at` decide org ativa default sem cookie.

### 5.4 Borda e rate limit

`proxy.ts`: valida JWT (`getUser`, nunca `getSession`), injeta `X-Request-Id`, redireciona/401-JSON não autenticados, gate antecipado de `/admin` (`fn_is_platform_admin`), impersonation edge (HMAC). `public-paths.ts`: paths que bypassam auth de cookie (auth mora dentro da rota). `rate-limit.ts`: por IP + por conta (`contaBloqueadaPorFalhas` — só conta senha errada). Sem IP identificável, limite por IP não entra (self-host sem proxy).

### 5.5 Tokens (`invite-token.ts`, `impersonate/`)

Convite: HMAC-SHA256 stateless (`INVITE_TOKEN_SECRET → INTERNAL_SECRET → "dev-fallback"`, TTL 24h). Impersonation: envelope HMAC aditivo à sessão (não troca o usuário Supabase; TTL 1h; secret ≥32 chars). Edge usa Web Crypto (middleware não tem `node:crypto`). Support session é autoritativa no banco.

### 5.6 Superfície de API (`lib/api/`)

`ok()`/`fail()` canônicos: `{data, meta?}` / `{error:{code,message,details?}}` + `X-Request-Id`. `ApiErrorCodes` catálogo por status HTTP. Padrão de handler: Zod → `requireRole` → query com `organization_id` → `audit` → `ok()`/`fail()`. `Actor` discrimina `user`/`ai_agent`/`webhook_source` (`id` ≠ `agent_id`, este é o que vai para FK).

---

## Grupo 6 — LGPD, Legal, Auditoria, Event-Log e Branding 🟢

### 6.1 `lib/event-log/` — barramento de eventos

`dispatchEvent` roteia por `consumed_by` (idempotência de retry) e `event_type`. `drainEventLog` (cron + laço no worker): claim otimista (`processing where pending`), reaper de órfãos (>10min), desfechos por precedência: **retry** (não incrementa attempts) > **error** (attempts++, dead em 5) > **sucesso** (`consumed_by += ok+skipped`). 15 handlers registrados em ordem deliberada (`followupReactivity` antes do LLM, `conversaoDeVenda` por último). `EventRow`/`EventHandler`/`HandlerResult` (status `ok|skipped|error|retry`).

### 6.2 `lib/audit/` — auditoria append-only

`audit(entry)`: fire-and-forget (nunca bloqueia a mutação), mas barulhento (console.error + Sentry). Escolhe admin client (service role) quando configurado. Enriquece com support session. `AUDIT_ACTIONS` array `as const` é fonte única (painel deriva dele). `hashEmail` para correlacionar sem PII.

### 6.3 `lib/lgpd/` — direitos do titular

`LgpdRequestType`: `data_request | redact | store_redact`. `LgpdScope`: `contact | tenant`. SLA em dias úteis BR (`computeDueAt`). Export worker: coleta multi-tabela PII-safe → PDF (PAdES stub sem chave) + JSON → Storage → signed URL → email (com marca da org) → `completed` + eventos. Redact worker: cascata por contato (`cascadeRedactContact` → RPC `fn_lgpd_cascade_redact_contact`, **enfileira avatar em `storage_redaction_queue` ANTES de zerar ponteiro**) ou por tenant (lotes de 100, `organizations.status='redacted'`). **PDF nomeia o CONTROLADOR (`organizations.legal_name`) e o DPO, nunca a marca** — decisão jurídica (revendedor é operador, não controlador).

### 6.4 `lib/legal/operador.ts`

`resolverOperador`: quem é o responsável legal desta instalação (usa client de SESSÃO, nunca service role — `/legal/*` é pública). `urlDePoliticaSegura` bloqueia `javascript:` na saída. Fallback `SEM_SESSAO` (nunca erro).

### 6.5 `lib/branding/` + `lib/branding.ts` — marca white-label

`resolverMarca`: camadas organização → instalação (banco) → `.env` → padrão. **Nunca lança** (roda em `app/layout.tsx`; throw = 500 em todas as telas). Precedência por campo (ausente numa camada não apaga a de baixo). `marcaDaSaida`: para saídas sem DOM (email, MFA issuer) — sempre tema claro, accent + contraste, degrada para padrão do produto (email de LGPD tem SLA legal). Fonte é o BANCO (`platform_branding`, `organizations.settings.branding`); `.env` é semente e piso de rollback. Envelope de cor grava só `{semente_hex, papel_da_semente}`, nunca os stops derivados (não congela a instalação).

---

## Grupo 7 — Integrações Externas 🟢

- **Nuvemshop** (`lib/nuvemshop/oauth.ts`): OAuth (tokens não expiram, `user_id` = `storeId`), webhook HMAC-SHA256 (`x-linkedstore-hmac-sha256`, `timingSafeEqual`, fail-closed).
- **Plataformas de anúncio** (`lib/plataformas-de-anuncio/`): registry (`meta_ads` → transporte; `google_ads` → `null` declarado, sem extrator de gclid). Meta `conversions.ts`: único fora de `lib/channels/` que escreve endpoint. `Purchase` exige value+currency, evento >7d recusado, identidade via `ctwa_clid` + telefone hasheado, `action_source: business_messaging`. Erro classificado transitório (5xx, throttle 613) vs permanente (token, evento velho).
- **Webhooks** (`lib/webhooks/captacao.ts`): registro durável de captação (nunca lança; org da fonte, nunca do body). Teto 60 campos × 2000 chars.
- **Opt-out** (`lib/opt-out/deteccao.ts`): `ehPedidoDeOptOut` (inequívoco, autoriza `is_blocked`) vs `ehOptOutProvavel` (ambíguo, só escala). PT + ES.
- **Notifications** (`lib/notifications/`): 🟡 pipeline emit → policy/prefs → deliver → web push VAPID (não lido em profundidade).

---

## Grupo 8 — Onboarding e Instalação 🟢

- `lib/onboarding/passos.ts`: wizard com fonte única (`PASSOS`) para roteador/indicador/resumo. Passo só existe se `existe(ctx)` (loja só se `lojaLigada`). Ordem: welcome → connect-whatsapp → connect-nuvemshop → setup-ai → funil → testar → invite-team.
- `lib/instalacao/ambiente.ts`: `lerAmbiente` detecta o que o instalador trouxe (chaves de provedor, gateway, email, transporte de WhatsApp). `NOME_PLACEHOLDER_DA_INSTALACAO = "Minha Empresa"`.

---

## Padrões transversais observados 🟢

1. **Fonte única de verdade repetida:** número de pacing, faixa de score, janela de risco, régua de orçamento — todos com uma implementação e a UI/lint proibindo cópias.
2. **Fail-closed em segurança, fail-open em telemetria:** guards de auth/envio negam na dúvida; logs/audit/activity nunca derrubam a operação primária.
3. **Idempotência onipresente:** `consumed_by` no event_log, `(jobId,seq)` no envio, `idempotency_key` no follow-up, dedup por episódio nos itens da Central.
4. **PII fora de log:** erros truncados, corpos nunca logados, hashes em vez de valores, `beforeSend` do Sentry.
5. **Regra pura + adapter de I/O:** lógica testável sem banco (`classifyRisk`, `processNode`, `decidePacing`, `calculaScore`), com adapters `supabase-js` (web) e `pg` (worker).
6. **Vocabulário aberto onde o self-host importa:** motivos sem CHECK no banco, para o `update.sh` de um clone não quebrar sobre linhas antigas.
