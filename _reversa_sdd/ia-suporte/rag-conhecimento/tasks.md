# Caso de Uso: RAG e Conhecimento — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas `ai_knowledge_sources`, `ai_kb_versions` e RPC `fn_buscar_trechos_das_fontes`
- [ ] Upstash Redis para debounce

## Tarefas
- [ ] T-01, Implementar chunking e content hash
  - Origem no legado: `lib/ai/rag/chunker.ts`
  - Critério de pronto: parágrafo→sentença→overlap 200; SHA-256 hex
  - Confiança: 🟢
- [ ] T-02, Implementar resolução de chave e embedding fixo
  - Origem no legado: `lib/ai/embeddings/chave.ts`, `embed.ts`
  - Critério de pronto: escada de chave; assere length 1536
  - Confiança: 🟢
- [ ] T-03, Implementar versionamento por fonte
  - Origem no legado: `lib/ai/rag/version.ts`
  - Critério de pronto: `activateVersion` desativa a anterior antes; 1 ativa por fonte
  - Confiança: 🟢
- [ ] T-04, Implementar busca com piso/limiar e debounce
  - Origem no legado: `lib/ai/knowledge/busca.ts`, `rag/debounce.ts`
  - Critério de pronto: piso −1 + limiar em memória; debounce fail-open
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Chunking respeita overlap
- [ ] TT-02, Embedding com length divergente lança
- [ ] TT-03, Só 1 versão ativa por fonte
- [ ] TT-04, Busca fraca expõe melhorSimilaridade

## Ordem Sugerida
1. T-01 → T-02 → T-03 → T-04.

## Lacunas Pendentes (🔴)
- Nenhuma bloqueante.
