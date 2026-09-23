# Integrações Externas — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas: `external_db_connections`, `ad_platform_connections`, `ad_insights_connections`, `ad_hierarchy_cache`, `extension_*`
- [ ] RPCs `fn_extensions_*` (migrations 0271/0340); `lib/crypto/aes_gcm`
- [ ] Env: `NUVEMSHOP_APP_ID/CLIENT_ID/CLIENT_SECRET`, `INTERNAL_SECRET`, `AI_CRED_AES_KEY`

## Tarefas
- [ ] T-01, Implementar external-db (conector read-only)
  - Origem no legado: `lib/external-db/*` (`limites`, `schemas`, `credenciais`, `guardas`, `acesso`, `conexao`, `leitura`, `introspeccao`)
  - Critério de pronto: read-only pelo Postgres; SSRF (LAN ok, metadata bloqueado, re-valida host); SELECT parametrizado; C-008 tolerante a espaço/caixa
  - Confiança: 🟢
- [ ] T-02, Implementar Nuvemshop (OAuth + cliente REST)
  - Origem no legado: `lib/nuvemshop/*` (`config`, `oauth`, `state`, `api-client`)
  - Critério de pronto: HMAC sobre rawBody; state CSRF; `Authentication: bearer` minúsculo; tokens não expiram
  - Confiança: 🟢
- [ ] T-03, Implementar plataformas de anúncio (conversões + leitura)
  - Origem no legado: `lib/plataformas-de-anuncio/*` (`types`, `credenciais`, `credenciais-de-leitura`, `registry`, `hierarquia-do-contato`, `meta/conversions`, `google/conversions`)
  - Critério de pronto: eixos independentes; dedup determinístico; transitorio≠permanente; Google deriva token; Meta rejeita >7d
  - Confiança: 🟢
- [ ] T-04, Implementar extensões declarativas
  - Origem no legado: `lib/extensions/*` (`capacidades`, `manifest`, `strict-json`, `download`, `service`, `http`, `vocabulario`, `erros-do-banco`, `versao`, `errors`)
  - Critério de pronto: capacidade → destino literal; SHA-256 bate; DNS pinning; anti prototype-pollution; persistência só via RPCs
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, INSERT no banco externo é recusado pelo Postgres
- [ ] TT-02, Host com rebinding é recusado
- [ ] TT-03, Coluna fora do catálogo → `coluna_inexistente`
- [ ] TT-04, HMAC Nuvemshop inválido → false
- [ ] TT-05, Conversão duplicada descartada por dedup
- [ ] TT-06, Artefato com SHA-256 divergente recusado

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Tabelas de conexão com RLS zero-policies (exigem admin client)

## Ordem Sugerida
1. T-01 (external-db) e T-02 (Nuvemshop) independentes.
2. T-03 (ads) independente.
3. T-04 (extensions) por último (superfície maior).

## Lacunas Pendentes (🔴)
- Corpos das RPCs `fn_extensions_*` e RLS das tabelas de conexão (Data Master).
