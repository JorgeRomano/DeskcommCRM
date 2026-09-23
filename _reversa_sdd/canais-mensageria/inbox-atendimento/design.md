# Caso de Uso: Inbox e Fronteira de Atendimento — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `comandoDaConversa` | `(fatos, agora)` | `Comando` |
| `silencioVigente` | `(valor, agora)` | `boolean` (ilegível = silenciado) |
| `assertCurrentServiceBoundary` | `(expected, current)` | `void` (throws) |

`Comando` = `humano | automatico | ninguem | aguardando | encerrada`. `ServiceBoundary` = `{organization_id, contact_id, conversation_id, service_revision, demanda_id, demanda_revision}`. Constantes: `INFINITO="infinity"`, `MENOS_INFINITO="-infinity"`, `STATUS_ENCERRADOS={closed, archived, resolved}`. 🟢

## Fluxo Principal — Comando da conversa
1. Atribuído a humano → `humano`. 🟢
2. Status encerrado → `encerrada`. 🟢
3. Silêncio vigente | force_human | blocked → `aguardando` (com precedência de motivo). 🟢
4. `automaticoDaOrg === false` → `ninguem`. 🟢
5. Caso contrário → `automatico`. 🟢

## Fluxo Principal — Fronteira de serviço
- `fronteira-server.ts` usa `AsyncLocalStorage` para o contexto de execução; `guardServiceTools` deixa tools read-only passarem e aplica `guardServiceEffect` antes do `execute` nas demais. 🟢
- `demandaTrocou` retorna false quando `expected.demanda_id===null` (abrir a 1ª demanda não é novo serviço — 0222 só bump `service_revision` ao TROCAR de demanda). 🟢

## Dependências
- `ai/agents/operation`, `ai/replies/delivery`, `agenda`, `agent-engine/queue`.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Janela 24h e status fechado ficam FORA do comando (é capability) | `inbox/comando-da-conversa.ts` | 🟢 |
| `silencioVigente` fail-closed | `inbox/comando-da-conversa.ts` | 🟢 |
| CAS por `service_revision` na fronteira | `atendimento/fronteira-server.ts` | 🟢 |
| Ordem da espera por `awaiting_since` (issue #990/#994) | `inbox/comando-da-conversa.ts` | 🟢 |

## Riscos e Lacunas
- 🟡 Arquivos menores de `inbox/` e `atendimento/` lidos por referência nesta passagem.
