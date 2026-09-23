# Caso de Uso: Borda e Sessão

> Sub-unit de `auth-tenancy-rbac`. Middleware de borda + resolução de sessão/organização ativa.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
O `proxy.ts` valida o JWT na borda e injeta correlação; `loadAuthUser`/`resolveActiveOrg` resolvem quem é e em que empresa está, com falha alto em erro de infra. 🟢

## Responsabilidades
- Injetar `X-Request-Id`/`x-pathname` e validar o JWT na borda. 🟢
- Decidir quais paths o proxy não avalia (`public-paths.ts`). 🟢
- Resolver a organização ativa (a mesma resposta para escopo de dados e idioma). 🟢

## Regras de Negócio
- `/api/*` sem sessão → 401 JSON (nunca redirect HTML). 🟢
- `isPublicPath` não significa "sem auth", significa "o proxy não decide". 🟢
- `loadAuthUser` falha alto (`auth_permissions_unavailable`). 🟢
- `escolherMembroAtivo`: cookie `active_org` válido, senão `memberships[0]` (só ESCOLHA porque a query ordena). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Validar JWT na borda | Must | Sessão null → 401 JSON em `/api/*`, redirect na UI |
| RF-02 | Allowlist ancorada | Must | Padrões com `$` para sub-path não nascer público de carona |
| RF-03 | Resolver org ativa consistente | Must | Escopo de dados e idioma usam a mesma resposta |

## Critérios de Aceitação
```gherkin
Dado uma requisição a /api/* sem sessão
Quando o proxy avalia
Então responde 401 unauthenticated em JSON
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `proxy.ts` | middleware de borda | 🟢 |
| `lib/auth/public-paths.ts` | `isPublicPath` | 🟢 |
| `lib/auth/server.ts` | `loadAuthUser`, `escolherMembroAtivo`, `resolveActiveOrg` | 🟢 |
