# Infra Transversal e Relatórios — Contratos

> Contratos: envelope de resposta, catálogo de erros, auth dual, idempotência, envelope cripto.
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Contrato 1 — Envelope de resposta
Sucesso: `{ data }` ou `{ data, meta }` (`meta` suporta cursor `{cursor, has_more, total}`). Erro: `{ error: { code, message, details? } }`. TODA resposta seta `X-Request-Id`. Status de sucesso restrito a `200|201|204`. `fail(code, message, status, opts)` — `code: ApiErrorCode | (string & {})` (não impõe o catálogo). JSON snake_case; dinheiro `_cents`+`currency`; datas ISO-8601 UTC; UUID v4. 🟢

## Contrato 2 — Catálogo de erros (`ApiErrorCodes`)
Agrupado por status: 400 (invalid_request, validation_failed, invalid_cursor); 401 (unauthenticated, token_expired, token_revoked, mfa_required, auth_in_query_forbidden); 403 (forbidden, forbidden_role, forbidden_tenant, lgpd_anonymization_irreversible); 404 (not_found, pipeline_not_found); Agenda (8); 409 (idempotency_conflict, idempotency_in_progress, state_conflict, contact_exists, duplicate_external_id, event_gone, next_action_changed, voice_already_paired); 422 semântica; 415 (unsupported_media_type, logo_svg_recusado); 413 payload_too_large; 429 rate_limited; Ads leitura (6); External DB (4); Voz (4); Negócios/funil (lead_stage_changed_concurrent, lost_reason_required/invalid, pipeline_immutable_use_clone); Aviso de caso (4); 500/upstream (internal_error, upstream_unavailable, unavailable, waha_error, wacalls_error, ai_provider_error, nuvemshop_error). Código nunca renomeado (versiona por `/api/v2/`). 🟢

## Contrato 3 — Auth dual
`resolveAuthDual(req, {requestId, resource, role, scope})` → `AuthDual`. Sucesso: `{organizationId, actor, supabase, idioma?, via:"session"|"token"}`. Bearer → org do token, admin client; senão → RBAC, client RLS. A rota bearer precisa de entrada em `public-paths.ts`. 🟢

## Contrato 4 — Idempotência
`comIdempotencia<T>(entrada)`. `TTL_MS=24h`, `JANELA_DA_RESERVA_MS=60s`. `DesfechoIdempotente = executou | replay | conflito | em_curso`. `hashDoCorpo` = SHA-256 de `JSON.stringify(ordenar(corpo))`. Corrida fechada (issue #778/migration 0321): reserva antes do efeito; quem perde recebe 409 `idempotency_in_progress`; hash divergente → conflito; formato desconhecido → conflito, nunca replay. 🟢

## Contrato 5 — Envelope cripto (AES-GCM)
`encryptKey(plaintext)` → `{ciphertext, iv (12 bytes), tag (16 bytes), last4}`. `aes-256-gcm`, chave de 32 bytes (`AI_CRED_AES_KEY`). `bufToBytea` → `\x<hex>`; `byteaToBuffer` aceita Buffer/Uint8Array/`\xHEX`. Plaintext nunca logado/persistido/devolvido; só `last4` via view `_safe`. 🟢

## Contrato 6 — Cliente de fetch (browser)
`apiClient.{get,post,patch,...}` gera `X-Request-Id` por request, carimba `Idempotency-Key` em métodos mutantes. Timeouts: 10s leitura, 30s escrita. Retry: MAX_ATTEMPTS=3, só `{429,503}`, honra `Retry-After`; método mutante NUNCA retentado em erro de rede/timeout. 🟢
