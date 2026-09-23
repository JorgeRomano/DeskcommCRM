# Caso de Uso: Borda e Sessão — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Supabase Auth configurado; RPC `fn_is_platform_admin`

## Tarefas
- [ ] T-01, Implementar o middleware de borda
  - Origem no legado: `proxy.ts`, `lib/auth/public-paths.ts`
  - Critério de pronto: correlação; `/api/*` 401 JSON; allowlist ancorada; impersonation no Edge
  - Confiança: 🟢
- [ ] T-02, Implementar resolução de sessão e org ativa
  - Origem no legado: `lib/auth/server.ts`
  - Critério de pronto: `getUser()`; falha alto; org ativa consistente entre dados e idioma
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, `/api/*` sem sessão → 401 JSON
- [ ] TT-02, Erro de permissões lança
- [ ] TT-03, Sub-path futuro não nasce público (allowlist `$`)

## Ordem Sugerida
1. T-02 (sessão) antes de T-01 (borda usa a mesma validação).

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
