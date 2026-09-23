# Voz / Telefonia — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Ligar voz para todas as orgs por engano 🟢
Inverter `&&` por `??` no opt-in entregaria a capacidade a todas as orgs da instalação — proibido pelo dono. `escolhaDaOrg` null = desligado.

## EC-02 — Chamar `/pair` separado 🟢
`POST .../pair` troca o cliente `whatsmeow` mas não refaz `s.calls`: quem pareia é o cliente novo, quem disca é o velho, e `startCall` responde `500 "websocket not connected"` por horas. O repo nunca chama `/pair`.

## EC-03 — Desparear com WaCalls fora 🟢
Se o WaCalls falhar, `desparear` lança e a linha NÃO é arquivada (arquivar faria a tela mentir "desconectado" com aparelho vinculado). Exceção `wacalls_404` tolerada (sessão inexistente = sem aparelho vinculado).

## EC-04 — Número sem o nono dígito 🟢
WhatsApp registra celular BR sem o nono dígito; o CRM guarda com. Discar o cadastro cru monta destino inexistente → "Chamando…" eterno. `resolverNumeroDiscavel` pergunta ao diretório do canal; falha aberta disca o cadastro.

## EC-05 — Sentido da chamada ausente no evento 🟢
`call-status` não traz `direction`. Inferência: declarado > `dono ? outbound : inbound`. `on conflict` só reescreve `direction` se declarado. Ligação feita FORA do CRM nasce sem dono → lida como recebida até o snapshot corrigir.

## EC-06 — Encerrar ligação de colega 🟢
Escopar só por org deixava qualquer `agent` derrubar a ligação de um colega num número compartilhado. `podeEncerrar`: só o dono; se ninguém assumiu e saiu do CRM, quem discou.

## EC-07 — Worker cai entre connected e call-ended 🟢
Silêncio de IA com teto de 2h (anti-morte); `'infinity'` deixaria a conversa muda para sempre. `devolverAVozDaIa` só desfaz o que esta ponte silenciou; nunca encurta handoff durável.

## EC-08 — Race do channelId no SIP 🟢
StasisStart chega por WS em processo separado, podendo preceder o retorno do POST. `channelId` gerado ANTES e passado no create fecha a race.

## EC-09 — `externalMedia` não relaya áudio 🟢
Bridge ARI "mixing" com `externalMedia` nunca relaya o áudio injetado de volta ao trunk. Por isso `originateCall` entrega ao dialplan (AudioSocket TCP), não à Stasis.

## EC-10 — Owner não-UUID do WaCalls 🟢
`donoValido` só aceita string com forma de UUID; gravar não-uuid numa coluna com FK para `auth.users` derrubaria a escrita.

## EC-11 — DID único ambíguo na entrada SIP 🟢
`resolveInboundNumber`: extensão `"s"` resolve só com exatamente 1 número ativo; senão recusa por ambiguidade e usa `fn_resolve_inbound_number`.

## EC-12 — Motivo de fim desconhecido 🟢
`motivoDaChamadaEmPortugues` devolve "terminou por um motivo que ainda não sabemos traduzir (<token>)" — não esconde o token; `end_reason` guarda o valor cru.
