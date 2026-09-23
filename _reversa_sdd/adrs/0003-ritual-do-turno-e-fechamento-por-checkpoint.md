# ADR-0003 — Ritual do turno do agente e fechamento por chamada de checkpoint

> ADR **retroativo** (Detetive/Reversa). Confiança: 🟢 CONFIRMADO.
> Evidência: `code-analysis.md` (Unidade 1.1), `agent-engine/agent/inbound-turn.ts`.

## Status
Aceito (vigente).

## Contexto
Um agente de IA que atende leads precisa de memória entre turnos, precisa não vazar estado entre
leads, e precisa que a saída passe sempre por guardrails. LLM é não-determinístico: se o registro do
estado (checkpoint) depender de o modelo lembrar de chamar uma tool, ele às vezes não chama, e a
memória se perde.

## Decisão
O runtime **impõe um ritual** por turno, sobre uma **sessão LLM fresca** (todo o estado do run vive
no closure — isolamento entre leads por construção):
1. **Abertura:** playbook (por ponteiro) + checkpoint anterior + estado do funil + últimas N mensagens.
2. **Loop:** o modelo chama tools livremente dentro de `maxSteps`. **Enviar é sempre `send_message`**
   (tool call); texto solto do modelo é descartado (RN-04).
3. **Fechamento:** uma **2ª chamada** com `purpose:'checkpoint'` que devolve **somente** o JSON do
   checkpoint, validado por Zod e persistido. Escolhido porque essa chamada **sempre** acontece.
4. **Pós-checkpoint:** enfileira `operator_turn` se o papel Operador estiver ligado.

A `declaracao` do turno distingue `undefined` (não declarou) de `{nada_a_declarar:true}` (avaliou,
nada há) — fronteira FALAR/OPERAR.

## Alternativas consideradas
1. **Checkpoint como tool (`save_checkpoint`)** — rejeitado: depende de o modelo lembrar de chamá-la;
   turnos sem checkpoint perderiam memória silenciosamente.
2. **Estado durável mutado a cada tool call** — rejeitado: acopla persistência ao meio do raciocínio;
   difícil garantir consistência num turno abortado.
3. **Sessão longa reutilizada entre leads** — rejeitado: risco de vazamento de contexto entre leads;
   quebra o prefixo estável de cache.
4. **Ritual + fechamento por 2ª chamada (escolhida).**

## Consequências
- **Positivas:** memória sempre gravada; isolamento entre leads por construção; prefixo estável de
  cache (superfície de 13 tools fixa); toda saída passa por `before_send`.
- **Negativas / custo:** duas chamadas de LLM por turno (a de fechamento custa tokens); complexidade
  alta (`inbound-turn.ts` ~4352 linhas, o coração do sistema); o escort de orçamento
  (`comHandoffSeOrcamentoAcabar`) precisa envolver o turno inteiro porque chamadas auxiliares
  ocorrem antes da chamada do modelo.
- **Derivada:** o Operador (que muta o CRM) e o agente que fala são **papéis separados por ausência
  de tool** — o Operador não tem `send_message` (ver ADR sobre separação FALAR/OPERAR no domínio).
