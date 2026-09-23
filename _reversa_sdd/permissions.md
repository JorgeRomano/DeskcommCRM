# Permissões e Papéis (RBAC/ACL) — DeskcommCRM

> Gerado pelo Detetive (Reversa) — fase de Interpretação · nível **detalhado**
> Escala de confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA
> Fontes: `lib/auth/types.ts`, `lib/auth/require-role.ts`, `lib/mcp/*`, `code-analysis.md`, `data-dictionary.md`.

O DeskcommCRM tem **duas camadas de autorização que somam**: RBAC de aplicação (`requireRole`,
guards de rota) e **RLS no banco** (a mesma função `SECURITY DEFINER` `fn_user_org_ids()` /
`fn_user_role_in_org()`). A tela nunca é a única defesa; o banco também barra. Serviço com service
role bypassa RLS e filtra `organization_id` manualmente (RN-02).

---

## 1. Papéis (`Role`) e hierarquia

`ROLE_RANK` (`lib/auth/types.ts`) — rank crescente, comparado por `requireRole`:

| Papel | Rank | Rótulo pt-BR | Quem é | Confiança |
|---|---|---|---|---|
| `viewer` | 1 | Somente leitura | Observador, sem mutação | 🟢 |
| `agent` | 2 | Atendente | Humano que atende conversas | 🟢 |
| `ai_operator` | 3 | Assistente com autonomia de operação | **Agente publicado**, só no token efêmero | 🟢 |
| `manager` | 4 | Gerente | Configura a operação | 🟢 |
| `admin` | 5 | Administrador | Dono da organização | 🟢 |

**Regras estruturais 🟢:**
- **`PAPEIS_HUMANOS = [viewer, agent, manager, admin]`** — espelha o CHECK
  `user_organizations_role_check`. `ai_operator` **nunca** entra em `user_organizations`; nenhuma
  pessoa o recebe, e ele não aparece em seletor de time (RN-03).
- `ai_operator` senta **entre** `agent` e `manager` de propósito: dá ao agente capacidades que um
  atendente humano não tem (configurar operação, mexer na régua de retorno) sem dar esse poder a uma
  pessoa pela tela. Cumpre o invariante 4 da doutrina (nenhuma demanda sem próximo passo).
- `fn_role_at_least` no banco **não conhece** `ai_operator` (lê `user_organizations`) — e está
  correto: o agente não é usuário, a RLS segue intacta.

**`requireRole()` é o único gate de rota 🟢** (`lib/auth/require-role.ts`): decide 401/403, aplica o
gate de MFA e resolve o role **efetivo** do banco via `fn_user_role_in_org` (não confia no cliente).
`roleAtLeast()` é helper informativo (ex.: `podeEditar`) usado **depois** que `requireRole` já
liberou — nunca decide acesso sozinho.

---

## 2. Escopo de visualização (`VisibilityMode`) 🟢

Restringe **quais conversas** um `agent` vê (G4-01, spec 13 §3.5). `lib/auth/types.ts`.

| Modo | Vê | Default |
|---|---|---|
| `all` | Todas as conversas da org | — |
| `own_and_unassigned` | As próprias + sem dono | ✅ `DEFAULT_VISIBILITY_MODE` (G1-06a) |
| `own` | Só as próprias | — |

**Regra 🟢:** só restringe o papel `agent`. `viewer`/`manager`/`admin` seguem org-wide. **Não é
fonte de autorização** — quem garante o escopo é a RLS (`fn_can_view_conversation`); o
`visibility_mode` no `ActiveOrg` só ajuda a UI do inbox a decidir visões visíveis.

---

## 3. Matriz de capacidades por papel (aplicação)

> 🟢 onde o gate está no código (guard de rota/tool); 🟡 onde é inferido do rótulo/uso.

