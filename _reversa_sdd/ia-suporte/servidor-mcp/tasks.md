# Caso de Uso: Servidor MCP — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas `api_tokens`, `api_audit_log`; admin client Supabase
- [ ] Handlers REST reutilizáveis em `app/api/v1/*/_handler`

## Tarefas
- [ ] T-01, Implementar auth Bearer e resolução de token/actor
  - Origem no legado: `lib/mcp/auth.ts`
  - Critério de pronto: hash SHA256; checa revoked/expired; actor `ai_agent`/`api_token`, nunca `user`; `ensureRole` via `ROLE_RANK`
  - Confiança: 🟢
- [ ] T-02, Implementar o pipeline por tool
  - Origem no legado: `lib/mcp/server.ts`, `types.ts`
  - Critério de pronto: higieniza UUID → ctx → ensureScope+ensureRole → handler → motivoDoVazio → audit → content[]+structuredContent
  - Confiança: 🟢
- [ ] T-03, Implementar auditoria com redação e recusa para o modelo
  - Origem no legado: `lib/mcp/audit.ts`, `recusa-para-o-modelo.ts`, `uuid-de-aterro.ts`
  - Critério de pronto: redige PII; trunca >500; recusa sem vazar vocabulário; UUID de aterro higienizado
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Token revogado/expirado recusado
- [ ] TT-02, RBAC recusa tool acima do papel
- [ ] TT-03, Auditoria redige chaves sensíveis
- [ ] TT-04, UUID de aterro: opcional apagado, obrigatório vira recusa

## Ordem Sugerida
1. T-01 → T-02 → T-03. As ~47 tools (ver `../tasks.md` T-07) vêm depois do núcleo.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante; revisar cada `requiresRole` por tool na reimplementação.
