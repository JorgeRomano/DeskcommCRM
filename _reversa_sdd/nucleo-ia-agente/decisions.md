# Núcleo de IA — Agente — Decisões

> Decisões arquiteturais desta unit (ADR-style local). Referências cruzadas: `_reversa_sdd/adrs/`.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## D-01 — Sessão LLM fresca por job, estado no closure 🟢
Cada job vira uma sessão nova; todo o estado do run vive no closure da invocação. Consequência: isolamento entre leads por construção, sem estado compartilhado entre runs. Ver ADR-0003 (ritual do turno e fechamento por checkpoint).

## D-02 — Fechamento por 2ª chamada de checkpoint, não por tool 🟢
A chamada de fechamento (`purpose:'checkpoint'`) sempre acontece; uma tool dependeria de o modelo lembrar de chamá-la. Trade-off: uma chamada extra de modelo por turno em troca de checkpoint garantido. — `inbound-turn.ts:573-607`.

## D-03 — Enviar é sempre `send_message` (nunca texto direto) 🟢
Texto direto do modelo é descartado. Garante que todo envio passa pela cadeia before-send e pela guarda de teto. Ver ADR-0004 (guardrails determinísticos before-send).

## D-04 — Cadeia before-send declarativa e versionada 🟢
`BEFORE_SEND_CHAIN_VERSION = 7`; 11 gates avaliados em ordem, curto-circuito no 1º veto, throttle acumulado. `runBeforeSend` serializa read-then-act por número via `pg_advisory_xact_lock(hashtext(channelSessionId))` e paga o atraso humano ANTES de tomar conexão (issue #654).

## D-05 — Validação de tool por whitelist, erro vira ensino 🟢
Schemas largos para o SDK, whitelist `.strict()` na aplicação. Campo extra/forjado não é stripado em silêncio: vira mensagem de ensino ao modelo.

## D-06 — Fila com claim de dois estágios e exactly-once 🟢
`DISTINCT ON (coalesce(contact_id,id))` (uma lane por vez) + `FOR UPDATE SKIP LOCKED` sob `pg_advisory_xact_lock(CLAIM_LOCK_KEY)` para `maxConcurrency`. `completeJob` com guard de lease garante efeito exactly-once. Ver ADR-0002 (event sourcing leve).

## D-07 — Orçamento nunca bloqueia sem aviso prévio 🟢
`decidirOrcamento` com escapes ordenados (modo off, kill switch, purpose isento, teto≤0, teto<piso) e limiar (`LIMIAR_PADRAO_PCT=80`). Purposes isentos: `connection_test`, `jailbreak_detect`, `promise_semantic`.

## D-08 — Operador separado do falante por AUSÊNCIA de tool 🟢
`operator-turn.ts` não tem `send_message` no toolset: a separação FALAR/OPERAR é imposta pela ausência da capacidade, não por convenção.

## D-09 — Adapter de canal nunca fala com o provedor direto 🟢
`WahaChannelAdapter.sessionHealth` lê o espelho `channel_session_health`; nunca consulta o WAHA diretamente. Ver ADR-0005 (restrição de canal).
