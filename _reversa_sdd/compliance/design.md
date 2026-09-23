# Compliance — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface (destaques)
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `audit` | `(entry)` | fire-and-forget (insert append-only) |
| `computeDueAt` | `(receivedAt, businessDays, holidays)` | `Date` (dias úteis) |
| `cascadeRedactContact` | `(args)` | resultado (RPC `fn_lgpd_cascade_redact_contact`) |
| `varrerRedacoesIncompletas` | `(db, teto)` | conserto por cron |
| `collectExportData` | `(args)` | `ExportPayload` |
| `drainStorageRedactionQueue` | `(opts)` | dreno idempotente |
| `perfilDaOrganizacao` | `(...)` | perfil legal do país |
| `ehPedidoDeOptOut` / `ehOptOutProvavel` | `(msg)` | `boolean` |
| `interpretarRetencao` | `(bruto, {chave, padrao, piso})` | dias + aviso |

## Fluxo Principal — Anonimização (cascata)
1. Enfileira a foto de perfil ANTES da cascata (upsert, caminho estável, falha fechada). 🟢
2. RPC `fn_lgpd_cascade_redact_contact` (transação: contacts irreversível, conversas, mensagens, atividades, leads; PII de `orders.payload` preservando valores; enfileira mídia; audit denso). 🟢
3. Passos 2-4 (`cascata.ts`) em lugar único (duas bocas escrevem a mesma redação): idempotência por `jaRedigida()`; passo 4 cancela a régua viva. 🟢
4. Cron `varrerRedacoesIncompletas`: detecta resíduo em bloco (leitura ≤5000, conserto ≤200 — números diferentes evitam starvation). 🟢

## Fluxo Principal — Export de acesso
`collectExportData` agrega todo dado pessoal; gate deriva a lista das duas pontas; campos obrigatórios fazem caminho novo não compilar. PDF (`pdf-renderer.tsx`) sem marca (nomeia controlador); PAdES stub degrada para não-assinado; e-mail via roteador (só sha256 no log). 🟢

## Fluxo Principal — Auditoria
`audit()` prefere admin client, cai no de sessão sem service role; resolve contexto de suporte (state OAuth ou cookie); `AUDIT_ACTIONS` array canônico deriva o filtro do painel. 🟢

## Dependências
- `lgpd` → RPCs security-definer, `@react-pdf/renderer`, `lib/email/roteador`, Supabase Storage. 🟢
- `audit` → `api_audit_log`, admin/session client, `lib/impersonate/support`. 🟢
- `legal` → `organizations.country`/`legal_name`, client de sessão (nunca service role em `/legal/*`). 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Audit fire-and-forget mas reportado | `audit/index.ts` | 🟢 |
| Export = o que se redige (gate deriva) | `lgpd/export-collector.ts` | 🟢 |
| PDF nomeia controlador, não marca | `lgpd/pdf-renderer.tsx` | 🟢 (ADR-0008) |
| País entra com citação revisada, sem fallback | `legal/perfil-do-pais.ts` | 🟢 |
| Opt-out por intenção (verbo + objeto de comunicação) | `opt-out/deteccao.ts` | 🟢 |
| Piso de retenção no SQL | `retencao/politica.ts` | 🟢 |

## Estado Interno
- `api_audit_log` (append-only), `lgpd_requests`, `storage_redaction_queue`, tabelas anonimizadas em cascata. 🟢

## Observabilidade
- Alarme de SLA D+5/D+10 (`sla-alarm.ts`) sem PII no Sentry; dedup fire-once-per-24h. 🟢

## Riscos e Lacunas
- 🔴 Funções SECURITY DEFINER (`fn_lgpd_cascade_redact_contact`, `fn_expurgar_auditoria_vencida`, `fn_podar_fila_de_jobs`) e policies RLS de `api_audit_log` — Data Master.
- 🟡 PAdES é stub (degrada para não-assinado mesmo com key; P12 pendente).
- 🟡 Só o perfil do Brasil está publicado (`paisesOferecidos()` devolve só BR).
- 🟡 Piso de retenção da captação vive só no TS (DELETE do admin client, sem função SQL).
