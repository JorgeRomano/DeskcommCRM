# C4 Nível 1 — Contexto — DeskcommCRM

> Gerado pelo **Arquiteto** (Reversa) · 🟢 CONFIRMADO

```mermaid
C4Context
  title Contexto do Sistema — DeskcommCRM

  Person(dono, "Dono do negócio / Operador da VPS", "Instala e opera o CRM self-host; é o controlador LGPD")
  Person(atendente, "Atendente (agent)", "Atende conversas e negócios no CRM")
  Person(gerente, "Gerente/Admin", "Configura funis, agentes, equipe, marca")
  Person(platform, "Platform admin", "Dono da instalação; único papel cross-tenant")
  Person(lead, "Lead / Cliente", "Conversa pelo WhatsApp")

  System(deskcomm, "DeskcommCRM", "CRM de vendas multi-tenant com agentes de IA nativos, WhatsApp-first, self-hosted, LGPD by-design")

  System_Ext(waha, "WAHA", "Gateway WhatsApp (engine NOWEB)")
  System_Ext(supabase, "Supabase", "Postgres + Auth + Realtime + Storage")
  System_Ext(redis, "Upstash Redis / SRH", "Rate limit, debounce")
  System_Ext(ia, "Provedores de IA", "Anthropic / OpenAI / Google (Vercel AI SDK)")
  System_Ext(resend, "Resend", "E-mail transacional")
  System_Ext(nuvem, "Nuvemshop", "E-commerce (OAuth + webhooks)")
  System_Ext(meta, "Meta Ads", "Conversions API (conversão offline)")
  System_Ext(sentry, "Sentry", "Observabilidade de erros")

  Rel(lead, waha, "Mensagens WhatsApp")
  Rel(waha, deskcomm, "Webhook message.any (HMAC-SHA512)")
  Rel(deskcomm, waha, "Envia mensagens (REST X-Api-Key)")
  Rel(atendente, deskcomm, "Usa inbox/kanban", "HTTPS")
  Rel(gerente, deskcomm, "Configura", "HTTPS")
  Rel(dono, deskcomm, "Opera / instala", "HTTPS")
  Rel(platform, deskcomm, "Administra plataforma / impersona", "HTTPS")
  Rel(deskcomm, supabase, "Dados/Auth/Realtime/Storage", "Postgres/REST")
  Rel(deskcomm, redis, "Rate limit/debounce", "REST")
  Rel(deskcomm, ia, "Turnos do agente / classificação / embeddings", "HTTP")
  Rel(deskcomm, resend, "E-mail (LGPD, convites)", "REST")
  Rel(nuvem, deskcomm, "Webhook pedidos/clientes (HMAC-SHA256)", "HTTPS")
  Rel(deskcomm, nuvem, "OAuth / API de loja", "REST")
  Rel(deskcomm, meta, "Conversão Purchase (ctwa_clid)", "Graph API")
  Rel(deskcomm, sentry, "Erros (PII higienizada)", "HTTP")
```

## Personas 🟢

- **Lead/Cliente:** conversa pelo WhatsApp; nunca acessa o CRM.
- **Atendente (`agent`):** trabalha inbox e kanban; escopo de visibilidade configurável.
- **Gerente/Admin (`manager`/`admin`):** configura funis, agentes, equipe, marca, anti-ban.
- **Dono/Operador:** instala na VPS; é o **controlador LGPD** (`organizations.legal_name`).
- **Platform admin:** único papel cross-tenant; pode impersonar tenants (sessão de suporte).

## Sistemas externos 🟢

Ver tabela de integrações em `architecture.md` §5. Todos os webhooks entrantes verificam HMAC (fail-closed). Google Ads é conhecido mas sem transporte (declarado `null`).
