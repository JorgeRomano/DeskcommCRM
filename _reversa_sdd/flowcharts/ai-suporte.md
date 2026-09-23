# Fluxogramas — IA de suporte (`lib/ai/` + `lib/mcp/`)

> 🟢 CONFIRMADO salvo indicação. Fontes: `lib/ai/**` e `lib/mcp/**`.
> Nível `detalhado`: um fluxo por módulo + fluxos por função principal com lógica não-trivial.

## Visão geral da unidade

```mermaid
flowchart LR
  subgraph AI[lib/ai]
    GW[gateway + gateway-binding]
    CAT[classifier-models / cost / credentials / validators]
    BUD[budget/check + usage/aggregate]
    RAG[rag: chunker/version/debounce + embed + knowledge/busca]
    AG[guardrails-schema + render-system-prompt + agents]
    DISP[dispatcher @deprecated]
  end
  subgraph MCP[lib/mcp]
    SRV[server]
    AUTH[auth]
    AUD[audit]
    TOOLS[tools/* ~47 handlers]
  end
  MODELO[Modelo LLM] --> TOOLS
  TOOLS --> SRV
  SRV --> AUTH
  SRV --> AUD
  GW --> MODELO
  RAG --> MODELO
  AG --> MODELO
  BUD -.gate.-> MODELO
```

## `resolveLanguageModel` — fallback de provider (`gateway.ts:74`)

```mermaid
flowchart TD
  A[model: ModelId] --> B{AI_GATEWAY_API_KEY?}
  B -- sim --> B1[devolve a string; SDK Vercel roteia] --> Z[LanguageModel]
  B -- nao --> C{OPENROUTER_API_KEY?}
  C -- sim --> C1[createOpenAI base OpenRouter] --> Z
  C -- nao --> D{prefixo anthropic/ e ANTHROPIC_API_KEY?}
  D -- sim --> D1[createAnthropic id sem prefixo] --> Z
  D -- nao --> E{prefixo openai/ e OPENAI_API_KEY?}
  E -- sim --> E1[createOpenAI id sem prefixo] --> Z
  E -- nao --> F[null: nenhum provider configuravel]
```

## `resolverModeloDoPonto` — binding → credencial → padrão (`gateway-binding.ts:60`)

```mermaid
flowchart TD
  A[purpose + organizationId + padrao] --> B{binding habilitado em ai_purpose_bindings?}
  B -- sim --> C[decifra credencial do binding aes_gcm]
  C --> D{provider conhecido e chave utilizavel?}
  D -- sim --> E[instanciar provider: origem=binding] --> Z[ModeloResolvido]
  D -- nao --> W[logger.warn + cai no padrao]
  B -- nao --> F{credencial da org ativa+validada?}
  F -- sim --> F1[origem=credencial_da_organizacao] --> Z
  F -- nao --> W
  W --> G{padraoDaInstalacao disponivel?}
  G -- sim --> G1[origem=padrao] --> Z
  G -- nao --> H[null]
```

## `getBudgetStatus` — nunca lança, degrada (`budget/check.ts:170`)

```mermaid
flowchart TD
  A[organization_id] --> B[le limite + modo de enforcement]
  B --> C{RPC fn_gasto_de_ia_do_mes ok?}
  C -- sim --> D[consumed = retorno da RPC]
  C -- nao --> E[log + fallback coluna current_month_consumed_cents]
  D --> F[pct = round consumed*10000/limit /100]
  E --> F
  F --> G{existe agent_inbox_items budget_exceeded status=open?}
  G -- sim --> H[blocked_now=true]
  G -- nao --> I[blocked_now=false]
  H --> J{ha llm_calls cost_cents null no mes?}
  I --> J
  J -- sim --> K[gasto_incompleto=true]
  J -- nao --> L[gasto_incompleto=false]
  K --> M[BudgetStatus]
  L --> M
```

## RAG — ingestão e versão por fonte (`rag/version.ts` + `rag/chunker.ts` + `embed.ts`)

