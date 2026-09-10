# Máquinas de Estado — DeskcommCRM

> Gerado pelo **Detetive** (Reversa) em 2026-09-10 · Nível: **Detalhado**
> 🟢 CONFIRMADO no código · 🟡 INFERIDO · 🔴 LACUNA

---

## 1. `crm_leads.status` — negócio/demanda 🟢

Estados: `open`, `won`, `lost`. Status e `closed_at` são escritos pelo trigger `fn_crm_lead_close_on_stage` ao mover para estágio com flag — nunca à mão (`encerramento.ts`, regra P-02).

```mermaid
stateDiagram-v2
  [*] --> open: garantirLeadDaConversa / criação manual
  open --> won: move p/ estágio is_won (encerraDemanda / kanban / bulk)
  open --> lost: move p/ estágio is_lost (exige lost_reason)
  won --> open: reabertura (move p/ estágio sem flag) — activity reopened
  lost --> open: reabertura — activity reopened
  won --> [*]: reportado como conversão (Purchase) se houver atribuição
  lost --> [*]
```

**Regras:** `lost` exige `lost_reason` (P-03, 422). Reabertura é auditada (P-04). Encerrar é idempotente (`jaEstava`). Lead `won` com atribuição de anúncio dispara `conversaoDeVendaHandler`.

---

## 2. `conversations.status` — conversa 🟢/🟡

Vocabulário (regra AT-01): `open`, `pending`, `ai_handling`, `claimed`, `closed`, `resolved`, `archived` (os 3 últimos terminais).

```mermaid
stateDiagram-v2
  [*] --> open: 1ª mensagem inbound
  open --> ai_handling: agente assume (active_ai_agent_id)
  ai_handling --> pending: handoff (bot_silenced_until=infinity)
  open --> pending: handoff / escalação
  pending --> claimed: atendente "eu cuido" (claim atômico)
  claimed --> resolved: atendente resolve
  open --> closed: UI Fechar
  claimed --> closed: UI Fechar
  closed --> [*]
  resolved --> [*]
  archived --> [*]
  closed --> open: inbound válido após terminal cria NOVA fronteira/demanda
```

**Silêncio do bot** (`bot_silenced_until`): `infinity` = handoff formal (nunca reassume); prazo finito = pausa por atendimento manual (60min, renova, nunca encurta maior). `service_revision` avança ao tocar estado terminal ou trocar de demanda (migration 0222). 🟡 alguns estados intermediários (`ai_handling`/`claimed`) inferidos do vocabulário do catálogo.

---

## 3. `followup_enrollments.status` — follow-up 🟢

```mermaid
stateDiagram-v2
  [*] --> active: enrollFollowupFlow (1 por lead/org)
  active --> waiting_reply: nó ai_classify/match_reply aguarda resposta
  waiting_reply --> active: resposta do lead (wokeEarly) ou timeout
  active --> paused_handoff: ai.handoff_triggered (policy=pause)
  waiting_reply --> paused_handoff: idem
  paused_handoff --> active: ai.handoff_resolved (+RESUME_GRACE_MS)
  active --> paused_manual: intervenção humana (migration 0145)
  paused_manual --> active: retomada
  active --> completed: nó end (outcome)
  active --> cancelled: opt-out / stale boundary / cancel_on_reply
  waiting_reply --> cancelled: idem
  active --> dead: steps_taken>80 OU action nunca completou (14 rechecks)
  completed --> [*]
  cancelled --> [*]
  dead --> [*]
```

**Outcomes** (só em terminais): `converted | replied | exhausted | opted_out | handoff`. **terminal** (denominador de conversão) = `completed + cancelled`; `dead` é excluído (falha de infra). `completeTurnForEnrollment` só avança se status ∈ {active, waiting_reply} (lista positiva — turno stale não sobrescreve intervenção humana).

---

## 4. `lead_state.stage` — funil interno do agente 🟢

Máquina de qualificação BANT que o agente marca via `update_lead_state` (só o próximo estágio válido; regressão rejeitada). Espelhada no CRM via `agent_stage_hint`.

```mermaid
stateDiagram-v2
  [*] --> new
  new --> contacted
  contacted --> qualifying
  qualifying --> qualified
  qualified --> negotiating
  negotiating --> won
  negotiating --> lost
  qualifying --> lost
  won --> [*]
  lost --> [*]
```

**Ponte para o funil do tenant** (`agent-stage-sync.ts`): o passo do agente → estágio nomeado via `crm_stages.agent_stage_hint`. `sem_mapeamento` não move (não inventa semântica); trava otimista impede atropelar movimento humano (`conflito_humano`).

---

## 5. `lgpd_requests.status` — pedido LGPD 🟢

```mermaid
stateDiagram-v2
  [*] --> received: createLgpdRequest (due_at em dias úteis BR)
  received --> processing: worker inicia
  processing --> completed: export/redact aplicado + eventos emitidos
  processing --> pending_review: sem email / sem footprint local / falha parcial de tenant
  processing --> failed: attempts >= 3
  received --> failed: attempts >= 3
  completed --> [*]
  failed --> [*]
  pending_review --> [*]
```

**Tipos:** `data_request` (export), `redact`, `store_redact`. **Scope:** `contact` | `tenant`. Redação de tenant marca `organizations.status='redacted'`.

---

## 6. `event_log.status` — barramento de eventos 🟢

```mermaid
stateDiagram-v2
  [*] --> pending: emit_event
  pending --> processing: claim otimista (where pending)
  processing --> done: só ok/skipped (consumed_by += keys)
  processing --> pending: retry (não incrementa attempts) OU error+backoff (attempts<5)
  processing --> pending: reaper (>10min órfão)
  processing --> dead: error e attempts>=5
  done --> [*]
  dead --> [*]
```

**Precedência de desfecho:** retry > error > sucesso. Backoff exponencial `2^n` min. `consumed_by` garante idempotência de retry por handler.

---

## 7. `channel_sessions.status` — sessão WhatsApp 🟡

Estados do WAHA repassados: `WORKING`, `SCAN_QR_CODE`, `STARTING`, `STOPPED`, `FAILED` (entre outros). 🟡 vocabulário do WAHA, não enum próprio.

```mermaid
stateDiagram-v2
  [*] --> SCAN_QR_CODE: sessão criada
  SCAN_QR_CODE --> WORKING: QR escaneado
  WORKING --> STOPPED: logout/stop
  WORKING --> FAILED: erro
  STARTING --> WORKING
  STARTING --> STARTING: preso >5min → alerta (W-11, não auto-recovery)
```

**W-11:** sessão presa em `STARTING` >5min gera alerta + escala (não tenta auto-recovery — perder sessão em silêncio é pior). O watchdog reconcilia `channel_sessions` × WAHA e redirige `queued`.

---

## 8. `messages.status` — mensagem 🟢

```mermaid
stateDiagram-v2
  [*] --> queued: agent-engine segura (sessão não WORKING)
  queued --> sending: despacho ao WAHA
  sending --> sent: ack do canal (external_id gravado)
  sending --> failed: cron recover-stuck (>5min) OU erro do canal
  queued --> failed
  sent --> [*]
  failed --> [*]
```

**W-12:** `sending` >5min vira `failed` (cron); `queued` não entra (tem dono — o engine reagenda por `SEND_QUEUED_RETRY_MS`).
