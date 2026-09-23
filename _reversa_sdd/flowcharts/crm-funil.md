# Fluxogramas — Unidade 5: CRM e Funil

> Gerado pelo Arqueólogo (Reversa) — nível **detalhado**
> Cobre leads (score/risco/ciclo de vida), funil, kanban, contatos, tags, conversões e prospecção.
> Escala: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

---

## 5.0 Visão geral — o ciclo de vida do negócio 🟢

```mermaid
flowchart LR
  Conv[Conversa / webhook] -->|garantirLeadDaConversa<br/>RPC advisory lock| Lead[(crm_leads open)]
  Lead --> Funil{Máquina do funil}
  Funil -->|agente / humano / agenda| Move[stage move<br/>trava otimista]
  Lead --> Score[score-writer<br/>fórmula + histerese]
  Lead --> Risco[risk-worker<br/>classifyRisk por estágio]
  Risco -->|esfriou| React[proposta de reativação]
  Score --> Card[Kanban card-state<br/>precedência estrita]
  Risco --> Card
  React --> Card
  Funil -->|won/lost| Fim[(fechado)]
  Fim -->|lead.won / stage_changed| Conversao[handler de conversão<br/>reporta ao anúncio]
```

---

## 5.1 `classifyRisk` — bucket por estágio 🟢 — `leads/risk-radar.ts`

```mermaid
flowchart TD
  Start([classifyRisk]) --> Win[resolveStageWindow:<br/>coldHours = expected_duration_hours ou 24<br/>criticalHours = coldHours*3]
  Win --> A{agenda.adiar?}
  A -->|sim| EmVoo1[em_voo]
  A -->|não| B{agenda = presenca_vencida?}
  B -->|sim| Crit1[critico]
  B -->|não| C{horas < coldHours?}
  C -->|sim| Dia[em_dia — sai do radar]
  C -->|não| D{inFlight follow-up?}
  D -->|sim| EmVoo2[em_voo]
  D -->|não| E{horas >= criticalHours?}
  E -->|sim| Crit2[critico]
  E -->|não| Risco[em_risco]
```

---

## 5.2 `calculaScore` — fórmula com lastro 🟢 — `leads/score-formula.ts`

```mermaid
flowchart TD
  Start([calculaScore]) --> St{status != open?}
  St -->|sim| N1[null — negocio_fechado]
  St -->|não| Cont[conta compromissos, objeções, BANT]
  Cont --> Min{substantivos < 2?<br/>recência NÃO conta}
  Min -->|sim| N2[null — sem_conteudo]
  Min -->|não| Chk{checkpointId null?}
  Chk -->|sim| N3[null — sem_lastro_citavel]
  Chk -->|não| Fat[fatores: compromisso/objeção com âncora checkpoint<br/>BANT SEM âncora + PESO_RISCO por bucket]
  Fat --> Soma[bruto = soma + BASE30; score = clamp 0..100]
  Soma --> Lastro{algum fator relevante tem âncora?}
  Lastro -->|não| N4[null — sem_lastro_citavel]
  Lastro -->|sim| Frase[frase: até 3 parcelas + N outros<br/>+ 'limitado a X' se houve clamp]
  Frase --> Band[resolveBand score, bandAnterior<br/>histerese degrau-a-degrau]
  Band --> Out([ScoreCalculado])
```

---

## 5.3 `resolveCardState` — precedência estrita da faixa ③ 🟢 — `kanban/card-state.ts`

```mermaid
flowchart TD
  Start([resolveCardState]) --> A{nextAction.label?}
  A -->|sim| Aw[awaiting — borda accent]
  A -->|não| R{reactivation viva?}
  R -->|sim| Re[reactivation — borda warning<br/>substitui cooling]
  R -->|não| C{isCooling?}
  C -->|sim| Co[cooling — borda warning]
  C -->|não| M{typeof probability = number E band?}
  M -->|sim| Me[meter — borda neutral<br/>até 3 fatores]
  M -->|não| Id[idle — faixa vazia mesma altura]
```

> Um card mostra SÓ o que exige decisão agora. `probability` null ≠ 0.

---

## 5.4 Movimentador de card com trava otimista 🟢 — `agent-stage-sync` / `handoff-stage-move` / `appointment-stage-move`

```mermaid
flowchart TD
  Start([mover card]) --> Rota[resolve destino<br/>hint do agente / slug chamar-humano / status do compromisso]
  Rota -->|sem destino| Neutro[sem_mapeamento / sem_etapa / status não avança]
  Rota --> Lead[resolveActiveLeadForContact]
  Lead -->|não-open| Fech[lead_fechado]
  Lead -->|ambíguo| Amb[ambiguous_open_leads — não adivinha]
  Lead --> Perda{destino é etapa de perda?}
  Perda -->|sim| Mot[decideMotivoDaPerda<br/>agente NÃO passa motivoAtual]
  Perda -->|não| Upd
  Mot -->|sem motivo| Recusa[lost_reason_required]
  Mot --> Upd[UPDATE ... WHERE stage_id = origem<br/>.select id — trava otimista]
  Upd -->|0 linhas| Conflito[conflito_humano]
  Upd -->|1 linha| Ev[emite stage_changed + emit_event lead.stage_changed<br/>entity_kind=crm_lead]
```

---

## 5.5 Nascimento do lead 🟢 — `leads/nascimento-do-lead.ts`

