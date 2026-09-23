# Caso de Uso: Guardrails Before-Send

> Sub-unit de `nucleo-ia-agente`. A costura determinística entre a decisão `send_message` do modelo e o canal.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Cadeia declarativa e versionada (`BEFORE_SEND_CHAIN_VERSION = 7`) de 11 gates que decide, para cada envio, se ele passa, é alterado, é adiado (throttle) ou vetado. 🟢

## Responsabilidades
- Avaliar os 11 gates em ordem fixa, curto-circuitando no 1º veto. 🟢
- Acumular `throttleWaitMs` e aplicar `amendBody` (reescritas). 🟢
- Serializar read-then-act por número (advisory lock) e gravar trace durável. 🟢

## Regras de Negócio
- Qualquer gate pode vetar; após o 1º veto, os demais ficam `skipped`. 🟢
- Atraso humano é pago ANTES de tomar conexão (issue #654). 🟢
- `aviso-de-escalacao` é o único caller que desarma spinning (`enforceSpinning:false`). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Avaliar os 11 gates em ordem e curto-circuitar no veto | Must | Contato `optedOut` → veto `contato_bloqueado`, gates seguintes `skipped` |
| RF-02 | Serializar por número via advisory lock | Must | Dois envios ao mesmo número não intercalam read-then-act |
| RF-03 | Gravar trace em `before_send_traces` | Should | Cada avaliação persiste seu trace |

## Regras dos Gates (ordem)
| # | Gate | Veto/efeito |
|---|------|-------------|
| 1 | `stop` | `contato_bloqueado` se `optedOut` |
| 2 | `lgpd` | `lgpd_anonymized` / `lgpd_missing_legal_basis` |
| 3 | `pacing` | throttle `waitMs` (espera, não veto) |
| 4 | `messaging_window` | `messaging_window_closed` |
| 5 | `spinning` | `mass_identical` |
| 6 | `promise` | promessa fora da tabela (preço/desconto/parcelas) |
| 7 | `semantic_promise` | `promise_semantic` (LLM assíncrono) |
| 8 | `case_promise` | `case_promise_without_case` |
| 9 | `internal_vocabulary` | `internal_vocabulary_leak` |
| 10 | `agenda_stall` | `agenda_stall_sem_ferramenta` |
| 11 | `disclosure` | 1º outbound sem disclosure (inject ou veto) |

## Critérios de Aceitação
```gherkin
Dado um contato opted-out
Quando runBeforeSend avalia o envio
Então o gate stop veta com contato_bloqueado
E os gates 2..11 ficam skipped
E nenhum envio ocorre
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/agent-engine/guardrails/before-send.ts` | `evaluateBeforeSend`, `runBeforeSend` | 🟢 |
