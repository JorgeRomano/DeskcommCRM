# C4 — Nível 3: Componentes — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) · nível **detalhado** · 🟢 CONFIRMADO salvo nota
> Detalha os dois containers mais relevantes: **app** (superfície HTTP + domínio) e **worker**
> (agent-engine). Fonte: `code-analysis.md`, `data-dictionary.md`, `domain.md`.

## A. Container `app` — superfície HTTP + domínio

```mermaid
C4Component
    title Componentes — app (Next.js)

    Container_Boundary(app, "app") {
        Component(proxy, "proxy.ts", "Edge middleware", "X-Request-Id, x-pathname, sessão, impersonation; public-paths")
        Component(rest, "app/api/v1/**", "Route handlers", "Zod → guard → org confiável → query → audit → ok/fail")
        Component(mcp, "app/api/mcp + lib/mcp", "Servidor MCP", "~47 tools, Bearer dsk_, RBAC, recusa-para-o-modelo, auditoria")
        Component(actions, "app/actions", "Server Actions", "auth, onboarding, team, settings")
        Component(apiwrap, "lib/api", "Infra HTTP", "ok()/fail(), errors, idempotency, auth-dual, wrappers")
        Component(authc, "lib/auth", "Auth/RBAC", "requireRole, server (getUser), public-paths, rate-limit, invite-token")
        Component(dominio, "lib/{leads,pipelines,agenda,financeiro,followup,automation,routing,...}", "Domínio", "Regras puras e serviços de aplicação")
        Component(channels, "lib/channels + waha", "Canais", "capabilities, adapters, pós-entrada, janela 24h")
        Component(aiplat, "lib/ai", "Plataforma de IA", "gateway, custo, orçamento, RAG, credenciais")
        Component(sb, "lib/supabase", "Clients", "browser / server (RLS) / admin (service role)")
        Component(audit, "lib/audit", "Auditoria", "audit() fire-and-forget, ~330 ações, redação PII")
        Component(crypto, "lib/crypto", "Cifra", "AES-GCM (credenciais, tokens SIP)")
    }

    ContainerDb(pg, "Supabase Postgres", "Postgres", "RLS + RPCs")
    ContainerDb(redis, "Redis", "Upstash/srh", "Rate limit")
    System_Ext(aigw, "Vercel AI Gateway", "LLM")

    Rel(proxy, rest, "Rota autenticada")
    Rel(rest, authc, "requireRole / requirePlatformAdmin")
    Rel(rest, dominio, "Casos de uso")
    Rel(rest, apiwrap, "Resposta padronizada")
    Rel(mcp, dominio, "tools → serviços")
    Rel(dominio, sb, "Query (RLS ou filtro manual)")
    Rel(sb, pg, "SQL")
    Rel(aiplat, aigw, "Chamada LLM/embedding")
    Rel(aiplat, redis, "Rate limit dispatcher")
    Rel(dominio, audit, "audit() em mutação")
    Rel(aiplat, crypto, "Decifra credencial")
    Rel(channels, dominio, "pós-entrada ordenado")
```

### Componentes-chave do `app` 🟢

| Componente | Arquivo canônico | Papel |
|---|---|---|
| Borda | `proxy.ts` | Auth de borda, `X-Request-Id`, impersonation |
| Guard RBAC | `lib/auth/require-role.ts` (`requireRole`) | Rank de papéis via `fn_user_role_in_org` |
| Sessão | `lib/auth/server.ts` (`loadAuthUser`, `resolveActiveOrg`) | `getUser()`, org ativa |
| Resposta | `lib/api/wrappers.ts` (`ok()`/`fail()`), `errors.ts` | Formato + `X-Request-Id` |
| Auth dual | `lib/api/auth-dual.ts` | Cookie OU bearer, org de fonte confiável |
| Idempotência | `lib/api/idempotency.ts` | Reserva antes do efeito (409 in_progress vs conflict) |
| Servidor MCP | `lib/mcp/server.ts`, `tools/*` | Pipeline higieniza→ctx→scope+role→handler→motivoDoVazio→audit |
| Clients | `lib/supabase/{browser,server,admin}.ts` | RLS vs service role |

## B. Container `worker` — agent-engine (o ritual do turno)

