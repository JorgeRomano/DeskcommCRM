# User Stories — Handoff e Retomada

> Passagem IA→humano e a volta. Confiança: 🟢 CONFIRMADO.
> Units relacionadas: `automacao-roteamento`, `nucleo-ia-agente`, `canais-mensageria`.

## US-01 — IA passa o atendimento para um humano
**Como** atendente
**Quero** receber o contexto quando a IA me passa uma conversa
**Para** continuar o atendimento sem pedir tudo de novo

```gherkin
Dado que a IA decide passar a conversa (handoff)
Quando performHumanHandoff roda
Então o contato fica force_human, o bot é silenciado, os crons são cancelados
E um item de inbox é criado com o briefing (palavras do cliente separadas da paráfrase da IA)
```

## US-02 — Rodízio distribui a conversa e o negócio
**Como** gestor
**Quero** que as conversas sejam distribuídas de forma justa
**Para** ninguém ficar sobrecarregado

```gherkin
Dado dois atendentes elegíveis (de plantão, com capacidade)
Quando o roteador escolhe
Então vem primeiro quem recebeu atribuição há mais tempo (rodízio real)
E o lead do contato também passa a ter dono (não só a conversa)
```

## US-03 — Conversa volta para a IA no prazo
**Como** dono da organização
**Quero** que a IA retome o atendimento após um tempo sem ação humana
**Para** nenhuma demanda ficar parada

```gherkin
Dado uma conversa com humano há mais que o prazo de devolução
Quando o cron de devolução avalia (contando do último sinal humano)
Então devolve à IA, soltando as três travas (incl. force_human)
E emite ai.handoff_resolved (que retoma follow-up pausado)
```

## US-04 — Operador responde pelo celular
**Como** atendente
**Quero** que a IA pare quando eu respondo pelo meu celular
**Para** não falar por cima de mim

```gherkin
Dado que respondi pelo celular (não pelo composer do CRM)
Quando o ingest detecta que não é eco (janela 60s)
Então pausa a IA por 60min a contar da minha última fala (extend-only)
```
