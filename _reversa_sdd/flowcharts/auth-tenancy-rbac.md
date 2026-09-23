# Fluxogramas — Auth, tenancy e RBAC

> Gerado pelo Arqueólogo (Reversa) — Unidade 8.
> Cobre `lib/auth/`, `lib/tenants/`, `lib/team/`, `lib/users/`, `lib/impersonate/`, `proxy.ts`.
> Nível de documentação `detalhado`: fluxograma por módulo + por função de lógica não-trivial.

---

## 1. Guard de borda — `proxy.ts` (middleware Next 16)

```mermaid
flowchart TD
  A[request] --> B[Injeta x-request-id + x-pathname]
  B --> C{isPublicPath?}
  C -- sim --> CX[return response: proxy não decide]
  C -- não --> D[createServerClient cookie sb-deskcomm-auth<br/>supabase.auth.getUser JWT validado]
  D --> E{user?}
  E -- não, /api/* --> EX[401 JSON unauthenticated]
  E -- não, UI --> EY[redirect /login?next=...]
  E -- sim --> F{pathname /app/*<br/>tem cookie impersonate?}
  F -- sim --> F1[verifyImpersonateCookieEdge<br/>HMAC + expiry, sem DB]
  F1 --> F2{válido?}
  F2 -- não --> F3[deleta cookie de apresentação<br/>banco segue autoritativo]
  F2 -- sim --> G
  F -- não --> G{/admin/* e não /admin/forbidden?}
  G -- sim --> G1[rpc fn_is_platform_admin]
  G1 --> G2{admin?}
  G2 -- não/erro --> G3[redirect /admin/forbidden]
  G2 -- sim --> H[return response]
  G -- não --> H
```

---

## 2. Sessão — `loadAuthUser` (`auth/server.ts`)

```mermaid
flowchart TD
  A[loadAuthUser cache/request] --> B[getUser: valida JWT no servidor<br/>NUNCA getSession]
  B --> C{error && não é<br/>AuthSessionMissingError?}
  C -- sim --> C1[logger.error: falha transitória<br/>rede/GoTrue/token ilegível]
  C -- não --> D
  C1 --> D{user?}
  D -- não --> DX[return null: falha FECHADA<br/>redireciona para login]
  D -- sim --> E[Promise.all:<br/>platform_admins + user_organizations<br/>ORDER BY accepted_at, organization_id]
  E --> F{erro em qualquer query?}
  F -- sim --> FX[THROW auth_permissions_unavailable<br/>FALHA ALTO: não rebaixa em silêncio]
  F -- não --> G[Monta memberships + combinarInterfaces<br/>empresa ∩ vínculo]
  G --> H[readSupportContext + metadados]
  H --> I[idioma = normalizarIdioma<br/>locale ?? support.locale ?? locale da org ativa]
  I --> J[return AuthUser]
```

---

## 3. Guard canônico de RBAC — `requireRole` (`auth/require-role.ts`)

```mermaid
flowchart TD
  A[requireRole min, opts] --> B[loadAuthUser]
  B --> C{user?}
  C -- não --> CX[401 unauthenticated]
  C -- sim --> D{support && status != active?}
  D -- sim --> DX[403 forbidden]
  D -- não --> E{opts.organizationId?}
  E -- sim --> E1[org da fonte confiável:<br/>support / membership / platformAdmin]
  E -- não --> E2[resolveActiveOrg]
  E1 --> F
  E2 --> F{org resolvida?}
  F -- não --> FX[403 forbidden_tenant]
  F -- sim --> G{allowPlatformAdmin<br/>&& is_platform_admin && !support?}
  G -- sim --> GX[OK: bypass do rank do tenant]
  G -- não --> H[rpc fn_user_role_in_org<br/>role EFETIVO do banco]
  H --> I{erro RPC?}
  I -- sim --> IX[500 internal_error]
  I -- não --> J[rank = ROLE_RANK effectiveRole]
  J --> K{rank >= min && mfaEmDivida?}
  K -- sim --> KX[audit authz.denied reason=mfa_required<br/>403 mfa_required]
  K -- não --> L{rank < min?}
  L -- sim --> LX[audit authz.denied required_role<br/>403 forbidden_role]
  L -- não --> M[OK: user, org com role efetivo]
```

> Ordem deliberada: o gate de MFA fica DEPOIS do rank e ANTES do sucesso — quem não tem papel leva 403
> por falta de papel, sem a resposta revelar o estado de MFA de quem nem chegaria lá.

---

## 4. MFA: as duas perguntas que não são a mesma

```mermaid
flowchart LR
  subgraph Cadastro["Preciso CADASTRAR? (POLÍTICA)"]
    A1[exigeCadastroDeMfa] --> A2{isPlatformAdmin<br/>&& plataformaExige?}
    A2 -- sim --> A3[obrigado]
    A2 -- não --> A4{role admin<br/>&& empresaExige?}
    A4 -- sim --> A3
    A4 -- não --> A5[não exige - default]
  end
  subgraph Sessao["Preciso PROVAR agora? (SESSÃO)"]
    B1[mfaEmDivida] --> B2{isMfaEnrolled?}
    B2 -- não --> B3[false: nada a provar]
    B2 -- sim --> B4{sessionAal != aal2?}
    B4 -- sim --> B5[true: em dívida]
    B4 -- não --> B3
  end
```

