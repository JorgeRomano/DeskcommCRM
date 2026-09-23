# Integrações Externas — Contratos

> Contratos externos: banco read-only, webhook/OAuth Nuvemshop, conversões Meta/Google, manifesto de extensão.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Contrato 1 — external-db (limites e schema)
`LIMITE_LINHAS={min:1,max:5000,padrao:200}`, `LIMITE_FILTROS={min:0,max:100,padrao:20}`, `LIMITE_RESPOSTA_BYTES={padrao:30000}`. `MODOS_TLS=[disable,prefer,require,verify-ca,verify-full]`. `leituraQuerySchema`: limit(coerce), offset, order_by, order_desc, colunas(csv). SEM parâmetro de filtro na querystring (PII/vazamento em log). 🟢

## Contrato 2 — Webhook + OAuth Nuvemshop
Webhook header `x-linkedstore-hmac-sha256` (hex, chave = client_secret), HMAC sobre rawBody. `SUBSCRIBED_EVENTS`: order/created|updated|paid|cancelled, product/created|updated|deleted, app/uninstalled. Token: `user_id` = storeId, não expira. `NuvemshopApiClient` usa `Authentication: bearer <token>` (minúsculo). State CSRF: `base64url(orgId.nonce.expMs[.userId.authSessionId]).hex(HMAC)`, TTL 10min. 🟢

## Contrato 3 — Conversão offline (Meta/Google)
`ConversaoOffline`: `{organizationId, leadId, evento:"Purchase", eventoId:"<leadId>:<evento>", ocorridoEm (=closed_at), cliqueDeOrigem, telefone (E.164 sem +, em claro — hash é do transporte), valorCentavos, moeda}`. `ResultadoDeEnvio = ok | transitorio | permanente`. Meta: `graph.facebook.com/<ver>/<datasetId>/events`, `action_source:"business_messaging"`, `event_time` em segundos, telefone `ph:[SHA-256 hex]`, rejeita >7 dias. Google: `customers/<id>/conversionActions/<actionId>`, `orderId=eventoId`, token derivado por refresh. 🟢

## Contrato 4 — Manifesto de extensão
`ExtensionManifest`: `{format_version:1, profile:"declarative", publisher/name(slug), version(semver), license:"MIT", host_api{min,max}, permissions[], dependencies:[] (vazio), data{mode:"none"}, display{...}, contributions.crm_cards[]}`. `HOST_API_VERSION=2`. `EXTENSION_LIMITS`: packageBytes=64KB, catalogBytes=512KB, jsonDepth=12, jsonNodes=20000, cards=4. `EXTENSION_CAPABILITIES`: tasks.open|inbox.open|kanban.open|contacts.open|agenda.open|radar.open → destino literal via `PORTA_DA_CAPACIDADE`. `checkCompatibility`: format/profile/host_api/permissões/dependências vazias/cobertura. `validateArtifact`: byte_length + SHA-256 + `mirrorsCatalog`. 🟢

## Contrato 5 — API de operação de extensão (`http.ts`)
`requireExtensionPlatform`: is_platform_admin && !support && scope=="full" && MFA aal2. `Idempotency-Key` tem que ser UUID (senão 422). `X-Expected-Organization-Id` tem que casar (precondição, nunca autoridade). `ExtensionServiceError{code,message,status=409}`. `SQL_ERRORS`: extension_removed(410), extension_forbidden(403), extension_idempotency_conflict(409)... código ausente → 503. 🟢
