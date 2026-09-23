# Automação e Roteamento — Fluxos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo 1 — Motor de regras por evento
```mermaid
flowchart TD
  A[evento no event_log] --> B{caused_by_rule?}
  B -->|sim| S[skipped: anti-loop]
  B -->|não| C[guard entity_kind]
  C --> D[seleciona regras ativas do trigger]
  D --> E[buildContext: event/lead/contact/appointment]
  E --> F[evaluateConditions AND]
  F --> G[pré-checagem postpone all-or-nothing]
  G -->|alguma adia| H[status=adiado, retry_at]
  G -->|nenhuma| I[executa cada ação]
  I --> J[status derivado: failed>partial>adiado>success]
```

## Fluxo 2 — Roteamento por rodízio
```mermaid
flowchart TD
  A[conversation.routing_requested] --> B[claim CAS pending→processing]
  B --> C[loadEligibleAttendants por canal]
  C --> D[decideRouting]
  D -->|assign| E[fn_channel_routing_claim + adotarLeadsDoContato]
  D -->|skip| F[idempotência/modo manual]
  D -->|requeue| G[backoff; esgotou → fn_routing_unassigned_notice]
```

## Fluxo 3 — Follow-up: nó de espera (estado por eventos)
```mermaid
flowchart TD
  A[processEnrollment] --> B[resolveWaitPhase por ${nodeId}:${steps-1}]
  B -->|1ª entrada| C[arma timer / enqueue_turn]
  B -->|2ª entrada após next_eval_at| D[avança pela edge]
  C --> E[applyResult: evento idempotente ${node}:${steps}]
  D --> E
```

## Fluxo 4 — Nó action (envio at-most-once)
```mermaid
flowchart TD
  A[entra no nó action] --> B{actionEnqueued ou actionCompleted?}
  B -->|não| C[enfileira turno 1× por ocupância]
  B -->|turno em voo| D[recheck: fica no nó, NÃO reenfileira]
  B -->|actionCompleted| E[avança]
  D --> F{dead-man esgotou?}
  F -->|sim| G[dead]
```

## Fluxo 5 — Passagem e devolução
```mermaid
flowchart TD
  A[IA decide passar a humano] --> B[prepararLinhaDaPassagem: tetos + briefing anti-injection]
  B --> C[registrarPassagem: nunca lança, sem dedup]
  C --> D[aviso ao cliente + aviso ao suporte]
  D --> E[humano atende]
  E --> F{prazo de devolução? último sinal humano}
  F -->|estourou| G[devolverAtendimentoAoAgente: solta 3 travas, emit ai.handoff_resolved]
```
🟢
