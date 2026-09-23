# ADR-0009 — Packaging: só imagem publicada; bump de versão nunca edita arquivo na VPS

> ADR **retroativo** (Detetive/Reversa). Confiança: 🟢 CONFIRMADO.
> Evidência: AGENTS.md (Packaging), `docs/doctrine/packaging.md`, commits `7ce379674`/`f025a62b6` (arquitetura sem imagem), `hostgator-setup-kit/`.

## Status
Aceito (vigente).

## Contexto
O produto é distribuído como código e roda numa VPS operada pelo cliente. Se um serviço do
`docker-compose.prod.yml` **construir** na máquina do cliente, a instalação fica cara e frágil
(build ARM local não roda na VPS amd64), e — pior — `docker compose pull` **pula** serviço `build:`-
only, de modo que ele **nunca é atualizado**. Qualquer passo que exija o operador editar um arquivo
à mão na VPS é uma fonte garantida de instalação quebrada.

## Decisão
- **Nenhum serviço de `docker-compose.prod.yml` constrói na máquina do cliente.** Todo serviço
  declara `image:` de imagem publicada; `build:` só existe **ao lado**, como escape.
- **Publicação é ato do CI** (`.github/workflows/publish-image.yml`), nunca da máquina do dev.
- **Instalação aponta para número de versão**; `latest` = topo da `main`, `stable` = última release.
- **Dependência upstream referenciada com tag fixa, nunca republicada** (WAHA é licenciado).
- **Bump de versão não pode exigir edição manual de arquivo na VPS.**
- `pnpm test:shell` é o **único** gate que exercita o kit (`install.sh`, `update.sh`).

## Alternativas consideradas
1. **Build na VPS do cliente** — rejeitado: lento, frágil, e `build:`-only nunca atualiza.
2. **Tag móvel (`latest`) na instalação** — rejeitado: instalação não-reproduzível; update
   imprevisível. `latest` reservado para "topo da main".
3. **Republicar imagem upstream (WAHA)** — rejeitado: licenciamento; referência por tag fixa.
4. **Imagens publicadas pelo CI + versão fixa na instalação (escolhida).**

## Consequências
- **Positivas:** instalação reproduzível; `update.sh` puxa imagem publicada; multi-arquitetura
  resolvido no CI (a VPS de outra arquitetura "se vira sozinha" — commit `7ce379674`).
- **Negativas / custo:** disciplina de release (fragmentos em `.changes/`, `release:conferir`,
  `release:cortar`); toda mudança de schema precisa entrar como **apêndice idempotente** em
  `supabase/baseline.sql` (senão não chega em quem instalou) além da migration versionada + MANIFEST.
- **Consequência de produto:** uma mudança que funciona no dev e quebra no clone fresco é **bug de
  produto** (a VPS do cliente é o ambiente de verdade), não detalhe de ambiente.
