# ADR-0010 — CRM exposto como servidor MCP com RBAC próprio e "recusa para o modelo"

> ADR **retroativo** (Detetive/Reversa). Confiança: 🟢 CONFIRMADO.
> Evidência: `code-analysis.md` (2.6, 2.7, edge/crm/mcp-client), `lib/mcp/*`, commit `4221996a5` (separar núcleo de validação de api_tokens do protocolo MCP).

## Status
Aceito (vigente). Nota: o transporte MCP **HTTP externo está morto pós-fusão**; o client MCP hoje
carrega o admin do Supabase (service role). As tools MCP seguem sendo a superfície de capacidade do
agente.

## Contexto
O agente de IA precisa agir sobre o CRM (ler contatos, mover leads, agendar, escalar). Expor isso
como chamadas ad-hoc espalharia autorização e auditoria por todo lado. E o modelo, ao receber uma
recusa por falta de permissão, não pode ver "role"/"agent"/"403" — isso vaza arquitetura e ensina o
modelo a falar errado com o lead.

## Decisão
O CRM é exposto como **servidor MCP** (`createMcpServer`), com pipeline uniforme por tool:
higienizar UUID de aterro → montar `McpContext` (org de `ctx`, nunca do arg) → `ensureScope` +
`ensureRole` (via `ROLE_RANK`) → handler → `motivoDoVazio` → auditoria com redação de PII. Autentica
por Bearer `dsk_...` contra `api_tokens` (hash SHA256). Cada tool declara `category`
(read/write/handoff), `requiresRole` e `requiresScope`. **"Recusa para o modelo"**
(`recusaDeCapacidadeParaOModelo`) traduz a recusa por papel em texto sem vazar termos internos,
distinguindo `apenasHumano` (deliberado) de restrição acidental — sem afrouxar o `ensureRole`. Tools
do harness (`crm_send_whatsapp_message`, `crm_request_human_handoff`) são **bloqueadas** de entrar
como MCP genérico (`BLOCKED_TOOL_IDS`).

## Alternativas consideradas
1. **Chamadas ad-hoc do agente ao banco** — rejeitado: autorização e auditoria espalhadas; sem
   contrato estável de capacidade.
2. **Expor mensagem de erro crua ao modelo** — rejeitado: vaza arquitetura; ensina o modelo a
   responder com jargão técnico ao lead.
3. **Servidor MCP com RBAC + recusa traduzida (escolhida).**

## Consequências
- **Positivas:** superfície de capacidade uniforme e auditável (~47 handlers, sanity-check 1:1
  catálogo↔handler); RBAC consistente (`ROLE_RANK`); "não achei" ≠ sucesso (`motivoDoVazio`, issue
  #484); PII redigida na auditoria.
- **Negativas / custo:** superfície de ataque grande (complexidade alta); o service role bypassa RLS,
  então cada handler filtra `organization_id` manualmente; o núcleo de validação de `api_tokens`
  precisou ser separado do protocolo MCP (commit `4221996a5`) para não acoplar auth ao transporte.
- **Estado atual:** o transporte HTTP externo está morto pós-fusão — o "MCP" hoje é a organização
  das capacidades e sua autorização, não um endpoint remoto vivo.
