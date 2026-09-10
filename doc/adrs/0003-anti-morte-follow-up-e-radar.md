# ADR 0003 — Follow-up como mecanismo anti-morte e Radar de Risco

> ADR retroativo · Status: **Aceito** · Confiança: 🟢 CONFIRMADO (`docs/doctrine/sistema-vivo.md`, `lib/followup/`, `lib/leads/risk-radar.ts`)

## Contexto

O invariante 4 da doutrina do Sistema Vivo: "nenhuma demanda aberta sem próximo passo definido e visível". Medido em produção (2026-08-06): 32 conversas para 15 leads, 13 dos 15 sem contato vinculado — pessoas escreviam e desapareciam sem que ninguém visse. O anti-morte era forte no engine (`schedule-followup`, `cron/scheduler`) mas **invisível para o humano**.

## Decisão

Duas peças complementares:
1. **Follow-up como máquina de estados por grafo** (`lib/followup/`): enrollment percorre nós (`trigger/wait/condition/ai_classify/match_reply/repeat/action/end`), com claim de worker, idempotência por evento, planejamento adaptativo de tempo e reatividade a inbound/handoff. 1 enrollment vivo por lead/org.
2. **Radar de Risco** (`/app/radar`, `lib/leads/risk-radar.ts`): tela que torna visível as demandas abertas que esfriaram (`critico`/`em_risco`/`em_voo`/`em_dia`), classificadas por janela de esfriamento **do estágio** (não constante global).
3. **Nascimento determinístico do lead** no ingest (`garantirLeadDaConversa`) — o sistema cria, não o modelo (entrada de funil que depende de o modelo lembrar falha no turno atípico).

## Alternativas consideradas

1. **Follow-up disparado por tool do modelo.** Rejeitada para o nascimento do lead: "entrada de funil que depende de o modelo lembrar é entrada que falha justamente no turno atípico" (mesma razão do checkpoint imposto pelo runtime).
2. **Janela de esfriamento global fixa.** Rejeitada: "sem resposta há 2 dias" é normal numa negociação e é abandono num agendamento — a janela vem de `crm_stages.expected_duration_hours`.
3. **Só o engine (sem tela).** Rejeitada: o anti-morte invisível viola o invariante 3 (log visível) — foi o "desilhamento C1" de 2026-07-24.

## Consequências

- **Positivas:** invariantes 4+5 satisfeitos e provados em tela (E2E); paridade humano↔IA (o radar e o agente leem o mesmo `classifyRisk`).
- **Negativas:** máquina de follow-up é complexa (dead-man de rechecks precisou subir de 5 para 14 porque matava follow-up da noite pela janela anti-ban — bug medido 2026-08-18).
- **Laço de retorno (invariante 7):** `outcome-stats` agrega converted/replied/exhausted/opted_out/handoff por fluxo; o flywheel consome.
