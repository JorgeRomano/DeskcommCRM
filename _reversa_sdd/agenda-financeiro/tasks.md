# Agenda e Financeiro — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas de agenda, financeiro e catálogo (ver `design.md`)
- [ ] RPCs security-definer do baseline (`fn_google_*`, `fn_meet_*`, `fn_finalizar_comanda`, `fn_agenda_ocupacao_google_do_dono`)
- [ ] App OAuth Google (`platform_google_oauth` ou `.env GOOGLE_CALENDAR_*`)

## Tarefas
- [ ] T-01, Implementar o vocabulário da agenda (fonte dos CHECK)
  - Origem no legado: `lib/agenda/tipos.ts`
  - Critério de pronto: consts `as const` como fonte; `SITUACAO_SEGURA_O_LEAD`, `CONEXAO_CONTA_COMO_OCUPACAO`; cor de trilha por hash estável
  - Confiança: 🟢
- [ ] T-02, Implementar o motor de horários livres e fuso
  - Origem no legado: `lib/agenda/horarios-livres.ts`, `fuso.ts`
  - Critério de pronto: grade fixa; faixas unidas; bloqueio = ocupado; DST de duas passagens; nada cruza meia-noite
  - Confiança: 🟢
- [ ] T-03, Implementar coleta do banco e ocupação
  - Origem no legado: `lib/agenda/consulta.ts`, `ocupados.ts`, `ocupacao-externa.ts`, `jornada.ts`
  - Critério de pronto: client injetado filtra org; recusa em duas vozes; na dúvida ocupa; Google por RPC em paralelo
  - Confiança: 🟢
- [ ] T-04, Implementar a sincronização Google Calendar
  - Origem no legado: `lib/agenda/google/*` (`config`, `oauth`, `token`, `evento`, `erros`, `sync-model`, `sync-executor`, `meet`, `lembretes`)
  - Critério de pronto: merge de três pontas; `id`+`iCalUID` nunca juntos; tabela de desfechos de erro; Meet com gate de entrega
  - Confiança: 🟢
- [ ] T-05, Implementar proteção de follow-up e fronteira de agenda
  - Origem no legado: `lib/agenda/protecao-followup.ts`, `efeito.ts`
  - Critério de pronto: adia follow-up com compromisso vivo; `assertAgendaEffect*` lança em fronteira obsoleta
  - Confiança: 🟢
- [ ] T-06, Implementar comanda e catálogo financeiro
  - Origem no legado: `lib/financeiro/comanda.ts`, `catalogo.ts`
  - Critério de pronto: precedência de comissão; total do item piso zero; schemas explícitos por entidade; cancelar é status
  - Confiança: 🟢
- [ ] T-07, Implementar catálogo de produtos (busca, planilha, moeda)
  - Origem no legado: `lib/catalogo/busca.ts`, `planilha.ts`, `moeda-da-org.ts`
  - Critério de pronto: palavra difusa/número exato; número filtra; relaxamento marcado; identidade estável na import; moeda única com rastro
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Bloqueio não desliza a grade
- [ ] TT-02, token_expired continua ocupando
- [ ] TT-03, Merge de três pontas classifica conflito
- [ ] TT-04, Comissão pessoa+serviço vence; empate pelo maior
- [ ] TT-05, 128GB não aparece para pedido de 256GB
- [ ] TT-06, DST: instanteDe nunca devolve Invalid Date

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Comanda (migration 0240): saldo derivado, lançamento pago imutável, comissão congelada

## Ordem Sugerida
1. T-01 (vocabulário) → T-02 (motor) → T-03 (coleta).
2. T-04 (Google) e T-05 (proteção) dependem de T-03.
3. T-06/T-07 (financeiro/catálogo) independentes.

## Lacunas Pendentes (🔴)
- Lógica interna das RPCs de agenda/comanda (Data Master).
- Nome exato da env var do TTL de entrega do Meet.
