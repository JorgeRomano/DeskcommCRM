# IA de Suporte — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Provider/chave inutilizável 🟢
`resolverModeloDoPonto` cai no padrão da instalação com `logger.warn`, nunca fallback silencioso. `idParaOProvider` devolve `null` se o id é de outro provider (não cruza chave de org com modelo alheio).

## EC-02 — Preço do catálogo ausente 🟢
`precoDoCatalogo` devolve `null` se ambos os preços são 0 (não inventa "de graça"). `computeCost` usa `Math.ceil` (nunca subfatura).

## EC-03 — RPC de gasto falha 🟢
`getBudgetStatus` degrada da RPC `fn_gasto_de_ia_do_mes` para a coluna materializada `current_month_consumed_cents` com log; nunca lança.

## EC-04 — Furo de medição de custo 🟡
`gasto_incompleto = true` quando há `llm_calls.cost_cents is null` no mês; `ai_pricing` casa por prefixo, então o gasto medido pode ser menor que o real.

## EC-05 — Dimensão de embedding divergente 🟢
`embedText` assere `embedding.length === 1536`; length divergente lança (recall quebraria em silêncio).

## EC-06 — Debounce de indexação perdido 🟢
`acquireDebounce` falha ABERTA (catch → `true`): perder debounce custa uma indexação extra; perder o evento custaria o material inteiro.

## EC-07 — Busca sem resultado vs resultado fraco 🟢
`buscarConhecimento` usa piso −1 na RPC e filtra o limiar em memória para expor `melhorSimilaridade` (distingue "não há nada" de "há algo perto mas fraco").

## EC-08 — "Não achei" tratado como sucesso 🟢
`motivoDoVazio` (issue #484): resultado vazio não é sucesso; o motivo desce para `api_audit_log.metadata.motivo`.

## EC-09 — UUID de aterro forjado 🟢
`higienizarUuidsDeAterro` apaga a chave (não escreve `null`) só quando o schema aceita ausência; campo obrigatório mantém o valor para virar recusa nomeada. Detecta NIL/MAX/mascarado por nibbles.

## EC-10 — IA tentando retomar atendimento 🟢
`crm_resume_ai_attendance` recusa se `actor.type === 'ai_agent'` (`resume_requires_person`).

## EC-11 — "Loja não tem o produto" 🟢
`crm_search_products` só afirma isso após varredura paginada completa (1000×10); varredura parcial vira motivo `varredura_parcial`.

## EC-12 — OpenRouter validando contra endpoint público 🟢
Validação usa `/api/v1/key` (exige credencial); `/api/v1/models` (público) gravava `validated_at` em chave falsa — evitado.
