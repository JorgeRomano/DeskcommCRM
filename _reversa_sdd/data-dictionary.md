# Dicionário de Dados — DeskcommCRM

> Gerado pelo Arqueólogo (Reversa) — fase de Escavação · nível **detalhado**
> Escala de confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA
> Entidades e DTOs extraídos do código, por unidade de análise. Campos: nome, tipo, obrigatório,
> valor padrão / observação.

---

## Unidade 2 — IA de suporte (`lib/ai/` + `lib/mcp/`)

### `ModeloResolvido` 🟢 — `gateway-binding.ts:31`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `model` | `LanguageModel` | sim | instância pronta do SDK |
| `modelId` | `string` | sim | id de modelo (com/sem prefixo por provider) |
| `origem` | `"binding" \| "credencial_da_organizacao" \| "padrao"` | sim | proveniência da chave usada |

### `ClassifierModelOption` 🟢 — `classifier-models.ts:32`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `provider` | `string` | sim | anthropic/openai/... |
| `model_id` | `string` | sim | só de `ai_models` com `deprecated_at is null` |
| `display_name` | `string` | sim | rótulo humano |
| `origem` | `"org" \| "plataforma"` | sim | BYOK ou chave de plataforma |

### `LoadedCredential` / `CredentialUnavailableError` 🟢 — `credentials.ts:23,56`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `apiKey` | `string` | sim | plaintext só no retorno; nunca logado |
| `provider` | `string` | sim | |
| `label` | `string` | sim | |
| _erro_ `reason` | `not_found \| inactive \| not_validated \| wrong_org \| decrypt_failed` | — | causa da indisponibilidade |

### `ProvedorSuportado` 🟢 — `pontos/provedores.ts:34` (5 registros: anthropic, openai, google, openrouter, deepseek)
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `id` | `string` | sim | chave canônica (substituiu CHECK, migration 0127) |
| `rotulo` | `string` | sim | |
| `quandoUsar` | `string` | sim | orientação de uso |
| `aceitaEndpointProprio` | `boolean` | sim | true: openai/openrouter/deepseek |
| `catalogoSincronizavel` | `boolean` | sim | true: openrouter/deepseek |
| `ondePegarAChave` | `string` | sim | |
| `prefixoDaChave` | `string` | sim | |

### `BudgetStatus` 🟢 — `budget/check.ts:47`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `organization_id` | `uuid` | sim | |
| `monthly_limit_cents` | `number` | sim | default `0` = sem teto |
| `current_month_consumed_cents` | `number` | sim | preferência à RPC `fn_gasto_de_ia_do_mes` |
| `pct` | `number` | sim | `round(consumed*10000/limit)/100` |
| `alarm_threshold_pct` | `number` | sim | `LIMIAR_PADRAO_PCT` |
| `enforcement_mode` | `ModoDeOrcamento` | sim | |
| `enforcement_effective_at` | `timestamptz \| null` | não | |
| `gasto_incompleto` | `boolean` | sim | há `llm_calls.cost_cents is null` no mês |
| `enforcement_env` | `string` | sim | kill switch `AI_BUDGET_ENFORCEMENT` |
| `blocked_now` | `boolean` | sim | lê inbox `budget_exceeded status=open` (não recalcula) |
| `current_period_start` | `timestamptz` | sim | |
| `last_alarm_sent_at` | `timestamptz \| null` | não | |
| `updated_at` | `timestamptz` | sim | |

### `UsagePayload` 🟢 — `usage/aggregate.ts:19`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `range` | `{start, end}` | sim | |
| `totals` | `{cost_cents, total_tokens, invocations, p50_latency_ms, p95_latency_ms, handoff_rate}` | sim | `percentile = ceil((p/100)*len)-1`; handoff_rate a 4 casas |
| `series` | objeto de séries diárias | sim | |
| `by_kind` | agregação por `InvocationKind` | sim | |

### `ChaveDeEmbedding` 🟢 — `embeddings/chave.ts:79`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `apiKey` | `string` | sim | |
| `baseUrl` | `string` | não | |
| `viaGateway` | `boolean` | sim | |
| `origem` | `OrigemDaChave` (4 valores) | sim | binding/credencial/gateway/env |
| `rotulo` | `string` | sim | |
| `avisos` | `string[]` | sim | |
| — const | `MODELO_DE_EMBEDDING = "openai/text-embedding-3-small"`; `DIMENSOES = 1536` | — | modelo FIXO, não configurável |

### `TrechoEncontrado` / `ResultadoDaBusca` 🟢 — `knowledge/busca.ts:16,30`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `chunk_id` | `uuid` | sim | |
| `knowledge_source_id` | `uuid` | sim | |
| `source_name` | `string` | não | |
| `content` | `string` | sim | |
| `similarity` | `number` | sim | cosseno |
| _resultado_ `trechos` | `TrechoEncontrado[]` | sim | filtrados por `limiar` em memória |
| _resultado_ `melhorSimilaridade` | `number \| null` | sim | melhor candidato **reprovado** |

### `Citation` 🟢 — `citations/types.ts:1`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `chunk_id` / `knowledge_source_id` | `uuid` | não | |
| `source_name` / `source_type` | `string` | não | |
| `score` | `number` (0..1) | não | |
| `content` | `string` | não | |

### `agentConfigSchema` (Zod, defaults) 🟢 — `guardrails-schema.ts:141,161`
| Campo | Tipo | Default | Faixa |
|---|---|---|---|
| `temperature` | number | 0.4 | 0..2 |
| `max_tokens` | number | 1024 | 64..4096 |
| `context_message_window` | number | 20 | 1..50 |
| `rag_top_k` | number | 5 | 1..20 |
| `rag_similarity_threshold` | number | 0.4 | 0..1 |
| `confidence_threshold` | number | 0.6 | 0..1 |
| `voice` | string | "marin" | 10 opções |
| `voice_speed` | number | 0.85 | 0.25..1.5 |
| `voice_model` | string | "gpt-realtime" | 6 opções |

### `guardrailItemSchema` (discriminated union) 🟢 — `guardrails-schema.ts:26`
| Kind | Campos próprios |
|---|---|
| `regex_output_block` | pattern |
| `regex_input_block` | pattern |
| `rag_must_hit` | `min_citations` (1..10, default 1) |
| `window_check` | `start_hour`/`end_hour` (0..23), `timezone` (default "America/Sao_Paulo") |
| `contact_flag` | `field ∈ force_human \| is_blocked \| is_vip` |
> `guardrailsSchema` = array `.max(50)`.

### `McpContext` 🟢 — `mcp/types.ts:15`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `organizationId` | `uuid` | sim | fonte confiável (token), nunca do arg |
| `role` | `Role` | sim | |
| `actor` | `Actor` (`ai_agent`\|`api_token`) | sim | nunca `user` |
| `apiTokenId` | `uuid` | sim | |
| `requestId` | `string` | sim | |
| `supabase` | admin client | sim | service-role (bypassa RLS) |
| `meetingBooking` | objeto | não | contexto de agendamento |

