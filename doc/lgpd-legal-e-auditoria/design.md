# LGPD, Legal e Auditoria — Design Técnico

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Interface 🟢

| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `processLgpdExport` | `(event: EventRow)` | `HandlerResult` |
| `processLgpdRedact` | `(event: EventRow)` | `HandlerResult` |
| `cascadeRedactContact` | `({organizationId, contactId, requestId})` | `CascadeResult {alreadyAnonymized, counts, mediaPaths}` |
| `renderLgpdPdf` | `(data: ExportPayload, options)` | `Promise<Buffer>` |
| `createLgpdRequest` | `(input)` | `{id, due_at}` (dias úteis BR) |
| `audit` | `(entry: AuditEntry)` | `Promise<void>` (fire-and-forget) |
| `resolverOperador` | `()` | `Operador` |
| `resolverMarca` | `(camadas, regua)` | `MarcaResolvida` |
| `marcaDaSaida` | `(organizationId: string\|null)` | `MarcaDeSaida` |

## Fluxo Principal — export LGPD 🟢

1. Evento `lgpd.data_request_received`; guarda tipo/attempts (cap 3).
2. `collectExportData` (agregador multi-tabela PII-safe).
3. `renderLgpdPdf` (controlador `legal_name` + DPO, sem marca; PAdES stub sem chave).
4. Upload PDF+JSON no bucket `lgpd-exports`; signed URL (72h).
5. Email via Resend com `marcaDaSaida(orgId)`; sem email → `pending_review`.
6. `completed` + emite `export_generated`/`export_delivered` + audit.

## Fluxo Principal — redação 🟢

- **Contato:** enfileira avatar em `storage_redaction_queue` (ANTES, fail-closed) → RPC `fn_lgpd_cascade_redact_contact` (TX única; muta contacts/conversations/messages/leads/orders) → `redact_applied`.
- **Tenant:** lotes de 100 (`is_anonymized=false`), checkpoint em `request_payload.progress` → `organizations.status='redacted'`.

## Fluxo Principal — auditoria 🟢

`audit(entry)`: escolhe admin client (service role) se configurado; enriquece com support session; try/catch global (`reportAuditFailure` → Sentry). Nunca propaga.

## Fluxo Principal — marca 🟢

`resolverMarca` junta camadas org→instalação(banco)→env→padrão (precedência por campo; cor inválida numa camada de cima não apaga a de baixo). `marcaDaSaida` para email/MFA (tema claro, accent+contraste). Envelope de cor grava só `{semente_hex, papel}`.

## Fluxos Alternativos 🟢

- **Redação sem footprint local:** `pending_review` + audit `redact_no_local_footprint` (L-03).
- **Já anonimizado:** `redact_skipped_already_anonymized`.
- **Export sem chave PAdES:** PDF com aviso `unsignedWarning`, não lança.
- **Marca: erro de leitura:** degrada para padrão do produto (email de LGPD tem SLA legal).

## Dependências 🟢

- `event-log` (handlers lgpdExport/lgpdRedact; emite eventos), `audit`, `email` (Resend), `branding`, `supabase`/Storage.
- `legal` usa client de sessão (não service role — rota pública).

## Decisões de Design Identificadas 🟢

| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| PDF nomeia controlador, não marca | `pdf-renderer.tsx` | 🟢 |
| Avatar enfileirado antes de zerar ponteiro | `redact-cascade.ts` | 🟢 |
| Audit é fonte única de AUDIT_ACTIONS | `actions.ts` + teste | 🟢 |
| Resolvedor de marca nunca lança | `resolve.ts`, `saida.ts` | 🟢 |
| `.env` é semente/piso; banco é fonte | `resolve.ts` (`camadaDaInstalacao`) | 🟢 |

## Estado Interno 🟢

`lgpd_requests` (status/scope/due_at/progress/result), `storage_redaction_queue`, `api_audit_log` (append-only), `organizations` (legal_name/status/redacted_at/settings.branding), `platform_branding`.

## Observabilidade 🟢

- Eventos LGPD (`export_generated/delivered`, `redact_applied`, `tenant_redacted`).
- Audit denso na TX de cascata (`lgpd.redact_executed`).
- `MotivoDaMarca` (forma, nunca hex) no logger.

## Riscos e Lacunas

- 🔴 SQL de `fn_lgpd_cascade_redact_contact` e `fn_expurgar_auditoria_vencida` (retenção) não lidos.
- 🟡 `computeDueAt`/`lib/lgpd/sla.ts` (dias úteis exatos) não lido.
- 🟡 `lib/branding/{rampa,contraste,regua-do-produto}` referenciados, não aprofundados.
