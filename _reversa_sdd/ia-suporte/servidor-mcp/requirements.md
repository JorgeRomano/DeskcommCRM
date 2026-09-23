# Caso de Uso: Servidor MCP

> Sub-unit de `ia-suporte`. Expõe o CRM inteiro como ~47 tools para o modelo, com auth, RBAC, auditoria e recusa segura.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Servidor MCP (`SERVER_NAME="deskcomm-crm"`) que autentica por Bearer `dsk_`, resolve papel/actor por escopos convencionais, aplica RBAC por tool e audita com redação de PII. 🟢

## Responsabilidades
- Autenticar Bearer `dsk_` contra `api_tokens` (hash SHA256). 🟢
- Higienizar UUID de aterro antes do handler. 🟢
- Aplicar `ensureScope` + `ensureRole` por tool. 🟢
- Auditar cada chamada com redação de PII e `motivoDoVazio`. 🟢
- Traduzir recusa por papel sem vazar vocabulário interno. 🟢

## Regras de Negócio
- Actor é `ai_agent` ou `api_token`, nunca `user` (evita quebrar FKs `_by_user_id` e furar `pre_go_live`). 🟢
- `motivoDoVazio`: "não achei" ≠ sucesso; desce para `api_audit_log.metadata.motivo`. 🟢
- Auditoria redige `authorization/api_key/token/password/cpf`; strings >500 truncadas; `resource_id null`. 🟢
- `higienizarUuidsDeAterro` apaga a chave só se o schema aceita ausência; obrigatório mantém valor para recusa nomeada. 🟢
- Org sempre de `ctx`, nunca do arg; usa admin client (service-role). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Autenticar Bearer com hash e checagens | Must | Token revogado/expirado é recusado |
| RF-02 | Aplicar RBAC por tool | Must | Tool `manager` chamada por `agent` recusa |
| RF-03 | Auditar com redação de PII | Must | Chaves sensíveis redigidas; strings >500 truncadas |
| RF-04 | Higienizar UUID de aterro | Should | Campo opcional forjado é apagado; obrigatório vira recusa |

## Critérios de Aceitação
```gherkin
Dado um Bearer dsk_ expirado
Quando validateBearerToken avalia
Então recusa a autenticação (-32001/401)

Dado uma tool que exige manager chamada por um token de agent
Quando o pipeline avalia ensureRole
Então recusa e traduz para o modelo sem citar "role"
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/mcp/server.ts` | `createMcpServer` | 🟢 |
| `lib/mcp/auth.ts` | `validateBearerToken`, `resolveApiToken`, `deriveActor`, `ensureRole` | 🟢 |
| `lib/mcp/audit.ts` | `auditMcpToolCall` | 🟢 |
| `lib/mcp/recusa-para-o-modelo.ts` | `recusaDeCapacidadeParaOModelo` | 🟢 |
| `lib/mcp/uuid-de-aterro.ts` | `higienizarUuidsDeAterro`, `ehUuidDeAterro` | 🟢 |
