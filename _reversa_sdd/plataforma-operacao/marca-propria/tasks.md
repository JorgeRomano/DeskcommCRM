# Caso de Uso: Marca Própria — Tarefas

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabela `platform_branding`; `organizations.settings.branding`; bucket `brand-logos`
- [ ] `REGUA_DO_PRODUTO` gerada no build a partir de globals.css

## Tarefas
- [ ] T-01, Implementar o núcleo de cor (rampa)
  - Origem no legado: `lib/branding/rampa.ts`
  - Critério de pronto: sRGB↔OKLab sem lib; clamp por redução de croma; semente no stop 600
  - Confiança: 🟢
- [ ] T-02, Implementar contraste e reconciliação
  - Origem no legado: `lib/branding/contraste.ts`
  - Critério de pronto: pisos WCAG; simular dicromacia; `escolherAccent`/`reconciliarSemanticas` nunca lançam
  - Confiança: 🟢
- [ ] T-03, Implementar resolvedor de camadas e serialização
  - Origem no legado: `lib/branding/resolve.ts`, `schema.ts`, `css.ts`, `instalacao.ts`, `saida.ts`, `logo.ts`
  - Critério de pronto: precedência por campo; guarda entrada; CSS fail-closed; logo por assinatura de byte; saídas sem DOM
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Cor da org vence por campo (não apaga camada de baixo)
- [ ] TT-02, Accent passa os pisos WCAG
- [ ] TT-03, CSS rejeita `<` e `;}`

## Ordem Sugerida
1. T-01 → T-02 → T-03.

## Lacunas Pendentes (🔴)
- `platform_branding` (Data Master).
