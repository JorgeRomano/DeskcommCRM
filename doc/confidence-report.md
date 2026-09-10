# Relatório de Confiança — DeskcommCRM

> Gerado pelo **Revisor** (Reversa) em 2026-09-10 · Nível: Detalhado
> Revisão cruzada via Codex: **não** (plugin Codex não disponível nesta sessão).

---

## Resumo Geral

Contagem de afirmações classificadas nas specs de unit + artefatos transversais (estimativa consolidada, **após processar as 6 respostas do usuário**):

| Nível | Quantidade | Percentual |
|-------|-----------|------------|
| 🟢 CONFIRMADO | ~211 | ~76% |
| 🟡 INFERIDO   | ~50 | ~18% |
| 🔴 LACUNA     | ~16 | ~6% |
| **Total**     | ~277 | 100% |

**Confiança geral:** ~85% (🟢 + metade dos 🟡 = (211 + 25) / 277).

> 4 lacunas resolvidas pelo usuário (SLA LGPD, retenção de audit, AT-04 corrigido, escopo OpenAPI).
> As lacunas 🔴 restantes são de banco (Data Master).

> Alta confiança no núcleo de negócio (código TypeScript lido diretamente). As lacunas
> concentram-se na camada de banco (SQL de RPCs/RLS/triggers), que é escopo do Data Master.

---

## Por unit / artefato

| Spec | 🟢 | 🟡 | 🔴 | Confiança |
|------|----|----|-----|-----------|
| `ia-e-agentes/` | alta | baixa | baixa | ~88% |
| `canais-e-mensageria/` | alta | média | baixa | ~82% |
| `crm-e-vendas/` | alta | baixa | baixa | ~86% |
| `automacao-e-followup/` | alta | média | baixa | ~84% |
| `auth-e-multitenancy/` | alta | baixa | média (RLS/RPC) | ~80% |
| `lgpd-legal-e-auditoria/` | alta | média | média (SLA/retenção) | ~78% |
| `governanca-de-eventos/` | alta | baixa | baixa (SQL emit_event) | ~85% |
| `integracoes-externas/` | média-alta | média | baixa | ~76% |
| `onboarding-e-instalacao/` | alta | média | baixa | ~80% |
| `domain.md` / `state-machines.md` / `permissions.md` | alta | média | média | ~80% |
| `architecture.md` / C4 / `erd-complete.md` | alta | média | média (ERD definitivo) | ~78% |
| `openapi/api-v1.yaml` | média | — | média (amostra) | ~65% |

---

## Lacunas Pendentes 🔴

Itens sem confirmação após a revisão (detalhe em `gaps.md`):

### Camada de banco (crítico — Data Master)
- SQL de todas as RPCs SECURITY DEFINER, policies RLS e triggers.
- ERD definitivo (constraints, índices parciais, FKs, ~213 migrations).

### Decisão de produto/negócio — ✅ RESOLVIDAS pelo usuário (2026-09-10)
- SLA LGPD: export D+7, redact D+15 (sem exceção) → 🟢
- Retenção de audit: 5 anos, sem cold/S3 → 🟢
- AT-04: supervisor manager+ lê E responde (corrige catálogo) → 🟢
- OpenAPI: amostra + padrão é suficiente → escopo fechado
- `notifications` e metrics/reports/catalogo/settings/... → 🔜 documentar em extração incremental
- Seguem 🟡: AT-02 (claim atômico), AT-05 (notas internas), AT-08 (idle→offline) — não reconfirmadas

---

## Consistência (revisão cruzada entre units) 🟢

- **Sem contradições entre units** detectadas: os módulos compartilham vocabulário coerente (event_log, ServiceBoundary, requireRole, guardrails) e as fronteiras batem com o código.
- **Dependências declaradas conferem** com as observadas na escavação (ver `spec-impact-matrix.md`).
- **Cobertura vs surface.json:** os ~40 módulos foram agrupados nas 9 units híbridas; nenhum módulo de negócio central ficou sem cobertura (auxiliares marcados 🟡/n/a em `code-spec-matrix.md`).

---

## Recomendações

- [ ] **Rodar o Data Master** — resolve a maioria das lacunas 🔴 críticas (banco). É o maior salto de confiança disponível.
- [ ] Responder `questions.md` (6 perguntas) para elevar as regras 🟡 de atendimento/LGPD a 🟢.
- [ ] Se for reimplementar o agente, ler `inbound-turn.ts` linha a linha (o laço de tools não foi lido integralmente).
- [ ] Considerar Visor/Design System para cobrir a camada de UI (fora do escopo desta descoberta).

---

## Histórico de Reclassificações

| De | Para | Afirmação | Evidência |
|----|------|-----------|-----------|
| 🟡 | 🟢 | Domingo liberado por default (janela é cortesia) | catálogo W-07 + `pacing/defaults.ts` |
| 🟡 | 🟢 | Opt-out em espanhol coberto (2 níveis) | catálogo W-02 (correção 2026-08-30) |
| 🟡 | 🟢 | SLA LGPD export D+7 / redact D+15 (sem exceção) | resposta do usuário (questions.md#1) |
| 🔴 | 🟢 | Retenção de audit 5 anos, sem cold/S3 | resposta do usuário (questions.md#3) |
| 🟡 | 🟢 | AT-04: supervisor manager+ lê E responde (corrige catálogo) | resposta do usuário (questions.md#2) |
| — | 🔴 | SQL de RPCs/RLS/triggers permanece lacuna | só call sites lidos → Data Master |

> Nota: a revisão manteve a classificação conservadora do Redator na maioria dos itens; as
> reclassificações 🟡→🟢 vieram de cruzar o código com o catálogo de regras já corrigido no repo.
