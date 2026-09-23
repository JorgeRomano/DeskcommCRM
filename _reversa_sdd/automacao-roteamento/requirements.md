# Automação e Roteamento (`automation`, `routing`, `followup`, `escalacao`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 7).

## Visão Geral
Quatro subsistemas que decidem **o que acontece sozinho** e **para quem vai**, disparados por eventos do `event_log` ou por crons, todos multi-tenant (service role que bypassa RLS; cada consulta filtra `organization_id` da linha do evento, nunca do body): 🟢
- **`automation`** — motor de regras "quando isto acontecer, faça aquilo" (condições em AND, 7 ações).
- **`routing`** — roteamento de conversa para atendente humano (rodízio real, elegibilidade tz-aware, fila).
- **`followup`** — motor de cadência como grafo de nós percorrido no tempo por worker de claim, estado em event sourcing.
- **`escalacao`** — passagem IA→humano e a volta (registro imutável, briefing, avisos, disponibilidade, retomada, devolução automática).

Padrão transversal: regra pura + adaptador(es) fino(s) de I/O (`supabase-js` nas rotas, `pg.Pool` no worker). 🟢

## Responsabilidades
- Avaliar regras de automação por evento com anti-loop e postpone all-or-nothing. 🟢
- Executar 7 ações (add_tag, assign_owner, create_or_move_lead, call_webhook, send_whatsapp_message, send_ai_message, start_message_flow). 🟢
- Rotear conversa por rodízio real com elegibilidade tz-aware e fila. 🟢
- Percorrer o grafo de follow-up no tempo, com reconstrução de estado por eventos. 🟢
- Registrar a passagem IA→humano como fato imutável e devolver automaticamente. 🟢

## Regras de Negócio
- Anti-loop de profundidade 1: evento com `caused_by_rule`/`request_id "rule:"` → `skipped`. — `automation/engine.ts` 🟢
- Guard de `entity_kind` evita rodar 2× por mudança de lead (legado `lead` vs novo `crm_lead`). 🟢
- Postpone all-or-nothing ANTES de executar qualquer ação (reexecução parcial no retry seria pior). 🟢
- Status do run é derivado; `skipped` entra junto de `failed` ("não saiu e a fila não resolve"). 🟢
- `contains` em lista = pertinência da tag inteira sem caixa; em texto = includes sem caixa. — `automation/conditions.ts` 🟢
- Consentimento é gate FIXO e lê `consent.marketing.declined_at` (a recusa registrada, não a ausência de `granted_at`). — `automation/guarda-do-contato.ts` 🟢
- `call_webhook`: anti-SSRF, `redirect:"manual"` (3xx = falha), projeta só campos públicos. 🟢
- Roteamento por rodízio REAL (quem recebeu há mais tempo primeiro), não random. — `routing/decide.ts` 🟢
- `windows` vazio no schedule = 24/7 (janelas RESTRINGEM, não habilitam). — `routing/eligibility.ts` 🟢
- Follow-up: 1 inscrição viva por lead (`23505` → conflict). `immune_to_reply` só em `wait fixed`. — `followup/*` 🟢
- Passagem nunca lança (o fato já aconteceu); sem dedup (quem deduplica é o aviso da Central). — `escalacao/passagem.ts` 🟢
- Devolução automática conta do último sinal humano; `force_human` irrevogável pelo agente. — `escalacao/retomada.ts` 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Motor de regras por evento com anti-loop | Must | Evento causado por regra é `skipped` |
| RF-02 | Postpone all-or-nothing | Must | Se uma ação adia, o run inteiro vira `adiado` (retry_at) |
| RF-03 | 7 ações com guardas de envio | Must | Consentimento recusado bloqueia (`consent_declined`) |
| RF-04 | Roteamento por rodízio real | Must | Quem recebeu há mais tempo vem primeiro; sem elegível → requeue |
| RF-05 | Grafo de follow-up com estado por eventos | Must | Nó entrado 2× (armar/avançar) via `resolveWaitPhase` |
| RF-06 | Envio at-most-once no nó action | Must | Recheck com turno em voo não reenfileira (evita mensagem dupla) |
| RF-07 | Passagem imutável IA→humano | Must | Passagem nunca lança; briefing separa palavras do cliente da paráfrase da IA |
| RF-08 | Devolução automática por prazo | Should | Conta do último sinal humano; `force_human` não é revogado pelo agente |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | Anti-SSRF no webhook (resolve IP, redirect manual) | `automation/actions` (`call_webhook`) | 🟢 |
| Segurança | Webhook projeta só campos públicos, HMAC opcional | `automation/actions` | 🟢 |
| Segurança | Briefing defende contra prompt injection (paráfrase separada) | `escalacao/briefing-da-passagem.ts` | 🟢 |
| Disponibilidade | Claim de follow-up nunca lança (DB fora não mata tudo) | `followup/engine.ts` | 🟢 |
| Disponibilidade | Janela de envio no fuso do tenant (régua única) | `automation/janela-do-canal.ts` | 🟢 |
| Disponibilidade | Plantão calculado por leitura (sem cron de auto-offline) | `routing/eligibility.ts` | 🟢 |
| Corretude | Aviso ao suporte reivindica ANTES da rede (INSERT único) | `escalacao/aviso-ao-suporte.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado um evento gerado por uma regra (caused_by_rule)
Quando runAutomationForEvent avalia
Então retorna skipped:caused_by_rule (anti-loop profundidade 1)

Dado um contato com consent.marketing.declined_at
Quando checarGuardasDeContato corre
Então bloqueia com consent_declined

Dado dois atendentes elegíveis
Quando selectRoundRobin escolhe
Então vem primeiro quem recebeu atribuição há mais tempo

Dado um nó action com turno em voo
Quando o recheck roda
Então fica no nó (não reenfileira), evitando mensagem dupla
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Motor de regras (RF-01/02/03) | Must | Automação central |
| Roteamento (RF-04) | Must | Distribuição de atendimento |
| Grafo de follow-up (RF-05/06) | Must | Cadência sem duplicata |
| Passagem/devolução (RF-07/08) | Must | Continuidade IA↔humano |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/automation/engine.ts` | `runAutomationForEvent` | 🟢 |
| `lib/automation/conditions.ts` | `evaluateConditions`, `resolveField` | 🟢 |
| `lib/automation/guarda-do-contato.ts` | `checarGuardasDeContato` | 🟢 |
| `lib/routing/decide.ts` | `decideRouting`, `selectRoundRobin` | 🟢 |
| `lib/routing/eligibility.ts` | `isAttendantEligible`, `isWithinSchedule` | 🟢 |
| `lib/followup/graph-schema.ts` | `flowGraphSchema`, `nodeBranches` | 🟢 |
| `lib/followup/node-handlers.ts` | `processNode`, `selectEdge` | 🟢 |
| `lib/followup/engine.ts` | `runFollowupTick`, `processEnrollment` | 🟢 |
| `lib/escalacao/passagem.ts` | `prepararLinhaDaPassagem`, `registrarPassagem` | 🟢 |
| `lib/escalacao/retomada.ts` | `devolverAtendimentoAoAgente` | 🟢 |
| `lib/escalacao/devolucao-automatica.ts` | `avaliarDevolucao` | 🟢 |
