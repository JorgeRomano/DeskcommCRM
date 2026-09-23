# Domínio — DeskcommCRM

> Gerado pelo Detetive (Reversa) — fase de Interpretação · nível **detalhado**
> Escala de confiança: 🟢 CONFIRMADO (extraído do código/dicionário) · 🟡 INFERIDO (padrão, pode estar errado) · 🔴 LACUNA (validação humana)
> Fontes primárias: `code-analysis.md`, `data-dictionary.md`, histórico Git (`git log`), doutrina em `docs/`.

Este documento captura o **porquê** do sistema: o vocabulário do negócio, as regras implícitas
que vivem no código (condicionais, invariantes, constantes nomeadas) e as decisões de domínio que
não estão escritas em nenhum requisito, mas que o código impõe.

O DeskcommCRM é um **sistema operacional de vendas open-source com agentes de IA nativos**,
multi-nicho (e-commerce, clínicas, imobiliárias, infoprodutos, serviços), tendo o **WhatsApp como
canal primário** e **multi-tenant com RLS desde o dia 1**. A monetização é **self-host em VPS** —
quem instala numa VPS *é* o usuário, então uma regra que só funciona na máquina do dev é bug de
produto.

---

## 1. Glossário de negócio

> Termos como o sistema os usa. Onde há divergência entre o vocabulário técnico e o de produto,
> ambos aparecem.

### 1.1 Pessoas, contas e organização

| Termo | Significado no sistema | Confiança |
|---|---|---|
| **Organização / Tenant** | A empresa que usa o CRM. Unidade de isolamento (RLS). Toda tabela tenant-aware tem `organization_id`. | 🟢 |
| **Contato** (`contacts`) | A pessoa do outro lado da conversa. Identidade canônica de WhatsApp resolvida por telefone e `wa_lid`. Pode existir sem lead. | 🟢 |
| **Lead / Negócio / Pedido** (`crm_leads`) | A oportunidade comercial de um contato dentro de **um** pipeline. O vocabulário exibido (`lead`/`deal`/`pedido`) é configurável por pipeline (`settings.vocabulary`). Default: `lead→cliente`, `deal→pedido`, `won→concluído`, `lost→cancelado`. | 🟢 |
| **Papel** (`Role`) | Nível de acesso de um membro na organização: `viewer < agent < ai_operator < manager < admin` (rank crescente). `ai_operator` **nunca** é papel humano — é o papel do agente publicado, só no token efêmero. | 🟢 |
| **Platform admin** | Dono do servidor (self-host). Escopo de plataforma, acima das organizações. | 🟢 |
| **Impersonation / Suporte** | Platform admin entra no contexto de uma org para dar suporte. Modo `full` ou `support_readonly`. | 🟢 |

### 1.2 Conversa, canal e atendimento

| Termo | Significado | Confiança |
|---|---|---|
| **Canal** (`ChannelProvider`) | Provedor de transporte: `waha` (WhatsApp via QR), `meta_cloud` (WhatsApp oficial), `zernio`/`zernio_social` (BSP), `wacalls` (voz — mora em canal mas **não** transporta mensagem). | 🟢 |
| **Capability** (`ChannelCapabilities`) | O que um canal *permite* (falar fora da janela, exigir template, risco de ban...). Regra dura: features perguntam a capability, **nunca** nomeiam o provedor. | 🟢 |
| **Janela de 24h** (`messaging window`) | Prazo em que o WhatsApp permite mensagem livre após a última entrada do cliente. `waha` ignora; `meta_cloud`/`zernio` exigem template fora dela. Derivada a cada leitura, sem coluna de expiração. | 🟢 |
| **Sessão de canal** (`channel_sessions`) | A conexão de um número/conta a um provedor. Estados: `STARTING`, `SCAN_QR_CODE`, `WORKING`, `STOPPED`, `FAILED`. Só `WORKING` é utilizável. | 🟢 |
| **Comando da conversa** | Quem está no controle: `humano` · `automatico` (IA) · `ninguem` · `aguardando` · `encerrada`. | 🟢 |
| **Handoff** | Passagem do atendimento da IA para um humano. De primeira classe: `force_human` (irrevogável) e `bot_silenced_until`. | 🟢 |
| **Fronteira de atendimento** (`ServiceBoundary`) | O recorte (org, contato, conversa, demanda) sobre o qual um turno/fluxo age. Versionada (CAS) — muda quando troca de demanda. | 🟢 |
| **Demanda** (`demandas`) | Um assunto/necessidade aberto de um contato. Aberta na entrada, fechada quando resolvida. | 🟢 |
| **Caso humano** (`human case`) | Loop assíncrono IA↔humano quando a IA não consegue resolver sozinha. Estados abertos: `awaiting_human`, `awaiting_lead`; terminais: `resolved`, `escalated`, `cancelled`. | 🟢 |
| **Passagem de atendimento** (`passagens_de_atendimento`) | O registro/briefing de uma escalação: qual motor passou, origem, motivo e se o cliente foi avisado. | 🟢 |

