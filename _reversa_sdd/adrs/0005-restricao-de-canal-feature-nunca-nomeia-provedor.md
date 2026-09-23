# ADR-0005 — Invariante de restrição de canal: feature nunca nomeia o provedor

> ADR **retroativo** (Detetive/Reversa). Confiança: 🟢 CONFIRMADO.
> Evidência: `code-analysis.md` (Unidade 3), `docs/doctrine/restricao-de-canal.md`, gate `pnpm lint:channels`.

## Status
Aceito (vigente).

## Contexto
O produto fala por vários canais: WhatsApp via WAHA (QR), WhatsApp oficial (Meta Cloud), BSP
(Zernio) e voz (WaCalls). Cada um tem regras diferentes (janela 24h, templates, risco de ban, custo
por mensagem). Se cada feature espalhar `if (provider === "waha")` pelo código, adicionar um canal
novo vira uma caçada por condicionais, e regras de um provedor vazam para onde não deviam.

## Decisão
**Nenhuma feature fora de `lib/channels/` pode nomear um provedor.** Features perguntam *o que o
canal permite* (`ChannelCapabilities`: `freeformOutsideWindow`, `requiresTemplates`, `banRisk`,
`minIntervalMs`, `voiceNote`, `groups`, `costPerMessage`), nunca *quem ele é*. O adapter é **tradutor
de formato puro** — nenhuma lógica de janela/cap/horário nele (isso vive na cadeia `before_send`).
`capabilitiesOf`/`transportaMensagem` são **fail-closed** (lançam em provedor desconhecido), e a
exaustividade é garantida em tempo de compilação (`ProviderNaoClassificado extends never`). O gate
`pnpm lint:channels` reprova quem viola.

## Alternativas consideradas
1. **`switch (provider)` nas features** — rejeitado: acoplamento; canal novo toca N arquivos;
   regras vazam.
2. **Herança de classe por provedor** — rejeitado: capabilities são dados, não comportamento; matriz
   é mais legível e testável que hierarquia.
3. **Matriz de capabilities + adapter puro + gate de lint (escolhida).**

## Consequências
- **Positivas:** canal novo = uma linha na matriz + um adapter; features imunes; `wacalls`
  (voz) mora em `channel_sessions` mas é excluído de `ProviderDeMensagem` no TIPO, forçando cada
  site "qual canal?" a decidir explicitamente.
- **Negativas / custo:** exige disciplina e um gate de lint dedicado; a distinção capability-vs-
  provider às vezes obriga a modelar uma capability nova em vez de um `if` rápido.
- **Efeito medido:** mover os efeitos pós-entrada para trás de um seam corrigiu que eles só rodavam
  no WAHA (806 despachos no QR, 0 no oficial) — ver ADR-0006.
