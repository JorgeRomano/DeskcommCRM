# Caso de Uso: Guardrails Before-Send — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `evaluateBeforeSend` | `(initial: GateContext, gates: Gate[])` | `{body, trace, veto, throttleWaitMs}` (puro) |
| `runBeforeSend` | `(args)` | `Promise<BeforeSendResult>` (stateful) |

`GateContext`: `{now, body, optedOut, provider, pacing, spinning, promise}`. 🟢

## Fluxo Principal
1. `runBeforeSend` paga o atraso humano (issue #654). 🟢
2. Pega `pg_advisory_xact_lock(hashtext(channelSessionId))` (serializa por número). 🟢
3. `evaluateBeforeSend` roda os 11 gates em ordem, curto-circuita no 1º veto, acumula throttle, aplica `amendBody`. 🟢
4. Grava trace durável em `before_send_traces`. 🟢
5. Em pass: dorme o throttle e envia pelo adapter. 🟢

## Detectores de Camada
- `vazamento-interno.ts` (determinístico), `human-promise.ts` (8 regex PT-BR), `sinal-de-urgencia.ts` (só prioriza), `messaging-window.ts` (fail-closed), `promise/` (engine determinístico + semantic LLM fail-open), `jailbreak/classifier.ts` (advisory, nunca veta sozinho). 🟢
- Camadas semânticas pagas: escolha três-estados (null/true/false) por org (`camadas-da-org.ts`), fail-open para default do ambiente. 🟢

## Dependências
- `pacing/engine`, `spinning/engine`, `disclosure/template`, `lgpd/legal-basis`, adapter de canal.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Cadeia versionada e declarativa | `before-send.ts` (`BEFORE_SEND_CHAIN_VERSION=7`) | 🟢 |
| Serialização por número via advisory lock | `before-send.ts` | 🟢 |
| Guardrails de saída não se desligam (só `semantic_promise`/`jailbreak` são opt-in) | `guardrails/lista-de-conferencia.ts` | 🟢 |

## Riscos e Lacunas
- 🟡 O gate `semantic_promise` custa +1 consulta por mensagem; ligado por org.
