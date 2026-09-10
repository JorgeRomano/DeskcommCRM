# LGPD, Legal e Auditoria — Tarefas de Implementação

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Pré-requisitos
- [ ] `event_log` + handlers lgpdExport/lgpdRedact
- [ ] Tabelas `lgpd_requests`, `storage_redaction_queue`, `api_audit_log`, `platform_branding`
- [ ] RPC `fn_lgpd_cascade_redact_contact` (SECURITY DEFINER)
- [ ] Buckets Storage `lgpd-exports` e `whatsapp-media`
- [ ] Resend configurado (email); `LGPD_SIGNING_KEY`/`LGPD_DPO_EMAIL` (opcionais)

## Tarefas

- [ ] T-01, Implementar export LGPD (coleta PII-safe → PDF → Storage → email)
  - Origem no legado: `workers/lgpd-export-worker.ts`, `lib/lgpd/export-collector.ts`
  - Critério de pronto: PDF nomeia controlador; sem email → pending_review; cap 3 attempts
  - Confiança: 🟢

- [ ] T-02, Implementar PDF de LGPD (controlador + DPO, sem marca)
  - Origem no legado: `lib/lgpd/pdf-renderer.tsx`
  - Critério de pronto: rodapé com `legal_name`/DPO; PAdES stub sem chave
  - Confiança: 🟢

- [ ] T-03, Implementar redação por contato (cascata, avatar antes)
  - Origem no legado: `lib/lgpd/redact-cascade.ts`, `workers/lgpd-redact-worker.ts`
  - Critério de pronto: enfileira avatar ANTES; RPC em TX única; fail-closed
  - Confiança: 🟢

- [ ] T-04, Implementar redação por tenant (lotes + status redacted)
  - Origem no legado: `workers/lgpd-redact-worker.ts`
  - Critério de pronto: 100/lote; checkpoint em progress; org.status='redacted'
  - Confiança: 🟢

- [ ] T-05, Implementar `createLgpdRequest` + SLA em dias úteis BR
  - Origem no legado: `lib/lgpd/repository.ts`, `lib/lgpd/sla.ts` 🟡
  - Critério de pronto: due_at correto (export D+7, redact D+15) 🟡
  - Confiança: 🟡

- [ ] T-06, Implementar auditoria append-only fire-and-forget
  - Origem no legado: `lib/audit/index.ts`, `lib/audit/actions.ts`
  - Critério de pronto: nunca bloqueia mutação; AUDIT_ACTIONS fonte única
  - Confiança: 🟢

- [ ] T-07, Implementar resolvedor de operador/controlador
  - Origem no legado: `lib/legal/operador.ts`
  - Critério de pronto: client de sessão (não service role); `urlDePoliticaSegura` bloqueia javascript:
  - Confiança: 🟢

- [ ] T-08, Implementar resolvedor de marca por camadas (nunca lança)
  - Origem no legado: `lib/branding/resolve.ts`, `saida.ts`, `branding.ts`, `schema.ts`
  - Critério de pronto: degrada para padrão; grava só semente+papel; email via marcaDaSaida
  - Confiança: 🟢

## Tarefas de Teste

- [ ] TT-01, Export gera PDF com controlador (não marca)
- [ ] TT-02, Redação enfileira avatar antes de zerar ponteiro
- [ ] TT-03, Audit falha não bloqueia a mutação
- [ ] TT-04, Marca resolve por camadas e degrada sem lançar
- [ ] TT-05, "Deskcomm" não aparece em código de usuário (`branding.test.ts`)

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `api_audit_log` sem GRANT de UPDATE/DELETE + expurgo com piso 90 dias

## Ordem Sugerida
1. T-06 (audit) e T-08 (marca) são transversais (usados por todos).
2. T-01/T-02/T-03/T-04 (LGPD) dependem de event-log + Storage.
3. T-07 (operador) alcança `/legal/*` pública.

## Lacunas Pendentes (🔴)
- SQL de `fn_lgpd_cascade_redact_contact` e do expurgo de auditoria.
- Dias exatos do SLA (`lib/lgpd/sla.ts`).
- `lib/branding/{rampa,contraste,regua-do-produto}` (geração de cor).
