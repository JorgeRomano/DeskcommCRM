# Fluxogramas — Unidade 6: Agenda e Financeiro

> Gerado pelo Arqueólogo (Reversa) — nível **detalhado**
> Cobre `lib/agenda/` (motor de horários, sync Google, Meet, lembretes), `lib/financeiro/`
> (comanda, catálogo financeiro) e `lib/catalogo/` (busca difusa, planilha, moeda).
> Escala: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

---

## 6.0 Visão geral — três domínios, um laço comercial 🟢

```mermaid
flowchart TB
  subgraph AGENDA["Agenda (quando)"]
    direction TB
    A1[Tela / Rota REST / Tool MCP] --> A2[consulta.ts<br/>client injetado]
    A2 --> A3[horarios-livres.ts<br/>motor PURO]
    A2 --> DBA[(calendar_appointments<br/>event_types / exceptions)]
    A2 -.->|RPC| G1[Google Calendar v3]
    G1 <--> A4[google/* sync 2 vias]
    A4 --> DBA
  end
  subgraph FIN["Financeiro (quanto)"]
    direction TB
    F1[Rota comanda] --> F2[comanda.ts<br/>comissão + total puros]
    F2 --> DBF[(sale_orders / sale_items)]
    F3[Rota catálogo] --> F4[financeiro/catalogo.ts<br/>5 schemas]
    F4 --> DBF2[(accounts / methods / plans<br/>commission_rules / recurring)]
  end
  subgraph CAT["Catálogo de produtos (o quê)"]
    direction TB
    C1[Import planilha] --> C2[catalogo/planilha.ts]
    C2 --> DBC[(catalog_products)]
    C3[Busca do agente] --> C4[catalogo/busca.ts<br/>token a token]
    C4 --> DBC
  end
  F2 -.->|faturar conclui compromisso<br/>fn_finalizar_comanda| DBA
```

---

## 6.1 Motor de horários livres — `horariosLivres()` 🟢 (PURO, sem banco, sem relógio)

```mermaid
flowchart TD
  Start(["horariosLivres(entrada)"]) --> P0{passo<=0 ou<br/>duracaoMin<=0?}
  P0 -->|sim| Vazio1[[return []]]
  P0 -->|não| Cortes[naoAntesDe = max de, agora+avisoMinimo<br/>naoDepoisDe = min ate, agora+janelaDias]
  Cortes --> P1{naoAntesDe ><br/>naoDepoisDe?}
  P1 -->|sim| Vazio2[[return []]]
  P1 -->|não| Loop[para cada DIA LOCAL<br/>1 dia de margem cada lado]
  Loop --> Ancora[meioDia local = âncora<br/>lê dow seguro do DST]
  Ancora --> Jan["janelasDoDia(jornada, excecoes, dow)"]
  Jan --> Bloq["bloqueiosDoDia =<br/>compromissos + exceções indisponíveis<br/>(DECISÃO 12: bloqueio É ocupado)"]
  Bloq --> Grade[grade fixa: início da faixa,<br/>+passo em +passo — NÃO se move]
  Grade --> Fim{slot cabe na faixa?<br/>fim > fimDaFaixa}
  Fim -->|não| Grade
  Fim -->|sim| Corte2{inicio dentro de<br/>naoAntesDe..naoDepoisDe?}
  Corte2 -->|não| Grade
  Corte2 -->|sim| Buffer["slotInflado(bufferAntes/Depois)"]
  Buffer --> Colide{"colide com algum<br/>bloqueio? (estrito)"}
  Colide -->|sim| Grade
  Colide -->|não| Push[adiciona slot]
  Push --> Grade
  Loop --> Dedup["dedup por instante<br/>(hora que desliza no DST)"]
  Dedup --> Ordena[[ordena por início asc]]
```

### `janelasDoDia()` — a régua do `windows` vazio 🟢

