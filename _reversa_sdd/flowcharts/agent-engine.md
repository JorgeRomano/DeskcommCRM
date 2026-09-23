# Fluxogramas — Núcleo de IA (`lib/agent-engine/`)

> 🟢 CONFIRMADO salvo indicação. Fonte: `lib/agent-engine/agent/inbound-turn.ts` e módulos irmãos.

## Turno inbound (ritual completo)

```mermaid
flowchart TD
  A[Job inbound_turn reivindicado da fila] --> B[claimJobs: DISTINCT ON contact + FOR UPDATE SKIP LOCKED]
  B --> C[runAgentTurn envolvido em comHandoffSeOrcamentoAcabar]
  C --> D[Abertura: loadPlaybook + checkpoint anterior + lead_state + get_lead_context]
  D --> E{maybeCompact / classifyStage / prune-tool-results}
  E --> F[Loop do modelo: generateText stopWhen stepCountIs maxSteps]
  F --> G{Modelo chamou tool?}
  G -- send_message --> H[Guarda falso-vazio + teto maxSendsPerTurn + promessa fora da tabela]
  H --> I[runBeforeSend: cadeia de 11 gates sob advisory lock por numero]
  I --> J{Algum gate vetou?}
  J -- sim --> K[rollback, erro de ENSINO ao modelo] --> F
  J -- nao --> L[atraso humano + envio pelo WahaChannelAdapter + recordSend] --> F
  G -- update_lead_state --> M[applyLeadStateUpdate: valida transicao do funil + mirror CRM] --> F
  G -- outras tools --> N[execute com tool-breaker + org filter] --> F
  G -- sem tool / fim --> O[2a chamada purpose checkpoint: JSON validado por Zod]
  O --> P[insertCheckpoint: commitments/objections/next_action/rolling_summary/declaracao]
  P --> Q{operatorEnabled e declaracao != nada_a_declarar?}
  Q -- sim --> R[enqueueJob operator_turn fire-and-forget]
  Q -- nao --> S[completeJob: guard lease, efeito exactly-once]
  R --> S
  K -.-> T{LlmBudgetExceededError?}
  T -- sim --> U[escort: avisa lead sem tokens + performHumanHandoff + re-lanca]
```

## Cadeia before-send (11 gates, versao 7)

```mermaid
flowchart TD
  A[body do modelo] --> G1[stop: opt-out / is_blocked / force_human]
  G1 --> G2[lgpd: anonimizado / base legal de prospecting]
  G2 --> G3[pacing: janela de horario + warmup cap + throttle]
  G3 --> G4[messaging_window: janela de 24h se banRisk]
  G4 --> G5[spinning: sha256 + Jaccard sobre janela]
  G5 --> G6[promise: preco/desconto/parcelas deterministico]
  G6 --> G7[semantic_promise: classificador LLM]
  G7 --> G8[case_promise: promessa de humano sem caso]
  G8 --> G9[internal_vocabulary: vazamento de vocabulario interno]
  G9 --> G10[agenda_stall: promessa de agenda sem tool]
  G10 --> G11[disclosure: 1o outbound - inject ou veto]
  G11 --> OK[send pelo adapter]
  G1 -. veto .-> V[curto-circuito: gates restantes = skipped, rollback]
  G2 -. veto .-> V
  G3 -. throttle .-> W[acumula waitMs, segue]
  W --> G4
```

## Máquina de estados do funil (`lead-state.ts`)

```mermaid
stateDiagram-v2
  [*] --> new
  new --> contacted
  new --> lost
  contacted --> qualifying
  contacted --> lost
  qualifying --> qualified
  qualifying --> lost
  qualified --> negotiating
  qualified --> lost
  negotiating --> won
  negotiating --> lost
  won --> [*]
  lost --> [*]
```

> Transições fora do grafo são rejeitadas com erro de ENSINO ao modelo. Regressão é proibida.
> `won` e `lost` são terminais. Fonte: `LEAD_STAGE_TRANSITIONS` em `lead-state.ts:37-45`.

## Ciclo de vida do job na fila (`queue/queue.ts`)

```mermaid
stateDiagram-v2
  [*] --> pending: enqueueJob (23505+sourceEventId = deduped)
  pending --> running: claimJobs (advisory lock + maxConcurrency)
  running --> done: completeJob (guard lease, exactly-once)
  running --> pending: failJob attempts<max (backoff exponencial) / rescheduleJob / reaper
  running --> dead: failJob attempts>=max (inbox job_dead)
  running --> failed: cancelJob (veto de negocio, terminal, sem retry)
  done --> [*]
  dead --> [*]
  failed --> [*]
```

## Circuit breaker de tools (`tool-breaker.ts`)

```mermaid
flowchart TD
  A[Modelo chama tool] --> B[assinatura = tool + sha256 args]
  B --> C{Modo}
  C -- exact_failure --> D[mesma tool+args repetida: warn -> block sintetico]
  C -- same_tool_failure --> E[mesma tool args variados: warn -> halt tool no run]
  C -- idempotent_no_progress --> F[tool read-only mesmo resultado: warn -> block]
  D --> G[erro breaker_* devolvido ao modelo]
  E --> G
  F --> G
```

## Escort de orçamento de IA (`comHandoffSeOrcamentoAcabar`)

```mermaid
flowchart TD
  A[Turno inteiro envolvido] --> B{LlmBudgetExceededError em qualquer chamada?}
  B -- nao --> C[turno normal]
  B -- sim --> D[le briefing do checkpoint durável - sem LLM]
  D --> E[avisa o lead com texto de codigo - sem tokens]
  E --> F[performHumanHandoff: ai_handling->pending, silencia, cancela crons, inbox]
  F --> G[re-lanca o erro: a fila decide o destino do job]
```
