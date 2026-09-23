# Caso de Uso: MFA

> Sub-unit de `auth-tenancy-rbac`. Duas perguntas distintas: preciso CADASTRAR (política) e preciso PROVAR agora (sessão).
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
MFA TOTP opcional, ligado por quem administra. Duas políticas que SOMAM (plataforma e organização) mais a regra de sessão de que quem tem fator prova sempre. 🟢

## Responsabilidades
- Decidir se o cadastro de MFA é exigido (`exigeCadastroDeMfa`). 🟢
- Decidir se a prova está em dívida na sessão (`mfaEmDivida`). 🟢
- Emitir e validar códigos de recuperação. 🟢

## Regras de Negócio
- As duas origens SOMAM, nunca se anulam; default de ambas é NÃO exigir. 🟢
- `mfaEmDivida` = tem fator (`isMfaEnrolled`) E sessão `aal1` — não pergunta a política. 🟢
- Recovery codes: 10 códigos, rejection sampling, alfabeto sem ambiguidade, guardados como sha256. 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Política de cadastro somando origens | Must | Platform admin obrigado mesmo em org que não exige |
| RF-02 | Prova de sessão para quem tem fator | Must | aal1 com fator → em dívida |
| RF-03 | Recovery codes seguros | Should | 10 códigos, sha256, sem viés de módulo |

## Critérios de Aceitação
```gherkin
Dado um platform admin com mfa_required numa org que não exige
Quando exigeCadastroDeMfa avalia
Então retorna true (as origens somam)

Dado um usuário com fator cadastrado e sessão aal1
Quando mfaEmDivida avalia
Então retorna true (independe da política)
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/auth/politica-mfa.ts` | `exigeCadastroDeMfa`, `empresaExigeMfa` | 🟢 |
| `lib/auth/server.ts` | `mfaEmDivida`, `sessionAal`, `requiresMfa` | 🟢 |
| `lib/auth/recovery-codes.ts` | geração/validação de códigos | 🟢 |
