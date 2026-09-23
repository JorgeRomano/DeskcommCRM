# Núcleo de IA — Agente — Casos Extremos

> Casos extremos extraídos do código legado. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Falso-vazio ("não recebi sua mensagem") 🟢
O modelo às vezes afirma que o inbound veio vazio. `claimsCurrentInboundIsEmpty` (`inbound-turn.ts`) é uma guarda por regex que exige referência-a-mensagem + alegação-de-vazio dentro de 90 chars. O pronome `ela` foi **deliberadamente excluído** por falso positivo medido. Teto: `MAX_VETOS_DE_FALSO_VAZIO=2` (`:449`).

## EC-02 — Orçamento estoura no meio do turno 🟢
Chamadas auxiliares (`classifyStage`, `maybeCompact`) ocorrem ANTES da chamada principal e lançariam primeiro; por isso `comHandoffSeOrcamentoAcabar` envolve o turno inteiro. Em `LlmBudgetExceededError`: lê briefing do checkpoint durável (sem LLM), avisa o lead com texto de código, roda handoff e re-lança.

## EC-03 — Job já resolvido pelo próprio run 🟢
`JobSettledError` (`:466-469`): quando o run resolve o job (ex.: veto `is_blocked`), o worker não re-tenta.

## EC-04 — Número BR quebrando divisão em bolhas 🟢
`splitSentences` (`split-message.ts`) protege número BR: `.` entre dígitos não é fim de frase ("R$ 10.990" fica inteiro).

## EC-05 — Promessa de humano sem caso aberto 🟢
Gate `case_promise` veta com `case_promise_without_case` quando o texto promete um humano mas não há caso aberto.

## EC-06 — Vazamento de vocabulário interno 🟢
`detectarVazamentoInterno` bloqueia snake_case, nomes de tool MCP, palavras de arquitetura, papéis/acesso, HTTP 403, UUID v4, SQLSTATE e stack traces. Teto: `MAX_VETOS_DE_VOCABULARIO_INTERNO=2` (`:434`).

## EC-07 — Janela de mensageria fechada (fail-closed) 🟢
`messaging-window.ts`: janela derivada de 24h (`WINDOW_MS=24h`); valor `null` = fechada. Fora da janela e não-template → veto `messaging_window_closed`.

## EC-08 — Cópia idêntica em massa 🟢
Spinning: `sha256` exato + Jaccard ≥ threshold sobre janela (`windowSize 20`, `similarity 0.8`); `matchCount ≥ repetitionThreshold (2)` → veto `mass_identical`.

## EC-09 — Mídia que falha no parse 🟢
`media-parts.ts`: try/catch pula o item de mídia (nunca aborta o turno); só o inbound mais recente com mídia é incluído.

## EC-10 — Classificador de estágio falha 🟢
`classifyStage` degrada para `null` em falha do provider, EXCETO `LlmBudgetExceededError` (re-lança). PII de candidato golden-set vai a arquivo, nunca a log.

## EC-11 — Loop de tool sem progresso 🟢
Circuit breaker `idempotent_no_progress` para tools read-only (`get_lead_context`, `get_lead_note`, `search_knowledge`, `read_skill_reference`) quando o mesmo resultado se repete.

## EC-12 — Job morto após retries 🟢
`failJob`: backoff `power(2,attempts-1)*10` cap 120s; ao virar `dead`, cria item de inbox `job_dead`.