| Capacidade | viewer | agent | ai_operator | manager | admin | Evidência |
|---|:---:|:---:|:---:|:---:|:---:|---|
| Ler conversas/leads/contatos | ✅ (escopo) | ✅ (visibility) | ✅ | ✅ | ✅ | RLS + tools MCP r·agent 🟢 |
| Enviar mensagem WhatsApp | — | ✅ | ✅ | ✅ | ✅ | `crm_send_whatsapp_message` w·agent 🟢 |
| Mover lead de estágio (mesmo pipeline) | — | ✅ | ✅ | ✅ | ✅ | `crm_move_lead_stage` w·agent 🟢 |
| Criar/atualizar lead | — | ✅ | ✅ | ✅ | ✅ | `crm_create/update_lead` w·agent 🟢 |
| Atribuir conversa / gerir tags | — | ✅ | ✅ | ✅ | ✅ | `crm_assign_conversation`, `crm_manage_tags` w·agent 🟢 |
| Abrir/atualizar/fechar caso humano | — | ✅ | ✅ | ✅ | ✅ | `escalacao.ts` w·agent 🟢 |
| Retomar atendimento IA (`resume_ai`) | — | ✅ (só pessoa) | ❌ | ✅ | ✅ | recusa ator `ai_agent` 🟢 |
| Iniciar conversa fria (cold-start) | — | — | — | ✅ | ✅ | `crm_start_conversation_and_send` w·manager 🟢 |
| Agendar/reagendar/cancelar compromisso | — | — | ✅ | ✅ | ✅ | `agendamento.ts` w·ai_operator 🟢 |
| Agendar/cancelar follow-up, inscrever fluxo | — | — | ✅ | ✅ | ✅ | `retencao.ts` w·ai_operator 🟢 |
| Salvar memória da org | — | — | ✅ | ✅ | ✅ | `crm_save_org_memory` w·ai_operator 🟢 |
| Criar/editar/arquivar stage, webhook, toggles | — | — | — | ✅ | ✅ | `operacao.ts` w·manager 🟢 |
| Aprovar proposta do flywheel | — | 🔴 | ❌ (não se auto-aprova) | 🟡 | 🟡 | gate humano (clique) — quem clica é 🔴 |
| Anonimizar / atender pedido LGPD | — | — | ❌ | 🟡 | 🟡 | IA só lê `lgpd_requests`; quem executa é 🔴 |
| Gerir time (convidar/mudar papel) | — | — | — | 🟡 | ✅ | `team/*`; nível exato 🔴 |
| Configurar marca, integrações, orçamento IA | — | — | — | 🟡 | ✅ | settings/admin UI 🔴 |

> ❌ = explicitamente negado no código. 🔴 = papel mínimo não confirmado nesta passagem (validação
> humana). A coluna `ai_operator` reflete o **agente**, não uma pessoa.

**Regras de negação explícita 🟢:**
- `crm_resume_ai_attendance` recusa ator `ai_agent` (só pessoa reativa a IA).
- A IA **não** cria regra de automação, não muda papel, não apaga (archive nunca apaga — FK
  RESTRICT), não se auto-aprova propostas e não anonimiza.
- `crm_search_contacts`/`crm_get_contact`: **CPF nunca em plaintext**; `crm_propose_contact_field`
  **não grava** — cria proposta para humano confirmar.
- `_users.resolveUserNames` expõe só `full_name` (LGPD).

---

## 4. Servidor MCP — autorização das tools 🟢

O CRM inteiro é exposto por MCP com RBAC próprio. Fonte: `lib/mcp/*`, `code-analysis.md` (2.6/2.7).

**Autenticação:** Bearer `dsk_<prefix>_<secret>` contra `api_tokens` (hash SHA256; checa
`revoked_at`/`expires_at`). Org sempre de `ctx`, nunca do arg. `createAdminClient()` (service role).

**Scopes convencionais (sem migration):** `role:<r>` (default `agent`), `actor:ai_agent`,
`agent_run:<uuid>`, `mcp:read`, `mcp:write`. `deriveActor` → `ai_agent` ou `api_token`, **nunca
`user`**.

**Pipeline por tool:** higienizar UUID de aterro → montar `McpContext` → `ensureScope` +
`ensureRole` (via `ROLE_RANK`) → handler → `motivoDoVazio` → auditoria.

**Códigos de erro MCP:** -32001/401 (auth), -32002/403 (role), -32603/500.

**Categorias:** `read` (exige `mcp:read`), `write` (`mcp:write`), `handoff`. Cada tool declara
`requiresRole`. Catálogo por domínio (resumo — detalhe em `code-analysis.md` 2.7):

| Domínio | Papel mínimo predominante | Exceções |
|---|---|---|
| contacts, conversations, leads (leitura) | agent (read) | — |
| messages, governance, escalacao (escrita) | agent (write) | `resume_ai` só pessoa |
| start-conversation (cold-start) | **manager** | apenasHumano |
| agendamento, retencao (escrita), evolucao (save memory) | **ai_operator** | — |
| operacao (create/update/archive) | **manager** | leituras em agent |
| privacidade | agent (só leitura) | IA não anonimiza |

**Redação de auditoria 🟢:** `ARGS_REDACT_KEYS = {authorization, api_key, token, password, cpf}`;
strings >500 chars truncadas.

**"Recusa para o modelo" 🟢:** recusa por papel vira texto para o modelo **sem vazar** "agent"/
"role"/"permissão"; distingue `apenasHumano` (deliberado) de restrição acidental. Não afrouxa o
`ensureRole`.

---

## 5. Superfícies HTTP e seus guards 🟢

Cada superfície tem guard **próprio** — nunca o cookie de sessão fora da UI. Fonte: AGENTS.md,
`code-analysis.md` (Unidade 14), `data-dictionary.md`.

