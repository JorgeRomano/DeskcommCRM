# Infra Transversal e Relatórios (`api`, `supabase`, `crypto`, `env`, `logger`, `i18n`, `schemas`, `query`, `http`, `net`, `reports`, `metrics`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 13).

## Visão Geral
O substrato compartilhado sobre o qual toda rota, worker e tela se apoiam: formato canônico de resposta HTTP, catálogo de códigos de erro, os três clients Supabase, auth dual, idempotência, cripto AES-GCM, contrato de env validado por Zod, log estruturado, i18n e dois construtores de relatório/métrica. 🟢

## Responsabilidades
- Prover `ok()`/`fail()`/`noContent()` (formato de resposta + `X-Request-Id`) e o catálogo de erros. 🟢
- Prover os três clients Supabase (server RLS, browser, admin service-role). 🟢
- Resolver auth dual (cookie OU bearer) e idempotência de escrita. 🟢
- Cifrar/decifrar segredos (AES-GCM) e validar o contrato de env (Zod). 🟢
- Log estruturado, i18n, schemas de validação, cliente de fetch e classificação de rede. 🟢
- Construir relatório de atividades e o Índice de Atrito. 🟢

## Regras de Negócio
- `fail()` aceita qualquer string (`ApiErrorCode | (string & {})`), NÃO impõe o catálogo — inventar código no call site vira contrato de wire calado. — `api/wrappers.ts` 🟢
- Código de erro nunca é renomeado (versiona via `/api/v2/`); todo código não-genérico existe porque a TELA age diferente por código. — `api/errors.ts` 🟢
- Sempre `getUser()` no server, nunca `getSession()`; cookie `sb-deskcomm-auth` `sameSite:strict`, `httpOnly`. — `supabase/server.ts` 🟢
- `createAdminClient` bypassa RLS → handlers filtram `organization_id` de fonte confiável, nunca do body. — `supabase/admin.ts` 🟢
- `cookieSecure` deriva do PROTOCOLO da URL, não de `NODE_ENV` (self-host HTTP não perde o cookie). — `supabase/cookie-secure.ts` 🟢
- Auth dual: org SEMPRE do token (bearer) ou do RBAC (cookie), nunca do cliente; rota bearer TAMBÉM precisa de `public-paths`. — `api/auth-dual.ts` 🟢
- Idempotência: reserva ANTES do efeito, corrida fechada (issue #778); hash do corpo ordenado; formato desconhecido → conflito, nunca replay. — `api/idempotency.ts` 🟢
- Cripto: plaintext nunca logado/persistido/devolvido; só `last4` exposto via view `_safe`. — `crypto/aes_gcm.ts` 🟢
- Env: knobs que uma pessoa digita usam `z.string()`, nunca `z.enum`/`z.coerce.number` (enum deixaria contêiner healthy com 500). — `env.ts` 🟢
- Método mutante NUNCA é retentado em erro de rede/timeout (write timeout é "não sei"; issue #783 triplicou cobrança). — `api/client.ts` 🟢
- Métrica de eficiência é publicada PAREADA com a medida de dano; dado ausente é `null`, nunca `0`. — `metrics/atrito.ts` 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Formato canônico de resposta | Must | Toda resposta seta `X-Request-Id`; sucesso `{data, meta?}`, erro `{error:{code,message,details?}}` |
| RF-02 | Três clients Supabase | Must | server usa `getUser()`; admin bypassa RLS com filtro manual de org |
| RF-03 | Auth dual | Must | Org do token/RBAC, nunca do cliente; `via` distinguido |
| RF-04 | Idempotência de escrita | Must | Reserva antes do efeito; corrida → 409 `idempotency_in_progress` |
| RF-05 | Cripto AES-GCM | Must | Plaintext nunca sai; só `last4` na view `_safe` |
| RF-06 | Contrato de env por Zod | Must | Knobs digitados usam `z.string()`; falha real lança no boot |
| RF-07 | i18n com fallback legível | Should | Tradução ausente degrada para português, nunca chave crua |
| RF-08 | Relatório + Índice de Atrito | Should | Eficiência sempre pareada com dano; `null` para dado ausente |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | `httpOnly` + `sameSite:strict` no cookie de sessão | `supabase/server.ts` | 🟢 |
| Segurança | Log nunca com segredo/token/CPF/telefone | `logger.ts` | 🟢 |
| Segurança | Realtime token via endpoint (cookie httpOnly esconde do supabase-js) | `supabase/browser.ts` | 🟢 |
| Corretude | Env lança no boot real, leniente só no build | `env.ts` | 🟢 |
| Corretude | Retry só 429/503; mutação nunca retenta | `api/client.ts`, `query/client.ts` | 🟢 |
| Corretude | `ipDoCliente` não é prova de origem (só rate-limit/exibição) | `http/ip-do-cliente.ts` | 🟢 |
| Disponibilidade | `classificarFalhaDeAlcance` distingue config de serviço-fora | `net/alcance.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado uma resposta de sucesso
Quando ok(data, {requestId}) monta
Então inclui X-Request-Id e o envelope {data}

Dado duas requisições simultâneas com a mesma Idempotency-Key
Quando comIdempotencia executa
Então uma reserva vence e a outra recebe 409 idempotency_in_progress

Dado um método mutante que sofre timeout de rede
Quando apiClient avalia o retry
Então NÃO retenta (timeout de escrita é "não sei")
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Resposta canônica + clients (RF-01/02) | Must | Base de toda rota |
| Auth dual + idempotência (RF-03/04) | Must | Integração externa sem duplicata |
| Cripto + env (RF-05/06) | Must | Segurança e boot correto |
| i18n (RF-07) | Should | Multi-idioma com fallback |
| Relatórios (RF-08) | Should | Métrica honesta (eficiência+dano) |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/api/wrappers.ts` | `ok`, `fail`, `noContent` | 🟢 |
| `lib/api/errors.ts` | `ApiErrorCodes`, `ApiErrorCode` | 🟢 |
| `lib/supabase/{server,browser,admin}.ts` | `createClient`, `createAdminClient` | 🟢 |
| `lib/supabase/cookie-secure.ts` | `cookieSecure` | 🟢 |
| `lib/api/auth-dual.ts` | `resolveAuthDual` | 🟢 |
| `lib/api/idempotency.ts` | `comIdempotencia` | 🟢 |
| `lib/crypto/aes_gcm.ts` | `encryptKey`, `decryptKey` | 🟢 |
| `lib/env.ts` | schema Zod, tiers | 🟢 |
| `lib/logger.ts` | `info`/`warn`/`error` | 🟢 |
| `lib/i18n/*` | `traduzir`, `normalizarIdioma` | 🟢 |
| `lib/reports/atividades.ts`, `lib/metrics/atrito.ts` | relatório, Índice de Atrito | 🟢 |