```mermaid
flowchart TD
  J(["janelasDoDia(jornada, excecoesDoDia, dow)"]) --> D{há exceção<br/>DISPONÍVEL no dia?}
  D -->|sim| Sub[base = exceções disponíveis<br/>SUBSTITUEM a jornada — sábado excepcional]
  D -->|não| Jor[base = janelas da jornada do dow<br/>vazio ⇒ ZERO horário, ≠ 24/7 do roteamento]
  Sub --> Unir["unirFaixas(base)<br/>funde sobrepostas e contíguas"]
  Jor --> Unir
  Unir --> Ret[[retorna faixas em minutos]]
  Note["⚠️ exceção INDISPONÍVEL 0..1440 vence<br/>a disponível do mesmo dia (subtrai depois).<br/>A TELA avisa ao salvar a 2ª linha."]
```

---

## 6.2 Conversão de fuso — `instanteDe()` (duas passagens, DST-safe) 🟢

```mermaid
flowchart TD
  I(["instanteDe(paredePedida, fuso)"]) --> A[chute = Date.UTC das partes]
  A --> B["offset1 = offsetEmMinutos(chute, fuso)"]
  B --> C[candidato1 = chute - offset1]
  C --> D["offset2 = offsetEmMinutos(candidato1, fuso)"]
  D --> E{offset1 == offset2?}
  E -->|sim| Ok[[instante = candidato1]]
  E -->|não| F["candidato2 = chute - offset2<br/>(fronteira de DST)"]
  F --> G{hora inexistente<br/>(primavera)?}
  G -->|sim| Max["return Math.max cand1, cand2<br/>(DECISÃO 15: min cairia no dia anterior<br/>em offset positivo — Beirute/Teerã)"]
  G -->|não| Min["hora ambígua (outono):<br/>return o primeiro"]
```

---

## 6.3 Sincronização Google — `reconcileAppointment()` (máquina de estados) 🟢

```mermaid
flowchart TD
  R(["reconcileAppointment(db, org, id, opts)"]) --> Claim["fn_google_appointment(claim)"]
  Claim --> Busy{claim negado?}
  Busy -->|sim| RetBusy[[return 'busy']]
  Busy -->|não| Cancel{cancelado antes<br/>de publicar?}
  Cancel -->|sim| Ack["commit ack → release"] --> RetProc[[return 'processed']]
  Cancel -->|não| Ident{tem event_id?}
  Ident -->|não| Pub[PUBLICAR: POST novo evento]
  Ident -->|sim| Get["GET exato (não list)<br/>revalida etag/identidade"]
  Get --> Sumiu{404/410?}
  Sumiu -->|sim| Erros["classificarErroDoGoogle"]
  Sumiu -->|não| IdConf{extendedProperties<br/>batem org/appt?}
  IdConf -->|não| Conf["conflito de identidade → error"]
  IdConf -->|sim| Merge["compare(base, local, remote)<br/>merge de três pontas"]
  Merge --> Kind{kind?}
  Kind -->|accept_remote| Adota[adota o remoto → checkpoint]
  Kind -->|publish| Pub
  Kind -->|converged| Nada[[return 'unchanged']]
  Kind -->|conflict| Espera[grava google_conflict<br/>aguarda resolução humana]
  Pub --> Meet{meeting_state<br/>== pending?}
  Meet -->|sim| CReq["injeta conferenceData.createRequest<br/>observeMeeting"]
  Meet -->|não| Send["send(): POST/PATCH/DELETE via transport"]
  CReq --> Send
  Send --> SendErr{erro?}
  SendErr -->|sim| Erros
  SendErr -->|não| Commit["commit → release"] --> RetProc
  Erros --> Desf{"deveTentarDeNovo(desfecho)?"}
  Desf -->|sim| RetFail[[return 'failed' — retry depois]]
  Desf -->|não| Term["error/terminal → release"] --> RetTerm[[return 'terminal']]
  Espera --> Rel[release no finally]
```

### `classificarErroDoGoogle()` — a tabela de desfechos 🟢

