# Fluxogramas — Automação e roteamento

> Gerado pelo Arqueólogo (Reversa) — Unidade 7.
> Cobre `lib/automation/`, `lib/routing/`, `lib/followup/`, `lib/escalacao/`.

---

## 1. Motor de regras — `runAutomationForEvent` (`automation/engine.ts`)

```mermaid
flowchart TD
  A[Evento do event_log] --> B{caused_by_rule<br/>ou request_id 'rule:'?}
  B -- sim --> BX[skipped: caused_by_rule]
  B -- não --> C{entity_kind bate<br/>com o esperado?}
  C -- não --> CX[skipped: entity_kind_mismatch]
  C -- sim --> D[Carrega automation_rules ativas<br/>trigger_event = event_type]
  D --> E{payload traz rule_id?<br/>regraDoEvento}
  E -- sim --> E1[Só a regra apontada]
  E -- não --> E2[Todas as regras do tipo]
  E1 --> F
  E2 --> F[buildContext: event + lead/contact/appointment<br/>sempre filtrando organization_id]
  F --> G[applicable = regras cujas<br/>evaluateConditions = true]
  G --> H{alguma ação tem<br/>postponeUntil que retorna ISO?}
  H -- sim --> HX[registrarAdiamento status='adiado'<br/>return retry + retry_at]
  H -- não --> I[Para cada regra: executa cada ação]
  I --> J[Agrega: naoEnviadas=failed+skipped<br/>adiados=postponed]
  J --> K{status derivado}
  K -->|naoEnviadas = total| K1[failed]
  K -->|naoEnviadas parcial| K2[partial]
  K -->|só adiados| K3[adiado]
  K -->|nada disso| K4[success]
  K1 --> L[Grava automation_rule_runs + audit se != success]
  K2 --> L
  K3 --> L
  K4 --> L
  L --> M[Atualiza last_run_at + run_count]
```

---

## 2. Ação de envio — `send_whatsapp_message` / `send_ai_message`

```mermaid
flowchart TD
  A[postponeUntil] --> B{guardas de contato OK?}
  B -- não --> B0[segue sem adiar por agenda]
  B -- sim --> B1{proteção de agenda pede adiar?}
  B1 -- leitura indisponível --> BX[throw: retry all-or-nothing]
  B1 -- adiar --> BR[retorna reavaliar_em]
  B1 -- não --> C{fora da janela do número?<br/>adiarAteAJanelaAbrir}
  B0 --> C
  C -- sim --> CR[retorna próxima abertura]
  C -- não --> D{checkDailyLimit allowed?}
  D -- não --> DR[retorna retry_at]
  D -- sim --> E[null: pode enviar agora]

  E --> X[execute]
  X --> F{checarGuardasDeContato}
  F -- no_contact/blocked/no_phone/consent_declined --> FX[skipped: reason]
  F -- ok --> G[assertAgendaEffect + serviceForAutomation]
  G --> H[espacarEnvio 1200ms+jitter]
  H --> I[sendMessageHandler texto/IA]
  I --> J[reportarEnvio: desfecho do ESTADO da mensagem]
  J -->|sent/delivered/read| J1[success]
  J -->|queued| J2[postponed + queued_reason]
  J -->|failed/desconhecido| J3[failed + abre aviso na Central]
```

---

## 3. Roteamento — `runRoutingWorker` + `decideRouting`

