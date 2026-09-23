# Caso de Uso: Restrição de Canal — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Gate `pnpm lint:channels` configurado no CI

## Tarefas
- [ ] T-01, Implementar tipos e matriz de capabilities
  - Origem no legado: `lib/channels/types.ts`, `capabilities.ts`
  - Critério de pronto: matriz por provider; fail-closed; exaustividade em compilação
  - Confiança: 🟢
- [ ] T-02, Implementar o gate de lint de canal
  - Origem no legado: script `lint:channels`
  - Critério de pronto: reprova nome de provider fora de `lib/channels/`
  - Confiança: 🟡

## Tarefas de Teste
- [ ] TT-01, Provider desconhecido lança
- [ ] TT-02, Nome de provider fora de `lib/channels/` reprova no lint

## Ordem Sugerida
1. T-01 → T-02.

## Lacunas Pendentes (🔴)
- Detalhe de implementação do `lint:channels` não lido nesta passagem (confirmar antes de reimplementar).
