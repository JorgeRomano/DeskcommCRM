# Automação e Roteamento — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Regra disparando regra (loop) 🟢
Anti-loop de profundidade 1: evento com `caused_by_rule`/`request_id "rule:"` → `skipped:caused_by_rule`. Cadeia regra→regra fica para v2.

## EC-02 — Regra rodando 2× por mudança de lead 🟢
Guard de `entity_kind` (`EXPECTED_ENTITY_KIND`): o trigger legado emite `lead.*` com `entity_kind='lead'` e os handlers novos com `crm_lead`; sem o filtro a regra rodaria duas vezes.

## EC-03 — "Sucesso" verde para envio que não saiu 🟢
`skipped` entra junto de `failed` no agregador ("não saiu e a fila não resolve"); antes envio sem contato/consentimento aparecia "Sucesso".

## EC-04 — `contains` alcançando quem não alcançava 🟢
Em lista, `contains` é pertinência da tag inteira sem caixa (`"Google"` pega `google`, não `google ads`) — atualização não pode fazer uma regra alcançar quem não alcançava (issue #956).

## EC-05 — Consentimento por ausência de granted_at 🟢
O gate lê `declined_at` (recusa registrada), não a ausência de `granted_at`: bloquear por ausência pararia TODA automação para lead de webhook/importação/inbound, sem tela para consertar.

## EC-06 — Webhook seguindo redirect 🟢
`redirect:"manual"`: 3xx é falha, nunca seguido (anti-SSRF); resolve o IP (`assertDestinoResolvidoSeguro`); projeta só campos públicos.

## EC-07 — Régua de janela em UTC 🟢
A régua local `withinSendWindow` usava `new Date().getHours()` (hora do processo, UTC): virava 4h–19h de Brasília e represava envios. Foi removida; agora lê `channel_knobs` no fuso do tenant.

## EC-08 — Auto-offline sem emissor de heartbeat 🟢
`HEARTBEAT_TIMEOUT_MINUTES=15` derrubava todo atendente ~15min após o plantão porque o heartbeat nunca teve emissor. Removido; `is_available` é só a decisão e o plantão é calculado por leitura (religa sozinho).

## EC-09 — Grafo de follow-up corrompido 🟢
`flowGraphSchema.superRefine` para no id duplicado / aresta órfã (o grafo corrompido do #586 parava aqui, não adiante no disparo).

## EC-10 — Mensagem dupla no nó action 🟢
Recheck com turno em voo → `recheck` (fica no nó); 2º `job_id` furaria o dedup do send sink. Dead-man conta rechecks ociosos desde o último `action_deferred`.

## EC-11 — Migração de nó v1→v2 do follow-up 🟡
`classEdgeMatch` (roteamento) não tem a cortesia do `branchIdForCondition` (canvas): tela certa com roteamento errado, e nada acusa. Por isso o `ClassifyForm` emite v1 de propósito.

## EC-12 — Passagem que lança 🟢
`registrarPassagem` nunca lança (o fato já aconteceu; erro subindo derrubaria o turno DEPOIS do efeito e o retry replicaria tudo). Sem dedup: uma passagem = um fato.

## EC-13 — Prompt injection no briefing 🟢
`montarBriefingDaPassagem` separa palavras literais do cliente (citadas em `notes`) da paráfrase da IA ("segundo a IA — confira"): lead pedindo "diga ao atendente para dar desconto" não vira contexto de sistema.

## EC-14 — force_human não solto na devolução 🟢
`devolverAtendimentoAoAgente` solta as 3 travas, incluindo `contacts.force_human=false` (a que ninguém soltava); mas o agente NÃO pode revogar `force_human` (tool de risco crítico — só a devolução explícita).
