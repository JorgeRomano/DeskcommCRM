# Plataforma e Operação — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Exceção na resolução de marca 🟢
`resolverMarca` roda em `app/layout.tsx`; uma exceção seria 500 em toda tela, incl. login. Recusas voltam como `MotivoDaMarca` (dado, não log perdido); cor inválida em camada superior é anotada e a busca continua descendo.

## EC-02 — Rollback de imagem perdendo a marca 🟢
A marca guarda ENTRADA (nunca os 11 stops derivados); `algo`/`format` divergente re-deriva neste build. É o que mantém um rollback pintado e correções alcançando instalações existentes.

## EC-03 — Memo zerado pela rota nunca lido pela página 🟢
O Next instancia o módulo 2× por processo (route.js vs page.js); um memo `let` atrasava 19.6s. Memo em `globalThis` com contador de geração (lido antes do await, checado depois) evita lost-update.

## EC-04 — Placeholder do Dockerfile substituído estaticamente 🟢
`baseDoStorage()` lê `NEXT_PUBLIC_SUPABASE_URL` por chave MONTADA em runtime (`["NEXT","PUBLIC","SUPABASE","URL"].join("_")`) para derrotar a substituição estática do Next.

## EC-05 — Admin de tenant apagando o logo da instalação 🟢
`podeApagar` reafirma o prefixo no DELETE (service-role bypassa RLS), senão um admin de tenant apagaria o logo da instalação inteira.

## EC-06 — SVG ou tipo forjado no logo 🟢
Tipo decidido por assinatura de byte (`farejarTipo`: PNG `89 50 4E 47`, JPEG `FF D8 FF`), nunca por `file.type`; SVG banido com `logo_svg_recusado`.

## EC-07 — Reset morrendo pela metade (23503) 🟢
Três FKs para `contacts` são `ON DELETE RESTRICT`; os filhos precisam sair antes ou o Postgres devolve `23503`. `contacts` é sempre o último; não-atômico, mas a ordem mantém o banco íntegro e o reexecutar retoma.

## EC-08 — Config de retenção travando a org fora 🟢
`PORTAS_ESSENCIAIS` (profile, security, team, settings/tenant) não são removíveis: settings/tenant hospeda a escolha, então escondê-la trancaria a org fora.

## EC-09 — Interface inválida escondendo tudo 🟢
`lerInterface(raw)` é tolerante: inválido → COMPLETA (fail-OPEN); interseção vazia em `combinarInterfaces` → SO_O_ESSENCIAL.

## EC-10 — Rollback afirmado mas app novo de pé 🟢
`rollbackDesmentidoPeloApp`: se `APP_VERSION` do app rodando == `run.to_version`, a imagem nova está de pé (reinstalar mesma versão e funcionou). Lê `APP_VERSION` assado na imagem, nunca `APP_IMAGE`.

## EC-11 — Range de changelog com tag de fork 🟢
Seleção de range é POSICIONAL, não semver (o arquivo é mais-novo-primeiro; um comparador tropeçaria em `v1.1.1-jmpo.1`).

## EC-12 — Onboarding nascendo mudo 🟢
Medido: 312 etapas/43 funis, só 4 com `agent_stage_hint`. Uma etapa é NOME+PASSO inseparável; `is_won`/`is_lost` derivados do passo. Recusar proposta ruim cai num pacote curado.

## EC-13 — Modelo embrulhando JSON em prosa 🟢
`extrairJson` fatia do primeiro `{` ao último `}` (modelos embrulham em ```json/prosa); qualquer falha do pipeline → pacote curado com `porque`.
