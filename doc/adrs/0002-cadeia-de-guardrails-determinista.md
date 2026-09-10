# ADR 0002 — Cadeia de guardrails de saída determinística e versionada

> ADR retroativo · Status: **Aceito** · Confiança: 🟢 CONFIRMADO (`lib/agent-engine/guardrails/before-send.ts`)

## Contexto

Um agente de IA que fala com clientes por WhatsApp pode: banir o número (template em massa), violar a LGPD, prometer preço fora da tabela, prometer humano sem abrir caso, vazar vocabulário interno, ou responder fora da janela de 24h. Confiar no prompt para evitar tudo isso é frágil — medições em produção (tenant YADEA, `gpt-5.6-terra`) mostraram o modelo ignorando instruções de texto.

## Decisão

Interpor uma **cadeia determinística de gates entre a decisão do modelo (`send_message`) e o canal**, no estilo "exit-2 do Claude Code": cada gate pode VETAR, e a razão volta ao modelo como erro instrutivo (ele reescreve no turno seguinte). A ordem é **código-constante e versionada** (`BEFORE_SEND_GATES` / `BEFORE_SEND_CHAIN_VERSION` v6), vigiada por `tests/unit/before-send-chain-shape.test.ts`. Roda sob `pg_advisory_xact_lock` por número (serialização anti-race).

Ordem: stop → lgpd → pacing → janela → spinning → promise → semantic_promise → case_promise → internal_vocabulary → agenda_stall → disclosure.

## Alternativas consideradas

1. **Só instrução no prompt.** Rejeitada: medido que o modelo desobedece; "instrução é ensino, não garantia".
2. **Um classificador LLM decidindo se pode enviar.** Rejeitada para as camadas duras: custo/latência por envio e não-determinismo. Usado só como camada complementar (`semantic_promise`).
3. **Ordem configurável em runtime.** Rejeitada: "stop primeiro" é invariante de segurança; ordem mutável em disco seria um footgun. Mudar a ordem exige bumpar a versão e passar pelo teste de shape.

## Consequências

- **Positivas:** proteção auditável (cada veredito → `before_send_traces`, exportável por run); veto vira ensino, não silêncio; ordem congelada por teste.
- **Negativas:** complexidade alta; defaults assimétricos exigem cuidado (`internalVocabularyEnforced` ausente=DESARMADO no caminho sem modelo, senão vira drop silencioso; `spinningEnforced` ausente=ARMADO para proteger o número).
- **Regra de manutenção:** acrescentar gate = bumpar versão + atualizar o teste de shape (o comentário do arquivo já corrigiu "7 gates" quando eram 9, depois 10).
