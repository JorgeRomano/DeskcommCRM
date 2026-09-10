# User Stories — Atendimento com IA

> Redator (Reversa) · Nível: Detalhado · derivado dos fluxos confirmados no código.

## US-01 — Lead escreve e o agente responde

**Como** lead que enviou uma mensagem no WhatsApp,
**quero** receber uma resposta útil e no tempo certo,
**para** ter minha dúvida resolvida sem esperar por um humano.

```gherkin
Dado que a organização tem um agente publicado apontando para a sessão
Quando eu envio uma mensagem
Então a ingestão persiste a mensagem, cria/atualiza meu lead e emite ai_agent.dispatch_requested
E o agente lê meu contexto, decide send_message, a cadeia before-send aprova e a resposta chega
E o ritmo respeita o anti-ban (throttle + janela horária)
```

## US-02 — Peço para falar com uma pessoa

**Como** lead,
**quero** pedir um atendente humano,
**para** resolver algo que o robô não consegue.

```gherkin
Dado que escrevo "quero falar com um atendente"
Quando a triagem síncrona (G1) roda antes do LLM
Então o handoff é disparado (requested_human), sou avisado que uma pessoa vai assumir
E a conversa vira pending, o bot é silenciado (infinity) e a Central recebe um item para o time
```

## US-03 — Agente não inventa promessa

**Como** dono do negócio,
**quero** que o agente nunca prometa preço/desconto fora da minha tabela,
**para** não vender no prejuízo por erro do robô.

```gherkin
Dado que configurei uma tabela de preços versionada
Quando o agente tenta enviar "faço por R$1"
Então o promiseGate veta e a razão instrutiva volta ao modelo para reescrever
E nada fora da tabela chega ao cliente
```

## US-04 — Atendente assume a conversa

**Como** atendente,
**quero** assumir uma conversa da fila,
**para** continuar o atendimento onde o agente parou.

```gherkin
Dado uma conversa pending com resumo de handoff pronto
Quando clico "eu cuido"
Então o claim é atômico (só se ninguém assumiu); recebo o contexto (rolling summary + compromissos + objeções)
E se alguém assumiu primeiro, recebo 409 already_assigned
```

## US-05 — Respondo pelo celular e o robô para

**Como** dono que respondeu direto no WhatsApp,
**quero** que a IA pare de responder por cima de mim,
**para** não haver duas vozes na mesma conversa.

```gherkin
Dado que respondo o cliente pelo meu celular (fromMe pelo canal)
Quando a ingestão detecta que não é eco de envio nosso
Então a IA é pausada por 60 minutos (renovável a cada fala), sem apagar a autorização do lead
```

## US-06 — Teto de gasto de IA atingido

**Como** dono,
**quero** que o atendimento vire humano quando o orçamento de IA acabar,
**para** não ficar sem resposta ao cliente nem gastar além do limite.

```gherkin
Dado que o teto de gasto do mês foi atingido
Quando qualquer chamada de modelo do turno acontece
Então a conversa é devolvida à fila humana (handoff orcamento_de_ia), o job não é descartado
E a Central mostra budget_exceeded
```
