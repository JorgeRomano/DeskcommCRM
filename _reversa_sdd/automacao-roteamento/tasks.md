# Automação e Roteamento — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas: `automation_rules`, `automation_rule_runs`, `conversation_assignment_events`, `followup_enrollments`, `followup_enrollment_events`, `passagens_de_atendimento`, `channel_knobs`
- [ ] RPCs: `fn_channel_routing_claim`, `fn_claim_due_followup_enrollments`, `fn_publish_followup_flow_version`, `fn_conversation_assign`, `fn_passagem_devolvida`
- [ ] Dispatcher de `event_log` com consumer keys

## Tarefas
- [ ] T-01, Implementar o motor de regras (engine + condições + guardas)
  - Origem no legado: `lib/automation/engine.ts`, `conditions.ts`, `guarda-do-contato.ts`, `desfecho-do-envio.ts`
  - Critério de pronto: anti-loop; guard de entity_kind; postpone all-or-nothing; status derivado; consentimento fixo por `declined_at`
  - Confiança: 🟢
- [ ] T-02, Implementar o catálogo de 7 ações
  - Origem no legado: `lib/automation/actions/*`, `register-all.ts`
  - Critério de pronto: contrato `ActionExecutor`; webhook anti-SSRF + campos públicos; create_or_move transfere cross-pipeline; envio at-most-once
  - Confiança: 🟢
- [ ] T-03, Implementar janela e throttle de envio
  - Origem no legado: `lib/automation/janela-do-canal.ts`, `throttle.ts`
  - Critério de pronto: régua única no fuso do tenant; falha aberta; espaçamento compartilhado com IA
  - Confiança: 🟢
- [ ] T-04, Implementar roteamento (decisão + elegibilidade + worker + fila)
  - Origem no legado: `lib/routing/decide.ts`, `eligibility.ts`, `worker.ts`, `eligibles.ts`, `queue.ts`, `channel-policies.ts`
  - Critério de pronto: rodízio real; `windows` vazio = 24/7; claim CAS; adota lead do contato; plantão calculado por leitura
  - Confiança: 🟢
- [ ] T-05, Implementar o grafo de follow-up (schema + nós + edges)
  - Origem no legado: `lib/followup/graph-schema.ts`, `node-handlers.ts`
  - Critério de pronto: `flowGraphSchema` com superRefine de integridade; `nodeBranches` único leitor de dialeto; envio at-most-once; dead-man timers
  - Confiança: 🟢
- [ ] T-06, Implementar o worker de follow-up e reatividade
  - Origem no legado: `lib/followup/engine.ts`, `turn-bridge.ts`, `reactivity.ts`, `enroll.ts`, `agent-followup-gate.ts`, `silence-sweep.ts`, `publish.ts`, `validate-publish.ts`, `timing-plan.ts`
  - Critério de pronto: claim nunca lança; estado por eventos; reatividade por status; 1 inscrição por lead; validate-publish
  - Confiança: 🟢
- [ ] T-07, Implementar escalação (passagem, retomada, devolução, aviso ao suporte)
  - Origem no legado: `lib/escalacao/passagem.ts`, `briefing-da-passagem.ts`, `retomada.ts`, `devolucao-automatica.ts`, `continuidade.ts`, `atendimento-manual.ts`, `aviso-ao-suporte.ts`, `estado-do-aviso.ts`
  - Critério de pronto: passagem imutável nunca lança; briefing anti-injection; devolução solta as 3 travas; aviso reivindica antes da rede
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Anti-loop: evento causado por regra é skipped
- [ ] TT-02, Consentimento recusado bloqueia
- [ ] TT-03, Rodízio real (mais antigo primeiro)
- [ ] TT-04, `windows` vazio = 24/7
- [ ] TT-05, Follow-up: nó entrado 2× (armar/avançar); envio at-most-once
- [ ] TT-06, Passagem nunca lança; briefing separa citação de paráfrase
- [ ] TT-07, Devolução automática conta do último sinal humano

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Migrar `followup_enrollment_events` (event sourcing) preservando chaves `${node}:${steps}`

## Ordem Sugerida
1. T-01/T-02/T-03 (automação) como base de eventos.
2. T-04 (roteamento) independente.
3. T-05/T-06 (follow-up) e T-07 (escalação) por último.

## Lacunas Pendentes (🔴)
- Lógica interna das RPCs (Data Master).
- Reanimar `throttle.ts` exige emissor para `channel_session_warmup` (hoje inalcançável).
