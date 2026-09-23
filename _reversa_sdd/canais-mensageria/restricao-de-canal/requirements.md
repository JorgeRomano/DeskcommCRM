# Caso de Uso: Restrição de Canal (invariante)

> Sub-unit de `canais-mensageria`. A lei que rege a unit: features perguntam capabilities, nunca o provedor.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Garante que nenhuma feature fora de `lib/channels/` nomeie um provedor. O código pergunta *o que o canal permite* (capabilities), nunca *quem ele é*. Gate: `pnpm lint:channels`. 🟢

## Responsabilidades
- Expor `ChannelCapabilities` por provider via matriz. 🟢
- Resolver adapter/capabilities fail-closed. 🟢
- Garantir exaustividade de providers em tempo de compilação. 🟢

## Regras de Negócio
- `capabilitiesOf`/`getAdapter`/`transportaMensagem` fail-closed (`unknown_channel_provider`). 🟢
- Exaustividade garantida em compilação (`ProviderNaoClassificado extends never`). 🟢
- `wacalls` (voz) mora em `channel_sessions` mas NÃO transporta mensagem (distinção no tipo `ProviderDeMensagem`). 🟢
- `DEFAULT_CHANNEL_PROVIDER = "waha"` (conservador, banRisk armado). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Feature externa não nomeia provider | Must | `pnpm lint:channels` reprova nome de provider fora de `lib/channels/` |
| RF-02 | Resolver capabilities fail-closed | Must | Provider desconhecido lança |
| RF-03 | Exaustividade em compilação | Should | Novo provider sem classificação quebra o build |

## Critérios de Aceitação
```gherkin
Dado uma feature que precisa saber se pode enviar fora da janela
Quando consulta o canal
Então usa capabilities.freeformOutsideWindow (nunca "if provider === waha")
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/channels/types.ts` | `ChannelProvider`, `ChannelCapabilities`, `ChannelAdapter` | 🟢 |
| `lib/channels/capabilities.ts` | `capabilitiesOf`, `CHANNEL_CAPABILITIES` | 🟢 |
