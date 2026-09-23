# Compliance — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Trilha inteira parando em silêncio 🟢
Toda tool MCP falhava ao auditar por "invalid input syntax for type uuid" e o único sinal era um console dentro do contêiner. `audit()` agora reporta ao Sentry (`reportAuditFailure`); segue fire-and-forget (não bloqueia a mutação).

## EC-02 — Chave de service role curta rejeitada 🟢
O corte antigo `length > 50` (assumia JWT) rejeitava a chave `sb_secret_...` (~41 chars) real; login/convite/atribuição falhavam com a chave certa. Agora só vazio/PLACEHOLDER são "não".

## EC-03 — 120 códigos de audit não-filtráveis 🟢
Havia cópia manual no painel (209 aqui, 89 lá). A cópia foi apagada; o painel deriva o array. Regra: acrescentar no fim, nunca renomear.

## EC-04 — Foto de perfil órfã 🟢
A RPC zera o ponteiro do avatar; se o arquivo não fosse enfileirado antes, ficaria órfão no bucket (pessoa "anonimizada" com o rosto guardado). Enfileira antes, falha fechada.

## EC-05 — Cascata reexecutada comendo o título 🟢
Rodar de novo sobre título já redigido produziria "Orçamento telhado (an (anonimizado)". `jaRedigida()` guarda o sufixo; passos 3-4 selecionam antes de escrever.

## EC-06 — Régua ressuscitando vínculo anonimizado 🟢
Passo 4 (issue #701) cancela a régua viva; sem ele a régua esgotava DEPOIS da redação e mandava mensagem a quem pediu para ser esquecido.

## EC-07 — Starvation da varredura de redação 🟢
`limit(200)` sem ordenação olhava sempre os mesmos 200 primeiros; o contato 201 ficava pendente indefinidamente. Leitura ≤5000 e conserto ≤200 são números diferentes de propósito.

## EC-08 — Export desalinhado da redação 🟢
Campos obrigatórios (`case_chat_messages`, `passagens`) fazem um caminho de export novo NÃO COMPILAR se esquecer; o gate deriva a lista das duas pontas.

## EC-09 — PDF descartando cor em silêncio 🟢
`@react-pdf` renderiza `var(--x)`/`oklch()` como PDF válido descartando a cor; o documento não recebe cor de marca (nomeia o controlador), então a armadilha não o alcança.

## EC-10 — Lei do país errado citada 🟢
`lei.revisada === false` faz o documento NÃO citar lei nenhuma (sem fallback para a LGPD): afirmar a lei brasileira para um titular em Angola é pior que nenhuma citação.

## EC-11 — `javascript:` numa URL pública 🟢
`z.string().url()` aceita `javascript:alert(1)`; `urlDePoliticaSegura` guarda na SAÍDA (só https/http), valendo inclusive para valor já gravado.

## EC-12 — "parar" no meio da frase 🟢
"tem como parar a dor?"/"posso sair antes das 15h?" bloqueavam o paciente com a regra antiga. Agora exige verbo de cessação + objeto de comunicação; palavra solta no meio não conta.

## EC-13 — Env de retenção digitada errada 🟢
`JOB_QUEUE_RETENTION_DAYS=noventa` às 2h derrubaria o produto se estivesse em `lib/env.ts` (que lança no import). `interpretarRetencao` usa `Number()` (não `parseInt`, que aceitaria "90dias"); o piso de verdade mora no SQL.