```mermaid
flowchart TD
  A[cron: drena conversation.routing_requested] --> B[recupera 'processing' abandonado > 5min]
  B --> C[claim CAS pending -> processing]
  C --> D[processEvent: valida payload/UUID/org]
  D --> E{conversa existe e status roteável?}
  E -- não --> EX[markDone: skipped_conv_missing]
  E -- sim --> F[Resolve settings.routing por Zod]
  F --> G{alreadyAssigned?}
  G -- sim --> GX[skip: already_assigned]
  G -- não --> H{mode}
  H -- manual --> HX[skip: manual_mode]
  H -- outro --> HY[skip: unsupported_mode]
  H -- round_robin --> I[loadEligibleAttendants por canal]
  I --> J[decideRouting]
  J -->|assign| K[fn_channel_routing_claim]
  K -->|assigned| K1[adotarLeadsDoContato + markDone]
  K -->|already/changed| K2[assign_lost_race]
  K -->|revoked/not_allowed/capacity| K3[requeue]
  J -->|requeue| L{attempts >= max_retries?}
  L -- sim --> L1[fn_routing_unassigned_notice]
  L -- não --> L2[requeue com backoff]
```

### Elegibilidade (pura, `routing/eligibility.ts`)

```mermaid
flowchart LR
  A[isAttendantEligible] --> B{isAvailable?}
  B -- não --> N[inelegível]
  B -- sim --> C{currentLoad < capacity?}
  C -- não --> N
  C -- sim --> D{isWithinSchedule?<br/>windows vazio = 24/7, tz-aware via Intl}
  D -- não --> N
  D -- sim --> Y[elegível]
```

---

## 4. Follow-up — tick do worker e decisão por nó

```mermaid
flowchart TD
  A[runFollowupTick] --> B[claimDueEnrollments lease 120s]
  B -->|claim falhou| BX[log + claim_falhou=true, NÃO lança]
  B --> C[processEnrollment por inscrição]
  C --> D[assertServiceBoundary + assertAgenda]
  D -->|stale boundary em 'dormente'| DX[cancela com aviso VISÍVEL na Central]
  D --> E{steps_taken > MAX_STEPS 80?}
  E -- sim --> EX[markDead: max_steps]
  E -- não --> F[carrega grafo pinado + nó + LeadFacts + eventos]
  F --> G[processNode: decisão pura]
  G --> H[applyResult]
  H --> H1[grava evento idempotente node:steps]
  H1 --> H2{replay? corrida action_sent x recheck?}
  H2 -- sim --> H3[aplica só o avanço, sem novo evento]
  H2 -- não --> I{result.kind}
  I -->|advance| I1[current_node = target, next_eval = agora]
  I -->|wait| I2[fica no nó, timer next_eval]
  I -->|enqueue_turn| I3[enfileira job de turno once-per-ocupância]
  I -->|recheck| I4[fica no nó, backoff, sem reenfileirar]
  I -->|complete| I5[status completed + outcome + aviso no-show?]
  I -->|dead| I6[markDead]
  I -->|fail| I7[applyHandlerFailure: backoff / dead]
```

### `processNode` por tipo de nó

```mermaid
flowchart TD
  N{node.type} -->|trigger| T[decide timing_plan: enqueue plan_timing<br/>ou advance; dead-man 3 -> segue sem plano]
  N -->|wait| W{wokeEarly?}
  W -- sim --> Wadv[advance]
  W -- não --> Wt[wait: fixed duration_ms ou smart planejado clampado<br/>immune_to_reply -> dormente]
  N -->|condition| C[per_check: 1ª regra que passa roteia<br/>combined: combinator; null NÃO satisfaz neq]
  N -->|ai_classify| AI{waitElapsed & !wokeEarly?}
  AI -- não --> AIe[enqueue classify]
  AI -- sim --> AIn[grace venceu -> no_reply sem LLM]
  N -->|match_reply| MR[casa branch por texto eq/contains<br/>save_to if_exists skip/overwrite/confirm]
  N -->|repeat| R[body enquanto taken<total, senão done]
  N -->|action| ACT[enqueue send once; recheck em voo;<br/>dead-man 14 ocioso -> dead]
  N -->|end| E[complete: converted/exhausted/custom]
```

### Reatividade (`followup/reactivity.ts`)

