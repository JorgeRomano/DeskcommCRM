# Integrações Externas — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface (destaques)
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `consultar` | `(pool, sql, params)` | linhas (transação read only) |
| `validarHostDeBanco` | `(host)` | ok/recusa (SSRF) |
| `montarConsulta` | `(catalogo, opts)` | SELECT parametrizado |
| `exchangeCodeForToken` | `(code, cfg)` | `{access_token, user_id}` |
| `verifyHmac` | `(rawBody, sigHex, clientSecret)` | `boolean` |
| `lerCredencial` | `(admin, orgId, plataforma)` | credencial ou `MotivoSemConexao` |
| `destinoDaCapacidade` | `(cap)` | destino literal ou `null` |
| `validateArtifact` | `(...)` | ok ou `extension_digest_mismatch` |
| `installExtension` | `(...)` | via RPCs `fn_extensions_*` idempotentes |

## Fluxo Principal — external-db (read-only)
`credenciais` (decifra) → `guardas` (rede) → `acesso` (orquestra, re-valida host) → `conexao` (pool + `begin read only` + `set local timeouts`) → `leitura` (SELECT seguro: identificadores quotados+validados, valores `$n`). Pool chaveado por credencial (editar derruba pool velho). 🟢

## Fluxo Principal — Nuvemshop
OAuth (`buildAuthorizeUrl` com state CSRF → `exchangeCodeForToken`, `user_id`=storeId). Webhook: header `x-linkedstore-hmac-sha256`, `verifyHmac` sobre rawBody. `NuvemshopApiClient` usa `Authentication: bearer <token>` (minúsculo, spec Nuvemshop). 🟢

## Fluxo Principal — Ads
Dois eixos independentes. Conversões: `lerCredencial` (completude por plataforma) → transporte (`registry`: meta/google) → `ResultadoDeEnvio` (ok/transitorio/permanente). Google deriva access token por refresh a cada envio. Meta hasheia PII e rejeita evento > 7 dias. Leitura: `lerCredencialDeLeitura`, `resolverHierarquiaDoContato` (cache-first, nunca lança, resolvido preguiçosamente). 🟢

## Fluxo Principal — Extensions
Capacidade nomeada → `PORTA_DA_CAPACIDADE` (Record exaustivo) → destino literal. `installExtension`: RPC prepare → `validateCatalogSnapshot` → `checkCompatibility` → `downloadArtifact` (DNS pinning) → `validateArtifact` (SHA-256) → RPC finish + audit. Persistência só via RPCs `fn_extensions_*` (idempotência por `applied_now`). 🟢

## Dependências
- `external-db` → `lib/crypto/aes_gcm`, `pg`, `dns`. 🟢
- `nuvemshop` → `crypto` (HMAC), `INTERNAL_SECRET` (state). 🟢
- `ads` → `ad_platform_connections`/`ad_insights_connections`/`ad_hierarchy_cache`, `graph.facebook.com`/Google Ads API. 🟢
- `extensions` → RPCs `fn_extensions_*`, `platform_admins`, `server-only`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Read-only pelo Postgres, não por string | `external-db/conexao.ts` | 🟢 |
| SSRF do banco permite LAN, bloqueia metadata; re-valida host | `external-db/guardas.ts` | 🟢 |
| Ads: transitorio ≠ permanente, nunca fundir | `ads/types.ts` | 🟢 |
| Capacidade → destino constante de código | `extensions/capacidades.ts` | 🟢 (ADR-0003 doutrina de extensões) |
| SHA-256 do artefato + DNS pinning | `extensions/manifest.ts`, `download.ts` | 🟢 |

## Estado Interno
- `external_db_connections`, `ad_platform_connections`, `ad_insights_connections`, `ad_hierarchy_cache`, `extension_catalogs`/`installations`/`operations`/`organization_extensions`. 🟢

## Observabilidade
- `plataforma_sem_transporte` logado (não omitido); `causaSegura` extrai só `{cause_code, cause_status}` (texto remoto nunca vaza). 🟢

## Riscos e Lacunas
- 🔴 Tabelas/RLS/CHECKs e corpos das RPCs `fn_extensions_*` — Data Master.
- 🟡 Comportamento de cada provedor externo sob erro (rate limit, saldo zero, API desativada) inferido do mapeamento `classificaErro`/`classifica4xx`.