```mermaid
flowchart TD
  A[material da fonte] --> B[acquireDebounce SET NX EX; falha ABERTA]
  B --> C[createKnowledgeVersion status=building version_number=max+1]
  C --> D[chunkText: paragrafo -> sentenca -> overlap 200]
  D --> E[computeContentHash SHA-256 por chunk]
  E --> F[embedText: assere length===1536 senao lanca]
  F --> G{tudo indexado?}
  G -- sim --> H[markVersionReady chunkCount]
  H --> I[activateVersion: desativa anterior ANTES; aponta active_kb_version_id]
  G -- erro --> J[markVersionFailed errorMessage]
  I --> K[releaseDebounce]
  J --> K
```

## `buscarConhecimento` — piso vs limiar (`knowledge/busca.ts:70`)

```mermaid
flowchart TD
  A[pergunta + knowledgeSourceIds + topK + limiar] --> B[embedText ponto=embedding_consultar]
  B --> C[RPC fn_buscar_trechos_das_fontes com p_threshold=PISO=-1]
  C --> D[ordena por similaridade desc]
  D --> E{trecho.similarity >= limiar?}
  E -- sim --> F[entra em trechos]
  E -- nao --> G[registra melhor candidato reprovado]
  F --> H[ResultadoDaBusca trechos + melhorSimilaridade]
  G --> H
```

## Servidor MCP — pipeline por chamada de tool (`server.ts:37`)

```mermaid
flowchart TD
  A[Modelo chama tool com args] --> B[higienizarUuidsDeAterro args]
  B --> C[monta McpContext org/role/actor/supabase admin]
  C --> D{ensureScope mcp:read|mcp:write ok?}
  D -- nao --> X[McpAuthError -32002/403] --> AUD
  D -- sim --> E{ensureRole >= requiresRole?}
  E -- nao --> X
  E -- sim --> F[tool.handler args, ctx]
  F --> G{handler lancou erro?}
  G -- sim --> H[content isError + code no metadata] --> AUD
  G -- nao --> I[motivoDoVazio? define success da auditoria]
  I --> AUD[auditMcpToolCall redige PII + trunca >500]
  AUD --> J[retorna content[] + structuredContent]
```

## `validateBearerToken` / `resolveApiToken` — auth MCP (`auth.ts:196`)

```mermaid
flowchart TD
  A[Authorization: Bearer dsk_prefix_secret] --> B{formato valido?}
  B -- nao --> E1[ApiTokenError malformed -32001/401]
  B -- sim --> C[hash SHA256 -> lookup token_hash em api_tokens]
  C --> D{encontrado?}
  D -- nao --> E2[not_found -32001/401]
  D -- sim --> F{revoked_at ou expires_at vencido?}
  F -- sim --> E3[revoked/expired -32001/401]
  F -- nao --> G[deriveActor scopes: ai_agent ou api_token, nunca user]
  G --> H[update last_used_at fire-and-forget]
  H --> I[McpAuthResult org/role/actor/scopes]
```

## `crm_search_products` — varredura paginada com motivo de vazio (`tools/comercio.ts`)

```mermaid
flowchart TD
  A[termo de busca] --> B[pagina=0; TAMANHO_DA_PAGINA=1000; count exact]
  B --> C[SELECT produtos ORDER BY codigo estavel]
  C --> D[buscarComRelaxamento: pontua em memoria]
  D --> E{alcancou o fim OU pagina >= PAGINAS_MAXIMAS=10?}
  E -- nao --> F[pagina++] --> C
  E -- sim --> G{houve resultado?}
  G -- sim --> H[topo ordenado; avisos: relaxamento + empate]
  G -- nao --> I{varredura completa?}
  I -- sim --> J[motivo=nao_encontrado / sem_estoque]
  I -- nao --> K[motivo=varredura_parcial]
  H --> L[structuredContent]
  J --> L
  K --> L
```

## `higienizarUuidsDeAterro` — UUID inventado pelo modelo (`uuid-de-aterro.ts:120`)

```mermaid
flowchart TD
  A[args + shape do schema] --> B[para cada campo uuid]
  B --> C{ehUuidDeAterro? NIL/MAX/mascarado}
  C -- nao --> B
  C -- sim --> D{schema aceita o campo ausente? safeParse undefined}
  D -- sim --> E[APAGA a chave; registra descartado]
  D -- nao --> F[mantem valor -> vira recusa nomeada no handler]
  E --> B
  F --> B
```
