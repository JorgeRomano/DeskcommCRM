# Fluxogramas — Superfície HTTP

> Gerado pelo Arqueólogo (Reversa) — Unidade 14.
> Cobre `proxy.ts`, `app/api/**` (route handlers) e Server Actions `app/actions/`.

---

## 1. Middleware de borda — `proxy` (`proxy.ts`)

```mermaid
flowchart TD
  A[request] --> B[X-Request-Id: lê ou gera UUID]
  B --> C[x-pathname na resposta e na request]
  C --> D{isPublicPath?}
  D -- sim --> DX[return response — proxy não decide]
  D -- não --> E[createServerClient cookie sb-deskcomm-auth<br/>sameSite strict, httpOnly, secure]
  E --> F[supabase.auth.getUser<br/>NUNCA getSession]
  F --> G{user existe?}
  G -- não, rota /api/ --> G1[JSON 401 unauthenticated + X-Request-Id]
  G -- não, rota UI --> G2[redirect /login?next=...]
  G -- sim --> H{pathname /app*?}
  H -- sim --> H1[verifyImpersonateCookieEdge<br/>HMAC+expiração; falha apaga cookie]
  H --> I{/admin/*?}
  I -- sim --> I1[rpc fn_is_platform_admin<br/>erro/false → /admin/forbidden]
  I -- não --> J[return response]
  H1 --> I
```

---

## 2. Receita canônica de route handler (6 passos)

```mermaid
flowchart TD
  A[request autenticada de tenant] --> B[1. Zod valida TODO input externo<br/>body, query, path]
  B --> C[2. Guard: requireRole /<br/>requirePlatformAdmin / secret / HMAC]
  C --> D[3. organization_id de FONTE CONFIÁVEL<br/>cookie/JWT/webhook secret/path token<br/>NUNCA do body]
  D --> E[4. Query: RLS pelo client de sessão<br/>OU filtro manual de org com service role]
  E --> F[5. audit fire-and-forget se mutação]
  F --> G[6. ok / fail — nunca Response na mão<br/>nem throw cru na borda]
```

---

## 3. As superfícies não-cookie e seus guards

```mermaid
flowchart LR
  A[Requisição HTTP] --> B{superfície}
  B -->|/api/v1/ cookie| C[requireRole sessão]
  B -->|/api/v1/cron/| D[Bearer INTERNAL_CRON_SECRET<br/>fail-closed]
  B -->|/api/internal/| E[x-internal-secret timingSafeEq]
  B -->|/api/mcp| F[Bearer dsk_ contra api_tokens<br/>erro → JSON-RPC 2.0]
  B -->|/api/v1/webhooks/| G[HMAC + path token<br/>timingSafeEqual]
  B -->|/api/v1/ dual| H[resolveAuthDual:<br/>cookie OU bearer]
  C --> I[org da sessão]
  D --> J[org: scan cross-tenant<br/>audit bypassedRls]
  E --> J
  F --> K[org do token]
  G --> L[org do webhook_sources<br/>por path_token]
  H --> M[org da sessão OU do token<br/>nunca do body]
```

---

## 4. Webhook de captação — `webhooks/in/[token]` (HMAC + idempotência)

```mermaid
flowchart TD
  A[POST /webhooks/in/token] --> B{token < 8 chars?}
  B -- sim --> BX[404]
  B -- não --> C[checkRateLimit por token 60/min]
  C --> D{estourou?}
  D -- sim --> DX[429 + Retry-After]
  D -- não --> E[busca webhook_sources<br/>.eq path_token]
  E --> F{fonte ativa?}
  F -- não --> FX[404]
  F -- sim --> G[verifyInboundSignature<br/>HMAC timingSafeEqual]
  G --> H{secret decifra?}
  H -- não, hmacSkipped --> I[PULA validação<br/>disponibilidade > defesa opcional]
  H -- sim --> J{assinatura bate?}
  J -- não --> JX[audit invalid_signature + 401]
  J -- sim --> K
  I --> K[dedup por external_id<br/>uniq + catch 23505]
  K --> L[audit lead_received]
  L --> M[ok lead_id ou 303 redirect]
```

---

## 5. Server Action — padrão (`app/actions/settings/updateProfile.ts`)

```mermaid
flowchart TD
  A[Server Action 'use server'] --> B[Zod safeParse input]
  B --> C{válido?}
  C -- não --> CX[ok:false, error:validation_failed, details]
  C -- sim --> D[loadAuthUser usa getUser]
  D --> E{autenticado?}
  E -- não --> EX[ok:false, error:unauthenticated]
  E -- sim --> F[lê x-request-id/ip/user-agent de headers]
  F --> G[muta via supabase]
  G --> H[audit action:profile.updated]
  H --> I[emit_event best-effort org-scoped]
  I --> J[revalidatePath]
  J --> K[devolve ok:true — objeto tipado<br/>não NextResponse]
```