### 1.3 Agente de IA

| Termo | Significado | Confiança |
|---|---|---|
| **Turno** (`turn`) | Uma invocação do agente sobre uma sessão LLM fresca. Tipos: `inbound_turn`, `followup_turn`, `operator_turn`, `case_reply_turn`, `approved_reply`. | 🟢 |
| **Ritual do turno** | A liturgia imposta pelo runtime: abrir (playbook + checkpoint + estado + histórico) → loop de tools → fechar (2ª chamada só com o JSON do checkpoint) → pós-checkpoint. | 🟢 |
| **Papel FALAR vs OPERAR** | O agente que fala com o lead (`inbound`) tem `send_message`; o **Operador** muta o CRM e **nunca** fala (separação por ausência de tool). | 🟢 |
| **Checkpoint** (`lead_checkpoints`) | Memória durável por lead: commitments, objections, next_action, rolling_summary. Sempre gravado na chamada de fechamento. | 🟢 |
| **Declaração do turno** (`declaracao`) | Fronteira entre FALAR e OPERAR: `undefined` (não declarou) ≠ `{nada_a_declarar:true}` (avaliou e nada há). | 🟢 |
| **Playbook** | Instruções em camadas versionadas por ponteiro: `platform` → `tenant` → `campaign`. | 🟢 |
| **Guardrail** | Gate determinístico entre a decisão do modelo e o canal (cadeia `before_send`, 11 conferências) ou de camada de IA (`guardrails-schema`). | 🟢 |
| **Pacing** | Janela de horário + anti-ban (warmup por idade do número, throttle, cap diário). | 🟢 |
| **Spinning** | Detecção de cópia em massa (mensagem idêntica repetida) — veto anti-ban. | 🟢 |
| **RAG / Conhecimento** | Base de conhecimento por tenant indexada por embedding; busca top-K com citações no turno. | 🟢 |
| **Skill situacional** | Trecho de orientação carregado só em hard-match de keyword (disclosure progressivo). | 🟢 |
| **Flywheel** | Ciclo de melhoria: juiz avalia higiene de memória → destilador propõe → humano aprova (gate). | 🟢 |
| **Orçamento de IA** (`ai_budgets`) | Teto mensal de gasto por org, medido em centavos. Bloqueio nunca acontece sem aviso prévio. | 🟢 |
| **Ponto** (`purpose`) | O uso de uma chamada de LLM: `bot_respond`, `stage_classifier`, `jailbreak_detect`, `promise_semantic`, `embedding_generate`, etc. Governa qual credencial/modelo resolve. | 🟢 |

### 1.4 Funil, agenda e financeiro

