# Caso de Uso: Onboarding — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Agente publicado (cérebro/modelo/chave) para a sugestão de IA; `NAV_CATALOG`

## Tarefas
- [ ] T-01, Implementar passos do wizard
  - Origem no legado: `lib/onboarding/passos.ts`
  - Critério de pronto: passo decide própria existência; `funil` depois de `setup-ai`
  - Confiança: 🟢
- [ ] T-02, Implementar proposta/pacotes/sugestão de funil
  - Origem no legado: `lib/onboarding/proposta-de-funil.ts`, `pacotes-de-funil.ts`, `sugerir-funil.ts`, `o-que-mais-existe.ts`
  - Critério de pronto: validar/recusar → pacote curado; won/lost derivados; `extrairJson` tolerante
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, nuvemshop só aparece com loja ligada
- [ ] TT-02, IA off → pacote curado com porque
- [ ] TT-03, Proposta sem won é reprovada

## Ordem Sugerida
1. T-01 → T-02.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