### `McpToolDefinition<TInput>` 🟢 — `mcp/types.ts:32`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `name` | `string` | sim | wire-contract imutável |
| `description` | `string` | sim | |
| `inputSchema` | Zod | sim | |
| `category` | `"read" \| "write" \| "handoff"` | sim | |
| `requiresRole` | `Role` | sim | gate RBAC via `ROLE_RANK` |
| `requiresScope` | `"mcp:read" \| "mcp:write"` | sim | |
| `motivoDoVazio` | fn | não | "não achei" ≠ sucesso (issue #484) |
| `handler` | fn | sim | |

### `McpAuthResult` / scopes 🟢 — `mcp/auth.ts:20`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `organizationId` | `uuid` | sim | |
| `role` | `Role` | sim | de scope `role:<r>`, default `agent` |
| `actor` | `Actor` | sim | `deriveActor` |
| `apiTokenId` | `uuid` | sim | |
| `scopes` | `string[]` | sim | `role:<r>`, `actor:ai_agent`, `agent_run:<uuid>`, `mcp:read`, `mcp:write` |

### Enums / listas canônicas 🟢
| Nome | Valores | Fonte |
|---|---|---|
| `InvocationKind` | bot_respond, sentiment_classify, triage_classify, embedding_generate | `log-invocation.ts:13` |
| `TIPOS_DE_FONTE` | faq, documento, conversas, catalogo | `rag/tipos-de-fonte.ts:71` |
| `CONFERENCIAS_DE_SAIDA` | stop, lgpd, pacing, messaging_window, spinning, promise, semantic_promise, case_promise, internal_vocabulary, agenda_stall, disclosure | `guardrails/lista-de-conferencia.ts:66` |
| `ROLE_RANK` (papéis MCP) | viewer < agent < ai_operator < manager < admin | `lib/auth/types` (ref.) |
| Códigos de erro MCP | -32001/401, -32002/403, -32603/500 | `mcp/auth.ts` |
| `ARGS_REDACT_KEYS` | authorization, api_key, token, password, cpf | `mcp/audit.ts:36` |

### Constantes numéricas de domínio (unidade 2) 🟢
| Constante | Valor | Fonte |
|---|---|---|
| `PRICING_TTL_MS` | 5 min | `cost.ts:23` |
| `TIMEOUT_MS` (validador) | 5000 ms | `provider-validators.ts:39` |
| `TIMEOUT_MS` (debounce) | 2000 ms | `rag/debounce.ts` |
| `maxChars` / `overlapChars` (chunk) | 1500 / 200 | `rag/chunker.ts:12` |
| `DIMENSOES_DO_EMBEDDING` | 1536 | `embeddings/chave.ts:61` |
| `LIMIAR_PADRAO` (busca MCP) | 0.4 | `tools/evolucao.ts:56` |
| `TAMANHO_DA_PAGINA` / `PAGINAS_MAXIMAS` | 1000 / 10 | `tools/comercio.ts` |
| `DAILY_LIMIT_BOUNDS` | {min:1, max:10000} | `pacing-knobs.ts:36` |
| `RATE_LIMIT_PER_MIN` / `WINDOW_SEC` (dispatcher) | 60 / 60 | `dispatcher/index.ts:44` |
| idempotência MCP (send/start) TTL | 24 h | `tools/messages.ts`, `tools/start-conversation.ts` |

---

## Unidade 3 — Canais e mensageria

### `ChannelCapabilities` 🟢 — `lib/channels/types.ts:33`
| Campo | Tipo | Observação |
|---|---|---|
| `freeformOutsideWindow` | boolean | fala fora da janela 24h? (waha true, meta false) |
| `requiresTemplates` | boolean | exige template p/ iniciar (meta/zernio true) |
| `canManageTemplates` | boolean | provedor gerencia templates |
| `banRisk` | boolean | risco de banimento (waha true) |
| `minIntervalMs` | number \| null | intervalo mínimo entre envios (meta 6000) |
| `voiceNote` | `"server-convert" \| "opus-only"` | conversão de áudio |
| `groups` | `"full" \| "limited" \| "none"` | suporte a grupos |
| `costPerMessage` | boolean | mensagem cobrada (meta/zernio true) |

### `ChannelProvider` / enums de canal 🟢
| Nome | Valores | Fonte |
|---|---|---|
| `ChannelProvider` | waha, meta_cloud, zernio, zernio_social, wacalls | `types.ts:13` |
| `ProviderDeMensagem` | ChannelProvider sem `wacalls` | `types.ts:31` |
| `ESTADOS_DO_CANAL` | STARTING, SCAN_QR_CODE, WORKING, STOPPED, FAILED | `estado.ts:37` |
| `STATUS_QUE_AVISAM` | SCAN_QR_CODE, FAILED, STOPPED | `health.ts:50` |
| `DEFAULT_CHANNEL_PROVIDER` | "waha" | `capabilities.ts:96` |

### `OutboundEnvelope` 🟢 — `lib/channels/types.ts:114`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `organizationId` | uuid | sim | de fonte confiável, nunca do body (issue #236) |
| `sessionRef` | ChannelSessionRef | sim | |
| `to` | string | sim | destinatário resolvido |
| `kind` | OutboundKind | sim | text/image/audio/... |
| `body` | string | não | |
| `media` | OutboundMedia `{url, mime, filename?, caption?}` | não | |
| `providerConversationId` | string | não | thread (Zernio) |
| `replyToExternalId` | string | não | wamid citado |
| `beforeSend` | fn | não | re-valida origem após preparo async |

### `ChannelSessionRef` (união marcada) 🟢 — `session-ref.ts:16`
| provider | ref |
|---|---|
| waha | `waha_session_name` |
| meta_cloud | `meta_phone_number_id` |
| zernio / zernio_social | `zernio_account_id` |

### `EstadoDaJanela` (união) 🟢 — `janela.ts:35`
| tipo | campos |
|---|---|
| `sem_restricao` | — |
| `aberta` | `restanteMs: number` |
| `fechada` | `fechadaHaMs: number \| null` |

### `wahaPayloadSchema` (Zod loose, campos `.nullish()`) 🟢 — `waha/envelope.ts:70`
| Campo | Tipo | Observação |
|---|---|---|
| `id` | string | id composto do provedor |
| `from` / `to` | string | chatId (`@c.us`/`@lid`/`@g.us`) |
| `fromMe` | boolean | roteia inbound vs outbound-from-phone |
| `body` | string | |
| `type` / `hasMedia` | string / boolean | mapeado por `WA_TYPE_MAP` |
| `ack` / `ackName` | number / string | ack≥2 delivered, ≥3 read |
| `timestamp` | number | unidade inferida por magnitude |
| `media` | `{url, mimetype}` | |
| `_data.key` | `{remoteJidAlt, participantAlt}` | telefone alternativo p/ `@lid` |
> Regra: schema loose em tudo — `.strict()` descartaria mensagem que deveria entrar.

### Máquina de comando da conversa 🟢 — `inbox/comando-da-conversa.ts`
| Tipo | Valores |
|---|---|
| `Comando` | humano · automatico · ninguem · aguardando · encerrada |
| `MotivoDoSilencio` | atendente_no_comando · contato_travado · pausado · resposta_humana_recente · contato_descadastrado |
| `STATUS_ENCERRADOS` | closed, archived, resolved |
| sentinelas | `INFINITO="infinity"`, `MENOS_INFINITO="-infinity"` |
| `ORDEM_DA_ESPERA` | `awaiting_since` asc, nullsFirst false (issue #990/#994) |

### `ServiceBoundary` 🟢 — `atendimento/fronteira.ts:2`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `organization_id` | uuid | sim | |
| `contact_id` | uuid | sim | |
| `conversation_id` | uuid | sim | |
| `service_revision` | number | sim | CAS; bump só ao TROCAR de demanda (0222) |
| `demanda_id` | uuid \| null | sim | |
| `demanda_revision` | number \| null | sim | |
> `CurrentServiceBoundary` adiciona `status`, `demanda_fechada_em`. Mismatch → `StaleServiceBoundaryError`.

### `EmailDeliveryError` / `SmtpConfig` 🟢 — `email/roteador.ts`, `email/config.ts`
| Nome | Valores / campos |
|---|---|
| `EmailDeliveryError` | not_configured, send_failed, rate_limited, sender_rejected, dominio_nao_verificado |
| `TransporteDeEmail` | smtp \| resend |
| `SmtpSecurity` | starttls \| tls \| none |
| `SmtpConfig` | host, port, security, username, password, fromEmail, fromName, source (database\|environment\|none) |

### `NOTIFY_KINDS` 🟢 — `notifications/kinds.ts`
`message_inbound`, `alerts_toggle`, `lead_assigned`, `lead_won`, `lead_lost`, `mention`,
`call_inbound` (cada um `{sound, tagPrefix}`). `BODY_MAX = 140` (`emit.ts`). Prefs em localStorage
`"notify.prefs.v1"`; push VAPID via `VAPID_PUBLIC_KEY`/`VAPID_PRIVATE_KEY`; 404/410 deleta
subscription morta (`web_push.ts`).

### Constantes de domínio (unidade 3) 🟢
| Constante | Valor | Fonte |
|---|---|---|
| `MIN_SECRET_LEN` (webhook) | 16 | `waha/webhook-auth.ts`, `channels/inbound.ts` |
| `JANELA_DO_ECO_MS` | 60000 | `waha/ingest.ts` |
| `TETO_PADRAO_MS` / `TETO_DE_MIDIA_MS` | 15000 / 30000 | `waha/client.ts` |
| `TETO_NOME_DE_SESSAO_WAHA` | 54 | `channels/nome-da-sessao.ts:34` |
| `LIMIAR_URGENTE_MS` (janela) | 7200000 (2h) | `channels/janela.ts:127` |
| `DIAS_ATE_AVISAR` (canal mudo) | 3 | `channels/canal-mudo.ts:44` |
| `minIntervalMs` meta/zernio | 6000 | `channels/capabilities.ts` |
| erro Meta janela fechada | 131047 | `channels/capabilities.ts` (doc) |

---

## Unidade 4 — Voz / Telefonia (`lib/voice/`, `lib/voip/`, `lib/wacalls/`)

### Tabela `voice_calls` 🟢 — unificada wacalls + sip (migration 0348)
Fonte da verdade das chamadas dos dois canais; `provider` discrimina a origem. Campos lidos no código
(`wacalls/events-bridge.ts`, `wacalls/calls.ts`, `app/api/v1/calls/route.ts`, `workers/voice-agent/index.ts`):

| Campo | Tipo | Obrig. | Notas |
|---|---|---|---|
| `id` | uuid | sim | nosso uuid; é o que vaza pro frontend (nunca `wacalls_call_id`) |
| `organization_id` | uuid | sim | tenant |
| `channel_session_id` | uuid | wacalls | FK para `channel_sessions` (só canal WaCalls) |
| `provider` | text | sim | `wacalls` \| `sip` |
| `wacalls_call_id` | text | wacalls | id da chamada no upstream WaCalls; `unique (organization_id, wacalls_call_id)` |
| `asterisk_channel_id` | text/uuid | sip | UUID do AudioSocket (correlação worker↔linha) |
| `direction` | text | sim | CHECK `outbound` \| `inbound` (migration 0347) |
| `status` | text | sim | CHECK `starting` \| `ringing` \| `connected` \| `ended` (vocabulário WaCalls upstream) |
| `end_reason` | text | não | valor **cru** do upstream, sem CHECK (`user_ended`, `timeout`, `busy`…) |
| `peer_phone` | text | sim | número do cliente (dígitos puros vindos do WhatsApp, ou E.164 no SIP) |
| `contact_id` | uuid | não | resolvido por duas grafias do celular; `null` se não casou |
| `owner_user_id` | uuid | não | FK `auth.users`; quem está na linha / quem discou; só grava valor com forma de UUID |
| `created_by` | uuid | não | quem discou pelo CRM (`null` em recebida) |
| `lead_id` | uuid | não | vínculo opcional (SIP outbound) |
| `handled_by` | text | não | CHECK `human` \| `ai` \| `ai_then_human` |
| `started_at` | timestamptz | sim | |
| `answered_at` | timestamptz | não | carimbado quando `status='connected'`; `null` = nunca atendida |
| `ended_at` | timestamptz | não | |
| `duration_ms` | bigint | não | `answered_at ? ended-answered : null`; **não é generated column** (era em `crm_calls`) |
| `transcript` | jsonb | não | array `{speaker, text, ts}` (SIP/OpenAI Realtime) |
| `updated_at` | timestamptz | sim | |

**Derivação do `status` da API (`mapStatusParaApi`) 🟢:** `connected→in_progress`; `≠ended → ringing`;
`ended` + `end_reason`: `timeout→no_answer`, `busy→busy`, `failed→failed`, `cancelled→canceled`, senão
`completed`.

### Tabela `org_voice_calls` 🟢 — opt-in de voz WhatsApp (migration 0234)
| Campo | Tipo | Obrig. | Notas |
|---|---|---|---|
| `organization_id` | uuid | sim | PK/uma linha por org |
| `enabled` | boolean | — | consentimento; **ausência de linha (`null`) = desligado** |
| `risco_aceito_em` | timestamptz | não | quando o admin aceitou o risco de vincular 2º aparelho |

### Tabela `voip_trunk_settings` 🟢 — trunk SIP por org (migration 0349), senha cifrada AES-GCM
| Campo | Tipo | Obrig. | Notas |
|---|---|---|---|
| `organization_id` | uuid | sim | PK (upsert `onConflict`) — é a config de UM trunk, não lista |
| `host` | text | sim | |
| `port` | int | sim | |
| `username` | text | sim | |
| `password_encrypted` | bytea | — | AES-GCM ciphertext; nunca plaintext em disco |
| `password_iv` | bytea | — | IV do AES-GCM |
| `password_tag` | bytea | — | tag de autenticação |
| `password_last4` | text | — | últimos 4 dígitos em claro (exibição) |
| `from_domain` | text \| null | não | |
| `endpoint_name` | text | sim | **derivado**: `org-<uuid>-trunk-endpoint` (bate com `pjsip.conf`) |
| `is_active` | boolean | sim | trunk inativo cai no fallback `VOIP_TRUNK_ENDPOINT` |
| `updated_by` | uuid | sim | FK `auth.users` |

### Colunas de `channel_sessions` usadas pela voz WhatsApp 🟢
`wacalls_session_id` (id da sessão no upstream; zerado ao desparear), `wacalls_paired_at` (timestamp do
pareamento; `null` = não pareado), `provider='wacalls'`, `archived_at` (arquivar = "desligado"),
`status` (`WORKING`/`STOPPED`), `last_status_change_at`.

### Tabela `agent_inbox_items` — aviso de chamada perdida 🟢
Inserida por `handleCallEnded` quando recebida não atendida: `kind='voice_call_missed'`,
`severity='warn'`, `title="Chamada perdida de <peer>"`, `body=motivoDaChamadaEmPortugues`,
`ref_kind='contact'` (ou `null`), `ref_id=contact_id`.

### DTOs de transporte (interfaces TS) 🟢
| Interface | Arquivo | Campos-chave |
|---|---|---|
| `EscolhaDeVoz` | `voice/opt-in.ts` | `boolean \| null` (`null`=nunca escolheu) |
| `EstadoDaVoz` | `voice/opt-in.ts` | `ligada`, `instalacaoOferece`, `escolhaDaOrg`, `motivo` (`ligada`\|`instalacao_nao_oferece`\|`organizacao_nao_ligou`) |
| `NumeroDiscavel` | `voice/numero-discavel.ts` | `digitos` (só dígitos, sem `+`), `fonte` (`whatsapp`\|`cadastro`) |
| `ResultadoDoDesparear` | `voice/desparear.ts` | `desapareado: boolean`, `channelSessionId: string\|null` |
| `WacallsSessionInfo` | `wacalls/client.ts` | `id, name, jid, state, paired` |
| `WacallsCallRecord` | `wacalls/client.ts` | `sessionId, callId, owner, direction, peer, startedAt, status(starting\|ringing\|connected\|ended), endedAt?, endReason?` |
| `VoiceCallWithSession` | `wacalls/calls.ts` | `id, wacallsCallId, wacallsSessionId, status, contactId, ownerUserId, createdBy` |
| `WacallsSessionRow` | `wacalls/session.ts` | `channelSessionId, wacallsSessionId` |
| `WacallsBridgeConfig` | `wacalls/events-bridge.ts` | `baseUrl, apiToken, maxBackoffMs` |
| `WacallsSessionMap` | `wacalls/events-bridge.ts` | `channelSessionId, organizationId` |
| `OriginateParams` | `voip/ariClient.ts` | `toNumber, fromNumber?, trunkEndpoint, callerLabel?, channelId` |
| `AriChannel` / `AriEvent` | `voip/ariClient.ts` | `id, dialplan.exten?, caller.{number,name}?` / `type, channel?, cause_txt?` |
| `RoteamentoInbound` | `workers/voice-agent/index.ts` | `organization_id, routing_mode(ai\|human\|ai_then_human), default_ai_agent_id, fallback_user_id` |
| `PedidoDeGuardarTrunk` | `voip/guardar-trunk.ts` | `admin, orgId, userId, host, port, username, password?, fromDomain, isActive` |

### Vocabulário (unidade 4) 🟢 — `voip/call-vocabulary.ts`
| Tipo | Valores | 1:1 com banco? |
|---|---|---|
| `CallDirection` | `outbound` \| `inbound` | sim (CHECK 0347) |
| `CallStatus` (API) | `ringing` \| `in_progress` \| `completed` \| `no_answer` \| `busy` \| `failed` \| `canceled` | não (derivado) |
| `CallHandledBy` | `human` \| `ai` \| `ai_then_human` | sim |
| `PhoneNumberRoutingMode` | `ai` \| `human` \| `ai_then_human` | sim |
| `AiAgentChannel` | `whatsapp` \| `voice` | sim |
| `EndCallReason` (upstream, traduzido) | `user_ended`, `declined`, `rejected`, `timeout`, `busy`, `cancelled`/`canceled`, `failed`, `do_not_disturb`, `unknown` | cru, sem CHECK |

### Constantes de domínio (unidade 4) 🟢
| Constante | Valor | Fonte |
|---|---|---|
| `PRAZO_DA_CONSULTA_MS` | 4000 | `voice/numero-discavel.ts` (race do lookup de grafia) |
| `MOTIVO_DO_SILENCIO` | "Ligação de voz em andamento" | `wacalls/events-bridge.ts` (predicado de reversão) |
| `TETO_DO_SILENCIO` | `now() + interval '2 hours'` | `wacalls/events-bridge.ts` (anti-morte, não estimativa) |
| backoff SSE inicial / teto | 1000 ms / `maxBackoffMs` | `wacalls/events-bridge.ts` (exponencial ×2) |
| `AUDIOSOCKET_PORT` | 9092 (default) | `workers/voice-agent/index.ts` |
| frame AudioSocket tipo UUID | `0x01` | `workers/voice-agent/index.ts` |
| `BIAS` / `CLIP` (μ-law) | `0x84` / `32635` | `voip/ulaw.ts` (ITU-T G.711) |
| clamp PCM float | `[-1, 1]`, `NaN→0` | `wacalls/pcm.ts` |
| nome da sessão de voz | `org_<uuid inteiro>` | `wacalls/nome-da-sessao.ts` |
| nome do endpoint SIP | `org-<uuid>-trunk-endpoint` | `voip/guardar-trunk.ts` |
| `ARI_APP` (default) | `voice-agent` | `voip/ariClient.ts` |

---

## Unidade 5 — CRM e Funil

### Tabela `crm_leads` 🟢 — o negócio
| Campo | Tipo | Obrig. | Notas |
|---|---|---|---|
| `id` | uuid | sim | |
| `organization_id` | uuid | sim | tenant |
| `contact_id` | uuid | sim | a pessoa |
| `pipeline_id` | uuid | sim | **imutável** (P-01); FK `ON DELETE RESTRICT`; mover cross-pipeline = clonar |
| `stage_id` | uuid | sim | FK `ON DELETE RESTRICT` |
| `title` | text | sim | via `rotuloDoContato` no nascimento |
| `description` | text | não | |
| `status` | text | sim | `open` \| `won` \| `lost` (escrito pelo trigger no fechamento) |
| `value_cents` | bigint | não | nullable → pendência mais comum de conversão |
| `currency` | text | não | ISO-4217, default BRL |
| `owner_user_id` | uuid | não | dono humano |
| `owner_agent_id` | uuid | não | dono agente (FK `ai_agents`) |
| `owner_kind` | text | não | `user` \| `ai` \| null (legado); coerente com a coluna preenchida (0070) |
| `last_activity_at` | timestamptz | não | carimbado por `fn_update_last_activity_at` (lista positiva 0079) |
| `created_at` | timestamptz | sim | |
| `closed_at` | timestamptz | não | trigger no fechamento |
| `position_in_stage` | numeric | não | indexação fracionária (`midpoint`, STEP 1000) |
| `tags` | text[] | não | |
| `source` | text | não | `whatsapp`/`voip`/`site`/`importacao_planilha`/... |
| `source_metadata` | jsonb | não | atribuição, `clonado_de`, `movido_para`, `telefone_em_conflito` |
| `custom_fields` | jsonb | não | herdado inteiro no clone |
| `lost_reason` | text | não | obrigatório em lost; vocabulário canônico ∪ `settings.lost_reasons` |
| `expected_close_date` | date | não | base do filtro `overdueOnly` |
| `external_id` | text | não | `uniq_crm_leads_org_source_external`; não herdado no clone |

### Tabela `crm_stages` 🟢 — a etapa
`id, organization_id, pipeline_id, name, slug, position (numeric), is_won, is_lost, is_archived,
agent_stage_hint, expected_duration_hours (janela de risco), last_change_actor_kind, last_change_at`.
Índices: `uniq_crm_stages_pipeline_slug` (não parcial), `uniq_crm_stages_pipeline_won/_lost/_hint`
(parciais `where is_archived=false`). CHECKs: `crm_stages_won_lost_mutex`, `crm_stages_hint_coerente_com_won_lost`,
`crm_stages_slug_format` (`^[a-z0-9_-]{2,40}$`).

### Tabela `crm_pipelines` 🟢 — o funil
`id, organization_id, name, slug, position, is_default, is_client_pipeline (0262), is_archived,
description, settings (jsonb: fields[], canonical_tags[], lost_reasons[], vocabulary{lead,deal,won,lost})`.
Índices únicos `uniq_crm_pipelines_org_default` (`where is_default=true`), `uniq_crm_pipelines_org_client`,
`uniq_crm_pipelines_org_slug`.

### Tabela `crm_lead_scores` 🟢 — score/probabilidade
`lead_id (PK), organization_id, ai_probability (0-100), ai_probability_reason, ai_probability_evidence
(jsonb {formula_v, factors[{pontos,frase,ancora?}]}), ai_probability_at, ai_probability_band
(frio|morno|quente), ai_probability_band_since, updated_at`. CHECK `evidence @? '$.factors[*].ancora'`
(exige lastro). Apagada quando o score vira null.

### Tabela `crm_lead_risk_states` 🟢 — estado de risco
`lead_id (PK), organization_id, bucket (critico|em_risco|em_voo|em_dia), since, cold_hours, detected_at
(default now())`. CHECK `since <= detected_at` (`crm_lead_risk_states_since_no_passado`). Na publicação
realtime — só escrita quando o bucket muda.

### Tabela `crm_lead_reactivations` 🟢
`id, lead_id, organization_id, status (pending|expired|accepted|dismissed), expires_at (= esfriamento + coldHours),
decided_at`. Índice único parcial por lead `pending`.

### Tabela `crm_lead_activities` 🟢 — timeline
`id, organization_id, lead_id, contact_id, type (ActivityType ~45 valores), source_module, source_id,
actor_kind (user|ai|system|rule|contact), actor_agent_id, performed_by_user_id, reason, evidence (jsonb:
run_ids→ai_agent_runs, trace_ids→trace, llm_call_ids→llm_calls), payload (jsonb), performed_at`.
Sem CHECK de tipo (clone com tipo legado não quebra update.sh). `fn_update_last_activity_at` (0079) decide
por `type` se carimba `crm_leads.last_activity_at`.

### Tabelas de apoio do funil 🟢
- **`lead_checkpoints`** (por contato): `id, contact_id, organization_id, seq, commitments[], objections[], next_action, rolling_summary` — lastro do score.
- **`lead_state`** (por contato): `contact_id, organization_id, qualification (jsonb BANT), next_action, next_action_seq, updated_at, stage`.
- **`demandas`**: `id, lead_id (nullable, ON DELETE SET NULL), contact_id, organization_id, aberta_em, fechada_em, proximo_passo, origem`.

### Tabela `contacts` (campos usados na unidade 5) 🟢
`id, name (editável), display_name (ingestão/pushName), email, email_normalized, phone_number, wa_lid,
is_blocked, is_merged_into, is_anonymized, source, source_metadata (jsonb: telefone_em_conflito), first_service_at,
custom_fields (jsonb: link_<tipo>), cpf_hash (sha256), cpf_encrypted (bytea, LACUNA), created_at, last_activity_at`.
Índices únicos parciais telefone/email_normalized/cpf_hash `where is_merged_into is null`.

### Tabelas de prospecção 🟢
- **`prospecting_campaigns`**: `id, organization_id, status (running|paused|completed), search_status (starting|running|unknown|...), config (jsonb: channel_session_id, agent_id, daily_limit, interval_minutes, instruction, qualification), next_send_at, error, created_at, updated_at`.
- **`prospecting_candidates`**: `id, organization_id, campaign_id, status (queued|sending|sent|failed), phone, contact_id, conversation_id, message_id, service_boundary, data (jsonb: name, category, address, website, rating, socials[]), attempted_at, error, created_at`.

### Tabela `conversion_ledger` (via `registro-de-envio`) 🟢
Livro-razão de conversões: `organization_id, lead_id, plataforma (meta_ads|google_ads), evento (Purchase),
status (sent|skipped|error), motivo, evento_id ("<leadId>:Purchase", idempotência), valor_centavos, moeda, detalhe`.

### Vocabulário / enums (unidade 5) 🟢
| Tipo | Valores |
|---|---|
| `RiskBucket` | `critico` \| `em_risco` \| `em_voo` \| `em_dia` |
| `ScoreBand` | `frio` \| `morno` \| `quente` |
| lead `status` | `open` \| `won` \| `lost` |
| `OwnerKind` | `user` \| `ai` \| null |
| `ActivityActorKind` | `user` \| `ai` \| `system` \| `rule` \| `contact` |
| `LeadStage` (passo do agente) | `new` \| `contacted` \| `qualifying` \| `qualified` \| `negotiating` \| `won` \| `lost` |
| classificação inicial | `desqualificado` \| `revisao_humana` \| `A` \| `B` \| `C` \| `D` \| `nao_avaliado` |
| `MotivoDeDuplicidade` | `telefone` \| `email` \| `telefone_em_conflito` |
| motivo perda canônico especial | `moved_to_another_pipeline` (excluído das métricas, nunca ofertado) |

### Constantes de domínio (unidade 5) 🟢
| Constante | Valor | Fonte |
|---|---|---|
| `BASE` (score) | 30 | `leads/score-formula.ts` |
| `POR_COMPROMISSO` / teto | +12 / 3 | `leads/score-formula.ts` |
| `POR_OBJECAO` / teto | -8 / 3 | `leads/score-formula.ts` |
| `POR_CAMPO_BANT` / teto | +5 / 4 | `leads/score-formula.ts` |
| `MINIMO_DE_SINAIS` | 2 | `leads/score-formula.ts` |
| `PESO_RISCO` | em_dia +10 / em_voo 0 / em_risco -10 / critico -20 | `leads/score-formula.ts` |
| `FORMULA_V` | 1 | `leads/score-formula.ts` |
| `LIMIAR_QUENTE` / `LIMIAR_MORNO` / `BANDA` | 70 / 40 / 5 | `kanban/score-band.ts` |
| `RISK_COLD_HOURS` / `RISK_CRITICAL_HOURS` | 24 / 72 (crítico = frio×3) | `leads/risk-radar.ts` |
| `SCAN_CAP` / `IDS_POR_CONSULTA` | 500 / 100 | `leads/radar-de-risco.ts` |
| `STEP` (posição) | 1000 | `kanban/fractional-indexing.ts` |
| `JANELA_MS` / `LIMITE_DE_BLOCOS` | 60000 / 12 | `leads/timeline-grouping.ts` |
| `FOLGA_MS` / `FALLBACK_MS` (echo) | 1000 / 4000 | `kanban/local-echo.ts` |
| `MAX_EVIDENCIAS` (card) | 3 | `kanban/card-state.ts` |
| `FATOR_DA_ESTEIRA_FRIA` | 4 | `prospecting/ritmo-da-esteira-fria.ts` |
| teto global de prospecção / 24h | 50 | `prospecting/worker.ts` |
| `PALETA_DE_ETIQUETAS` | 8 tons fixos (medidos) | `tags/cor-da-etiqueta.ts` |
| `maxScoreConhecido` (classif.) | 100 (🟡 inferido) | `leads/config-classificacao-inicial.ts` |
| `ETAPAS_INICIAIS` | Novo / Em andamento / Ganho / Perdido | `pipelines/pipeline-editing.ts` |
| `SLUG_MIN` / `SLUG_MAX` | 2 / 40 | `leads/stage-editing.ts` |

---

## Unidade 6 — Agenda e Financeiro

### Tabela `calendar_appointments` 🟢 — o compromisso
| Campo | Tipo | Obrig. | Notas |
|---|---|---|---|
| `id` | uuid | sim | |
| `organization_id` | uuid | sim | tenant |
| `owner_user_id` | uuid | não | dono do compromisso (resolve `default_owner_user_id` do tipo) |
| `contact_id` | uuid | não | a pessoa (pode ser contato sem lead) |
| `title` | text | sim | |
| `description` | text | não | |
| `starts_at` | timestamptz | sim | |
| `ends_at` | timestamptz | sim | `ends_at > starts_at` |
| `time_zone` | text | sim | fuso da regra do tipo |
| `status` | text | sim | `pending`\|`confirmed`\|`cancelled`\|`completed`\|`no_show` (CHECK migr. 0176) |
| `location_kind` | text | sim | `in_person`\|`phone`\|`whatsapp`\|`video_link`\|`google_meet` |
| `location_details` | text | não | endereço ou url conforme `CAMPO_EXIGIDO_PELO_LOCAL` |
| `meeting_url` | text | não | link do Meet quando `meeting_state=ready` |
| `meeting_state` | text | não | `not_requested`\|`pending`\|`ready`\|`failed`\|`cancelled` |
| `meeting_allowed_types` | text[] | não | ex. `hangoutsMeet` |
| `revision` | int | não | otimista; base do merge |
| `google_connection_id` | uuid | não | FK `calendar_connections` |
| `google_calendar_id` | text | não | |
| `google_event_id` | text | não | identidade estável `deskcomm...` |
| `google_etag` | text | não | If-Match |
| `google_local_revision` / `google_synced_local_revision` | int | não | controle de sync |
| `google_base_projection` | jsonb | não | base do merge de três pontas (hashes) |
| `google_conflict` | jsonb | não | conflito pendente + resolução |
| `google_pending_write` | jsonb | não | escrita reservada/pendente |
| `claim` | jsonb | não | lease do worker (token/epoch/lease_until) |
| Índice parcial | | | `calendar_appointments_org_vivos_idx` (só `SITUACOES_VIVAS`) |

Vínculo com lead: **polimórfico** via `crm_lead_links` (`target_kind='appointment'`, `link_kind='scheduled'`)
— não há coluna `lead_id` na tabela (DECISÃO 6).

### Tabela `calendar_event_types` 🟢 — o molde do atendimento
`id, organization_id, name, slug, description, category (CATEGORIAS_DE_AGENDAMENTO), duration_minutes,
buffer_before_minutes, buffer_after_minutes, minimum_notice_minutes, slot_interval_minutes (null⇒duração),
booking_window_days, default_owner_user_id, location_kind, location_details, requires_confirmation,
is_active, reminder_enabled, reminder_minutes_before, reminder_extra_offsets_minutes (int[], teto 20),
reminder_body, reminder_bodies (Record<minuto,texto>), default_price_cents`.

### Tabela `calendar_availability_exceptions` 🟢 — exceções de disponibilidade
`id, organization_id, user_id, exception_date (YYYY-MM-DD; ⚠️ NÃO `date`), is_unavailable (bool),
start_minute (0-1440), end_minute`. Dia inteiro = `(0, 1440)` **sempre preenchido** (NULL não colide com
NULL na UNIQUE). Disponível SUBSTITUI a base do dia; indisponível SUBTRAI.

### Tabela `attendant_availability` 🟢 — a jornada
`user_id, organization_id, schedule (jsonb: {timezone, windows:[{dow, start:"HH:MM", end:"HH:MM"}]})`.
Fonte ÚNICA da jornada — a agenda lê, não duplica. `windows` vazio = nada publicado (≠ 24/7 do roteamento).

### Tabela `calendar_connections` 🟢 — a agenda conectada
`id, organization_id, user_id, provider (`google_calendar`), account_email, status (SITUACOES_DA_CONEXAO:
connecting|healthy|token_expired|scope_missing|disconnected|rate_limited|error, CHECK migr. 0177),
oauth_access_token_encrypted, oauth_refresh_token_encrypted, last_sync_at, calendar_selection_revision,
sync_cursor (jsonb: generation/mode/base_sync_token/page_token/window)`. RLS
`calendar_connections_dono_ou_manager_read` (esconde de agent/viewer — leituras via RPC).
Unique key `(organization_id, user_id, provider, account_email)`.

### Tabela `calendar_external_events` 🟢 — eventos vindos do Google
`external_event_id, organization_id, connection_id (dono via join, sem user_id próprio), external_calendar_id,
title, starts_at, ends_at, is_all_day, status (SITUACOES_EXTERNAS: confirmed|tentative|cancelled),
transparency (opaque|transparent), external_updated_at, ical_uid, sequence, ocupa (bool)`.
View `calendar_google_reconcilable_appointments` (candidatos a push).

### Tabela `platform_google_oauth` 🟢 — o app OAuth da instalação
`id (=1, singleton), client_id, client_secret_encrypted`. DB-first; fallback `.env`
(`GOOGLE_CALENDAR_CLIENT_ID/SECRET`). 42P01 = clone antes da migration 0201.

### Tabelas financeiras 🟢
- `financial_accounts` — `id, organization_id, name, kind (cash|bank|other), opening_balance_cents
  (int assinado), currency (3 chars, def BRL), is_active, created_at`.
- `payment_methods` — `id, organization_id, name, account_id (uuid nullish), is_active, created_at`.
- `account_plans` — `id, organization_id, name, direction (in|out, SEM default), is_active, created_at`.
- `commission_rules` — `id, organization_id, name, attendant_user_id (nullish), event_type_id (nullish),
  percent (0-100), is_active, created_at`. CHECK: ao menos um alvo (pessoa OU serviço).
- `recurring_entries` — `id, organization_id, name, account_id, account_plan_id (nullish), direction,
  amount_cents (1..1e9), day_of_month (1-31; cron ajusta p/ último dia existente), is_active, created_at`.
- `sale_orders`/`sale_items` (comanda, migration 0240) 🟡 — invariantes garantidos pelo schema: nada
  apagado, saldo derivado, lançamento pago imutável, comissão congelada na linha do item. Colunas exatas
  são lacuna para o Data Master. Item guarda `event_type_id, attendant_user_id, quantity, unit_price_cents,
  discount_cents, commission_percent` (congelado). Abertura idempotente por `appointment_id`.

### Tabela `catalog_products` 🟢 — o catálogo de produtos
`id, organization_id, codigo (≤60 chars, identidade estável), nome, preco_cents, custo_cents (nullish),
marca, categoria, quantidade, controla_estoque (bool), moeda (de `organizations.currency`, nunca do corpo)`.

### DTOs puros da unidade 6 🟢
| DTO | Onde | Campos-chave |
|---|---|---|
| `Slot` | `horarios-livres.ts` | `inicio:Date, fim:Date` (instantes) |
| `Ocupado` | `horarios-livres.ts` | `inicio, fim` |
| `JornadaDaAgenda` | `horarios-livres.ts` | `timezone, windows[]` |
| `ExcecaoDeData` | `horarios-livres.ts` | `data, indisponivel, inicioMinuto, fimMinuto` |
| `TipoDeAgendamento` | `horarios-livres.ts` | `duracaoMin, bufferAntes/DepoisMin, avisoMinimoMin, intervaloMin?, janelaDias` |
| `TipoDeAtendimento` | `consulta.ts` | 20+ campos do molde (id, nome, slug, categoria, buffers, lembretes...) |
| `AgendamentoListado` | `consulta.ts` | `id, titulo, iniciaEm, terminaEm, situacao, donoId, contatoNome, meetingUrl?` |
| `TokenDoGoogle` | `google/oauth.ts` | `access_token, refresh_token?, scope[], expira_em(ISO abs)` |
| `EventoDoGoogle` / `CorpoDeEventoDoGoogle` | `google/evento.ts` | recurso events v3 / corpo de escrita |
| `Projection`/`Base`/`Comparison` | `google/sync-model.ts` | merge de três pontas (grupos como hashes) |
| `RegraDeComissao` | `financeiro/comanda.ts` | `attendant_user_id, event_type_id, percent` |
| `LinhaImportada` / `ErroDaLinha` | `catalogo/planilha.ts` | resultado da leitura de planilha |
| `ProdutoBuscavel` / `Achado` / `BuscaRelaxada` | `catalogo/busca.ts` | busca difusa e relaxamento |

### Constantes e parâmetros configuráveis (unidade 6) 🟢
| Constante / parâmetro | Valor | Onde |
|---|---|---|
| `TRILHAS_DA_AGENDA` | 1..8 (cor mora em `--agenda-pessoa-N` no CSS) | `agenda/tipos.ts` |
| `MAXIMO_DE_DIAS` (consulta) | 62 | `agenda/consulta.ts` |
| `PASSO_DA_CELULA_MIN` (grade) | 30 | `agenda/grade-interativa.ts` |
| `LEMBRETE_MIN_MINUTOS` / `_MAX_` | 15 / 10080 (7d) | `agenda/lembretes.ts` |
| `TETO_DE_LEMBRETES_EXTRAS` | 20 | `agenda/lembretes.ts` |
| `ESCOPOS_OBRIGATORIOS` (OAuth) | `calendar.events`, `calendar.readonly` | `agenda/google/oauth.ts` |
| `FOLGA_DE_RENOVACAO_MS` | 60000 | `agenda/google/oauth.ts` |
| `VALIDADE_DO_ESTADO_MS` | 10min | `agenda/google/estado.ts` |
| `VALIDADE_DO_VINCULO_S` | 600 | `agenda/google/vinculo.ts` |
| `TTL_MS` (memo do app) | 30000 | `agenda/google/config.ts` |
| `CAMINHO_DO_CALLBACK` | `/api/v1/agenda/google/callback` | `agenda/google/config.ts` |
| `JANELA_INICIAL_DIAS` (1º sync) | 90 | `agenda/google/eventos-remotos.ts` |
| `TETO_DE_PAGINAS` | 20 | `agenda/google/eventos-remotos.ts` |
| endpoint Google | `https://www.googleapis.com/calendar/v3` | `agenda/google/transport.ts` |
| timeout HTTP Google | 15000ms (transport) / 10000ms (token/conta) | `google/*.ts` |
| `SUFIXO_ICAL_UID` | `deskcomm.app` (FIXO, identidade técnica) | `agenda/google/evento.ts` |
| `TIPOS_DE_CONTA` / `DIRECOES` | cash\|bank\|other / in\|out | `financeiro/catalogo.ts` |
| `LIMITE_DO_CODIGO` (planilha) | 60 chars + assinatura FNV-1a 8 hex | `catalogo/planilha.ts` |
| `MOEDA_PADRAO` (fallback) | BRL | `catalogo/moeda-da-org.ts` → `lib/money` |
| env do app OAuth | `GOOGLE_CALENDAR_CLIENT_ID`, `GOOGLE_CALENDAR_CLIENT_SECRET`, `NEXT_PUBLIC_APP_URL` | `agenda/google/config.ts` |
| HMAC de state/vínculo | `INTERNAL_SECRET` (🟡 nome inferido do padrão) | `agenda/google/{estado,vinculo}.ts` |

---

## Unidade 7 — Automação e roteamento (`lib/automation/`, `lib/routing/`, `lib/followup/`, `lib/escalacao/`)

### `RuleCondition` 🟢 — `automation/conditions.ts`
| Campo | Tipo | Obrig. | Observação |
|---|---|---|---|
| `field` | string (path `a.b.c`) | sim | resolvido em `context` por `resolveField` |
| `op` | `eq \| neq \| contains` | sim | `contains` difere por tipo: lista=pertinência da tag inteira, texto=`includes` |
| `value` | string | sim | vem sempre como string da UI; coerção via `String()` |
> Avaliadas em **AND**. Campo ausente/null ⇒ falso, exceto `neq` (ausente satisfaz `neq`).

### `ActionResultDetail` / `ActionExecutor` / `ActionCtx` 🟢 — `automation/types.ts`
| Campo | Tipo | Obrig. | Observação |
|---|---|---|---|
| `ActionResultDetail.status` | `success \| failed \| skipped \| postponed` | sim | `postponed` = adiado (ainda a caminho) |
| `ActionExecutor.postponeUntil` | `(ctx,cfg)=>Promise<string\|null>` | não | ISO ⇒ adia o EVENTO INTEIRO (all-or-nothing) |
| `ActionExecutor.execute` | `(ctx,cfg)=>Promise<ActionResultDetail>` | sim | |
| `ActionCtx` | `{admin, organizationId, ruleId, ruleName, event, context, requestId, serviceBoundaries?}` | — | `context` é o MESMO objeto avaliado pelas condições |

### Catálogo de ações 🟢 — `automation/actions/*`
| `type` | `postponeUntil`? | Config esperada |
|---|---|---|
| `add_tag` | não | `tags: string[]` |
| `assign_owner` | não | `user_id: uuid` |
| `create_or_move_lead` | não | `pipeline_id`, `stage_id` |
| `call_webhook` | não | `url`, `secret?`/`secret_enc?` |
| `send_whatsapp_message` | sim | `channel_session_id`, `template` |
| `send_ai_message` | sim | `channel_session_id`, `agent_id`, `instruction` |
| `start_message_flow` | não | `flow_pointer_id` |

### `MotivoDeBloqueio` (guardas de contato) 🟢 — `automation/guarda-do-contato.ts`
`no_contact | contact_blocked | no_phone | consent_declined`. O gate de consentimento lê
`consent.marketing.declined_at` (recusa registrada), **não** a ausência de `granted_at`.

### `RoutingCandidate` / `RoutingAction` / `DecideRoutingInput` 🟢 — `routing/decide.ts`
| Campo | Tipo | Obrig. | Observação |
|---|---|---|---|
| `RoutingCandidate.userId` | uuid | sim | |
| `RoutingCandidate.currentLoad` | number | sim | conversas abertas atribuídas |
| `RoutingCandidate.lastAssignedAt` | `number \| null` | sim | epoch ms; null = nunca (prioridade no rodízio) |
| `RoutingCandidate.scheduleSnapshot` | Json | não | jornada relida no claim |
| `RoutingAction` | `assign{userId} \| skip{reason} \| requeue{nextAttemptAt,attempts}` | — | união |
| `DecideRoutingInput` | `{mode, alreadyAssigned, eligibles[], config, attempts, now}` | — | `now` injetado |

### `AttendantEligibilityInput` 🟢 — `routing/eligibility.ts`
| Campo | Tipo | Obrig. | Observação |
|---|---|---|---|
| `isAvailable` | boolean | sim | só a DECISÃO durável (não presença) |
| `capacity` | number | sim | teto de conversas |
| `currentLoad` | number | sim | carga atual |
| `schedule` | `{timezone, windows[]} \| null` | não | `windows` vazio ⇒ 24/7 |
> `OPEN_LOAD_STATUSES = [open, pending, claimed, ai_handling]`. Elegível = `isAvailable ∧ load<capacity ∧ isWithinSchedule`.

### `QueueStatus` 🟢 — `routing/queue.ts`
| Campo | Tipo | Observação |
|---|---|---|
| `queue_size` | number | sem dono ∧ `status='open'` |
| `avg_wait_seconds` | number | média de `now - awaiting_since` (não `last_inbound_at`, #990) |
| `online_eligible_count` | number | elegíveis agora |

### `FlowNode` / `FlowEdge` / `FlowGraph` (Zod) 🟢 — `followup/graph-schema.ts`
| Elemento | Forma |
|---|---|
| `NodeType` | `trigger \| wait \| condition \| ai_classify \| match_reply \| repeat \| action \| end` |
| nó | `{id, type, label(1..60), position{x,y}, config}` |
| `wait.config` | `fixed{duration_ms 5min..90d, immune_to_reply?}` \| `smart{min_ms, max_ms, guidance?}` |
| `ai_classify.config` | `{classes[1..8], branches?, grace_timeout_ms≥15min, target:last_reply\|summary, hint?}` |
| `match_reply.config` | `{branches[1..8]{id,label,op:eq\|contains,pattern}, grace_timeout_ms≥15min, save_to?, if_exists?}` |
| `repeat.config` | `{max_count 1..20}` |
| `condition.config` | `{combinator:and\|or, branching?:combined\|per_check, checks[1..10]{id?,label?,field,op,value}}` |
| `condition.checks[].field` | `lead_stage \| tag \| steps_taken \| last_outcome` |
| `condition.checks[].op` | `eq \| neq \| gte \| lte \| contains` |
| `action.config` | `text{body 1..4000}` \| `ai_message{prompt_hint, fallback_template_id?}` \| `template{template_id uuid}` |
| `end.config` | `{outcome:converted\|exhausted\|custom, note?}` |
| `replySaveToSchema` | `contact_name` \| `lead_custom{key}` |
| `ifExistsSchema` | `skip \| overwrite \| confirm` |
| grafo | `nodes[2..60]`, `edges[..120]` + superRefine (ids únicos, arestas apontam a nó existente) |
| `edge.condition` | `always \| class_match{value} \| cond_result{boolean} \| branch{branch_id}` |
| branches reservados | `else`(FALLBACK), `no_reply`, `true`, `false`, `body`, `done` |

### `NodeResult` (união) 🟢 — `followup/node-handlers.ts`
`advance{next_node_id, next_eval_at, reason?, repeat?} | wait{next_eval_at, wake_status?} |
enqueue_turn{purpose:send_message|classify|plan_timing, wake_status, fixed_body?} | recheck{next_eval_at} |
dead{reason} | complete{outcome, cancel_reason?} | fail{error}`.

### `EnrollmentRow` 🟢 — `followup/node-handlers.ts` (tabela `followup_enrollments`)
| Campo | Tipo | Obrig. | Observação |
|---|---|---|---|
| `id` / `organization_id` | uuid | sim | |
| `pointer_id` / `version_id` | uuid | sim | ponteiro do fluxo / versão pinada |
| `contact_id` | uuid | sim | 1 inscrição viva por (org,pointer,contact) — unique |
| `conversation_id` | `uuid \| null` | sim | |
| `current_node_id` | string | sim | nó atual do grafo |
| `status` | `active \| waiting_reply \| dormente \| paused_handoff \| paused_manual \| completed \| cancelled \| dead` | sim | |
| `next_eval_at` / `claimed_until` | `timestamptz \| null` | sim | agendamento / lease |
| `attempts` / `max_attempts` | number | sim | backoff |
| `steps_taken` | number | sim | +1 por passo aplicado; base da chave de idempotência |
| `outcome` | `converted \| exhausted \| replied \| handoff \| opted_out \| null` | não | |
| `cancel_reason` / `last_error` | `string \| null` | não | |
| `timing_plan` | jsonb (`unknown`) | não | migration 0144; null = antes da feature |
| `service_boundary` / `revision` | ServiceBoundary / number | não | fronteira congelada |
| `appointment_id` / `appointment_revision` | `uuid \| null` / number | não | cadência de no-show |

### `LeadFacts` / `EnrollmentEventRef` 🟢 — `followup/node-handlers.ts`
- `LeadFacts` = `{lead_stage:string|null, tags:string[], steps_taken, last_outcome:string|null, contact_name?, custom_fields?}`.
- `EnrollmentEventRef` = `{node_id, idempotency_key, event_type?, payload?}` (linha de `followup_enrollment_events`).

### Constantes de domínio — follow-up 🟢
| Constante | Valor | Fonte |
|---|---|---|
| `MAX_STEPS` | 80 | `engine.ts` |
| `CLAIM_LEASE_SECONDS` | 120 | `engine.ts` |
| `DEFAULT_CLAIM_LIMIT` | 20 | `engine.ts` |
| `BACKOFF_MS` | `[30s, 60s, 5m, 15m, 1h]` | `node-handlers.ts` |
| `ACTION_RECHECK_MS` / `_MAX_MS` | 5min / 1h | `node-handlers.ts` |
| `MAX_ACTION_RECHECKS` | 14 (ocioso desde último `action_deferred`) | `node-handlers.ts` |
| `MAX_PLAN_RECHECKS` | 3 (→ segue sem plano) | `node-handlers.ts` |
| `EVENTO_ACAO_ADIADA` | `"action_deferred"` (prova de vida) | `node-handlers.ts` |
| `RESUME_GRACE_MS` | carência ao retomar handoff | `reactivity.ts` |
| `wait fixed duration_ms` | 300_000 (5min) .. 7_776_000_000 (90d) | `graph-schema.ts` |
| `grace_timeout_ms` | ≥ 900_000 (15min) | `graph-schema.ts` |

### Tabelas do follow-up 🟢
| Tabela | Colunas-chave |
|---|---|
| `followup_flow_pointers` | `id, organization_id, status, active_version_id, trigger_config, handoff_policy` |
| `followup_flow_versions` | `id, organization_id, graph(jsonb)` (escrita por `fn_publish_followup_flow_version`) |
| `followup_enrollments` | ver `EnrollmentRow` acima; unique (org,pointer,contact) |
| `followup_enrollment_events` | `organization_id, enrollment_id, node_id, event_type, payload(jsonb), idempotency_key` (unique por enrollment) — o event sourcing |

### `PassagemNova` / `LinhaDaPassagem` 🟢 — `escalacao/passagem.ts` (tabela `passagens_de_atendimento`)
| Campo | Tipo | Obrig. | Observação |
|---|---|---|---|
| `organization_id`/`contact_id`/`conversation_id` | uuid | sim | |
| `caso_id` | `uuid \| null` | não | só quando nasce de "não consigo → escalar" |
| `motor` | `engine \| crm` (`MOTORES_DA_PASSAGEM`) | sim | qual motor de IA passou |
| `origem` | `OrigemDaPassagem` (13 valores) | sim | origem de código |
| `motivo_codigo` | `MotivoDaPassagem` (9 valores) | sim | razão de gente (superset de `HandoffReason`) |
| `title`/`notes`/`content` | string (tetos 300/2000/1200) | não | sanitizados por `sanitizarTextoDoLead` |
| `body` | string (teto 8000) | sim | montagem própria; vazio → `PISO_DO_BRIEFING`; NÃO higienizado |
| `tentativas` | jsonb (array, max 10; itens trim/1..280) | sim | escrito pelo modelo/MCP |
| `cliente_avisado` | `boolean \| null` | não | null = ninguém tentou avisar |
| `aviso_motivo_codigo` | `MotivoDoAviso \| null` | não | por que NÃO avisou |

### Enums / tuplas de vocabulário — escalação 🟢 (espelhadas contra CHECK do banco)
| Nome | Valores |
|---|---|
| `MOTORES_DA_PASSAGEM` | `engine`, `crm` |
| `ORIGENS_DA_PASSAGEM` (13) | `pedido_explicito`, `opt_out_provavel`, `ferramenta_do_modelo`, `teto_de_gasto`, `caso_escalado`, `sentimento`, `legado_pedido`, `legado_juridico`, `legado_etapa`, `legado_confianca`, `legado_teto`, `mcp_externo`, `runtime_nativo` |
| `MOTIVOS_DA_PASSAGEM` (9) | `requested_human`, `suspected_optout`, `orcamento_de_ia`, `low_sentiment`, `low_confidence`, `critical_stage`, `legal_mention`, `refund_mention`, `caso_escalado` |
| `MOTIVOS_DO_AVISO` (6) | `na_fila_canal_fora`, `falhou_no_envio`, `sem_telefone`, `pre_go_live`, `canal_arquivado`, `fora_da_janela` |
| `CODIGOS_DO_ESTADO_DO_AVISO` (11) | `sem_conexao`, `so_canal_oficial`, `sem_endereco_publico`, `conexao_removida`, `conexao_fora_do_ar`, `conexao_atende_clientes`, `casos_desligados`, `agente_assistido`, `atendimento_externo`, `aquecimento`, `descarte_acontecendo` |
| `STATUS_DA_ENTREGA_DE_AVISO` (4) | `pendente`, `enviado`, `falhou`, `cancelado` |
| `ERROS_DA_ENTREGA_DE_AVISO` (11) | `canal_desconectado`, `canal_arquivado`, `canal_nao_aceita_aviso_livre`, `transporte_ausente`, `sem_endereco_publico`, `destino_invalido`, `falha_no_envio`, `teto_diario_do_numero`, `titular_anonimizado`, `expirou`, `indeterminado` |

### `DesfechoDoAvisoAoCliente` / `ConversaEmHandoff` / `Selecao` 🟢 — `escalacao/*`
- `DesfechoDoAvisoAoCliente` = `{avisado:true} | {avisado:false, porque:string, motivoCodigo?:MotivoDoAviso}` (união discriminada — força dizer o porquê).
- `ConversaEmHandoff` (`devolucao-automatica.ts`) = `{id, organization_id, channel_session_id, status, assignee_kind, assigned_to_user_id, assigned_at, bot_silenced_until, last_handoff_at, last_outbound_at, status_changed_at}`.
- `Selecao` = `{prazoPorOrg: Map<org,minutos>, sessoesComAgente: Map<org,Set<sessão>>, agoraMs}`.
- `MotivoDeNaoDevolver` = `sem_prazo | nao_esta_com_humano | status_nao_devolvivel | sessao_sem_agente | sem_relogio | dentro_do_prazo`.

### Constantes de domínio — escalação 🟢
| Constante | Valor | Fonte |
|---|---|---|
| `PRAZO_DO_SILENCIO_MS` (atendimento manual) | 60min, extend-only | `atendimento-manual.ts` |
| `PRAZO_MIN_MINUTOS` / `_MAX_` (devolução) | 5 / 1440 | `devolucao-automatica.ts` |
| `IDADE_MAXIMA_DO_EVENTO_MS` (aviso) | 30min | `aviso-ao-suporte.ts` |
| `VALIDADE_DA_ENTREGA_MS` | 24h | `aviso-ao-suporte.ts` |
| `JANELA_DE_REIVINDICACAO_MS` | 2min | `aviso-ao-suporte.ts` |
| `TETO_DE_TENTATIVAS_DO_CANAL` / `_DE_ENVIO` / `_ABSOLUTO` | 6 / 3 / 10 | `aviso-ao-suporte.ts` |
| `ADIAMENTO_DO_DRENO_EM_REQUEST_MS` / `_DO_CANAL_MS` | 15s / 5min | `aviso-ao-suporte.ts` |
| `JANELA_DO_DESCARTE_DIAS` | 7 | `estado-do-aviso.ts` |
| `TTL_DO_CACHE_MS` (número interno) | 30s | `numero-interno-de-aviso.ts` |
| `TETOS_DA_PASSAGEM` | title 300 / notes 2000 / content 1200 / body 8000 | `passagem.ts` |
| `TETOS_DO_TEXTO_DO_LEAD` | title 90 / summary 160 / blocker 160 | `sanitizar-texto-do-lead.ts` |

### Constantes de domínio — automação e roteamento 🟢
| Constante | Valor | Fonte |
|---|---|---|
| `AUTOMATION_CONSUMER_KEY` | `"automation-rules"` | `automation/engine.ts` |
| `AUTOMATED_SEND_SPACING_MS` | 1200 | `automation/throttle.ts` |
| `jitterMs()` | 0..800 | `automation/throttle.ts` |
| `daily_message_limit` (default) | 300 | `automation/throttle.ts` |
| `PACING_DEFAULTS` janela | 7h–22h, allowSunday, fuso do tenant | `agent-engine/pacing/defaults` |
| webhook: `TIMEOUT_MS` / `RETRY_DELAYS_MS` | 10s / `[1s, 5s]` (3 tentativas) | `automation/actions/call-webhook.ts` |
| `ROUTING_WORKER_KEY` / `ROUTING_EVENT_TYPE` | `worker.routing.v1` / `conversation.routing_requested` | `routing/worker.ts` |
| routing `DEFAULT_BATCH_SIZE` | 100 (clamp 1..500) | `routing/worker.ts` |
| recuperação de `processing` abandonado | > 5min | `routing/worker.ts` |

---

## Unidade 8 — Auth, tenancy e RBAC

### Tipos e enums de domínio (`lib/auth/types.ts`)

| Nome | Valores / rank | Origem |
|---|---|---|
| `Role` | `viewer(1)`, `agent(2)`, `ai_operator(3)`, `manager(4)`, `admin(5)` | `auth/types.ts` — `ROLE_RANK` |
| `PAPEIS_HUMANOS` | `viewer, agent, manager, admin` (espelha `user_organizations_role_check`) | `auth/types.ts` |
| `ai_operator` | papel do AGENTE PUBLICADO — só no token efêmero, **nunca** em `user_organizations` | `auth/types.ts` |
| `ROTULO_DO_PAPEL` | viewer=Somente leitura, agent=Atendente, manager=Gerente, admin=Administrador | `auth/types.ts` |
| `VisibilityMode` | `all`, `own_and_unassigned` (default, G1-06a), `own` — só restringe `agent` | `auth/types.ts` |
| `ModoDeCadastro` | `aberto` (default), `so_convite` | `auth/politica-de-cadastro.ts` |
| `StatusConvite` | `pendente`, `aceito`, `expirado`, `revogado` (DERIVADO, nunca coluna) | `team/convite-status.ts` |
| `access_mode` (support) | `full`, `support_readonly` | `impersonate/support.ts` |
| `status` (support) | `active`, `expired`, `revoked` | `impersonate/support.ts` |

### Entidades / DTOs

| Entidade | Campos principais | Origem |
|---|---|---|
| `AuthUser` | `id`, `email`, `full_name`, `avatar_url`, `is_platform_admin`, `locale?`, `idioma`, `timezone?`, `organizations: UserOrgMembership[]`, `support?` | `auth/types.ts` |
| `UserOrgMembership` | `organization_id`, `organization_name`, `role`, `interface_settings?`, `locale?`, `timezone?` | `auth/types.ts` |
| `ActiveOrg` | `orgId`, `name`, `role`, `interface_settings?`, `timezone?`, `visibility_mode?`, `cliente_pela_agenda?`, `marca?` | `auth/types.ts` |
| `InvitePayload` | `invite_id(uuid)`, `email`, `organization_id(uuid)`, `role`, `exp(epoch s)`, `iat?`, `invited_by?`, `interface_settings?` | `auth/invite-token.ts` |
| `ConviteDeTime` (`team_invites`) | `id`, `organization_id`, `email`, `role`, `interface_settings`, `invited_by`, `inviter_name`, `email_dispatched`, `created_at`, `last_sent_at`, `resend_count`, `expires_at`, `accepted_at`, `revoked_at` | `team/convites.ts` (migration 0238) |
| `ImpersonatePayload` | `sessionId?`, `tenantId`, `platformAdminId`, `exp(epoch s)` | `impersonate/cookie.ts` |
| `SupportContext` | `id`, `organization_id`, `actor_user_id`, `auth_session_id`, `previous_organization_id?`, `expires_at`, `name`, `locale?`, `access_mode`, `status` | `impersonate/support.ts` |
| `PlatformAdminInfo` | `user_id`, `scope`, `mfa_required` | `auth/requirePlatformAdmin.ts` |
| `PoliticaDeMfa` | `role?`, `isPlatformAdmin`, `plataformaExige: bool\|null`, `empresaExige: bool` | `auth/politica-mfa.ts` |

### Constantes e parâmetros configuráveis

| Nome | Valor | Origem |
|---|---|---|
| `INVITE_TTL_SECONDS` | 86400 (24h) | `auth/invite-token.ts` |
| secret do convite | `INVITE_TOKEN_SECRET → INTERNAL_SECRET → "dev-fallback"` | `auth/invite-token.ts` |
| `ACTIVE_ORG_COOKIE` | `"active_org"` (httpOnly, strict, maxAge 30d) | `auth/server.ts`, `aplicar-convite.ts` |
| `COOKIE_NAME` (sessão) | `"sb-deskcomm-auth"` (SameSite=strict, httpOnly) | `proxy.ts` |
| `AUTH_LIMITS.login` | `{ ip: 60, id: 5, windowSec: 300 }` | `auth/rate-limit.ts` |
| `AUTH_LIMITS.signup` | `{ ip: 20, windowSec: 3600 }` | `auth/rate-limit.ts` |
| `AUTH_LIMITS.reset` | `{ ip: 30, id: 3, windowSec: 3600 }` | `auth/rate-limit.ts` |
| `AUTH_LIMITS.invite_accept` | `{ ip: 60, windowSec: 3600 }` | `auth/rate-limit.ts` |
| `AUTH_LIMITS.org_recovery` | `{ ip: 5, id: 3, windowSec: 3600 }` | `auth/rate-limit.ts` |
| `LOGIN_IP_DEFAULT` | 60 (override: `AUTH_RATE_LIMIT_LOGIN_IP`) | `auth/rate-limit.ts` |
| recovery code | alfabeto 31 chars sem ambiguidade, 8 chars, 10 códigos, `sha256` bytea | `auth/recovery-codes.ts` |
| `IMPERSONATE_COOKIE_NAME` | `"deskcomm-impersonate"` (HttpOnly, Secure, SameSite=Lax) | `impersonate/cookie.ts` |
| `IMPERSONATE_TTL_SECONDS` | 3600 (1h) | `impersonate/cookie.ts` |
| secret de impersonation | `IMPERSONATE_COOKIE_SECRET` (mínimo 32 chars) | `impersonate/cookie.ts` |
| secret de cron | `INTERNAL_CRON_SECRET`, `INTERNAL_SECRET` (fail-closed) | `auth/cron-auth.ts` |
| API key de org | `dsk_<prefix>_<secret>` (prefix 4 bytes hex, secret 32 bytes b64url, SHA256 em `token_hash`) | `tenants/api-key.ts` |
| escopos default da API key | `["mcp:read", "mcp:write", "role:agent", <integrationScope>]` | `tenants/api-key.ts` |
| `TTL_MS` (memo cadastro) | 30000 (30s) | `auth/politica-de-cadastro.ts` |
| `SIGNUP_MODE` (`.env`) | semente/piso; banco (`platform_settings.signup_mode`) manda | `auth/politica-de-cadastro.ts` |
| senha (schemas) | mínimo 8 chars; nome da empresa 2–120 chars | `auth/schemas.ts` |

---

## Unidade 9 — Compliance (LGPD, legal, opt-out, retenção, audit)

### Tipos e enums de domínio

| Nome | Valores | Origem |
|---|---|---|
| `LgpdRequestType` | `data_request`, `redact`, `store_redact` (o CHECK do banco) | `lgpd/types.ts` |
| `LgpdScope` | `contact`, `tenant` | `lgpd/types.ts` |
| `LgpdRequestStatus` | `received`, `processing`, `completed`, `failed`, `pending_review` | `lgpd/types.ts` |
| `LgpdRequest.source` | `nuvemshop`, `admin_panel`, `api` | `lgpd/types.ts` |
| `AlarmThreshold` | `data_request_d5`, `redact_d10` | `lgpd/sla-alarm.ts` |
| `storage_redaction_queue.status` | `pending`, `processing`, `deleted`, `failed`, `skipped` | `lgpd/storage-redaction-queue.ts` |
| `AuditAction` | ~330 códigos derivados de `AUDIT_ACTIONS` (array = fonte única) | `audit/actions.ts` |
| `ModoDeCadastro` (referência) | — ver Unidade 8 | — |

### Entidades / DTOs

| Entidade | Campos principais | Origem |
|---|---|---|
| `LgpdRequest` | `id`, `organization_id`, `request_type`, `source`, `contact_id?`, `external_customer_id?`, `status`, `attempts`, `received_at`, `due_at`, `completed_at?`, `request_payload(jsonb, PII aqui)`, `result?`, `error_message?`, `cascaded_to?`, `emergency`, `scope` | `lgpd/types.ts` |
| `AuditEntry` | `action`, `actorUserId?`, `actorApiTokenId?`, `actorAuthSessionId?`, `organizationId?`, `resourceType?`, `resourceId?`, `metadata?`, `requestId?`, `ip?`, `userAgent?`, `bypassedRls?`, `actingAsPlatformAdmin?` | `audit/index.ts` |
| `CascadeResult` | `alreadyAnonymized`, `counts`, `mediaPaths` | `lgpd/redact-cascade.ts` |
| `ResultadoDaRedacao` | `leadsRedigidas[]`, `atividadesRedigidas`, `tabelas[]`, `falhas[]` | `lgpd/cascata.ts` |
| `ExportPayload` | controlador (`organization_legal_name`, `dpo_email`, `lei_citada?`, `documento_rotulo`), `contact?`, `consents[]`, `conversations[]`, `messages_*`, `leads[]`, `orders[]`, `activities[]`, `appointments[]`, `sales[]`, `tasks[]`, `webhook_captures[]`, `voice_calls[]`, `cases[]`, `case_events[]`, `demandas[]`, `case_chat_messages[]`, `passagens[]`, `avisos_de_caso[]`, `audit_log_extract[]` | `lgpd/export-collector.ts` |
| `SignResult` | `signed(Buffer)`, `sha256`, `signed_pades`, `warning?` | `lgpd/pades-signer.ts` |
| `PerfilDoPais` | `codigo`, `nome`, `documento`, `lei?`, `calendario`, `padroesDePii[]` | `legal/perfil-do-pais.ts` |
| `DocumentoDoTitular` | `rotulo`, `exemplo`, `regra`, `mensagemInvalido`, `confereDigito`, `apelidosDoCabecalho[]`, `valida()`, `normaliza()` | `legal/perfil-do-pais.ts` |
| `LeiCitada` | `nome`, `numero`, `artigo`, `revisada` | `legal/perfil-do-pais.ts` |
| `Operador` | `sistema`, `nome?`, `razaoSocial?`, `cnpj?`, `dpoEmail?`, `politicaPropria?`, `resolvido` | `legal/operador.ts` |
| `RetencaoInterpretada` | `dias`, `aviso?` | `retencao/politica.ts` |

### Constantes e parâmetros configuráveis

| Nome | Valor | Origem |
|---|---|---|
| `DEDUP_MS` (alarme SLA) | 24h | `lgpd/sla-alarm.ts` |
| limiares SLA | D+5 (data_request), D+10 (redact) | `lgpd/sla-alarm.ts` |
| `MAX_ATTEMPTS` / batch (fila storage) | 3 / 50 | `lgpd/storage-redaction-queue.ts` |
| `SUFIXO_ANONIMIZADO` / `TITULO_PRESERVADO` | `" (anonimizado)"` / 20 chars | `lgpd/cascata.ts` |
| `STATUS_DA_REGUA_VIVA` | active, waiting_reply, dormente, paused_handoff, paused_manual | `lgpd/cascata.ts` |
| `MAX_CONTATOS_EXAMINADOS` / `MAX_CONTATOS_POR_VARREDURA` / `CONTATOS_POR_BLOCO` | 5000 / 200 / 100 | `lgpd/cascata.ts` |
| `RECENT_MESSAGES_LIMIT` / `AUDIT_LIMIT` (export) | 100 / 200 | `lgpd/export-collector.ts` |
| retenção fila | padrão 90d / piso 7d | `retencao/politica.ts` |
| retenção auditoria | padrão 1825d (L-10, 5 anos) / piso 90d | `retencao/politica.ts` |
| retenção captação | padrão 365d / piso 30d (piso só no TS) | `retencao/politica.ts` |
| retenção espelho agenda | padrão 90d / piso 7d | `retencao/politica.ts` |
| retenção conversa do caso | padrão 365d / piso 90d | `retencao/politica.ts` |
| retenção passagem | padrão 1825d / piso 90d (+ só `reconhecido_em IS NOT NULL`) | `retencao/politica.ts` |
| retenção aviso de caso | padrão 180d / piso 30d | `retencao/politica.ts` |
| `HOLIDAYS_BR_ISO` | feriados nacionais BR 2026-2030 (fixos + móveis) | `lgpd/holidays-br.ts` |
| CPF | mod-11 Receita Federal (`isValidCpf`) | `legal/perfil-do-pais.ts` |
| PADRÃO país | `BR` (`null` na coluna = Brasil) | `legal/perfil-do-pais.ts` |
| `LGPD_SIGNING_KEY` | assinatura PAdES (stub enquanto ausente/pendente) | `lgpd/pades-signer.ts` |
| `LGPD_DPO_EMAIL` | encarregado da instalação (piso do da org) | `lgpd/sla-alarm.ts`, `legal/operador.ts` |
| máscara PII | email `a***@dominio.com`, phone `(**) ****-1234`, CPF omitido | `lgpd/mask.ts` |

---

## Unidade 11 — Integrações externas

### `ConexaoExterna` 🟢 — `external-db/types.ts:31`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `id` | `string` | sim | uuid da conexão |
| `organizationId` | `string` | sim | dono da conexão |
| `label` | `string` | sim | 1-80 chars |
| `host` | `string` | sim | validado por `guardas.ts` |
| `port` | `number` | sim | 1-65535, default 5432 |
| `database` | `string` | sim | de `database_name` |
| `username` | `string` | sim | |
| `password` | `string` | sim | decifrado JIT (AES-GCM) |
| `sslMode` | `ModoTls` | sim | disable\|prefer\|require\|verify-ca\|verify-full |
| `maxRows` / `maxFilters` / `maxResponseBytes` | `number` | sim | limite da conexão ≤ absoluto |
| `versao` | `string` | sim | `= updated_at`, invalida o pool |

### `OperadorDeFiltro` 🟢 — `external-db/types.ts:73` (vocabulário FECHADO)
`eq` \| `ne` \| `gt` \| `gte` \| `lt` \| `lte` \| `contem` \| `comeca_com` \| `in` \| `nulo` \| `nao_nulo`

### `TokenResult` (Nuvemshop) 🟢 — `nuvemshop/oauth.ts`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `ok` | `boolean` | sim | discriminante |
| `accessToken` | `string` | quando ok | token NÃO expira |
| `scope` | `string` | quando ok | default "" |
| `storeId` | `string` | quando ok | `= user_id` da resposta |
| `error` / `status` / `raw` | `string`/`number` | quando falha | network_error \| token_exchange_failed \| invalid_token_response |

### `VerifiedState` (CSRF Nuvemshop) 🟢 — `nuvemshop/state.ts`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `orgId` | `string` | sim | do payload assinado |
| `nonce` | `string` | sim | uso único |
| `expMs` | `number` | sim | TTL 10min |
| `userId` / `authSessionId` | `string` | não | só payload de 5 segmentos |

### `ConversaoOffline` 🟢 — `plataformas-de-anuncio/types.ts:63`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `organizationId` / `leadId` | `string` | sim | |
| `evento` | `NomeDoEvento` | sim | "Purchase" (Lead = Fase 2) |
| `eventoId` | `string` | sim | `= <leadId>:<evento>` dedup determinístico |
| `ocorridoEm` | `Date` | sim | `= closed_at` do lead, nunca `now()` |
| `cliqueDeOrigem` | `string` | sim | `ad_source_id` do contato |
| `telefone` | `string \| null` | não | E.164 sem `+`, EM CLARO (hash é do transporte) |
| `valorCentavos` | `number` | sim | |
| `moeda` | `string` | sim | |

### `CredencialDeConversao` 🟢 — `plataformas-de-anuncio/types.ts:143`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `datasetId` | `string` | sim | Meta: dataset; Google: customerId |
| `accessToken` | `string` | sim | Google: `""` (derivado a cada envio) |
| `testEventCode` | `string \| null` | não | preenchido = teste |
| `google` | objeto | não | `{refreshToken, customerId, loginCustomerId(null=sem MCC), conversionActionId}` |

### `ResultadoDeEnvio` 🟢 — `plataformas-de-anuncio/types.ts:117`
`{tipo:"ok"}` \| `{tipo:"transitorio", tentarEmMs?}` \| `{tipo:"permanente"}` — física da falha declarada, nunca booleano.

### `ExtensionManifest` 🟢 — `extensions/manifest.ts:30`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `format_version` | `1` | sim | literal |
| `profile` | `"declarative"` | sim | literal |
| `publisher` / `name` | `string` (slug) | sim | `^[a-z0-9]+(-[a-z0-9]+)*$` |
| `version` | `string` (semver) | sim | sem zeros à esquerda |
| `license` | `"MIT"` | sim | literal |
| `host_api` | `{min,max}` | sim | `min ≤ 2 ≤ max` |
| `permissions` | `ExtensionPermission[]` | sim | ≥1, sem duplicata |
| `dependencies` | `[]` | sim | `z.tuple([])` — sempre vazio |
| `data` | `{mode:"none"}` | sim | fixo |
| `display` | objeto | sim | title/summary LocalizedText, category, icon |
| `contributions.crm_cards` | array | sim | ≤4 cards, ≤8 blocks cada |

### `EXTENSION_LIMITS` 🟢 — `extensions/manifest.ts:14`
| Nome | Valor |
|---|---|
| packageBytes | 64 KB |
| catalogBytes | 512 KB |
| jsonDepth / jsonNodes | 12 / 20000 |
| catalogEntries | 128 |
| cards / blocksPerCard | 4 / 8 |

---

## Unidade 12 — Eventos e tempo real

### `EventRow` 🟢 — `event-log/dispatcher.ts:20`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `id` | `string` | sim | |
| `organization_id` | `string` | sim | |
| `event_type` | `string` | sim | ex "message.received" |
| `entity_kind` | `string` | sim | |
| `entity_id` | `string \| null` | sim | |
| `payload` / `metadata` | `Record<string,unknown>` | sim | |
| `consumed_by` | `string[]` | sim | keys dos handlers que já consumiram |
| `attempts` | `number` | sim | |
| `created_at` | `string` | não | OPCIONAL de propósito; quem descarta por idade falha ABERTO se faltar |

### `HandlerResult` 🟢 — `event-log/dispatcher.ts:54`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `consumer_key` | `string` | sim | vai para `consumed_by` |
| `status` | `"ok"\|"skipped"\|"error"\|"retry"` | sim | |
| `retry_at` | `string` (ISO) | quando retry | reagenda sem contar attempt |
| `detail` | `string` | não | preservado em `last_error` |

### `JobRow` 🟢 — `agent-engine/queue/queue.ts:37`
| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `id` / `organization_id` | `string` | sim | |
| `contact_id` | `string \| null` | sim | lane da fila |
| `kind` | `JobKind` | sim | inbound_turn\|followup_turn\|watchdog\|flywheel\|case_reply_turn\|operator_turn\|transactional_delivery\|approved_reply |
| `source_event_id` | `string \| null` | sim | dedup evento→job (unique + 23505) |
| `status` | `JobStatus` | sim | pending\|running\|done\|failed\|dead |
| `priority` | `number` | sim | default 100 |
| `run_after` | `Date` | sim | default now() |
| `attempts` / `max_attempts` | `number` | sim | max default 5; attempts++ no CLAIM |
| `locked_by` / `locked_at` | `string`/`Date` | não | lease |

### `CronJobRow` / `CronSpec` 🟢 — `agent-engine/cron/scheduler.ts:29`, `schedule.ts`
| Campo | Tipo | Observação |
|---|---|---|
| `kind` | `'at'\|'every'\|'cron'` | |
| `interval_ms` | `string \| null` | bigint chega como string |
| `cron_expr` | `string \| null` | 5 campos |
| `tz` | `string` | default 'UTC' |
| `job_kind` | `JobKind` | default 'followup_turn' |
| `next_run_at` | `Date` | com stagger FNV-1a aplicado |
| `enabled` | `boolean` | |

### Constantes de tempo/fila 🟢
| Nome | Valor | Origem |
|---|---|---|
| `FUSO_PADRAO` | `America/Sao_Paulo` | `tempo/fusos.ts` |
| `MAX_ATTEMPTS` (event_log) | 5 | `event-log/drain.ts` |
| `PROCESSING_STALE_MS` | 10min | `event-log/drain.ts` |
| `QUEUE_MAX_CONCURRENCY` | 8 | `agent-engine/env.ts` |
| `QUEUE_VISIBILITY_TIMEOUT_MS` | 600_000 (10min) | `agent-engine/env.ts` |
| `CRON_SCAN_HORIZON_MIN` | 366×24×60 | `cron/schedule.ts` |
| `CLAIM_LOCK_KEY` | 727258 (advisory) | `queue/queue.ts` |

---

## Unidade 13 — Infra transversal e relatórios

### `ApiSuccess<T>` / `ApiError` 🟢 — `api/wrappers.ts`
| Campo | Tipo | Observação |
|---|---|---|
| `data` | `T` | sucesso |
| `meta` | `{cursor?, has_more?, total?}` | paginação por cursor |
| `error.code` | `string` | ⚠️ `ApiErrorCode \| (string & {})` — não protege |
| `error.message` | `string` | frase de produto |
| `error.details` | `unknown` | só quando presente |

### `EncryptedSecret` 🟢 — `crypto/aes_gcm.ts:46`
| Campo | Tipo | Observação |
|---|---|---|
| `ciphertext` / `iv` / `tag` | `Buffer` | IV 12B, tag 16B |
| `last4` | `string` | única parte exposta na UI |

### `DesfechoIdempotente<T>` 🟢 — `api/idempotency.ts:137`
`{tipo:"executou"}` \| `{tipo:"replay"}` \| `{tipo:"conflito"}` \| `{tipo:"em_curso"}` — 409 in_progress vs conflict pedem ações diferentes.

### `Medida` / `Par` (Índice de Atrito) 🟢 — `metrics/atrito.ts`
| Campo | Tipo | Observação |
|---|---|---|
| `chave` / `rotulo` | `string` | |
| `valor` | `number \| null` | ausente é `null` NUNCA `0` |
| `unidade` | `"contagem"\|"segundos"\|"razao"\|"media"` | |
| `Par.eficiencia` | `Medida` | |
| `Par.danos` | `Medida[]` | não-vazio — tipo proíbe eficiência sozinha |

### Constantes de infra 🟢
| Nome | Valor | Origem |
|---|---|---|
| `TTL_MS` (idempotência) | 24h | `api/idempotency.ts` |
| `JANELA_DA_RESERVA_MS` | 60s | `api/idempotency.ts` |
| `KEY/IV/TAG_LENGTH_BYTES` | 32 / 12 / 16 | `crypto/aes_gcm.ts` |
| `DEFAULT_TIMEOUT_MS` / `MUTATION_TIMEOUT_MS` | 10s / 30s | `api/client.ts` |
| `ABANDONO_HORAS_DEFAULT` | 72h (limites 1..2160) | `metrics/atrito.ts` |
| `IDIOMA_PADRAO` | `pt-BR` | `i18n/idiomas.ts` |
| cookie de sessão | `sb-deskcomm-auth`, sameSite strict, httpOnly | `supabase/server.ts` |

---

## Unidade 14 — Superfície HTTP

### `AuthDual` 🟢 — `api/auth-dual.ts:32`
| Campo | Tipo | Observação |
|---|---|---|
| `ok` | `boolean` | discriminante |
| `organizationId` | `string` | do TOKEN ou da sessão, NUNCA do body |
| `actor` | `Actor` | `{type:"user"\|"ai_agent"\|"api_token", id}` |
| `supabase` | `SupabaseClient` | admin (token) ou RLS (sessão) |
| `via` | `"session" \| "token"` | por onde a identidade entrou |
| `response` | `Response` | quando falha |

### `RoleCheck` 🟢 — `auth/require-role.ts`
| Campo | Tipo | Observação |
|---|---|---|
| `ok` | `boolean` | |
| `user` | `AuthUser` | quando ok |
| `org` | `ActiveOrg` | `{orgId, name, role}` — role EFETIVO do banco via `fn_user_role_in_org` |
| `response` | `Response` | quando falha (fail com código) |

### `PUBLIC_PATHS` 🟢 — `auth/public-paths.ts`
`RegExp[]`, precedência por ordem do array. `isPublicPath(pathname)` = `.some(re => re.test(pathname))`. Duas categorias: público real e "o proxy não decide" (auth mora dentro da rota).

### Convenções de resposta 🟢
| Convenção | Regra | Origem |
|---|---|---|
| `X-Request-Id` | em TODA resposta (do requestId ou UUID novo) | `api/wrappers.ts` |
| JSON | snake_case | doutrina |
| dinheiro | `_cents` + `currency` | doutrina |
| datas | ISO-8601 UTC | doutrina |
| erro na borda | só `fail(code, message, status)`, nunca `throw` cru | `api/wrappers.ts`, `api/recusa.ts` |
