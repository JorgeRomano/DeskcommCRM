# Automação e Follow-up — Tarefas de Implementação

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Pré-requisitos
- [ ] `event_log` + handlers registrados (automation-rules, reactivity, gatilho-etapa)
- [ ] Tabelas `automation_rules`/`automation_rule_runs`, `followup_flow_pointers`/`versions`, `followup_enrollments`/`enrollment_events`
- [ ] RPC `fn_claim_due_followup_enrollments`
- [ ] `attendant_availability`, `calendar_appointments`

## Tarefas

- [ ] T-01, Implementar motor de automação (anti-loop, condições, postpone, agregação honesta)
  - Origem no legado: `lib/automation/engine.ts`, `conditions.ts`, `guarda-do-contato.ts`
  - Critério de pronto: `caused_by_rule` não reprocessa; skip+fail não viram success
  - Confiança: 🟢

- [ ] T-02, Implementar ações da automação (registry + executores)
  - Origem no legado: `lib/automation/actions/*`
  - Critério de pronto: `create-or-move-lead` reusa handlers de leads; actor webhook_source
  - Confiança: 🟡

- [ ] T-03, Implementar decisões puras de nó do grafo
  - Origem no legado: `lib/followup/node-handlers.ts`
  - Critério de pronto: 8 tipos de nó; action at-most-once; dead-man 14 rechecks
  - Confiança: 🟢

- [ ] T-04, Implementar tick do worker de follow-up (claim + isolamento)
  - Origem no legado: `lib/followup/engine.ts`
  - Critério de pronto: claim falho não confunde com "nada vencido"; falha de 1 não derruba o tick
  - Confiança: 🟢

- [ ] T-05, Implementar ponte de turno (só active/waiting_reply avançam)
  - Origem no legado: `lib/followup/turn-bridge.ts`
  - Critério de pronto: turno stale não sobrescreve intervenção humana; idempotência por key
  - Confiança: 🟢

- [ ] T-06, Implementar enrollment (1 por lead/org)
  - Origem no legado: `lib/followup/enroll.ts`, `lib/followup/gatilho-etapa.ts`
  - Critério de pronto: 23505 → 409; gatilho por etapa com idempotency_key
  - Confiança: 🟢

- [ ] T-07, Implementar reatividade (inbound/handoff)
  - Origem no legado: `lib/followup/reactivity.ts`
  - Critério de pronto: opt-out cancela tudo; handoff allow/cancel/pause; resolved retoma
  - Confiança: 🟢

- [ ] T-08, Implementar escalação (disponibilidade + pausa manual)
  - Origem no legado: `lib/escalacao/disponibilidade.ts`, `atendimento-manual.ts`
  - Critério de pronto: frase conservadora sem ninguém; pausa 60min renovável
  - Confiança: 🟢

- [ ] T-09, Implementar proteção de agenda no follow-up
  - Origem no legado: `lib/agenda/protecao-followup.ts`
  - Critério de pronto: compromisso futuro adia; leitura indisponível não cobra
  - Confiança: 🟢

- [ ] T-10, Implementar agregação de outcomes por fluxo
  - Origem no legado: `lib/followup/outcome-stats.ts`
  - Critério de pronto: `conversion_rate = converted/terminal`; dead excluído
  - Confiança: 🟢

## Tarefas de Teste

- [ ] TT-01, Regra casa e agrega honestamente (skip não vira success)
- [ ] TT-02, action nunca completa → dead após 14 rechecks
- [ ] TT-03, opt-out cancela todos enrollments (`opted_out`)
- [ ] TT-04, turno stale não reativa enrollment pausado por humano
- [ ] TT-05, 1 enrollment por lead/org (2º → 409)

## Ordem Sugerida
1. T-03 (nós puros) e T-04/T-05 (tick/ponte) são o núcleo do follow-up.
2. T-01/T-02 (automação) em paralelo.
3. T-07 (reatividade) depende de T-04; T-08/T-09 transversais.

## Lacunas Pendentes (🔴)
- Ações concretas da automação (`actions/*`) além de create-or-move-lead.
- `lib/followup/{silence-sweep,intervencao,timing-plan}` e o `graph-schema` completo.
- Integração Google Calendar (`lib/agenda/google/*`).