| Termo | Significado | Confiança |
|---|---|---|
| **Pipeline / Funil** (`crm_pipelines`) | A sequência de etapas. Um por org pode ser default (`is_default`) e um pode ser o pipeline de cliente (`is_client_pipeline`). | 🟢 |
| **Etapa / Estágio** (`crm_stages`) | Um passo do funil. Marcada `is_won` ou `is_lost` (mutuamente exclusivas). | 🟢 |
| **Estágio do agente** (`LeadStage`) | O funil abstrato que o agente de IA navega: `new → contacted → qualifying → qualified → negotiating → won/lost`. Distinto das etapas configuráveis do CRM. | 🟢 |
| **Score / Banda** | Probabilidade de conversão 0-100 com evidência ancorada; banda `frio`/`morno`/`quente`. | 🟢 |
| **Risco** (`RiskBucket`) | Estado de negligência de um lead: `em_dia`, `em_voo`, `em_risco`, `critico` (frio × 3). | 🟢 |
| **Prospecção** | Esteira fria: campanha que aborda candidatos novos, com autorização de contato por allowlist. | 🟢 |
| **Compromisso** (`calendar_appointments`) | Agendamento. Estados: `pending`, `confirmed`, `cancelled`, `completed`, `no_show`. "Marcado ≠ confirmado". | 🟢 |
| **Tipo de agendamento** (`calendar_event_types`) | O molde do atendimento: duração, buffers, aviso mínimo, janela de reserva, lembretes. | 🟢 |
| **Comanda / Ordem de serviço** (`sale_orders`) | Venda vinculada (idempotente por `appointment_id`). Em espanhol: "orden de servicio". | 🟢 |
| **Follow-up / Fluxo** | Grafo de nós (`trigger`, `wait`, `condition`, `ai_classify`, `match_reply`, `action`, `end`) percorrido por uma inscrição (`followup_enrollment`). | 🟢 |
| **Automação / Regra** | Condições (avaliadas em AND) → ações (`add_tag`, `assign_owner`, `create_or_move_lead`, `call_webhook`, `send_whatsapp_message`, `send_ai_message`, `start_message_flow`). | 🟢 |
| **Roteamento** | Distribuição de conversas a atendentes elegíveis (round-robin por carga + jornada). | 🟢 |

### 1.5 Compliance e plataforma

| Termo | Significado | Confiança |
|---|---|---|
| **LGPD request** (`lgpd_requests`) | Pedido do titular: `data_request` (acesso), `redact` (anonimização), `store_redact`. Escopo `contact` ou `tenant`. | 🟢 |
| **Opt-out** | Descadastro do contato (`is_blocked=true`, `blocked_reason='stop_keyword'`). Primeiro efeito pós-entrada. | 🟢 |
| **Anonimização / Redação** | Apagamento irreversível de PII (cascata). A IA **nunca** anonimiza. | 🟢 |
| **Retenção** | Janela de vida de cada tipo de dado (fila 90d, auditoria 5 anos, captação 365d...). | 🟢 |
| **Auditoria** (`audit()`) | Trilha fire-and-forget de toda mutação, com redação de PII. ~330 códigos de ação. | 🟢 |
| **Marca própria (white-label)** | O produto é revendido; o nome resolve do banco, nunca do `.env`. Código que alcança o usuário nunca escreve "Deskcomm". | 🟢 |
| **Extensão declarativa** | Módulo de nicho que pede capacidade nomeada, não importa código interno nem toca o banco. Núcleo continua útil com zero extensões. | 🟢 |

---

## 2. Regras de negócio implícitas

> Regras que o código impõe mas que nenhum requisito declara explicitamente. Cada uma cita a
> evidência. Muitas são 🟡 porque a *intenção* é inferida do comportamento.

### 2.1 Multi-tenancy e origem confiável

- **RN-01 🟢 — `organization_id` nunca vem do body.** Em toda superfície (envelope de canal, MCP,
  auth-dual, turno), a org é resolvida de fonte confiável (cookie/JWT/webhook secret/path token).
  Origem: fix da issue #236 (`OutboundEnvelope`), `McpContext`, `AuthDual`. *Por quê:* body é
  controlado pelo cliente; aceitar org do body fura o isolamento de tenant.
