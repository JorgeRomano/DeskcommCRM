# Caso de Uso: Marca Própria (white-label)

> Sub-unit de `plataforma-operacao`. Resolve a marca do revendedor/tenant do banco, deriva rampa de cor e serializa CSS, sem lançar.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Motor de marca que resolve nome/logo/cor de destaque em três camadas (org → instalação → `.env`), deriva uma rampa de 11 tons com contraste WCAG e correção de dicromacia, e serializa para CSS. Roda em `app/layout.tsx` e nunca lança. 🟢

## Responsabilidades
- Resolver a marca por CAMPO nas três camadas. 🟢
- Derivar rampa OKLab, accent com contraste e semânticas reconciliadas. 🟢
- Serializar CSS fail-closed e resolver logo por assinatura de byte. 🟢

## Regras de Negócio
- Nunca lança; recusas voltam como `MotivoDaMarca`. 🟢
- Guarda ENTRADA, nunca SAÍDA (correções alcançam instalações existentes). 🟢
- Semente ancorada no stop 600 (papel), não pela lightness (navy não vira periwinkle). 🟢
- PDF/e-mail/MFA usam `marcaDaSaida` (tema claro sempre); PDF de LGPD não leva marca (nomeia o controlador). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Resolver sem lançar | Must | Cor inválida anota motivo e continua descendo |
| RF-02 | Rampa com contraste WCAG | Must | Accent passa pisos texto 4.5 / componente 3.0 |
| RF-03 | CSS fail-closed | Must | Rejeita `<` e `;}`; allowlist de forma de valor |

## Critérios de Aceitação
```gherkin
Dado uma cor de marca só na camada da organização
Quando resolverMarca compõe as camadas
Então a cor da org vence por campo, sem apagar as de baixo
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/branding/resolve.ts` | `resolverMarca`, `camadaDaOrganizacao` | 🟢 |
| `lib/branding/rampa.ts` | `rampaDeSemente` | 🟢 |
| `lib/branding/contraste.ts` | `derivarMarca`, `escolherAccent` | 🟢 |
| `lib/branding/css.ts` | `cssDaMarca` | 🟢 |
| `lib/branding/saida.ts` | `marcaDaSaida` | 🟢 |
