# User Stories — Captação e Conversão de Lead

> Redator (Reversa) · Nível: Detalhado · derivado dos fluxos confirmados no código.

## US-01 — Lead chega por formulário/webhook

**Como** operador,
**quero** que todo formulário enviado vire registro visível,
**para** não perder nenhum lead que chegou.

```gherkin
Dado um webhook de captação de uma fonte configurada
Quando registrarCaptacao roda
Então o envio é registrado em webhook_lead_captures (criado/duplicado/recusado)
E campos grandes são cortados com reticências (nunca recusam a linha inteira)
E a organização é resolvida da FONTE (token), nunca do body
```

## US-02 — Lead nasce da conversa

**Como** operador,
**quero** que quem escreve no WhatsApp vire um negócio no funil,
**para** que ninguém fique fora do radar.

```gherkin
Dado um contato sem lead aberto que envia a primeira mensagem
Quando garantirLeadDaConversa roda
Então nasce um lead no funil padrão, na primeira etapa não-terminal
E uma segunda mensagem NÃO cria um segundo lead (um por demanda)
```

## US-03 — Classificação inicial do lead

**Como** operador,
**quero** que leads sejam triados na entrada,
**para** priorizar quem tem mais chance e barrar spam/sem-consentimento.

```gherkin
Dado um lead vindo do Respondi
Quando classificarLeadInicial roda
Então retorna desqualificado (contato inválido/sem consentimento), revisao_humana (conflito/spam/incoerência) ou classificado (A/B/C/D)
E D só vem do critério combinado do score ou da frase exata, nunca de um corte de R$ isolado
```

## US-04 — Lead esfria e aparece no radar

**Como** operador,
**quero** ver as demandas abertas que esfriaram,
**para** agir antes de perder o negócio.

```gherkin
Dado um lead aberto sem resposta além da janela do estágio e sem follow-up agendado
Quando o Radar de Risco carrega
Então o lead aparece como em_risco/critico, com link para a conversa
E se há follow-up agendado, aparece como em_voo (o sistema o mantém vivo)
```

## US-05 — Venda fechada é reportada ao anúncio

**Como** dono,
**quero** que a venda de um lead de anúncio seja reportada à plataforma,
**para** o otimizador aprender e trazer mais leads como esse.

```gherkin
Dado um lead won com atribuição de anúncio (ctwa_clid) e valor > 0
Quando o handler de conversão consome lead.won
Então envia Purchase à Meta (business_messaging), dedup por <leadId>:Purchase
E se não há atribuição, é skip (não é pendência); se não há valor, registra sem_valor
```

## US-06 — Lead pede para sair

**Como** lead,
**quero** parar de receber mensagens quando peço,
**para** exercer meu direito de opt-out.

```gherkin
Dado que envio "não quero mais receber nada" (pedido inequívoco)
Quando a ingestão detecta o opt-out
Então contacts.is_blocked=true, nenhum envio automatizado sai, e follow-ups vivos são cancelados (opted_out)
E um pedido ambíguo ("me deixa em paz") apenas escala à Central, sem bloquear
```