- **RN-02 🟢 — Service role bypassa RLS; o filtro de `organization_id` é manual e responsabilidade
  de quem escreve.** `lib/supabase/admin.ts`. Não há gate automático. Boa parte de `app/api/**` usa
  service role.
- **RN-03 🟢 — `ai_operator` jamais entra em `user_organizations`.** É o papel do agente publicado,
  só no token efêmero (`deriveActor` nunca emite `user`), para não quebrar FKs `_by_user_id` nem
  furar o gate `pre_go_live`.

### 2.2 Agente de IA — o turno

- **RN-04 🟢 — Texto direto do modelo NUNCA é enviado.** Enviar é sempre `send_message` (tool call);
  texto solto do modelo é descartado. Origem: `inbound-turn.ts`. *Por quê:* toda saída tem de passar
  pela cadeia `before_send`.
- **RN-05 🟢 — O fechamento do turno SEMPRE acontece.** O checkpoint é uma 2ª chamada com
  `purpose:'checkpoint'`, não uma tool (uma tool dependeria de o modelo lembrar de chamá-la).
- **RN-06 🟢 — Bloqueio de orçamento nunca sem aviso prévio.** `decidirOrcamento` só bloqueia depois
  de já ter avisado; e ao estourar, avisa o lead com texto de código (sem tokens), roda handoff e
  re-lança. `PISO_DE_TETO_CENTS=100`; purposes isentos: `connection_test`, `jailbreak_detect`,
  `promise_semantic`.
- **RN-07 🟢 — Máximo de envios por turno = 3** (`DEFAULT_MAX_SENDS_PER_TURN`). Teto anti-flood.
- **RN-08 🟢 — Um follow-up vivo por lead** (guarda anti-empilhamento em `schedule-followup.ts`) e
  **um retorno/reativação vivo por cliente** (MCP `retencao.ts`, índice único parcial).
- **RN-09 🟢 — Prazos relativos são convertidos no servidor.** O modelo pede `in_hours`; o servidor
  calcula a data absoluta (o modelo não sabe "hoje"). Mensagens de rejeição declaram o horário atual.
- **RN-10 🟢 — Campo forjado/extra vira erro de ENSINO, não strip silencioso.** Schemas das tools são
  largos (`.passthrough()`) para o SDK, mas a validação real é whitelist `.strict()` dentro de cada
  `apply*`.
- **RN-11 🟢 — Vocabulário interno nunca vaza para o lead.** Guardrail `internal_vocabulary`
  detecta snake_case, nomes de tool, UUID, SQLSTATE, stack trace, HTTP 403, palavras de arquitetura.
