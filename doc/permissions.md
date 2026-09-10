# Matriz de Permissões (RBAC) — DeskcommCRM

> Gerado pelo **Detetive** (Reversa) em 2026-09-10 · Nível: **Detalhado**
> 🟢 CONFIRMADO no código (`lib/auth/types.ts`, `lib/auth/require-role.ts`, `lib/mcp/auth.ts`) · 🟡 INFERIDO · 🔴 LACUNA

---

## 1. Papéis e ranking 🟢

Fonte: `lib/auth/types.ts`. Ranking numérico determina a hierarquia (`requireRole(min)` compara rank).

| Papel | Rank | Rótulo (pt-BR) | Natureza |
|---|---|---|---|
| `viewer` | 1 | Somente leitura | humano |
| `agent` | 2 | Atendente | humano |
| `ai_operator` | 3 | Assistente com autonomia de operação | **máquina** (token efêmero) |
| `manager` | 4 | Gerente | humano |
| `admin` | 5 | Administrador | humano |
| `platform_admin` | — | (transversal) | **cross-tenant**, dono da instalação |

**Regras estruturais:**
- `PAPEIS_HUMANOS` = `[viewer, agent, manager, admin]` — espelha o CHECK `user_organizations_role_check`. `ai_operator` **nunca** aparece em `user_organizations` (nenhuma pessoa o recebe). 🟢
- `ai_operator` senta entre `agent` e `manager`: descreve a faixa que o agente publicado precisa (configurar operação, mexer na régua de retorno) e que um atendente humano não tem. 🟢
- `fn_user_role_in_org` (banco, SECURITY DEFINER) NÃO conhece `ai_operator` — ela lê `user_organizations`. A RLS segue intacta. 🟢
- `platform_admin` é o único papel cross-tenant (T-04); bypassa o rank do tenant quando `allowPlatformAdmin` (e não está em sessão de suporte). 🟢

---

## 2. Como a autorização é decidida 🟢

`requireRole(min, opts)` (`lib/auth/require-role.ts`) — gate canônico, obrigatório em toda rota `/api/v1`:

1. `loadAuthUser()` valida JWT (`getUser`, nunca `getSession`). Sem user → **401**.
2. Sessão de suporte inativa → **403**.
3. Resolve org ativa (cookie validado / `opts.organizationId` de fonte confiável). Sem org → **403 forbidden_tenant**.
4. `platform_admin` + `allowPlatformAdmin` + sem suporte → sucesso (bypass).
5. **Role efetivo do BANCO** (`fn_user_role_in_org`) — não do snapshot do cookie.
6. **Gate de MFA de sessão**: se rank suficiente E `mfaEmDivida` (papel exige + fator cadastrado + sessão aal1) → **403 mfa_required**.
7. `rank < min` → **403 forbidden_role** + audit `authz.denied`.

**Anti-padrão proibido:** comparar `ROLE_RANK` direto em rota ("matriz advisória"). `roleAtLeast` só computa campos informativos após `requireRole` já ter autorizado.

---

## 3. Matriz de permissões por superfície 🟡

> Rank mínimo observado nos handlers lidos e inferido do catálogo (`docs/business-rules`, spec 13).
> Superfícies não lidas em profundidade marcadas 🟡; a RLS do banco é a garantia final.

| Capacidade | Papel mínimo | Fonte |
|---|---|---|
| Ler equipe (`GET /api/v1/team`) | `manager` | `app/api/v1/team/route.ts:39` 🟢 |
| Ler radar de risco | `viewer`/`agent` | `app/api/v1/leads/at-risk` 🟢 |
| Ler leads | `viewer` | 🟡 |
| Criar/editar/mover lead | `agent` | 🟡 (rotas de leads usam `requireRole("agent")`) |
| Editar funil/etapas | `manager` | 🟡 |
| Configurar agente / publicar versão | `admin` | 🟡 |
| Criar token de API / convidar membro / LGPD anonymize | `admin` | 🟢 (citado como gateado por `requireRole("admin")`) |
| Anti-ban / pacing por canal | `manager`+ | 🟡 |
| Impersonar tenant | `platform_admin` | 🟢 (`lib/impersonate/`) |
| Superfície `/admin/*` | `platform_admin` | 🟢 (`proxy.ts` + `requirePlatformAdmin`) |

> 🔴 **LACUNA:** a matriz completa por rota (169 handlers) não foi enumerada. O padrão canônico é
> `requireRole(<min>, {resource})` no início de cada handler; a lista exata deve sair de uma
> varredura `grep -rn 'requireRole(' app/api` (recomendado para o Redator/Arquiteto).

---

## 4. Escopo de visualização de conversas (G4-01) 🟢

Além do papel, o `agent` tem escopo de visibilidade (`VisibilityMode` em `lib/auth/types.ts`):

| Modo | Quem vê |
|---|---|
| `all` | todas as conversas da org |
| `own_and_unassigned` | próprias + não atribuídas (**default**, G1-06a) |
| `own` | só as próprias |

Só restringe `agent`; `viewer/manager/admin` seguem org-wide. **Não é fonte de autorização** — a RLS (`fn_can_view_conversation`) é quem garante o escopo. 🟢

---

## 5. Autorização não-cookie 🟢

### MCP (Bearer token) — `lib/mcp/auth.ts`
- Token `dsk_<prefix>_<secret>` → SHA256 contra `api_tokens.token_hash`.
- Atributos codificados em `scopes` (sem migration): `role:<role>` (default `agent`), `actor:ai_agent` (default `user`), `agent_run:<uuid>`, `mcp:read`, `mcp:write`.
- `ensureRole` (rank) + `ensureScope` (mcp:read/write). Erros JSON-RPC: `-32001` (401), `-32002` (403), `-32603` (500).

### Sessão de suporte (impersonation) — `lib/impersonate/support.ts`
- `access_mode`: `full` (age como admin) ou `support_readonly` (age como viewer).
- `requireSupportWrite()` é guarda de EFEITO antes dos clients service role (não substitui RBAC/MFA).
- Banco é autoritativo: sessão expirada/revogada bloqueia o app mesmo com cookie válido.

### Convite (stateless) — `lib/auth/invite-token.ts`
- HMAC-SHA256; `role` restrito a `viewer|agent|manager|admin` (Zod). TTL 24h.

### Superfícies públicas (auth dentro da rota) — `lib/auth/public-paths.ts`
- `/api/v1/webhooks/`, `/api/v1/cron/`, `/api/internal/`, `/api/mcp`, callbacks OAuth (state HMAC), `/api/v1/system/*` (Bearer `INTERNAL_SECRET`/`INTERNAL_CRON_SECRET`).
- ⚠️ Adicionar path aqui remove a checagem de auth de borda — só com guard próprio na rota.

---

## 6. MFA como política 🟢

- `mfaEmDivida` (papel exige MFA + fator cadastrado + sessão `aal1`) → gate de sessão em `requireRole`.
- Exigência vem da política (`platform_admins.mfa_required`, `organizations.settings.security.mfa_required`), padrão **não exige** — o `install.sh` cria o dono como platform admin e não pode trancá-lo antes de usar o produto.

---

## 7. Lacunas 🔴

- 🔴 SQL de `fn_user_role_in_org`, `fn_can_view_conversation`, `fn_is_platform_admin` e as policies RLS não foram lidos (só call sites) — Data Master.
- 🔴 Matriz rota-a-rota completa (169 handlers) não enumerada.
- 🟡 Papéis mínimos de várias capacidades inferidos do catálogo, não confirmados handler a handler.
