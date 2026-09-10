# Fluxograma — Turno do agente de IA e guardrails de saída

> Gerado pelo **Arqueólogo** · Módulos: `lib/agent-engine/agent/inbound-turn.ts` + `guardrails/before-send.ts` 🟢

## Turno do agente rico (`inbound_turn`)

```mermaid
flowchart TD
  A[job inbound_turn na fila] --> B[abre sessão FRESCA do LLM]
  B --> C[lê playbook + lead_checkpoints + lead_state + últimas N msgs]
  C --> D[comHandoffSeOrcamentoAcabar envolve tudo]
  D --> E[loop de tools do modelo]
  E --> F{tool escolhida}
  F -- send_message --> G[cadeia before-send]
  F -- update_lead_state --> H[máquina new→...→won/lost + espelha no CRM]
  F -- search_knowledge --> I[RAG]
  F -- schedule_followup --> J[agenda retorno + encerra turno]
  F -- open_human_case --> K[abre caso p/ retaguarda]
  F -- request_human_handoff --> L[triggerHandoff]
  F -- send_template --> M[template Meta fora da janela 24h]
  G --> N{gates passaram?}
  N -- veto --> O[razão volta ao modelo como erro instrutivo]
  O --> E
  N -- pass --> P[ChannelAdapter envia]
  E --> Q[2ª chamada purpose=checkpoint]
  Q --> R[JSON validado por Zod → lead_checkpoints]
```

**Regras duras:** texto direto do modelo nunca vira mensagem (só `send_message`). Orçamento estourado em QUALQUER chamada (mesmo `classifyStage`) devolve à fila humana. Teto `DEFAULT_MAX_SENDS_PER_TURN=3`.

## Cadeia `before-send` (ordem versionada v6)

```mermaid
flowchart TD
  A[candidata do modelo] --> L[pg_advisory_xact_lock por número]
  L --> G1[1. stop — opt-out/blocked/force_human]
  G1 -- veto --> V[razão ao modelo]
  G1 --> G2[2. lgpd — anonimização/base legal]
  G2 --> G3[3. pacing — janela/warm-up/cap → waitMs]
  G3 --> G35[3.5 messaging_window — 24h]
  G35 --> G4[4. spinning — Jaccard ≥ 0.8]
  G4 --> G5[5. promise — preço/desconto/parcela vs tabela]
  G5 --> G6[6. semantic_promise — texto livre]
  G6 --> G65[6.5 case_promise — humano sem caso]
  G65 --> G67[6.7 internal_vocabulary]
  G67 --> G69[6.9 agenda_stall]
  G69 --> G7[7/8 disclosure — pode emendar corpo]
  G7 -- todos pass --> S[ChannelAdapter.send]
```

**Defaults assimétricos:** `internalVocabularyEnforced` ausente = DESARMADO; `spinningEnforced` ausente = ARMADO; `messagingWindow` ausente = FECHADA. Cada veredito vira linha em `before_send_traces` (auditável por run).

## Handoff bot→humano (`triggerHandoff`)

```mermaid
flowchart TD
  A[gatilho G1/G2/G3/G4/orçamento] --> B{idempotente 5s mesma reason?}
  B -- sim --> Z[skip idempotent_5s]
  B -- não --> C{elegível? IA poderia atender}
  C -- não --> Z2[skip nao_elegivel]
  C -- sim --> D[avisarLeadDoCrm]
  D --> E[conversations: status=pending<br/>bot_silenced_until=infinity<br/>zera active_ai_agent_id]
  E --> F[crm_lead_activities handoff_triggered]
  F --> G[move card p/ etapa chamar-humano — opt-in]
  G --> H[emit_event ai.handoff_triggered]
  H --> I[broadcast realtime org:queue]
  I --> J[audit + agent_inbox_items dedup por episódio]
```
