# C4 Nível 3 — Componentes — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) · 🟢 CONFIRMADO. Foco nos dois containers mais relevantes: `app` e `worker`.

## Container `app` (Next.js) — componentes internos

```mermaid
C4Component
  title Componentes — container app

  Container_Boundary(app, "app (Next.js)") {
    Component(proxy, "proxy.ts", "Middleware", "Auth de borda (getUser), X-Request-Id, gate /admin, impersonation edge")
    Component(apiv1, "app/api/v1/*", "Route handlers", "REST versionado; Zod → requireRole → org → audit → ok/fail")
    Component(apiint, "app/api/internal + mcp + cron", "Superfícies não-cookie", "Bearer/secret/state HMAC")
    Component(actions, "app/actions/*", "Server Actions", "auth, onboarding, team, settings")
    Component(ui, "app/app/* + components", "UI autenticada", "inbox, kanban, radar, ai, settings...")
    Component(authlib, "lib/auth", "RBAC", "requireRole, loadAuthUser, rate-limit, tokens")
    Component(apilib, "lib/api", "Contrato", "ok()/fail(), errors, Actor/HandlerCtx")
    Component(domain, "lib/<dominio>", "Regras", "leads, pipelines, kanban, waha, channels, lgpd, branding...")
    Component(eventlog, "lib/event-log", "Barramento", "dispatcher + drain (cron)")
    Component(sb, "lib/supabase", "Clients", "browser/server/admin")
  }

  Rel(proxy, apiv1, "encaminha autenticado")
  Rel(apiv1, authlib, "requireRole")
  Rel(apiv1, apilib, "ok/fail")
  Rel(apiv1, domain, "regras de negócio")
  Rel(domain, sb, "queries")
  Rel(domain, eventlog, "emit_event / handlers")
  Rel(ui, apiv1, "fetch (react-query)")
```

## Container `worker` (agent-engine) — componentes internos

```mermaid
C4Component
  title Componentes — container worker

  Container_Boundary(w, "worker (agent-engine)") {
    Component(main, "workers/agent-worker/main.ts", "Processo", "boot, healthz, loops, dispatch de jobs")
    Component(queue, "agent-engine/queue", "Fila", "job_queue: claim, lease, reaper")
    Component(inbound, "agent/inbound-turn.ts", "Turno", "sessão fresca do LLM, tools, checkpoint")
    Component(guard, "guardrails/before-send.ts", "Guardrails", "cadeia v6 de 10 gates")
    Component(pacing, "pacing + spinning", "Anti-ban", "decidePacing / decideSpinning")
    Component(flywheel, "flywheel/live.ts", "Aprendizado", "judge + distiller (gate humano)")
    Component(drain, "edge/crm/drain.ts", "Drain", "event_log → jobs")
    Component(cron, "cron/scheduler.ts", "Cron", "follow-up por contato")
    Component(watchdog, "edge/crm/session-*.ts", "Watchdog", "reconcilia channel_sessions × WAHA")
    Component(elog, "event-log/drain-loop.ts", "Handlers", "mídia, branding, follow-up, LGPD...")
    Component(channel, "edge/channel/waha-adapter.ts", "Canal", "ChannelAdapter (envio idempotente)")
  }

  Rel(main, queue, "claim/complete/fail")
  Rel(main, drain, "runDrainLoop")
  Rel(main, cron, "runCronLoop")
  Rel(main, watchdog, "runSessionWatchdogLoop")
  Rel(main, flywheel, "runFlywheelLoop")
  Rel(main, elog, "runEventLogDrainLoop")
  Rel(queue, inbound, "handler inbound_turn")
  Rel(inbound, guard, "send_message")
  Rel(guard, pacing, "pacingGate/spinningGate")
  Rel(guard, channel, "envia se todos pass")
```

## Responsabilidades-chave 🟢

- **`proxy.ts`** é o gate de borda; **`requireRole`** é o gate por rota (role do banco + MFA de sessão).
- **`before-send`** é o único caminho de saída do agente ao canal (10 gates, advisory lock por número).
- **`drain`** transforma `event_log` em `job_queue`; o `event-log/drain-loop` roda os handlers de eventos (mídia/branding/LGPD/follow-up) no ritmo do worker além do cron.
- **`watchdog`** mantém `channel_sessions` coerente com o WAHA e redirige `queued`.
