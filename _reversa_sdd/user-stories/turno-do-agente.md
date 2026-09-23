# User Stories — Turno do Agente de IA

> Fluxo inbound WhatsApp → resposta do agente. Confiança: 🟢 CONFIRMADO.
> Units relacionadas: `nucleo-ia-agente`, `ia-suporte`, `canais-mensageria`.

## US-01 — Cliente manda mensagem e recebe resposta da IA
**Como** cliente no WhatsApp
**Quero** enviar uma mensagem e receber uma resposta útil
**Para** ser atendido sem esperar um humano

```gherkin
Dado que a organização tem um agente publicado e canal conectado
Quando eu envio uma mensagem no WhatsApp
Então o webhook autentica (fail-closed), persiste a mensagem (idempotente)
E dispara ai_agent.dispatch_requested
E o worker roda um turno (abrir → tools → checkpoint)
E a resposta passa pela cadeia before-send antes de chegar a mim
```

## US-02 — A IA não envia texto fora de tool
**Como** operador
**Quero** que só mensagens deliberadas cheguem ao cliente
**Para** evitar vazamento de raciocínio interno

```gherkin
Dado que o modelo produz texto livre (sem chamar send_message)
Quando o turno termina
Então nenhuma mensagem é enviada (texto descartado)
```

## US-03 — A IA respeita o teto de envios e o pacing
**Como** operador
**Quero** que a IA não dispare muitas mensagens seguidas
**Para** não arriscar banimento do número

```gherkin
Dado que a IA já enviou o máximo de mensagens no turno
Quando tenta enviar de novo
Então o envio é bloqueado (maxSendsPerTurn)

Dado que é fora da janela de envio do tenant
Quando a cadeia before-send avalia
Então o envio é vetado (outside_window) ou adiado (throttle)
```

## US-04 — Orçamento estoura no meio do atendimento
**Como** dono da organização
**Quero** que o atendimento não pare em silêncio quando o orçamento acaba
**Para** o cliente não ficar sem resposta

```gherkin
Dado que o orçamento mensal de IA estourou
Quando um turno tenta chamar o modelo
Então a IA avisa o cliente com texto de código (sem tokens)
E passa o atendimento para um humano (handoff)
```
