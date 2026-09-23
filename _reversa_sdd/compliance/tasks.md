# Compliance — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas: `api_audit_log` (append-only), `lgpd_requests`, `storage_redaction_queue`
- [ ] RPCs security-definer: `fn_lgpd_cascade_redact_contact`, `fn_expurgar_auditoria_vencida`, `fn_podar_fila_de_jobs`, `fn_redigir_captacoes_do_contato_anonimizado`
- [ ] `@react-pdf/renderer`; `lib/email/roteador`; Supabase Storage
- [ ] Env: `LGPD_SIGNING_KEY` (P12, pendente), retenção `*_RETENTION_DAYS`

## Tarefas
- [ ] T-01, Implementar a trilha de auditoria
  - Origem no legado: `lib/audit/index.ts`, `actions.ts`
  - Critério de pronto: fire-and-forget reportado; `isServiceRoleConfigured` não infere por comprimento; array canônico deriva o painel; proibido importar no `actions.ts`
  - Confiança: 🟢
- [ ] T-02, Implementar SLA em dias úteis
  - Origem no legado: `lib/lgpd/sla.ts`, `holidays-br.ts`
  - Critério de pronto: pula fim de semana/feriado; feriados parametrizáveis por país
  - Confiança: 🟢
- [ ] T-03, Implementar a cascata de anonimização e retomada
  - Origem no legado: `lib/lgpd/redact-cascade.ts`, `cascata.ts`
  - Critério de pronto: foto enfileirada antes (falha fechada); idempotência por sufixo; passo 4 cancela régua; cron sem starvation
  - Confiança: 🟢
- [ ] T-04, Implementar o export de acesso (coletor, PDF, PAdES, e-mail, máscara)
  - Origem no legado: `lib/lgpd/export-collector.ts`, `pdf-renderer.tsx`, `pades-signer.ts`, `email-delivery.ts`, `mask.ts`
  - Critério de pronto: export = o que se redige (gate); PDF nomeia controlador; PAdES stub honesto; e-mail só sha256 no log
  - Confiança: 🟢
- [ ] T-05, Implementar fila de storage e alarme de SLA
  - Origem no legado: `lib/lgpd/storage-redaction-queue.ts`, `sla-alarm.ts`
  - Critério de pronto: claim idempotente; "not found" = skipped; alarme sem PII, dedup 24h
  - Confiança: 🟢
- [ ] T-06, Implementar perfil de país e operador
  - Origem no legado: `lib/legal/perfil-do-pais.ts`, `operador.ts`
  - Critério de pronto: país com citação revisada ou não cita; operador por client de sessão; `urlDePoliticaSegura` na saída
  - Confiança: 🟢
- [ ] T-07, Implementar opt-out e retenção
  - Origem no legado: `lib/opt-out/deteccao.ts`, `lib/retencao/politica.ts`
  - Critério de pronto: verbo + objeto de comunicação; inequívoco bloqueia, ambíguo escala; piso no SQL, cópia no TS para log
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Audit falha reporta ao Sentry, não bloqueia mutação
- [ ] TT-02, `computeDueAt` pula feriado
- [ ] TT-03, Cascata idempotente (sem duplicar sufixo)
- [ ] TT-04, Gate export × cascata deriva a mesma lista
- [ ] TT-05, Opt-out por intenção (a dor vs a mensagem)
- [ ] TT-06, Retenção nunca abaixo do piso

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `api_audit_log` append-only sem GRANT de UPDATE/DELETE (nem service role)

## Ordem Sugerida
1. T-01 (audit) como base.
2. T-02/T-03/T-04/T-05 (LGPD) na ordem.
3. T-06/T-07 (legal/opt-out/retenção) por último.

## Lacunas Pendentes (🔴)
- Funções SECURITY DEFINER e policies RLS (Data Master).
- Provisionar `LGPD_SIGNING_KEY` (P12) para tirar o PAdES do stub.
- Publicar perfis de outros países com citação revisada.
