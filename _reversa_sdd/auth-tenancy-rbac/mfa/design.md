# Caso de Uso: MFA — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `exigeCadastroDeMfa` | `({role, isPlatformAdmin, plataformaExige, empresaExige})` | `boolean` (política) |
| `mfaEmDivida` | `()` | `boolean` (sessão) |
| `requiresMfa` | `(role, isPlatformAdmin, userId, orgId)` | `boolean` |
| `sessionAal` | `()` | `aal1 \| aal2 \| null` |

## Fluxo Principal
1. **Política (cadastrar):** `requiresMfa` carrega `platform_admins.mfa_required` e `organizations.settings.security.mfa_required`, delega a `exigeCadastroDeMfa` (as duas SOMAM; default não exige). 🟢
2. **Sessão (provar):** `mfaEmDivida` = `isMfaEnrolled && sessionAal ≠ 'aal2'`. Não consulta a política — quem tem fator prova sempre. 🟢
3. **Recovery:** 10 códigos de 8 chars, rejection sampling, sha256 em `user_recovery_codes`. 🟢

## Dependências
- Supabase Auth (`getAuthenticatorAssuranceLevel`), `platform_admins`, `organizations.settings`, `user_recovery_codes`.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Cadastro é política (soma), prova é sessão | `politica-mfa.ts`, `server.ts` | 🟢 |
| Quem tem fator prova sempre (fator opcional não vira buraco) | `server.ts` (`mfaEmDivida`) | 🟢 |
| `platform_admins.mfa_required` passou de decorativo a lido | `politica-mfa.ts` | 🟢 |

## Riscos e Lacunas
- 🟡 Aplicação do gate em rota de API vive em `requireRole` (ver sub-unit `rbac-requireRole`).
