# Onboarding e Instalação — Tarefas de Implementação

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO

## Pré-requisitos
- [ ] `OnboardingState` (schema) persistido por org
- [ ] `.env` da instalação (chaves de provedor, gateway, email, WhatsApp)
- [ ] Integração de loja com flag `lojaLigada`

## Tarefas

- [ ] T-01, Implementar fonte única de passos + resolução de próximo/resumo
  - Origem no legado: `lib/onboarding/passos.ts`
  - Critério de pronto: passo só existe se aplicável; roteador/indicador/resumo derivam de PASSOS
  - Confiança: 🟢

- [ ] T-02, Implementar detecção de ambiente da instalação
  - Origem no legado: `lib/instalacao/ambiente.ts`
  - Critério de pronto: env vazio = ausente; Google sem variável; placeholder detectado
  - Confiança: 🟢

- [ ] T-03, Implementar sugestão de funil por IA (com fallback)
  - Origem no legado: `lib/onboarding/sugerir-funil.ts`, `pacotes-de-funil.ts`
  - Critério de pronto: nunca funil vazio; cai em PACOTE_PADRAO por nicho
  - Confiança: 🟡

## Tarefas de Teste

- [ ] TT-01, Loja desligada → connect-nuvemshop não aparece
- [ ] TT-02, `.env` sem RESEND_API_KEY → email=false
- [ ] TT-03, Org "Minha Empresa" → nomeAindaEhPlaceholder=true
- [ ] TT-04, Sugestão de funil falha → PACOTE_PADRAO

## Ordem Sugerida
1. T-01 (passos) é o esqueleto do wizard.
2. T-02 (ambiente) adapta os passos ao que o instalador trouxe.
3. T-03 (sugestão) no passo `setup-ai`/`funil`.

## Lacunas Pendentes (🔴)
- `hostgator-setup-kit/` (instalação assistida) não analisado.
- `lib/onboarding/{pacotes-de-funil,proposta-de-funil}` e `lib/instalacao/{prova-de-credito,retrato}`.
