# Caso de Uso: RBAC (`requireRole`) — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] RPC `fn_user_role_in_org`; `lib/audit`; gate `pnpm lint:role-rank`

## Tarefas
- [ ] T-01, Implementar `requireRole`
  - Origem no legado: `lib/auth/require-role.ts`
  - Critério de pronto: 8 passos do fluxo; role do banco; gate de MFA após rank; bypass de platform admin
  - Confiança: 🟢
- [ ] T-02, Implementar os tipos de papel e o gate de lint
  - Origem no legado: `lib/auth/types.ts`, script `lint:role-rank`
  - Critério de pronto: `Role` 1..5; `ai_operator` fora de user_organizations; lint reprova comparação de rank na rota
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, rank < min → 403 forbidden_role
- [ ] TT-02, aal1 com fator → 403 mfa_required após rank
- [ ] TT-03, lint reprova matriz advisória na rota

## Ordem Sugerida
1. T-02 → T-01.

## Lacunas Pendentes (🔴)
- `fn_user_role_in_org` (Data Master).
