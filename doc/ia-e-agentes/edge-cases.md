# IA e Agentes — Casos Extremos

> Redator (Reversa) · Nível: Detalhado · casos extraídos de comentários e correções no código
> 🟢 CONFIRMADO · 🟡 INFERIDO

## EC-01 — Modelo sem provider atendível 🟢
Numa instalação self-host padrão (só `ANTHROPIC_API_KEY`, sem `AI_GATEWAY_API_KEY`), passar o modelo como string caía no gateway da Vercel e abortava com "Unauthenticated". `resolverModeloDoPonto` resolve o provider da chave existente. Fica DEPOIS de G1/G4 (pedido de humano e menção legal geram handoff mesmo sem LLM atendível). Skip, não erro (config, não falha transitória).

## EC-02 — Orçamento estourado no meio do turno 🟢
`comHandoffSeOrcamentoAcabar` envolve o turno INTEIRO — inclusive chamadas indiretas (`classifyStage`, `maybeCompact`) que rodam antes da principal. A conversa é devolvida à fila humana antes do relance. `operator_turn` não tem handoff (perde só o `registrarDesfecho` daquele turno). Erro `terminal===true` → `cancelJob`, não `failJob` (senão N×5 alertas críticos afogariam o `budget_exceeded`).

## EC-03 — Promessa nomeando pessoa em vez de cargo 🟢
Agente cujo prompt nomeia "Fernando" em vez de "gerente" escapava 100% do detector (medido tenant YADEA: dezenas de promessas, 1 detecção em 3 dias). `detectHumanPromise` aceita `extraHumanNames` (de `ai_agent_versions.handoff_keywords`) para estender o alvo.

## EC-04 — Falso vazio / vocabulário interno teimoso 🟢
Um regex teimoso podia calar o turno inteiro (cliente sem resposta por frase nossa). Fail-safe com teto (`MAX_VETOS_DE_FALSO_VAZIO=2`, `MAX_VETOS_DE_VOCABULARIO_INTERNO=2`): a 1ª vez ensina, a 2ª libera com `runLog.warn` — troca erro invisível (silêncio) por visível.

## EC-05 — 8 mensagens seguidas no mesmo turno 🟢
Nenhum gate limita CONTAGEM (pacing limita ritmo). `DEFAULT_MAX_SENDS_PER_TURN=3` impede o modelo tratar lista de perguntas como uma mensagem por pergunta (medido: 1 lead recebeu 8).

## EC-06 — Envio fora da janela de 24h 🟢
`messagingWindowGate` veta texto livre; a saída é `send_template` (Meta). Sem a flag `isTemplate`, o template seria vetado pelo próprio gate que ele resolve, ou pularia a cadeia (viraria bypass de opt-out/LGPD).

## EC-07 — Reindexar FAQ derrubava o catálogo 🟢
Quando a versão de conhecimento era por agente e havia uma ativa por agente, duas rotinas competiam pelo mesmo ponteiro. Agora indexa a FONTE que o evento nomeia; reindexar a FAQ não derruba o catálogo.

## EC-08 — Sentimento com agente errado 🟢
Antes o limiar vinha do "primeiro agente por is_default", não do agente da conversa (issue #486): numa clínica cliente triste é problema, numa assistência é normal. `resolverAgenteDaConversa` usa `active_ai_agent_id` + versões publicadas na sessão.

## EC-09 — Eco de envio próprio vs digitação humana 🟢
Mensagem `fromMe` pode ser eco do nosso envio ou digitação no celular do operador. `ehEcoDeEnvioNosso` (janela 60s, mesmo corpo, sem `external_id`, `sent_via in [ai,user]`) decide; na dúvida silencia (desfecho seguro do atendente humano).

## EC-10 — Confirmação de agenda sem checar 🟢
Modelo (`gpt-5.6-terra`) ignorava a instrução e afirmava "está confirmado" sem chamar a tool. `agendaStallGate` (regex determinística, com `semAcento` porque `\b` é ASCII) veta "vou verificar horário"/"está confirmado" sem `crm_book_appointment`/`crm_find_free_slots` no turno.
