# ADR 0001 — Dois runtimes de IA convivendo (agent-engine + workers legados)

> ADR **retroativo** gerado pelo Detetive (Reversa) a partir de código + arqueologia Git.
> Status: **Aceito** (vigente) · Confiança: 🟢 CONFIRMADO no código · 🟡 datas/motivação parciais

## Contexto

O produto nasceu da fusão de dois sistemas (commits `feat`/`docs` de "vendaval-fusion", `docs/vendaval-fusion-plan.md`): um CRM com IA sobre `event_log` (workers `ai-response-worker`, `ai-sentiment-worker`) e o "Vendaval" — um runtime de agente rico sobre fila Postgres (`lib/agent-engine/`, `workers/agent-worker/main.ts`). Os dois consomem a mesma cañería de eventos mas resolvem o turno de formas diferentes.

## Decisão

Manter **os dois runtimes convivendo**, com fronteira clara:
- O **agent-engine** é o runtime canônico/rico (fila durável, turnos com tools, guardrails, pacing/spinning, flywheel), usado por organizações com versão de agente **publicada**.
- Os **workers legados** (`workers/ai-*.ts`) atendem organizações **sem** versão publicada, consumindo `message.received`/`ai.sentiment_alert`.
- Ambos compartilham o orquestrador de handoff (`triggerHandoff`), a decisão de orçamento (`decidirOrcamento`) e a trava de multi-tenancy.
- `lib/ai/dispatcher/` foi marcado `@deprecated` (Fase 0) e saiu do caminho quente.

## Alternativas consideradas

1. **Migrar tudo para o agent-engine de uma vez.** Rejeitada: quebraria as instalações sem agente publicado (o caso de toda instalação nova pelo kit).
2. **Manter só os workers legados.** Rejeitada: não suportam tools, memória durável, guardrails determinísticos nem flywheel — o núcleo de valor do produto.
3. **Um adapter único que esconda os dois.** Adiada: a fronteira "tem versão publicada?" é simples o bastante para viver no drain (`lib/agent-engine/edge/crm/drain.ts`).

## Consequências

- **Positivas:** migração incremental sem downtime; compatibilidade com self-host fresco; reuso das partes compartilhadas (handoff, orçamento).
- **Negativas:** duas fontes de comportamento de resposta ao cliente; correções de defeito precisam ser feitas nos dois lados (vários commits `fix(ia)` corrigem "a cópia do padrão que ficou para trás" — ex.: resolução de modelo por provider no sentiment E no response worker).
- **Dívida declarada:** o `dispatcher` deprecado ainda existe por completude.
