# Caso de Uso: MFA — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Supabase Auth com TOTP; tabela `user_recovery_codes`

## Tarefas
- [ ] T-01, Implementar a política de cadastro de MFA
  - Origem no legado: `lib/auth/politica-mfa.ts`
  - Critério de pronto: as duas origens somam; default não exige; ausência = false
  - Confiança: 🟢
- [ ] T-02, Implementar a regra de sessão e recovery codes
  - Origem no legado: `lib/auth/server.ts` (`mfaEmDivida`, `sessionAal`), `recovery-codes.ts`
  - Critério de pronto: quem tem fator prova sempre; rejection sampling; sha256
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Origens somam (platform admin obrigado em org que não exige)
- [ ] TT-02, aal1 com fator → em dívida
- [ ] TT-03, Recovery codes sem viés de módulo

## Ordem Sugerida
1. T-01 → T-02.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