> As duas origens de política SOMAM, nunca se anulam. A sessão NÃO consulta a política: quem TEM fator
> prova sempre (senão o cadastro opcional viraria buraco).

---

## 5. Provisionamento externo — `provisionExternalTenant` (`auth/provision.ts`)

```mermaid
flowchart TD
  A[provisionExternalTenant] --> B[slug = slugDoProvisionamento hash]
  B --> C[reencontrarECompletar: busca org por slug]
  C --> D{achou?}
  D -- sim --> D1{marcador bate<br/>integration+external_id?}
  D1 -- não --> D2[ProvisionConflictError 409]
  D1 -- sim --> D3[garantirAdminDaOrganizacao<br/>completa vínculo faltante]
  D3 --> D4[return replay:true]
  D -- não --> E[ensureExternalOwnerUser]
  E --> E1{email já existe?}
  E1 -- sim --> E2[donoOrfaoDesteProvisionamento:<br/>marcador em app_metadata + sem vínculo vivo]
  E2 --> E3{órfã deste provisionamento?}
  E3 -- não --> E4[EmailJaTemContaError 409]
  E3 -- sim --> F
  E1 -- não --> F[cria conta ativa senha aleatória]
  F --> G[insert organizations settings.provisioning=marcador]
  G --> H{23505?}
  H -- sim --> H1[reencontrarECompletar corrida] --> D4
  H -- não --> I[insert user_organizations role=admin]
  I --> J[audit tenant.created_by_provisioning actor=null]
  J --> K[return replay:false]
```

---

## 6. Convite no signup — `decidirConviteDoSignup` (`auth/convite-no-signup.ts`) — PURA

```mermaid
flowchart TD
  A[decidirConviteDoSignup user] --> B{user_metadata.invite_token<br/>é string não-vazia?}
  B -- não --> BX[provisionar: caminho normal]
  B -- sim --> C[verifyInviteToken: HMAC + Zod + exp]
  C --> D{token válido?}
  D -- não --> DX[recusar: token_invalido<br/>FALHA FECHADA]
  D -- sim --> E{payload.email ==<br/>email confirmado pelo provedor?}
  E -- não --> EX[recusar: email_divergente]
  E -- sim --> F[convite: token + payload<br/>mandar para o aceite]
```

> ⚠️ `user_metadata` é gravável pelo próprio usuário — nada dali é autoridade. Quem manda é a assinatura
> HMAC + a comparação com o e-mail que o provedor confirmou. Sem a comparação, um token alheio vira porta
> de entrada.

---

## 7. Aceite de convite — `aplicarConvite` (`auth/aplicar-convite.ts`)

```mermaid
flowchart TD
  A[aplicarConvite user, payload] --> B[busca team_invites por id+org]
  B --> C{revoked_at preenchido?}
  C -- sim --> CX[invalid_or_expired]
  C -- não --> D[rpc fn_accept_team_invite<br/>org/papel/convidador SÓ do token]
  D --> E{erro?}
  E -- 42501 --> EX[invalid_or_expired: recusa da função]
  E -- outro --> EY[internal_error]
  E -- não --> F{resultado.changed?}
  F -- sim --> F1[audit member.accepted]
  F -- não --> G
  F1 --> G[update team_invites accepted_at<br/>idempotente: is accepted_at null + revoked_at null]
  G --> H[set cookie active_org]
  H --> I[ok: membershipId, mudou]
```

---

## 8. Emissão / rotação de API key — `rotateIntegrationApiKey` (`tenants/api-key.ts`)

```mermaid
flowchart TD
  A[rotateIntegrationApiKey] --> B[busca chave anterior<br/>.contains scopes integrationScope]
  B --> C{erro na busca?}
  C -- sim --> CX[THROW: falha FECHADA 500]
  C -- não --> D{há chave anterior viva?}
  D -- sim --> D1[update revoked_at<br/>.is revoked_at null + .select id]
  D1 --> D2[audit token.revoked por linha, actor=null]
  D --> E[gera dsk_prefix_secret<br/>SHA256 em token_hash]
  D2 --> E
  E --> F[insert api_tokens<br/>scopes mcp:read, mcp:write, role:agent, integrationScope]
  F --> G{erro?}
  G -- sim --> GX[THROW insert falhou]
  G -- não --> H[audit token.created actor=null]
  H --> I[return plaintext: mostrado uma vez]
```

> Sem `actor:ai_agent` de propósito: o parceiro é integração, não IA — a ausência faz `deriveActor`
> devolver `api_token`. Papel `agent` (menor privilégio) cobre 46 das 63 tools.
