# Fluxograma — Follow-up, automação e barramento de eventos

> Gerado pelo **Arqueólogo** · Módulos: `lib/followup/`, `lib/automation/engine.ts`, `lib/event-log/` 🟢

## Barramento de eventos (`event_log` drain)

```mermaid
flowchart TD
  A[emit_event grava linha pending] --> B[drainEventLog: cron + laço no worker]
  B --> C[reaper: processing >10min → pending]
  C --> D[claim otimista: processing where pending]
  D -- não reivindicou --> Z[pula — outra instância pegou]
  D -- reivindicou --> E[dispatchEvent: handlers não em consumed_by]
  E --> F{desfecho por precedência}
  F -- algum retry --> G[pending, NÃO incrementa attempts, next_attempt_at]
  F -- algum error --> H[attempts++; ≥5 → dead; senão pending+backoff 2^n]
  F -- só ok/skipped --> I[done; consumed_by += ok+skipped]
```

**15 handlers em ordem deliberada:** `followupReactivity` (antes do LLM), aiResponse, aiSentiment, aiHandoffFromSentiment, ragIndexer, lgpdExport, lgpdRedact, automationRules, followupGatilhoEtapa/Caso/Presenca, mediaPersist/Derive, webPushInbound, `conversaoDeVenda` (por último — não pode segurar quem escreve no banco).

## Motor de automação (`runAutomationForEvent`)

```mermaid
flowchart TD
  A[evento-gatilho] --> B{caused_by_rule OU request_id rule:?}
  B -- sim --> Z[skip anti-loop]
  B -- não --> C{entity_kind esperado?}
  C -- não --> Z2[skip entity_kind_mismatch]
  C -- sim --> D[carrega automation_rules ativas por trigger_event]
  D --> E[buildContext lead/contact filtrado por org]
  E --> F[evaluateConditions eq/neq/contains em AND]
  F --> G{alguma ação com postponeUntil?}
  G -- fora da janela --> H[registra adiamento + retry all-or-nothing]
  G -- não --> I[executa ações em sequência]
  I --> J[agrega: skipped+failed juntos; falha vence adiamento]
  J --> K[automation_rule_runs + audit se ≠ success]
```

## Máquina de follow-up (`processNode`)

```mermaid
flowchart TD
  A[claimDueEnrollments lease 120s] --> B[processEnrollment isolado]
  B --> C{service_boundary / agenda ok?}
  C -- stale --> Z1[cancela: atendimento encerrado]
  C -- agenda adiar --> Z2[reagenda next_eval_at]
  C -- ok --> D{steps_taken > 80?}
  D -- sim --> Z3[markDead max_steps]
  D -- não --> E[processNode por tipo]
  E --> N1[trigger: decide timing_plan uma vez]
  E --> N2[wait: wokeEarly corta espera; senão duration/plano]
  E --> N3[condition: eq/neq/contains → branch]
  E --> N4[ai_classify: enqueue turn; grace vencido = no_reply sem LLM]
  E --> N5[match_reply: casa branches; save_to skip/overwrite/confirm]
  E --> N6[action: at-most-once send; dead-man 14 rechecks]
  E --> N7[repeat: parseReplyCount → body/done]
  E --> N8[end: complete outcome]
  N6 --> F[completeTurnForEnrollment: só active/waiting_reply avançam]
```

## Reatividade do follow-up (`applyReactivityEvent`)

```mermaid
flowchart TD
  A[evento] --> B{tipo}
  B -- message.received --> C{is_blocked?}
  C -- sim --> C1[cancelAll opted_out — hard stop LGPD]
  C -- não --> C2[waiting_reply: cancel_on_reply → replied<br/>senão acorda inbound_woke]
  B -- ai.handoff_triggered --> D{handoff_policy}
  D -- allow --> D0[ignora]
  D -- cancel --> D1[cancela outcome handoff]
  D -- pause --> D2[paused_handoff]
  B -- ai.handoff_resolved --> E[retoma paused_handoff → active +grace]
```

**Outcomes:** `converted | replied | exhausted | opted_out | handoff`. `conversion_rate = converted / terminal` (terminal = completed+cancelled; `dead` excluído — é falha de infra).