- **RN-12 🟢 — "Não achei" ≠ sucesso.** Tools MCP com `motivoDoVazio` descem a razão do vazio para
  a auditoria (issue #484). No comércio, só afirma "a loja não tem" após varredura paginada completa.
- **RN-13 🟢 — A IA lê propostas de melhoria mas não se auto-aprova.** `crm_save_org_memory` exige
  papel `ai_operator`; aplicar proposta do flywheel é gate humano (um clique).
- **RN-14 🟢 — A falha de classificador auxiliar não cala o agente.** Origem: commit
  `29eb60b78`/`5dfb6799a`. Exceção: `LlmBudgetExceededError` sempre re-lança.
- **RN-15 🟢 — Ausência de citação não trava a conversa.** Origem: commit `9ad41ffd1`.

### 2.3 Guardrails de saída (a ordem é lei)

- **RN-16 🟢 — Cadeia `before_send` de 11 gates, curto-circuito no 1º veto.** Ordem: `stop`, `lgpd`,
  `pacing`, `messaging_window`, `spinning`, `promise`, `semantic_promise`, `case_promise`,
  `internal_vocabulary`, `agenda_stall`, `disclosure`. Versão `BEFORE_SEND_CHAIN_VERSION = 7`.
- **RN-17 🟢 — 9 das 10 conferências de saída não se desligam.** Só `semantic_promise` é opcional
  (custa +1 consulta por mensagem enviada). `jailbreak_detect` (entrada) também custa +1.
- **RN-18 🟢 — Anti-ban armado por default.** `spinningEnforced` default ARMADO; `messagingWindow`
  ausente = fechado (fail-closed); mas `internalVocabularyEnforced` e `agenda` default NO-OP.
- **RN-19 🟢 — O atraso humano é pago ANTES de tomar conexão** (issue #654), e o read-then-act é
  serializado por `pg_advisory_xact_lock(hashtext(channelSessionId))` por número.
- **RN-20 🟢 — Enviar template não reabre a janela de 24h.** Só a resposta do cliente reabre; a Meta
  devolve 200+wamid mas recusa entrega por webhook (erro **131047**).

### 2.4 Canais e ingestão

- **RN-21 🟢 — Efeitos pós-entrada têm ORDEM fixa e nenhum pode lançar.** Sequência (guardada por
  testes): opt-out (passo 1) → origem da página → abrir demanda → avaliar campanha → acelerar
  pipeline → pedir despacho do agente. Um 500 dispararia tempestade de reentrega do provedor.
- **RN-22 🟢 — Opt-out é o primeiro efeito.** Abrir demanda recusa contato bloqueado, por isso o
  opt-out precede.
- **RN-23 🟢 — Schema do webhook WAHA é *loose* em tudo.** Um `.strict()` transformaria "mensagem
  entra no CRM" em "mensagem descartada".
- **RN-24 🟢 — Webhook fail-closed.** WAHA e Zernio verificam assinatura HMAC (`timingSafeEqual`);
  assinatura presente e errada → sempre rejeita. `MIN_SECRET_LEN = 16`. (Era fail-open, buraco de
  forja provado com curl.)
- **RN-25 🟢 — Gravar mensagem é tolerante (dupe > perda); silenciar o bot é estrito (não silencia
  na dúvida).** Direções opostas de propósito em `handleOutboundFromUserPhone`.
- **RN-26 🟢 — O adapter de canal é tradutor de formato puro.** Nenhuma lógica de janela/cap/horário
  vive no adapter — só na cadeia `before_send`.

### 2.5 Funil, score e risco

- **RN-27 🟢 — Pipeline de um lead é imutável (P-01).** Mover cross-pipeline = **clonar** (herda
  custom_fields inteiro; `external_id` não é herdado). FK `ON DELETE RESTRICT`. Motivo de perda
  especial `moved_to_another_pipeline` é excluído das métricas e nunca ofertado.
- **RN-28 🟢 — Lead perdido exige motivo** (`lost_reason`), do vocabulário canônico ∪
  `settings.lost_reasons`.
- **RN-29 🟢 — Score exige lastro.** CHECK `evidence @? '$.factors[*].ancora'`; precisa de
  `MINIMO_DE_SINAIS = 2`. Fórmula: `BASE=30`, +12/compromisso (teto 3), -8/objeção (teto 3),
  +5/campo BANT (teto 4), peso de risco (em_dia +10 … crítico -20). Bandas: quente ≥70, morno ≥40.
- **RN-30 🟢 — Risco: frio = 24h sem atividade, crítico = 72h (frio × 3).** Escrita só quando o
  bucket muda.
- **RN-31 🟢 — Prospecção tem teto global de 50 envios/24h** e a esteira fria anda a
  `FATOR_DA_ESTEIRA_FRIA = 4` do ritmo normal. Contato só entra para a IA quando a campanha autoriza
  (`metadata.ai_gate === "allowlist"`).
- **RN-32 🟢 — Posição no funil usa indexação fracionária** (`midpoint`, `STEP=1000`), sem
  reordenação em massa.

### 2.6 Agenda e financeiro

- **RN-33 🟢 — "Marcado ≠ confirmado".** Compromisso nasce `pending` (`aguarda_confirmacao`);
  recusa de negócio da tool é RESPOSTA ao modelo, nunca exceção.
- **RN-34 🟢 — O relógio é o da ORGANIZAÇÃO, não o do servidor.** A semana da agenda abre no fuso de
  quem olha. Origem: commits `60a125e05`, `5ca0e7a86`, `f6f860...` (semana semente como função pura
  com fuso como parâmetro).
- **RN-35 🟢 — Compromisso cancelado não enfileira entrega** (a guarda olhava o link do Meet; commit
  `d43a2ac14`). E o compromisso chega ao cliente mesmo sem Google Meet (`f220852c3`).
- **RN-36 🟢 — Remarcar um compromisso já enviado corrige o cliente sozinho** (commit `fc5f02219`).
- **RN-37 🟢 — Disponibilidade: `windows` vazio na agenda = nada publicado; no roteamento = 24/7.**
  Semânticas opostas do mesmo vazio, deliberadas.
- **RN-38 🟢 — Custo em centavos com `Math.ceil` (nunca subfatura).** Dinheiro é `_cents` + moeda;
  moeda vem de `organizations.currency`, nunca do corpo.
- **RN-39 🟢 — Comanda é imutável no que já foi pago; comissão é congelada na linha do item.**
  Abertura idempotente por `appointment_id`.

### 2.7 Fila, eventos e idempotência

- **RN-40 🟢 — Idempotência evento→job por `unique(organization_id, external_id/source_event_id)` +
  captura de `23505`.** Duplicata devolve a linha existente (`deduped:true`).
- **RN-41 🟢 — Efeito exactly-once na fila.** `completeJob` exige `status='running' AND locked_by AND
  locked_at=acquiredAt`; `rowCount≠1` lança. Uma lane por contato de cada vez
  (`DISTINCT ON coalesce(contact_id,id)` + `FOR UPDATE SKIP LOCKED`).
- **RN-42 🟢 — Trigger Postgres nunca faz HTTP.** Event sourcing leve: `event_log` + workers drenados
  por cron. `message.received` é emitido pelo trigger `trg_messages_emit_event`, não pelo ingest
  (era emissão dupla, medido 805 msgs com 2 eventos).
- **RN-43 🟢 — Backoff exponencial com teto.** Job: `power(2,attempts-1)*10` cap 120s; 'dead' → inbox.
  Follow-up: `[30s, 60s, 5m, 15m, 1h]`. Webhook: `[1s, 5s]` (3 tentativas, timeout 10s).
- **RN-44 🟢 — `created_at` opcional de propósito no event_log:** quem descarta por idade falha
  ABERTO se faltar.

### 2.8 Compliance

- **RN-45 🟢 — A IA nunca anonimiza** (`crm_list_privacy_requests` é só leitura). Anonimização é
  irreversível.
- **RN-46 🟢 — SLA LGPD: aviso em D+5 (data_request) e D+10 (redact),** dedup 24h. Feriados BR
  2026-2030 embutidos.
- **RN-47 🟢 — O PDF de LGPD não leva marca própria.** Nomeia o controlador
  (`organizations.legal_name`) e o DPO, não o nome do produto.
- **RN-48 🟢 — Consentimento é lido pela RECUSA registrada** (`consent.marketing.declined_at`),
  não pela ausência de `granted_at`.
- **RN-49 🟢 — Retenção tem piso.** Ex.: auditoria default 1825d (5 anos, L-10) piso 90d; fila 90d
  piso 7d.

### 2.9 Marca própria e packaging

- **RN-50 🟢 — Código que alcança o usuário nunca escreve "Deskcomm".** `tests/unit/branding.test.ts`
  varre `app|components|lib|workers|hooks`. Marca resolve do banco; `.env` é só semente e piso de
  rollback. Fora do DOM (e-mail, ícone, `issuer` MFA), usa `marcaDaSaida()`.
- **RN-51 🟢 — O resolvedor de marca nunca lança** (roda em `app/layout.tsx`; throw ali = 500 em
  todas as telas).
- **RN-52 🟢 — Bump de versão não pode exigir edição manual de arquivo na VPS.** Todo serviço de
  `docker-compose.prod.yml` declara `image:` publicada; publicação é ato do CI.

---

## 3. Constantes de negócio (com significado de domínio)

> Números que codificam decisões de negócio. Detalhe completo em `data-dictionary.md`.

| Constante | Valor | Significado de negócio | Fonte |
|---|---|---|---|
| `DEFAULT_MAX_SENDS_PER_TURN` | 3 | Máximo de mensagens por turno do agente | `inbound-turn.ts` |
| `PISO_DE_TETO_CENTS` | 100 | Piso do teto de orçamento de IA | `edge/llm/orcamento.ts` |
| `LIMIAR_PADRAO_PCT` | 80 | % de gasto que dispara o aviso de orçamento | `edge/llm/orcamento.ts` |
| `RAG_SIMILARITY_THRESHOLD` | 0.40 | Limiar de relevância do conhecimento (calibrado migr. 0097) | `agent-config.ts` |
| janela de pacing | 7h–22h | Horário permitido para envio (fuso do tenant) | `pacing/defaults.ts` |
| warmup | 20/50/100/200 por idade | Cap diário de mensagens por idade do número (anti-ban) | `pacing/defaults.ts` |
| `WINDOW_MS` (janela 24h) | 24h | Janela de mensagem livre do WhatsApp | `messaging-window.ts` |
| erro Meta 131047 | — | Janela fechada: Meta recusa entrega | `capabilities.ts` |
| `LEAD_STAGES` | 7 estágios | Funil abstrato do agente | `lead-state.ts` |
| score `BASE` | 30 | Ponto de partida da probabilidade | `score-formula.ts` |
| `LIMIAR_QUENTE`/`MORNO` | 70 / 40 | Fronteiras das bandas de score | `kanban/score-band.ts` |
| `RISK_COLD_HOURS`/`CRITICAL` | 24 / 72 | Frio e crítico (crítico = frio × 3) | `risk-radar.ts` |
| teto de prospecção/24h | 50 | Limite global da esteira fria | `prospecting/worker.ts` |
| `FATOR_DA_ESTEIRA_FRIA` | 4 | Prospecção anda a 1/4 do ritmo normal | `ritmo-da-esteira-fria.ts` |
| SLA LGPD | D+5 / D+10 | Prazos de aviso (data_request / redact) | `lgpd/sla-alarm.ts` |
| retenção auditoria | 1825d (5 anos) | Guarda legal L-10 | `retencao/politica.ts` |
| `INVITE_TTL_SECONDS` | 86400 (24h) | Validade do convite de time | `auth/invite-token.ts` |
| `IMPERSONATE_TTL_SECONDS` | 3600 (1h) | Validade da sessão de suporte | `impersonate/cookie.ts` |

---

## 4. Lacunas para validação humana 🔴

- 🔴 **Catálogo de modelos divergente.** `AGENT_MODELS` (`claude-sonnet-4-6`/`-haiku-4-5`/`-opus-4-7`
  em `guardrails-schema.ts`) diverge de `DEFAULT_*` (`claude-sonnet-5`/`-haiku-4-5` em `gateway.ts`).
  Qual é canônico? Impacta qual modelo o agente realmente usa.
- 🔴 **`maxScoreConhecido = 100`** na classificação inicial é 🟡 inferido — confirmar se é teto real
  ou placeholder.
- 🔴 **Colunas exatas de `sale_orders`/`sale_items`** são lacuna (Data Master). Invariantes de
  imutabilidade e comissão congelada são 🟡 inferidos do comportamento.
- 🔴 **Intenção de negócio de vários motivos de escalação** (`ORIGENS_DA_PASSAGEM` legadas:
  `legado_*`) — confirmar se ainda são caminhos vivos ou resíduo de migração.
- 🔴 **README obsoleto de `lib/ai/`** descreve arquivos/modelos que não existem — não confiar nele
  como fonte de regra.

---

> Máquinas de estado detalhadas em `state-machines.md`; matriz de permissões em `permissions.md`;
> decisões arquiteturais retroativas em `adrs/`.
