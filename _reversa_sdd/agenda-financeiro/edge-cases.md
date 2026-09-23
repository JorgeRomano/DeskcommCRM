# Agenda e Financeiro — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Bloqueio que desliza a tarde 🟢
A versão anterior SUBTRAÍA o bloqueio da janela (reparte): reunião 10:30-11:30 deslizava 12:30/13:30/14:30 e o 17:00 sumia. A correção trata bloqueio como OCUPADO (remoção de instantes), igual a um compromisso.

## EC-02 — `windows` vazio 🟢
Significa opostos: `isWithinSchedule` (roteamento) lê vazio como 24/7; a agenda lê vazio como zero horário (senão ofereceria consulta às 3h). Por isso `janelasDoDia` existe.

## EC-03 — Hora inexistente do DST 🟢
`instanteDe` usa duas passagens e devolve `Math.max` dos candidatos (`Math.min` cairia no dia anterior em offsets positivos). Nunca Invalid Date. Dedup final por instante protege a hora que desliza (Santiago/Assunção têm DST; Brasil não).

## EC-04 — Conexão Google com token expirado 🟢
`token_expired`/`scope_missing`/`error`/`rate_limited` continuam bloqueando (o compromisso segue no Google; o que parou foi a atualização). Só `disconnected`/`connecting` não contam como ocupação.

## EC-05 — Corte por dia UTC perdendo linhas 🟢
Exceções filtradas pelo dia LOCAL (issue #878); corte por dia UTC perdia linhas em offset negativo.

## EC-06 — Compromisso remarcado ocupando o destino 🟢
`coletaOQueOcupa` usa `neq id` (issue #1084): o remarcado ocupa a ORIGEM, não o destino.

## EC-07 — `id` + `iCalUID` juntos no Google 🟢
Enviar os dois juntos dá HTTP 400 (produção 2026-09-01). São enviados separados.

## EC-08 — syncToken morto (410) 🟢
410 tem duplo sentido: DELETE = `ja_esta_feito`; sync incremental = `ressincronizar` (syncToken morreu, risco de apagar tudo). Tratado por operação.

## EC-09 — Recusa do Meet como 5xx 🟢
`motivoDoMeet` nunca dá 5xx a recusa conhecida (5xx faria o cliente repetir 3× e travar ~20s); casa por nome primeiro, SQLSTATE depois.

## EC-10 — Desconto maior que o item 🟢
`totalDoItem` tem piso zero (item vira de graça, não erro). Desconto da COMANDA não entra no item (abatimento de caixa não reduz o combinado com quem atendeu).

## EC-11 — 128GB para quem pediu 256GB 🟢
Número FILTRA, não ranqueia: número dito que o produto não tem ELIMINA o produto (`pontuar` devolve null). `buscarComRelaxamento` refaz sem os números desconhecidos e marca como relaxado.

## EC-12 — Reimportar planilha 🟢
`codigoDoProduto` dá identidade estável (reimportar ATUALIZA); corte em 60 chars com assinatura FNV-1a evita mesmo código para "…Cor Manhã" e "…Cor Noite". Coluna de estoque ausente ≠ estoque zero.

## EC-13 — Org em MXN gravando BRL 🟢
`moedaDaOrganizacao` é função única com fallback BRL mas com rastro em console+Sentry; nunca aceita a moeda de quem chamou.
