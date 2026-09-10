# Fluxograma — Autorização (RBAC) e LGPD

> Gerado pelo **Arqueólogo** · Módulos: `lib/auth/require-role.ts`, `proxy.ts`, `lib/lgpd/` 🟢

## `requireRole` — gate canônico de rota

```mermaid
flowchart TD
  A[requireRole min, opts] --> B[loadAuthUser via getUser JWT]
  B -- null --> Z1[401 unauthenticated]
  B --> C{support ativo?}
  C -- inativo --> Z2[403 forbidden]
  C --> D[resolve org: opts.organizationId ou cookie active_org]
  D -- sem org --> Z3[403 forbidden_tenant]
  D --> E{allowPlatformAdmin && is_platform_admin && !support?}
  E -- sim --> OK1[sucesso — bypass]
  E -- não --> F[fn_user_role_in_org — role efetivo do BANCO]
  F --> G{rank >= min?}
  G -- sim --> H{mfaEmDivida? papel exige + fator + sessão aal1}
  H -- sim --> Z4[403 mfa_required + audit]
  H -- não --> OK2[sucesso — role efetivo]
  G -- não --> Z5[403 forbidden_role + audit authz.denied]
```

## Auth de borda (`proxy.ts`)

```mermaid
flowchart TD
  A[request] --> B[injeta X-Request-Id / x-pathname]
  B --> C{isPublicPath?}
  C -- sim --> OK[segue — auth mora dentro da rota]
  C -- não --> D[getUser valida JWT]
  D -- sem user + /api/ --> Z1[401 JSON envelope]
  D -- sem user + UI --> Z2[redirect /login?next=]
  D --> E{/app/*?}
  E -- sim --> E1[valida impersonation cookie edge HMAC]
  D --> F{/admin/*?}
  F -- sim --> F1[fn_is_platform_admin senão /admin/forbidden]
```

## LGPD — export (Art. 18 II)

```mermaid
flowchart TD
  A[lgpd.data_request_received] --> B{request tipo data_request?}
  B -- não --> Z[skip]
  B -- sim --> C{attempts < 3?}
  C -- não --> Z2[failed + audit export_failed]
  C --> D[collectExportData multi-tabela PII-safe]
  D --> E[renderLgpdPdf — nomeia CONTROLADOR legal_name + DPO, SEM marca]
  E --> F[signPdfPades — stub se sem chave]
  F --> G[upload PDF+JSON no bucket lgpd-exports]
  G --> H[signed URL 72h]
  H --> I{tem email?}
  I -- não --> J[pending_review]
  I -- sim --> K[sendExportEmail com marcaDaSaida da org]
  K --> L[completed + emit export_generated/delivered + audit]
```

## LGPD — redação (cascata)

```mermaid
flowchart TD
  A[lgpd.redact_received] --> B{scope}
  B -- contact --> C[resolve contact_id ou por external_id]
  C -- sem contato --> C1[pending_review + audit no_local_footprint]
  C -- achou --> D[cascadeRedactContact]
  D --> D1[ENFILEIRA avatar em storage_redaction_queue ANTES]
  D1 --> D2[RPC fn_lgpd_cascade_redact_contact TX única]
  D2 --> D3[muta contacts/conversations/messages/leads/orders]
  D3 --> E[completed + emit redact_applied]
  B -- tenant --> F[lotes de 100 contatos is_anonymized=false]
  F --> G[organizations.status=redacted + redacted_at]
```

**Invariante crítica:** o avatar é enfileirado ANTES de zerar o ponteiro — senão a imagem fica órfã no bucket ("mesmo que não ter anonimizado"). Falha ao enfileirar aborta a cascata (fail-closed).
