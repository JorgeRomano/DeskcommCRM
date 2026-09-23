# Eventos e Tempo Real — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — PDF envenenado derrubando o worker 313× 🟢
Crash não incrementava attempts, então o mesmo evento reprocessava para sempre. Agora evento órfão em `processing` CONTA como tentativa (reaper), com exceção da 1ª volta (reprocessa no mesmo tique).

## EC-02 — Worker + cron processando o mesmo evento 🟢
Claim otimista (`update status='processing' where id=? and status='pending'`); `if (!claimed) continue`. É isto que torna worker-loop + cron seguros em paralelo.

## EC-03 — Handler que lança 🟢
`dispatchEvent` roda cada handler em try/catch; um que lança vira `{status:"error"}`, não aborta o lote.

## EC-04 — Env faltando matando o worker 🟢
`drain-loop.ts` usa imports dinâmicos (a cadeia termina em `@/lib/env`, que lança no topo); import estático mataria o worker inteiro. Falha → `log.error` (não warn — a #648 ficou muda 10 dias).

## EC-05 — Boot sem tabelas do harness 🟢
`assertHarnessSchema` recusa boot se faltar `job_queue`, `lead_checkpoints`, `agent_inbox_items`, `send_ledger` (o boot não aplica migrations).

## EC-06 — Veto de orçamento gerando N×5 alertas 🟢
`ehVetoPermanenteDeNegocio` lê `err.terminal===true` (`LlmBudgetExceededError`): veto de negócio → cancelJob + warn; incidente de sistema → failJob + error+Sentry.

## EC-07 — Runs de cron perdidos após downtime 🟢
`computeNextRunAt` 'every' colapsa runs perdidos (`while(next<=now) next+=interval`, sem stampede); 'cron' recalcula do agora (auto-corretivo).

## EC-08 — Thundering herd de crons 🟢
`staggerOffsetMs` (FNV-1a 32-bit do contact_id mod windowMs) espalha o disparo sem estado; `windowMs<=0` desliga.

## EC-09 — Regra dom/dow do cron 🟢
`matches` implementa a regra Vixie: ambos restritos → OR, senão AND.

## EC-10 — SIM que chega antes do próximo next_eval_at 🟢
`aplicarRespostasQueChegaram` lê `followup_enrollments` `waiting_reply` e aplica a última inbound se `inboundEhDestaPergunta` (fecha o buraco do relógio HTTP).

## EC-11 — Fuso inválido (acento) 🟢
`organizations.timezone` é `z.string()` sem CHECK; `Intl.DateTimeFormat` lança com fuso inválido (ex.: `America/Asunción`). `fusoUtilizavel` degrada aberto para `America/Sao_Paulo`.

## EC-12 — Hora inexistente/ambígua do DST 🟢
`instanteDe` (2 passadas): spring-forward → `Math.max` (`Math.min` daria véspera em offset positivo); fall-back ambíguo → primeiro. Nunca Invalid Date.

## EC-13 — Tick que explode 🟢
`runEventLogDrainLoop`: tick que explode mantém a espera ociosa (banco fora não pode virar tempestade de tentativas); `scanned` fora da conta de `proximaEspera`.
