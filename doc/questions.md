# Perguntas para Validação — DeskcommCRM

> Gerado pelo **Revisor** (Reversa) em 2026-09-10 · `answer_mode = file`
> Preencha o campo **Resposta** de cada pergunta e me avise (digite `reversa`) quando terminar.
>
> Estas são as lacunas 🔴 que **só você pode resolver** — pontos onde a extração
> não pôde confirmar o comportamento a partir do código lido, ou que dependem de
> uma decisão de produto/negócio. As demais lacunas técnicas (SQL de RPCs, triggers,
> policies RLS) estão em `gaps.md` e são endereçadas pelo **Data Master**, não aqui.

---

## Pergunta 1 ✅ Respondida

**Contexto:** SLA de LGPD — `lib/lgpd/repository.ts:computeDueAt` (não lido em detalhe; `lib/lgpd/sla.ts`)
**Spec afetada:** `doc/lgpd-legal-e-auditoria/requirements.md` (RF/regra L-02/L-03)
**Pergunta:** Os prazos de SLA em dias úteis BR são exatamente **7 dias (export/data_request)** e **15 dias (redact)**, como o catálogo de regras sugere? Há algum caso que use prazo diferente (ex.: `store_redact` de tenant / uninstall com prazo próprio)?
**Impacto:** Confirma se as regras L-02/L-03 sobem para 🟢 ou permanecem 🟡 nas specs; define o `due_at` na reimplementação.

**Resposta:** 7 e 15 dias uteis. Nao há casos que use diferente. 

---

## Pergunta 2 ✅ Respondida (⚠️ contradiz AT-04 do catálogo — resposta do usuário prevalece)

**Contexto:** Atendimento/roteamento — regras AT-02, AT-04, AT-05, AT-08 do catálogo (UI/handlers de mensagens não lidos em profundidade)
**Spec afetada:** `doc/canais-e-mensageria/*`, `doc/domain.md` (§3.5)
**Pergunta:** As seguintes regras continuam vigentes no código atual? (a) "eu cuido" é claim atômico com 409 `already_assigned`; (b) supervisor manager+ lê conversa não atribuída em modo somente-leitura; (c) notas internas nunca vão ao WhatsApp; (d) atendente `online` vira `offline` após 15 min de inatividade.
**Impacto:** Reclassifica essas regras de 🟡 para 🟢 (ou corrige/remove) nas specs de atendimento.

**Resposta:** Supervisor Manager, pode ler e responder.

---

## Pergunta 3 ✅ Respondida

**Contexto:** Retenção de auditoria — regra L-10; `fn_expurgar_auditoria_vencida` (não lida)
**Spec afetada:** `doc/lgpd-legal-e-auditoria/*`, `doc/domain.md`
**Pergunta:** A retenção do `api_audit_log` é mesmo **5 anos por padrão, com piso de 90 dias** e configurável por `AUDIT_LOG_RETENTION_DAYS`? Não há mesmo camada cold/S3 (o catálogo diz que nunca foi construída)?
**Impacto:** Confirma a regra de retenção como 🟢 e evita documentar um arquivamento inexistente.

**Resposta:** Nao foi construida, sim 5 anos por padrao.

---

## Pergunta 4 ✅ Respondida

**Contexto:** Notificações push — `lib/notifications/*` (pipeline VAPID não aprofundado)
**Spec afetada:** `doc/integracoes-externas/*`
**Pergunta:** O pipeline de push (emit → policy/prefs → deliver → web push VAPID) é uma feature ativa e suportada no produto, ou é secundária/experimental? Deve ganhar uma unit própria numa próxima extração?
**Impacto:** Define se `notifications` merece elevação de 🟡 para spec dedicada ou permanece como nota.

**Resposta:** provavelmente ganhe uma unit própria.

---

## Pergunta 5 ✅ Respondida

**Contexto:** Superfície de API v1 — 166 route handlers; só uma amostra representativa foi enumerada no OpenAPI
**Spec afetada:** `doc/openapi/api-v1.yaml`, `doc/permissions.md`
**Pergunta:** Você quer que uma próxima passagem gere o **OpenAPI completo** (todos os 166 handlers, com corpos Zod e papel mínimo por rota) e a **matriz RBAC rota-a-rota**? Ou a amostra + padrão documentado é suficiente para o seu uso?
**Impacto:** Decide o escopo de uma extração incremental sobre `app/api/v1/*`.

**Resposta:** É suficiente.

---

## Pergunta 6 ✅ Respondida

**Contexto:** Módulos sem unit dedicada (`lib/metrics`, `lib/reports`, `lib/catalogo`, `lib/retencao`, `lib/operacao`, `lib/settings`, `lib/tarefas`) — marcados `n/a` na code-spec-matrix
**Spec afetada:** `doc/traceability/code-spec-matrix.md`
**Pergunta:** Algum desses módulos carrega regra de negócio relevante que você quer documentada como unit própria (ex.: métricas/relatórios com fórmulas, catálogo com regras de produto)? Ou são transversais/infra o bastante para ficarem fora?
**Impacto:** Define se a cobertura de specs deve ser ampliada para além dos 9 grupos atuais.

**Resposta:** SIm, preferencialmente documentadas.
