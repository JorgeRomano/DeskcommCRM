# Caso de Uso: RAG e Conhecimento

> Sub-unit de `ia-suporte`. Pipeline de conhecimento por tenant: ingestão → chunking → embedding → versão → busca.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Transforma fontes de conhecimento (FAQ, documento, conversas, catálogo) em índice versionado por fonte e responde consultas top-K com citações. 🟢

## Responsabilidades
- Fazer chunking (parágrafo → sentença → overlap 200 chars). 🟢
- Gerar embeddings com modelo fixo 1536 dims. 🟢
- Versionar o índice por fonte (só 1 ativa por fonte). 🟢
- Buscar top-K distinguindo vazio de fraco. 🟢

## Regras de Negócio
- `TIPOS_DE_FONTE`: `faq`, `documento`, `conversas`, `catalogo`; `canonizarTipoDeFonte` traduz legados. 🟢
- `activateVersion` desativa a anterior ANTES (índice único `ai_kbv_uma_ativa_por_fonte`). 🟢
- Debounce de indexação falha ABERTA. 🟢
- Busca com piso −1 na RPC + limiar em memória expõe `melhorSimilaridade`. 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Chunking determinístico com overlap | Must | `chunkText` respeita `maxChars=1500`, `overlapChars=200` |
| RF-02 | Embedding com dimensão fixa | Must | Length ≠ 1536 lança |
| RF-03 | Versionamento por fonte | Must | Só 1 versão ativa por fonte após `activateVersion` |
| RF-04 | Busca distingue vazio de fraco | Should | `melhorSimilaridade` reflete melhor candidato reprovado |

## Critérios de Aceitação
```gherkin
Dado uma nova versão de índice para uma fonte
Quando activateVersion é chamada
Então a versão anterior é desativada antes de a nova ser ativada

Dado uma busca sem trecho acima do limiar
Quando buscarConhecimento executa
Então retorna trechos vazios mas melhorSimilaridade preenchido
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/ai/rag/chunker.ts` | `chunkText`, `computeContentHash` | 🟢 |
| `lib/ai/embed.ts` | `embedText` | 🟢 |
| `lib/ai/rag/version.ts` | `createKnowledgeVersion`, `activateVersion` | 🟢 |
| `lib/ai/knowledge/busca.ts` | `buscarConhecimento` | 🟢 |
