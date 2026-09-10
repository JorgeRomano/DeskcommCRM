# C4 Nível 2 — Containers — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) · 🟢 CONFIRMADO (`docker-compose.prod.yml`, `next.config.ts`, `workers/`)

```mermaid
C4Container
  title Containers — DeskcommCRM (stack self-host)

  Person(lead, "Lead", "WhatsApp")
  Person(usuario, "Atendente/Gerente/Dono", "Navegador")

  System_Boundary(vps, "VPS (docker compose)") {
    Container(caddy, "caddy", "Caddy 2", "HTTPS/Let's Encrypt; único que publica 80/443")
    Container(app, "app", "Next.js 16 standalone (Node 22)", "UI + API REST v1 + Server Actions + middleware (proxy.ts)")
    Container(worker, "worker", "tsx + pg.Pool", "Agent-engine: fila job_queue, turnos, loops (drain/cron/watchdog/health/flywheel)")
    Container(scheduler, "scheduler", "crond + curl", "Dispara crons 1/min contra a API interna")
    Container(waha, "waha", "WAHA NOWEB", "Sessões WhatsApp, envio, webhooks")
    ContainerDb(redis, "redis", "Redis 7", "Efêmero: rate limit, debounce")
    Container(srh, "srh", "serverless-redis-http", "REST Upstash-compatível sobre o redis")
  }

  System_Ext(supabase, "Supabase", "Postgres (RLS) + Auth + Realtime + Storage")
  System_Ext(ia, "Provedores de IA", "Anthropic/OpenAI/Google")
  System_Ext(resend, "Resend", "E-mail")
  System_Ext(meta, "Meta Ads", "Conversions API")
  System_Ext(nuvem, "Nuvemshop", "E-commerce")

  Rel(lead, waha, "Mensagens", "WhatsApp")
  Rel(usuario, caddy, "HTTPS")
  Rel(caddy, app, "proxy reverso", "HTTP interno")
  Rel(waha, app, "webhook /api/v1/webhooks/waha", "HTTP HMAC")
  Rel(app, waha, "envia mensagens", "REST X-Api-Key")
  Rel(worker, waha, "envia (WahaChannelAdapter)", "REST")
  Rel(app, supabase, "queries (supabase-js, RLS)", "Postgres/REST")
  Rel(worker, supabase, "queries (pg.Pool)", "Postgres")
  Rel(app, srh, "rate limit/debounce", "REST")
  Rel(worker, srh, "debounce", "REST")
  Rel(srh, redis, "comandos", "Redis")
  Rel(scheduler, app, "crons (Bearer INTERNAL_SECRET)", "HTTP interno")
  Rel(app, ia, "workers legados / embeddings", "HTTP")
  Rel(worker, ia, "turnos do agente", "HTTP")
  Rel(app, resend, "e-mail", "REST")
  Rel(worker, meta, "conversão offline", "Graph API")
  Rel(nuvem, app, "webhook loja", "HTTP HMAC")
```

## Notas de topologia 🟢

- **Só o Caddy publica portas** (80/443); todo o resto vive na rede interna `internal`.
- **Tetos de memória medidos** (VPS 3,8 GB): app 768m, worker 512m, waha 1280m, scheduler — para o OOM killer não derrubar o CRM inteiro.
- **Healthcheck do app é TCP puro** (não `/api/v1/health`, que dá 503 se WAHA/Redis caem — derrubaria o Caddy junto).
- **Imagens publicadas com `:stable`** (última release), não `:latest` (topo da main). `build:` fica ao lado como escape.
- **Supabase é externo** ao compose (serviço gerenciado ou instância própria) — 🟡 o modo exato de provisão não está no compose de prod.
- **`docker-compose.traefik.yml`** adiciona labels de roteamento quando a VPS já tem proxy próprio (Hostinger/Coolify/Dokploy); esquecer o 2º `-f` deixa o domínio em 404 com container `healthy`.
