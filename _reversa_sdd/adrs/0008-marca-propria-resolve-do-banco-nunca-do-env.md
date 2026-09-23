# ADR-0008 — Marca própria (white-label) resolve do banco, nunca do `.env`

> ADR **retroativo** (Detetive/Reversa). Confiança: 🟢 CONFIRMADO.
> Evidência: `code-analysis.md`, AGENTS.md (Marca própria), `tests/unit/branding.test.ts`, `lib/branding/*`.

## Status
Aceito (vigente).

## Contexto
O produto é open-source e **revendido**: quem instala numa VPS pode vendê-lo com o próprio nome. O
nome "Deskcomm" não pode aparecer para o usuário final de um revendedor. Além disso, a marca pode
variar **por organização** dentro da mesma instalação. Fixar o nome em constante ou só no `.env`
tornaria o white-label impossível ou dependente de reinstalação.

## Decisão
A marca **resolve do banco** em camadas (`platform_branding` → `organizations.settings.branding`),
com `APP_NAME`/`APP_LOGO_URL`/`APP_ACCENT_HEX` do `.env` servindo apenas como **semente e piso de
rollback**. **Código que alcança o usuário nunca escreve "Deskcomm"** — `tests/unit/branding.test.ts`
varre `app|components|lib|workers|hooks` e reprova (allowlist só encolhe). Fora do DOM (e-mail,
ícone, `issuer` do MFA) usa `marcaDaSaida()` (`lib/branding/saida.ts`). O resolvedor **nunca lança**
(roda em `app/layout.tsx`; um throw ali seria 500 em todas as telas). O PDF de LGPD **não** leva
marca: nomeia o controlador (`organizations.legal_name`) e o DPO.

## Alternativas consideradas
1. **Nome em constante/código** — rejeitado: impossibilita white-label.
2. **Marca só no `.env`** — rejeitado: não varia por organização; mudar exigiria mexer no ambiente
   da VPS (contra a doutrina de packaging).
3. **Resolução em camadas do banco, `.env` como semente (escolhida).**

## Consequências
- **Positivas:** white-label por instalação e por organização; instalador grava a marca no banco e
  o usuário já entra no idioma/marca certos; rollback pelo piso do `.env`.
- **Negativas / custo:** o resolvedor precisa ser à prova de falha (nunca lança) porque roda no
  layout raiz; um teste de varredura precisa policiar strings da marca no código, e a allowlist é
  um débito que só pode encolher.
- **Consistência:** documentos legais (LGPD) deliberadamente ignoram a marca e nomeiam a entidade
  jurídica — a marca é comercial, o controlador é legal.