| Superfície | Guard | Fail-mode |
|---|---|---|
| UI do tenant (`app/app/`), Server Actions | cookie de sessão (`sb-deskcomm-auth`, SameSite=strict) + `requireRole` | 401/403 |
| UI de plataforma (`app/admin/`) | `requirePlatformAdmin` (+ gate MFA) | 403 |
| `app/api/v1/cron/` | Bearer `INTERNAL_CRON_SECRET` | fail-closed |
| `app/api/internal/` | `x-internal-secret` | fail-closed |
| `app/api/mcp/` | Bearer `dsk_...` contra `api_tokens` | 401/403 |
| `app/api/v1/webhooks/` | HMAC (`timingSafeEqual`) + path token | fail-closed |
| parte de `app/api/v1/` | `lib/api/auth-dual.ts` (cookie OU bearer) | 401 |

**Regras 🟢:**
- Rota que usa `auth-dual` **também** precisa de entrada em `lib/auth/public-paths.ts`, senão o
  `proxy.ts` devolve 401 antes do handler.
- `proxy.ts` (middleware de borda, Next 16) autentica a sessão e injeta `X-Request-Id`/`x-pathname`
  antes da rota.
- Sempre `getUser()` no server; **nunca `getSession()`** (confia no cookie sem revalidar).
- API key/token só em header, nunca em query string. Plaintext do bearer mostrado uma vez; no banco
  só hash SHA256.
- A guarda de id vem **depois** da autorização, não antes (commit `a4b4bbf05`).

---

## 6. Platform admin, impersonation e MFA 🟢

**Platform admin** (`requirePlatformAdmin` → `PlatformAdminInfo {user_id, scope, mfa_required}`) —
o dono do servidor self-host, escopo acima das organizações. UI em `app/admin/`.

**Impersonation / Suporte** (`impersonate/*`):
- `ImpersonatePayload {sessionId?, tenantId, platformAdminId, exp}`; cookie `deskcomm-impersonate`
  (HttpOnly, Secure, SameSite=Lax), TTL 3600s (1h), secret `IMPERSONATE_COOKIE_SECRET` (≥32 chars).
- `SupportContext.access_mode ∈ {full, support_readonly}`; `status ∈ {active, expired, revoked}`.
- Auditoria marca `actingAsPlatformAdmin` / `bypassedRls`.

**Política de MFA (`PoliticaDeMfa`, `lib/auth/politica-mfa.ts`) 🟢** — **duas políticas que somam**:
plataforma (`plataformaExige: bool|null`) e organização (`empresaExige: bool`). TOTP, opcional,
ligado por quem administra. `issuer` do MFA usa `marcaDaSaida()` (marca própria fora do DOM).
Recovery codes: 10 códigos de 8 chars (alfabeto sem ambiguidade), guardados como `sha256`.

---

## 7. Convite de time e cadastro 🟢

- **Convite** (`InvitePayload`, `auth/invite-token.ts`): HMAC assinado, TTL `86400s` (24h), secret
  `INVITE_TOKEN_SECRET → INTERNAL_SECRET → "dev-fallback"` (fallback inalcançável em produção). O
  convite carrega o `role` a conceder. `StatusConvite` (pendente/aceito/expirado/revogado) é
  **derivado** de timestamps, nunca coluna.
- **Modo de cadastro** (`ModoDeCadastro`): `aberto` (default) ou `so_convite`. Banco
  (`platform_settings.signup_mode`) manda; `.env SIGNUP_MODE` é só semente/piso.

---

## 8. Rate limiting de autenticação 🟢

`lib/auth/rate-limit.ts` — por IP **e** por identificador hasheado (`AUTH_LIMITS`):

| Fluxo | Limite |
|---|---|
| login | IP 60, id 5, janela 300s |
| signup | IP 20, janela 3600s |
| reset (recuperação de senha) | IP 30, id 3, janela 3600s |
| invite_accept | IP 60, janela 3600s |
| org_recovery | IP 5, id 3, janela 3600s |

**Limitação conhecida (de AGENTS.md/threat-model):** crons e MCP seguem **sem** rate limit HTTP; o
webhook de captação e o dispatcher de IA usam `checkRateLimit`.

---

## 9. Lacunas 🔴

- 🔴 **Papel mínimo exato** para: gerir time (convidar/mudar papel), configurar marca/integrações/
  orçamento de IA, aprovar proposta do flywheel, executar pedido LGPD (anonimizar). O código
  confirma que a **IA** não faz nada disso; qual papel **humano** faz (manager vs admin) precisa de
  leitura direta dos guards em `app/actions/*` e `app/admin/*`.
- 🔴 **Matriz RLS por tabela** (quais políticas `SELECT/INSERT/UPDATE/DELETE` cada tabela expõe por
  papel) é trabalho do **Data Master** — aqui documentamos a camada de aplicação e os helpers RLS,
  não cada política. Ex. confirmado: `calendar_connections_dono_ou_manager_read` esconde a conexão
  de agent/viewer (leituras via RPC).
- 🔴 **Escopos da API key além dos default** (`["mcp:read","mcp:write","role:agent",<integrationScope>]`)
  — quais integrações elevam o `role:<r>` do token e até onde.
