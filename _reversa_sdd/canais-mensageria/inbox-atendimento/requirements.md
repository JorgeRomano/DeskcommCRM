# Caso de Uso: Inbox e Fronteira de Atendimento

> Sub-unit de `canais-mensageria`. Decide quem comanda a conversa e protege a fronteira IA↔humano.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Máquina de estados do comando da conversa (quem atende) e a fronteira de serviço que impede efeitos sobre um serviço obsoleto. 🟢

## Responsabilidades
- Decidir o comando da conversa (`humano | automatico | ninguem | aguardando | encerrada`). 🟢
- Ordenar a fila de espera pelo inbound mais antigo sem resposta. 🟢
- Validar a fronteira de serviço (CAS por revision) antes de efeitos. 🟢

## Regras de Negócio
- `comandoDaConversa`: atribuído→humano; encerrada→encerrada; (silêncio|force_human|blocked)→aguardando; `automaticoDaOrg===false`→ninguem; else automatico. 🟢
- Precedência do motivo: bloqueado (`contato_descadastrado`) > `contato_travado` > silêncio da conversa. 🟢
- Dois gates deliberadamente FORA: janela 24h (é capability) e status fechado. 🟢
- `silencioVigente`: valor ilegível = SILENCIADO (fail-closed). 🟢
- `assertCurrentServiceBoundary` lança `StaleServiceBoundaryError` em qualquer divergência; abrir a 1ª demanda não é novo serviço. 🟢
- `ORDEM_DA_ESPERA` por `awaiting_since` (não `last_inbound_at`). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Decidir o comando da conversa | Must | Atribuição → humano; silêncio → aguardando |
| RF-02 | Precedência de motivo | Must | Bloqueado antes de travado antes de silêncio |
| RF-03 | Proteger a fronteira de serviço | Must | Divergência de revision lança `StaleServiceBoundaryError` |

## Critérios de Aceitação
```gherkin
Dado uma conversa com contato bloqueado e também travado
Quando comandoDaConversa avalia o motivo
Então retorna contato_descadastrado (bloqueado tem precedência)

Dado um efeito sobre uma demanda que já trocou
Quando assertCurrentServiceBoundary valida
Então lança StaleServiceBoundaryError
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/inbox/comando-da-conversa.ts` | `comandoDaConversa`, `silencioVigente`, `ORDEM_DA_ESPERA` | 🟢 |
| `lib/atendimento/fronteira-server.ts` | `assertCurrentServiceBoundary`, `guardServiceTools` | 🟢 |
