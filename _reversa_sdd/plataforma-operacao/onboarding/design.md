# Caso de Uso: Onboarding — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal
1. `passosVisiveis(ctx)` / `proximoPasso(state, ctx)`: passos = welcome, connect-whatsapp, connect-nuvemshop, setup-ai, funil, testar, invite-team; cada passo decide a própria existência. 🟢
2. `sugerirFunil(ctx, gerar)`: `escolherPacotePorTexto` → `gerar` (único ponto de rede, injetado) → `extrairJson` → `normalizarProposta` → `validarProposta`. 🟢
3. Qualquer falha → pacote curado (`PACOTES`: clinica, imobiliaria, servicos, curso, loja, generico) com `porque`. 🟢
4. `etapasParaGravar`: `position = (i+1)*1000`; `is_won`/`is_lost` derivados do passo. 🟢

## Dependências
- Mesmo cérebro/modelo/chave do agente publicado; `NAV_DESTINATIONS` (para `o-que-mais-existe`).

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Passo decide a própria existência | `onboarding/passos.ts` | 🟢 |
| Pacote é plano B e régua da sugestão | `onboarding/pacotes-de-funil.ts` | 🟢 |
| Recusar > auto-completar | `onboarding/proposta-de-funil.ts` | 🟢 |
| `PACOTE_PADRAO` por id, lança no load se ausente | `pacotes-de-funil.ts` | 🟢 |

## Riscos e Lacunas
- 🟡 `sugerirFunil` faz chamada de rede real (comportamento sob erro inferido).
