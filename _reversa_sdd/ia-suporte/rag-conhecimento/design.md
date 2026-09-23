# Caso de Uso: RAG e Conhecimento — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal
1. `chunkText`: split por parágrafo (`\n\n+`) → sub-split de parágrafo grande por sentença → overlap dos últimos 200 chars do chunk anterior. `computeContentHash` = SHA-256 hex. 🟢
2. `resolverChaveDeEmbedding`: escada binding → credencial OpenAI ativa+validada (desempate pela mais antiga) → gateway → `OPENAI_API_KEY` → `null`. 🟢
3. `embedText`: com gateway passa a string do modelo; sem gateway usa `createOpenAI(...).textEmbeddingModel`; assere length 1536. 🟢
4. Versão: `createKnowledgeVersion` (status `building`, `version_number=max+1`) → `markVersionReady`/`markVersionFailed` → `activateVersion` (desativa a anterior, aponta `active_kb_version_id`). 🟢
5. `buscarConhecimento`: embeda a pergunta (`ponto embedding_consultar`), RPC `fn_buscar_trechos_das_fontes` com `p_threshold = -1`, filtra o `limiar` em memória. `resolverAcervoDoAgente` usa a versão publicada com fallback legado. 🟢

## Fluxos Alternativos
- `formatProductForRag`: strip HTML + template rotulado PT-BR para produtos Nuvemshop.
- Debounce `acquireDebounce` (SET NX EX, TTL 2s) protege reindexação; fail-open.

## Dependências
- `@ai-sdk/openai`/gateway, Upstash Redis (debounce), Supabase (RPC + tabelas de KB).

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Piso −1 + limiar em memória | `knowledge/busca.ts:70` | 🟢 |
| Desativa-antes-de-ativar | `rag/version.ts:135` | 🟢 |
| Modelo/dimensão de embedding fixos | `embeddings/chave.ts`, `embed.ts` | 🟢 |

## Estado Interno
- Versões em `ai_kb_versions`; ponteiro em `ai_knowledge_sources.active_kb_version_id`. 🟢

## Riscos e Lacunas
- 🟡 `elegibilidade/` e `anonymize/` lidos só por nome nesta passagem.
