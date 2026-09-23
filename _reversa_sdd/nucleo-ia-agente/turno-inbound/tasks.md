# Caso de Uso: Turno Inbound — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Fila e máquina de estados implementadas (ver `../tasks.md` T-01, T-04)

## Tarefas
- [ ] T-01, Carregar inbound pelo id exato
  - Origem no legado: `lib/agent-engine/agent/inbound-turn.ts` (`loadInboundBodyForJob`, `:506`)
  - Critério de pronto: usa `corpoDaMensagem`, imune a corrida com eventos concorrentes
  - Confiança: 🟢
- [ ] T-02, Montar a abertura do turno
  - Origem no legado: `inbound-turn.ts:1386-1556`
  - Critério de pronto: bloco de abertura inclui playbook, checkpoint, lead_state e histórico N
  - Confiança: 🟢
- [ ] T-03, Loop de tools com guarda de envio
  - Origem no legado: `inbound-turn.ts:2843-2962+`
  - Critério de pronto: envio só via `send_message`, teto respeitado, texto livre descartado
  - Confiança: 🟢
- [ ] T-04, Fechamento por checkpoint
  - Origem no legado: `inbound-turn.ts:573-607`, `:1355-1382`
  - Critério de pronto: 2ª chamada devolve só o JSON; validado por Zod; persistido
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Ritual completo gera checkpoint
- [ ] TT-02, Teto de envios bloqueia o 4º envio (default)

## Ordem Sugerida
1. T-01 → T-02 → T-03 → T-04.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante; confirmar knobs de env.
