# Caso de Uso: Marca Própria — Design

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo Principal
1. `resolverMarca(camadas, regua)`: `primeiroDefinido` por campo (org → instalação → env); cor inválida anotada, busca continua. 🟢
2. `derivarMarca`/`rampaDeSemente`: 11 stops OKLab; semente ancorada no stop 600; `oklchParaHex` clampa gamut por redução de croma (não clipping). 🟢
3. `escolherAccent`: desloca a rampa inteira por offset; fallback "menos ruim" nunca lança. `reconciliarSemanticas` rotaciona success/warning/error/info sob dicromacia (≤60°). 🟢
4. `cssDaMarca`: allowlist de FORMA de valor; `:root:root` (0,2,0) vence globals.css; stops já deslocados. 🟢
5. Cache `WeakMap<Regua, Map<seed, Marca>>`, FIFO teto 64. 🟢

## Fluxos Alternativos
- Semente acromática (C < LIMIAR) mantém a rampa do produto; hex vai para `--color-brand` (`marca_acromatica`). 🟢
- Saídas sem DOM (`saida.ts`): tema claro sempre; degrada para o padrão do produto; PDF de LGPD nomeia o controlador. 🟢

## Dependências
- `schema`, `contraste`, `rampa`, `regua-do-produto`, `lib/instalacao/config`, `lib/supabase/admin`.

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Semente ancorada por papel (stop 600) | `rampa.ts` | 🟢 |
| Clamp de gamut por redução de croma | `rampa.ts` (`oklchParaHex`) | 🟢 |
| CSS fail-closed | `css.ts` | 🟢 |
| Guarda entrada, nunca saída | `schema.ts` | 🟢 (ADR-0008) |

## Riscos e Lacunas
- 🟡 `platform_branding` e a rampa do produto congelada no build vivem fora deste TS — validar no Data Master/build.
