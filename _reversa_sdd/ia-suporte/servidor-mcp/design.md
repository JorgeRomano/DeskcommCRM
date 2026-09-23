# Caso de Uso: Servidor MCP — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `createMcpServer` | `(auth: McpAuthResult, requestId)` | `McpServer` |
| `validateBearerToken` | `(authHeader)` | `Promise<McpAuthResult>` |
| `resolveApiToken` | `(plaintext)` | `Promise<ResolvedApiToken>` |
| `ensureRole` | `(actual, minimum)` | `void` (throws `McpAuthError`) |
| `higienizarUuidsDeAterro` | `(shape, args)` | `{limpos, descartados}` |

`McpContext`: `{organizationId, role, actor, apiTokenId, requestId, supabase (admin), meetingBooking?}`. 🟢

## Fluxo Principal (pipeline por tool)
1. `higienizarUuidsDeAterro(args)`. 🟢
2. Monta `McpContext` (org de `ctx`). 🟢
3. `ensureScope` (`mcp:read`/`mcp:write`) + `ensureRole` (via `ROLE_RANK`). 🟢
4. `tool.handler(args, ctx)`. 🟢
5. `motivoDoVazio` → `success` da auditoria. 🟢
6. `auditMcpToolCall` (fire-and-forget, `action='mcp.tool_called'`, redação de PII). 🟢
7. Devolve `content[]` + `structuredContent`. 🟢

## Fluxos Alternativos
- Falta de papel → `recusaDeCapacidadeParaOModelo` (texto sem vazar vocabulário interno).
- Token inválido → erros MCP `-32001/401`, `-32002/403`, `-32603/500`.

## Dependências
- `api_tokens`, `api_audit_log`, `@modelcontextprotocol/sdk`, admin client Supabase.
- Handlers REST `app/api/v1/*/_handler` (reuso da lógica de negócio).

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Actor nunca é `user` | `mcp/auth.ts` | 🟢 |
| Scopes convencionais sem migration | `mcp/auth.ts` | 🟢 |
| Org de `ctx`, nunca do arg (service-role) | `mcp/server.ts` | 🟢 |

## Estado Interno
- `last_used_at` do token atualizado fire-and-forget. 🟢

## Riscos e Lacunas
- 🟡 Superfície de ataque grande (~47 tools); RBAC por tool é a defesa principal — validar cada `requiresRole` na reimplementação.