```mermaid
flowchart TD
  E(["classificarErroDoGoogle(erro, operacao)"]) --> App{motivo de app errado?<br/>invalid_client/redirect_uri_mismatch}
  App -->|sim| Perm[permanente]
  App -->|não| IG{invalid_grant?}
  IG -->|sim| Reauth[reautenticar → status token_expired]
  IG -->|não| Full{fullSyncRequired?}
  Full -->|sim| Ress[ressincronizar → não toca conexão]
  Full -->|não| S{status HTTP?}
  S -->|401| Reauth
  S -->|429| Recuar[recuar → status rate_limited]
  S -->|403| Cota{motivo de cota?}
  Cota -->|sim| Recuar
  Cota -->|não| SemPerm[sem_permissao → status scope_missing]
  S -->|404/410| Op{operação + alvo}
  Op -->|apagar| Feito[ja_esta_feito → healthy]
  Op -->|criar / alvo=calendario| CalSumiu[calendario_sumiu → error]
  Op -->|listar/sincronizar + 410| Ress
  Op -->|senão| EvSumiu[evento_sumiu → não toca conexão]
  S -->|>=500 ou null| Trans[transitorio → não toca conexão]
  S -->|outro| Perm
```

> ⚠️ **O 410 tem duplo sentido** (o coração do módulo): num DELETE significa "já foi apagado"
> (`ja_esta_feito`); num sync incremental significa que o `syncToken` morreu (`ressincronizar`) —
> tratar como sucesso apagaria a agenda inteira em silêncio.

---

## 6.4 O que ocupa o horário — `ocupadosDoDono()` 🟢

```mermaid
flowchart TD
  O(["ocupadosDoDono(agendamentos, externos)"]) --> LoopA[para cada agendamento]
  LoopA --> Lib{status em<br/>LIBERAM_O_HORARIO?<br/>cancelled/no_show}
  Lib -->|sim| SkipA[ignora]
  Lib -->|não| AddA["ocupa (inclui pending<br/>e status desconhecido)"]
  O --> LoopE[para cada evento externo]
  LoopE --> Transp{transparent?}
  Transp -->|sim| SkipE[ignora]
  Transp -->|não| GCancel{google status<br/>cancelled?}
  GCancel -->|sim| SkipE
  GCancel -->|não| ConexOcupa{"CONEXAO_CONTA_<br/>COMO_OCUPACAO[situacao] ?? true"}
  ConexOcupa -->|false: disconnected/connecting| SkipE
  ConexOcupa -->|true| AddE["ocupa + (se != healthy)<br/>marca fontesDefasadas"]
```

> Princípio: **oferecer de menos se recupera; marcar em dobro não** → na dúvida, OCUPA.
> "BLOQUEIA, A MENOS QUE UM HUMANO TENHA MANDADO PARAR" (só `disconnected` é decidido por gente).

---

## 6.5 Comissão da comanda — `percentualDaComissao()` 🟢

```mermaid
flowchart TD
  C(["percentualDaComissao(regras, {pessoa, servico})"]) --> Ex{pessoa E serviço?<br/>match exato}
  Ex -->|há regras| RE[return MAIOR percent das exatas]
  Ex -->|nenhuma| Pe{pessoa?<br/>event_type_id null}
  Pe -->|há regras| RP[return MAIOR percent por pessoa]
  Pe -->|nenhuma| Se{serviço?<br/>attendant null}
  Se -->|há regras| RS[return MAIOR percent por serviço]
  Se -->|nenhuma| Zero[return 0<br/>zero é resposta legítima]
```

> Precedência **pessoa+serviço > pessoa > serviço > zero**; empate no mesmo nível → MAIOR percentual
> (errar a favor de quem trabalhou). Comissão errada é dinheiro errado no bolso de quem trabalhou.

---

## 6.6 Busca difusa de produto — `buscarComRelaxamento()` 🟢

