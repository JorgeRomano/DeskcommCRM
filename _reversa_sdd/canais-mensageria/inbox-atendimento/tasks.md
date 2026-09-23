# Caso de Uso: Inbox e Fronteira de Atendimento — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas de conversa/demanda com `service_revision`/`demanda_revision`

## Tarefas
- [ ] T-01, Implementar a máquina de comando da conversa
  - Origem no legado: `lib/inbox/comando-da-conversa.ts`
  - Critério de pronto: 5 estados; precedência de motivo; `silencioVigente` fail-closed; ordem por `awaiting_since`
  - Confiança: 🟢
- [ ] T-02, Implementar a fronteira de serviço
  - Origem no legado: `lib/atendimento/fronteira-server.ts`
  - Critério de pronto: `assertCurrentServiceBoundary` (CAS/revision); `AsyncLocalStorage`; `guardServiceTools`
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Precedência de motivo (bloqueado > travado > silêncio)
- [ ] TT-02, Fronteira obsoleta lança StaleServiceBoundaryError
- [ ] TT-03, Abrir 1ª demanda não é novo serviço

## Ordem Sugerida
1. T-01 → T-02.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