```mermaid
flowchart TD
  Start([garantirLeadDaConversa]) --> Blk{contato bloqueado?}
  Blk -->|sim| R1[recusa contato_bloqueado]
  Blk -->|não| Open{lead aberto existe?}
  Open -->|sim| R2[ja_existe]
  Open -->|não| Cli[ehCliente = first_service_at != null<br/>E lerClientePelaAgenda]
  Cli --> Funil[funilDeEntrada:<br/>cliente -> is_client_pipeline<br/>senão is_default<br/>falha -> funil de entrada]
  Funil --> Etapa[primeiraEtapa: menor position, não won/lost]
  Etapa --> Ins[RPC fn_nascer_lead_da_conversa<br/>advisory lock org+contato]
  Ins -->|NULL corrida| R3[segunda mensagem achou o card]
  Ins -->|linha| Emit[emite lead_created]
```

---

## 5.6 Detecção de duplicados — union-find 🟢 — `contacts/duplicados.ts`

```mermaid
flowchart TD
  Start([encontrarContatosDuplicados]) --> Vivos[filtra vivos: is_merged_into null E não anonimizado]
  Vivos --> Init[cada contato é sua própria raiz]
  Init --> Tel[une por chaveDeTelefone canônica BR]
  Tel --> Email[une por chaveDeEmail]
  Email --> Conf[une por telefone_em_conflito<br/>o que a ingestão parkou]
  Conf --> Grp[agrupa por raiz; grupos com >1 membro]
  Grp --> Ord[ordena por created_at; motivos ordenados]
  Ord --> Out([GrupoDeDuplicados])
```

> Critérios se encadeiam (A~B tel, B~C email → os três). `principalSugerido` = atividade mais recente.

---

## 5.7 Handler de conversão — duas portas 🟢 — `conversoes/envio.handler.ts`

```mermaid
flowchart TD
  Start([evento lead.won OU lead.stage_changed]) --> Read[re-lê crm_leads<br/>banco é verdade, payload é dica]
  Read -->|erro| Retry1[retry backoff 5min]
  Read -->|não achou| Skip1[skipped]
  Read --> Won{status = won?}
  Won -->|não| Skip2[skipped nao_e_ganho]
  Won -->|sim| Ja{já enviada? eventoId leadId:Purchase}
  Ja -->|sim| Skip3[skipped ja_enviada]
  Ja -->|não| Attr{tem atribuição de anúncio?}
  Attr -->|não| Skip4[skipped sem_atribuicao — não suja o livro]
  Attr -->|sim| Val{value_cents > 0?}
  Val -->|não| Skip5[skipped sem_valor — pendência mais comum]
  Val -->|sim| Cred{credencial ok?}
  Cred -->|não| Skip6[skipped]
  Cred -->|sim| Env[transporte.enviar Purchase]
  Env -->|ok| Sent[registra sent]
  Env -->|transitório| Retry2[retry]
  Env -->|recusado| Err[registra error]
```

---

## 5.8 Tick de prospecção fria 🟢 — `prospecting/worker.ts::sendNextCandidate`

```mermaid
flowchart TD
  Start([sendNextCandidate]) --> Due{next_send_at no futuro?}
  Due -->|sim| Ret[retorna]
  Due -->|não| Jan{janela de envio aberta?}
  Jan -->|não| Ad1[adia p/ próxima abertura]
  Jan -->|sim| Pac{decidePacing permite?}
  Pac -->|não| Ad2[adia p/ nextAllowedAt]
  Pac -->|sim| Warm{count >= tetoDiarioDaEsteiraFria?<br/>warmupCap/4, piso 1, falha fechada}
  Warm -->|sim| Ad3[adia p/ retry_at]
  Warm -->|não| Lim{count >= daily_limit OU total >= 50?}
  Lim -->|sim| Ad4[adia]
  Lim -->|não| Cand[pega candidato queued]
  Cand -->|nenhum| Comp[campanha completed]
  Cand --> Tx[BEGIN: candidato sending;<br/>next_send_at = agora + intervalo + JITTER;<br/>recordSend reserva cota; COMMIT]
  Tx --> Guards[preflight canal, serviceBoundary,<br/>agente publicado, autoriza contato]
  Guards --> Gen[gerarAbordagemDeFormulario<br/>origem=prospeccao_fria + comSaida idioma]
  Gen --> Send[sendMessageHandler]
  Send -->|sent| OK[candidato sent + audit approach_sent]
  Send -->|não confirmado| Fail[message failed; candidato failed]
```

> Falha `escopo=candidato` marca o item e segue; qualquer outra pausa a campanha.

---

## 5.9 Worker de risco — travessia narrada 🟢 — `leads/risk-worker.ts`

```mermaid
flowchart TD
  Start([observaTravessias]) --> Calc[calculaBucketsAtuais]
  Calc --> Loop[por lead: compara com crm_lead_risk_states gravado]
  Loop --> Eq{de = bucket?}
  Eq -->|sim| Cont[continue — sem escrita, board não pisca]
  Eq -->|não| Narra[narra de,para]
  Narra -->|em_risco de em_dia/null| Cool[lead_cooled + proposta reativação no mesmo tick]
  Narra -->|critico| Cool2[lead_cooled parado há muito tempo]
  Narra -->|volta a em_dia| Reat[lead_reactivated]
  Narra -->|entra em_voo| Silen[silenciosa — quem agendou já registrou]
  Cool --> Grava[UPSERT crm_lead_risk_states novo bucket]
  Grava -->|falha| Conta[conta falhasDeGravacao, não aborta o laço]
```