```mermaid
flowchart TD
  A[applyReactivityEvent] --> B{event_type}
  B -->|message.received| C{contato is_blocked? STOP/opt-out}
  C -- sim --> C1[cancela TUDO vivo incl. dormente: opted_out]
  C -- não --> C2{cancel_on_reply no ponteiro?}
  C2 -- sim --> C3[cancela waiting_reply + active: replied]
  C2 -- não --> C4[acorda: inbound_woke, next_eval=agora]
  B -->|ai.handoff_triggered| D{handoff_policy por contato}
  D -->|allow| D0[nada]
  D -->|cancel| D1[cancela: handoff]
  D -->|pause| D2[paused_handoff]
  B -->|ai.handoff_resolved| E[retoma paused_handoff do contato + RESUME_GRACE]
```

---

## 5. Escalação — devolução à IA e devolução automática

```mermaid
flowchart TD
  A[devolverAtendimentoAoAgente] --> B[1. lê continuidade ANTES de mutar]
  B --> C[2. solta dono humano: fn_conversation_assign release]
  C --> D[3. UPDATE conversations: assignee_kind=ai, status=ai_handling<br/>guard .is assigned_to_user_id null]
  D -->|0 linhas| DX[assignment_conflict]
  D --> E[3. contacts.force_human=false — a trava esquecida]
  E --> F[3b. autorizarContatoParaIA re-autoriza gate allowlist]
  F --> G[3c. fn_passagem_devolvida destrava dedup do aviso]
  G --> H[4. AWAITED emit_event ai.handoff_resolved<br/>único produtor do sinal que retoma follow-up]
  H -->|erro| HX[resume_signal_failed]
  H --> I[5. checkpoint de retomada acrescentado]
  I --> J[6. atividade handoff_resolved na timeline]
  J --> K[audit ai.reactivated_by_agent]
```

### Devolução automática (cron, pura — `devolucao-automatica.ts`)

```mermaid
flowchart TD
  A[avaliarDevolucao] --> B{prazo em settings.routing?}
  B -- não --> B0[sem_prazo]
  B -- sim --> C{está com humano durável?}
  C -- não --> C0[nao_esta_com_humano]
  C -- sim --> D{status devolvível?}
  D -- não --> D0[status_nao_devolvivel]
  D -- sim --> E{sessão tem agente publicado?}
  E -- não --> E0[sessao_sem_agente]
  E -- sim --> F{tem último sinal humano?}
  F -- não --> F0[sem_relogio]
  F -- sim --> G{agora - sinal >= prazo?}
  G -- não --> G0[dentro_do_prazo]
  G -- sim --> Y[DEVOLVER -> devolverAtendimentoAoAgente automática]
```

---

## 6. Escalação — aviso ao suporte (`aviso-ao-suporte.ts`)

```mermaid
flowchart TD
  A[ai.case_opened / ai.case_closed] --> Z{caso fechado?}
  Z -- sim --> ZC[cancela pendentes do caso]
  Z -- não --> B[0. dreno em request? adia sem rede]
  B --> C[1. payload tem case_id?]
  C --> D[2. config ligada + canal? senão sem_configuracao]
  D --> E[3. evento < 30min?]
  E --> F[4. caso ainda aberto e origem aceita?]
  F --> G[5. REIVINDICA entrega antes da rede — unique 23505]
  G --> H[6. entrega < 24h?]
  H --> I[7. titular anonimizado? cancela]
  I --> J[8. canal WORKING, aceita livre, não arquivado? retry até 6]
  J --> K[9. URL pública OK?]
  K --> L[10. transporte configurado? retry até 3]
  L --> M[11. pacing: espaçamento + cap warm-up — SEM janela de horário]
  M --> N[12. resolve destino]
  N --> O[13. monta texto sem PII]
  O --> P[14. ENVIA]
  P -->|erro| PX[condena ou retry conforme tentativas]
  P --> Q[15. marca enviado + corpo_hash sha256 — daqui falha ABERTO]
  Q --> R[depoisDoEnvio: pacing ledger, JID, timeline do caso]
```
