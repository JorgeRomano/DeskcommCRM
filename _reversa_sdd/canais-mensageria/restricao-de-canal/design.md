# Caso de Uso: Restrição de Canal — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal
1. Feature chama `capabilitiesOf(provider)` (nunca compara com nome de provider). 🟢
2. Decide comportamento por capability (`freeformOutsideWindow`, `requiresTemplates`, `banRisk`, `minIntervalMs`, `voiceNote`, `groups`, `costPerMessage`). 🟢
3. Providers não classificados quebram em compilação (`ProviderNaoClassificado extends never`). 🟢

## Dependências
- Consumido por todo o resto do código (via capabilities). Gate `pnpm lint:channels` no CI.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Restrição de canal como invariante com gate de lint | doutrina + `capabilities.ts` | 🟢 |
| `wacalls` distinto no tipo (voz não transporta mensagem) | `channels/types.ts:31` | 🟢 |
| Fail-closed em provider desconhecido | `capabilities.ts:189` | 🟢 |

Ver ADR-0005 (restrição de canal — feature nunca nomeia provedor).

## Riscos e Lacunas
- 🟡 Ao adicionar provider novo, atualizar a matriz e a exaustividade juntas.