```mermaid
flowchart TD
  B(["buscarComRelaxamento(produtos, consulta)"]) --> Tok["tokenizar: palavras (difuso)<br/>vs números (exato + unidade)"]
  Tok --> Empty{sem palavra<br/>e sem número?}
  Empty -->|sim| VazioB[[achados: [], ignorados: []]]
  Empty -->|não| Ex["pontuarTodos (filtro numérico DURO)<br/>número ausente ELIMINA o produto"]
  Ex --> Achou{achou algo OU<br/>sem números?}
  Achou -->|sim| RetEx[[achados exatos, ignorados: []]]
  Achou -->|não| Conhece["numerosQueOCatalogoConhece"]
  Conhece --> Ign[ignorados = números que o catálogo NÃO conhece]
  Ign --> TemIgn{há ignorados?}
  TemIgn -->|não| RetEx2[[achados exatos vazio, ignorados: []]]
  TemIgn -->|sim| Relax["pontuarTodos só com números conhecidos<br/>(256 numa loja com 256 e 128 ainda elimina o 128)"]
  Relax --> SoPalavra{sobrou palavra?}
  SoPalavra -->|não| VazioR[[[] — 'quero 2' não vira catálogo inteiro]]
  SoPalavra -->|sim| RetRelax[["achados relaxados + ignorados<br/>→ agente CONFIRMA, não responde preço"]]
```

> **PALAVRA é difusa, NÚMERO é exato.** O número filtra (não ranqueia): quem pede 256GB nunca vê o
> 128GB. `ignorados` não vazio é contrato: não responda como se fosse o pedido — confirme.
> Notas medidas: inicial vale mais que distância; prefixo ≥3 letras; distância 1 (≥4), distância 2 (≥5).

---

## 6.7 Leitura de planilha de produtos — `lerPlanilha()` 🟢

```mermaid
flowchart TD
  L(["lerPlanilha(conteudo, t?)"]) --> Parse["parseCsv (RFC 4180, detecta ;)"]
  Parse --> Vazia{planilha vazia?}
  Vazia -->|sim| ErrV[[erro: 'planilha vazia']]
  Vazia -->|não| Map[mapeia colunas por sinônimo pt/es<br/>colunas desconhecidas → ignoradas]
  Map --> Falta{falta nome OU preço?}
  Falta -->|sim| ErrF[[erro nomeia a coluna faltante<br/>ANTES de processar 300 linhas]]
  Falta -->|não| LoopL[para cada linha de dados]
  LoopL --> SemNome{sem nome?}
  SemNome -->|sim| RecN[recusa: sem nome]
  SemNome -->|não| Preco{"precoParaCentavos = null?"}
  Preco -->|sim| RecP[recusa: preço não reconhecido + valor cru]
  Preco -->|não| Cod["codigo = codigoDoProduto(codigo ?? nome)<br/>≤60 chars + assinatura FNV-1a"]
  Cod --> Dup{código já visto?}
  Dup -->|sim| RecD[recusa: código repetido]
  Dup -->|não| Add["produto (controla_estoque =<br/>coluna presente, ≠ estoque zero)"]
```

> A planilha vem suja e recusar é a função: o ambíguo vira linha recusada com o valor cru, nunca chute
> (chute aqui é preço errado dito ao cliente). Código estável faz reimportar ATUALIZAR, não duplicar.

---

## Lacunas de fluxo 🔴🟡
- 🔴 A lógica interna das RPCs (`fn_google_appointment`, `fn_google_calendar`, `fn_meet_action`,
  `fn_finalizar_comanda`, etc.) vive em `supabase/baseline.sql` — os fluxogramas acima param na
  fronteira da chamada. Detalhamento é do Data Master.
- 🟡 O gate de entrega do Meet (`fn_meet_delivery_policy` + fronteira de serviço + comando humano/gate
  de IA) é referenciado como caixa única; o desenho completo depende de `lib/atendimento/fronteira`
  e `lib/ai/elegibilidade` (unidades vizinhas).
- 🟡 O cron mensal que materializa `recurring_entries` em lançamentos é citado em `financeiro/catalogo.ts`
  mas vive em `workers/` (cobertura na unidade de eventos/workers).
