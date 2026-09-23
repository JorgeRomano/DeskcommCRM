# Caso de Uso: RBAC (`requireRole`)

> Sub-unit de `auth-tenancy-rbac`. O guard único de autorização por role nas rotas `/api/v1`.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
`requireRole(min, opts)` é o único ponto de autorização por role (invariante G2-01). Reimplementar a comparação de rank na rota é anti-padrão proibido (gate `pnpm lint:role-rank`). 🟢

## Responsabilidades
- Resolver usuário, support e org (do recurso ou ativa). 🟢
- Ler o role efetivo do banco (`fn_user_role_in_org`). 🟢
- Aplicar o gate de MFA após o rank e antes do sucesso. 🟢

## Regras de Negócio
- Role efetivo do banco, nunca do cookie; membership revogada falha fechada. 🟢
- Gate de MFA depois do rank (403 por falta de papel não revela estado de MFA). 🟢
- `allowPlatformAdmin && is_platform_admin && !support` → bypass do rank do tenant. 🟢
- `roleAtLeast` é leitura de rank, NÃO gate de rota. 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Role do banco | Must | `fn_user_role_in_org` decide; revogado → forbidden |
| RF-02 | Gate de MFA após rank | Must | aal1 com fator → 403 mfa_required |
| RF-03 | Bypass de platform admin | Should | `allowPlatformAdmin` fura o rank do tenant |

## Critérios de Aceitação
```gherkin
Dado um usuário com rank abaixo do mínimo
Quando requireRole avalia
Então audit authz.denied + 403 forbidden_role

Dado rank suficiente mas sessão aal1 com fator cadastrado
Quando requireRole avalia
Então 403 mfa_required (após o rank)
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/auth/require-role.ts` | `requireRole` | 🟢 |
| `lib/auth/types.ts` | `Role`, `roleAtLeast`, `VisibilityMode` | 🟢 |
