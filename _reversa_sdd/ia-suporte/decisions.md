# IA de Suporte — Decisões

> Decisões arquiteturais desta unit. Referências: `_reversa_sdd/adrs/`.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## D-01 — Privacy-by-default por tenant (ZDR) 🟢
Toda chamada de gateway injeta `X-AI-Gateway-Zero-Retention` e `X-AI-Gateway-Tenant-Id`. Opt-out de treino é o default, não opção. — `gateway.ts:108`.

## D-02 — Resolução de modelo por ponto, sem fallback silencioso 🟢
Ordem: binding → credencial da org ativa+validada → padrão da instalação. Provider/chave inutilizável cai no padrão com `warn`. Chave da org nunca cruza com modelo de outro provider (freio PR #151). — `gateway-binding.ts`.

## D-03 — Custo em centavos com `Math.ceil` 🟢
Nunca subfatura; preços 0/0 devolvem `null` em vez de "de graça". Cache de preços TTL 5min. — `cost.ts`.

## D-04 — Orçamento nunca lança, lê a decisão do inbox 🟢
`getBudgetStatus` degrada para coluna materializada com log; `blocked_now` lê `agent_inbox_items kind='budget_exceeded'`, não recalcula. — `budget/check.ts`.

## D-05 — Embedding com modelo e dimensão fixos 🟢
`text-embedding-3-small` 1536 dims; length divergente lança. O binding governa a chave, não o modelo. — `embed.ts`, `embeddings/chave.ts`.

## D-06 — Versionamento de índice RAG por fonte 🟢
`activateVersion` desativa a anterior ANTES (índice único uma-ativa-por-fonte). — `rag/version.ts`.

## D-07 — CRM exposto como servidor MCP com RBAC 🟢
Pipeline por tool: higieniza UUID → ctx → ensureScope+ensureRole → handler → motivoDoVazio → audit. Actor nunca é `user` (evita quebrar FKs `_by_user_id` e furar gate `pre_go_live`). Ver ADR-0010 (CRM como servidor MCP).

## D-08 — Recusa para o modelo sem vazar vocabulário interno 🟢
`recusaDeCapacidadeParaOModelo` traduz falta de papel sem citar "role"/"permissão"; distingue `apenasHumano` de restrição acidental; não afrouxa o `ensureRole`. — `recusa-para-o-modelo.ts`.

## D-09 — Dispatcher legado isolado do caminho quente 🟡
`dispatcher/index.ts` é `@deprecated`; o runtime real é `lib/agent-engine`. Persiste só para orgs em modo externo (G6-02). Não confundir com o caminho quente.