```mermaid
C4Component
    title Componentes — worker (agent-engine)

    Container_Boundary(worker, "worker") {
        Component(queue, "queue/", "Fila", "enqueueJob (dedup 23505), claimJobs (SKIP LOCKED), completeJob (exactly-once)")
        Component(inbound, "agent/inbound-turn.ts", "Ritual do turno", "abrir (playbook+checkpoint+estado+histórico) → loop tools → fechar")
        Component(turns, "agent/{operator,followup,case-reply}-turn.ts", "Outros turnos", "OPERAR (muta CRM, não fala), follow-up, resposta de caso")
        Component(leadstate, "agent/lead-state.ts", "Máquina de funil", "LEAD_STAGES; só avança; won/lost terminais")
        Component(handoff, "agent/human-handoff.ts", "Handoff", "force_human, bot_silenced_until, casos humanos")
        Component(breaker, "agent/tool-breaker.ts", "Circuit breaker", "exact_failure / same_tool / idempotent_no_progress")
        Component(guard, "guardrails/before-send.ts", "Guardrails", "cadeia v7 de 11 gates, advisory lock, curto-circuito")
        Component(pacing, "pacing/engine.ts", "Pacing", "janela horário + warmup por idade + throttle+jitter")
        Component(spinning, "spinning/engine.ts", "Spinning", "sha256 exato + Jaccard (anti-cópia em massa)")
        Component(edge, "edge/llm/*", "Edge LLM", "run-model-call, orcamento (decidirOrcamento), providers")
        Component(compaction, "compaction.ts + memória", "Anti-context-rot", "rolling summary + poda determinística")
        Component(cron, "cron/scheduler.ts", "Cron", "scheduleCronJob (stagger), fireOneDue")
    }

    ContainerDb(pg, "Postgres", "Postgres", "jobs, event_log, checkpoints, llm_calls")
    System_Ext(aigw, "Vercel AI Gateway", "LLM")
    Container(waha, "waha/meta/zernio", "Adapters", "Envio de saída")

    Rel(queue, inbound, "Claim → handler do job")
    Rel(inbound, edge, "runModelCall (LLM)")
    Rel(edge, aigw, "Chamada de modelo")
    Rel(inbound, guard, "send_message → before-send")
    Rel(guard, pacing, "gate pacing")
    Rel(guard, spinning, "gate spinning")
    Rel(guard, waha, "Aprovado → adapter")
    Rel(inbound, leadstate, "applyLeadStateUpdate")
    Rel(inbound, handoff, "Gatilho → performHumanHandoff")
    Rel(inbound, breaker, "wrapToolsWithBreaker")
    Rel(inbound, compaction, "Compacta contexto")
    Rel(cron, queue, "Enfileira jobs agendados")
    Rel(queue, pg, "SQL (lease/lock)")
    Rel(inbound, pg, "Checkpoint durável")
```

### Regras estruturais do `worker` 🟢

- **Enviar é sempre `send_message` (tool call)** — texto direto do modelo é descartado (RN-04). Tudo
  passa pela cadeia `before_send` (11 gates, versão 7): `stop, lgpd, pacing, messaging_window,
  spinning, promise, semantic_promise, case_promise, internal_vocabulary, agenda_stall, disclosure`.
- **Fechamento sempre acontece** — o checkpoint é uma 2ª chamada de LLM (`purpose:'checkpoint'`), não
  uma tool (RN-05).
- **Idempotência evento→job** por `unique(organization_id, source_event_id)` + captura `23505`;
  **efeito exactly-once** no `completeJob` (guard de lease). Uma lane por contato por vez.
- **Papel FALAR vs OPERAR** — o operador muta o CRM e nunca fala (separação por ausência da tool
  `send_message`).
- **Máx. 3 envios físicos por turno** (`DEFAULT_MAX_SENDS_PER_TURN`).
- **Orçamento nunca bloqueia sem aviso prévio**; ao estourar, avisa o lead com texto de código, roda
  handoff e re-lança.

## Componentes transversais (ambos os containers) 🟢

| Componente | Onde | Papel |
|---|---|---|
| `lib/event-log` | app + worker | `event_log` + dispatcher; `EventRow`/`HandlerResult` |
| `lib/audit` | app + worker | Trilha fire-and-forget, redação de PII |
| `lib/crypto/aes_gcm` | app + worker | `EncryptedSecret` (IV 12B, tag 16B), `last4` exposto |
| `lib/tempo` / `lib/relogio` | app + worker | Relógio da org (fuso), DST-safe via `Intl` |
| `lib/env` | app + worker | Contrato de env vars (Zod); app não sobe sem obrigatórias |
| `lib/branding` | app | Marca resolve do banco (nunca lança; roda em `layout.tsx`) |

## Nota de confiança
🟢 Componentes e arquivos vêm de `code-analysis.md`. Os diagramas são uma visão de dependência
lógica; a ordem exata dos gates e as regras numéricas estão em `domain.md` e `data-dictionary.md`.
