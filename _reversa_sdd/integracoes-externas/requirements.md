# Integrações Externas (`external-db`, `nuvemshop`, `plataformas-de-anuncio`, `extensions`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 11).

## Visão Geral
Quatro subsistemas independentes que ligam o CRM ao mundo de fora, cada um com sua fronteira de confiança. Padrão comum: `organization_id` sempre no filtro (service-role bypassa RLS), segredo cifrado em repouso (AES-GCM) e decifrado just-in-time (nunca logado, nunca devolvido por rota), e falha-fechada (DNS que não resolve, cifra ausente, código de banco desconhecido → recusa segura). 🟢

## Responsabilidades
- `external-db`: acesso read-only a um Postgres externo do cliente (grade + tools do agente). 🟢
- `nuvemshop`: OAuth + cliente REST da Tiendanube/Nuvemshop, com webhook HMAC. 🟢
- `plataformas-de-anuncio`: conversões (escreve) e leitura/insights (lê) de Meta/Google Ads. 🟢
- `extensions`: sistema de extensões declarativas (modelo de capacidade nomeada, download endurecido). 🟢

## Regras de Negócio
- external-db: só-leitura imposto pelo POSTGRES (`begin read only`), não por análise de string. — `external-db/conexao.ts` 🟢
- external-db: faixas RFC1918 PERMITIDAS (banco na LAN é caso real do dono); link-local/metadata 169.254.169.254 sempre bloqueado; host re-validado a cada leitura (anti-rebinding). — `external-db/guardas.ts` 🟢
- external-db: `contem`/`comeca_com` tolerantes a espaço e caixa (regra C-008: "cb250" acha "CB 250 F Twister"). — `external-db/leitura.ts` 🟢
- nuvemshop: `user_id` da resposta de token É o storeId; tokens não expiram; HMAC sobre rawBody (não re-stringificar). — `nuvemshop/oauth.ts` 🟢
- ads: conversões e leitura são eixos INDEPENDENTES; nunca fundir falha `transitorio` com `permanente`. — `plataformas-de-anuncio/types.ts` 🟢
- ads: `eventoId=<leadId>:<evento>` (dedup determinístico); Google não armazena access token (deriva por refresh). 🟢
- ads: Meta rejeita evento > 7 dias (permanente); telefone hasheado SHA-256 no transporte. 🟢
- extensions: capacidade nomeada → destino CONSTANTE DE CÓDIGO; pacote nunca monta endereço nem importa código do núcleo; `data.mode="none"`. — `extensions/capacidades.ts` 🟢
- extensions: SHA-256 do artefato tem que bater; download com DNS pinning (anti-rebinding); anti prototype-pollution no parse. 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Acesso read-only ao Postgres externo | Must | Qualquer INSERT/UPDATE/DDL recusado pelo Postgres |
| RF-02 | Guarda SSRF do banco externo | Must | Link-local bloqueado; RFC1918 permitido; re-valida a cada leitura |
| RF-03 | Montador de SELECT seguro | Must | Colunas validadas contra o catálogo; valores parametrizados |
| RF-04 | OAuth + webhook Nuvemshop | Should | HMAC sobre rawBody com `timingSafeEqual`; state CSRF |
| RF-05 | Conversões idempotentes por evento | Must | `eventoId=<leadId>:<evento>`; transitorio ≠ permanente |
| RF-06 | Leitura de insights sem interruptor | Should | Desconectar = apagar linha; sem `enabled` |
| RF-07 | Extensões declarativas com capacidade nomeada | Must | Capacidade → destino literal; SHA-256 bate; download endurecido |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | Segredo cifrado AES-GCM, decifrado JIT, nunca logado | `external-db/credenciais.ts`, `ads/credenciais.ts` | 🟢 |
| Segurança | `organization_id` sempre no filtro (admin client) | todos (lição #236) | 🟢 |
| Segurança | HMAC com `timingSafeEqual` + length-guard | `nuvemshop/oauth.ts`, `state.ts` | 🟢 |
| Segurança | DNS pinning anti-rebinding no download de extensão | `extensions/download.ts` | 🟢 |
| Segurança | Anti prototype-pollution (`__proto__/prototype/constructor`) | `extensions/strict-json.ts` | 🟢 |
| Segurança | Bearer só em header, nunca querystring | `ads/*/conversions.ts`, `nuvemshop/api-client.ts` | 🟢 |
| Escalabilidade | Pool LRU (`MAX_POOLS=32`) chaveado por credencial | `external-db/conexao.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado uma tentativa de INSERT no banco externo
Quando consultar() roda em transação read only
Então o Postgres recusa (não por análise de string)

Dado um host que resolve para IP público e privado (rebinding)
Quando validarHostDeBanco avalia
Então recusa (qualquer endereço em faixa proibida)

Dado uma conversão já enviada com o mesmo <leadId>:<evento>
Quando o transporte reenvia
Então a plataforma descarta pela dedup determinística

Dado um artefato de extensão com SHA-256 divergente
Quando validateArtifact roda
Então recusa com extension_digest_mismatch
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Read-only + SSRF + SELECT seguro (RF-01/02/03) | Must | Superfície de ataque crítica |
| Conversões idempotentes (RF-05) | Must | Reporte financeiro |
| Extensões declarativas (RF-07) | Must | Modelo de capacidade, download endurecido |
| OAuth/webhook Nuvemshop (RF-04) | Should | Integração de e-commerce |
| Leitura de insights (RF-06) | Should | Métricas de anúncio |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/external-db/conexao.ts` | `consultar`, `obterPool` | 🟢 |
| `lib/external-db/guardas.ts` | `validarHostDeBanco`, `ipDeBancoProibido` | 🟢 |
| `lib/external-db/leitura.ts` | `montarConsulta`, `clausulaDeFiltro` | 🟢 |
| `lib/nuvemshop/oauth.ts` | `exchangeCodeForToken`, `verifyHmac` | 🟢 |
| `lib/nuvemshop/api-client.ts` | `NuvemshopApiClient` | 🟢 |
| `lib/plataformas-de-anuncio/credenciais.ts` | `lerCredencial` | 🟢 |
| `lib/plataformas-de-anuncio/meta/conversions.ts` | transporteMeta | 🟢 |
| `lib/plataformas-de-anuncio/google/conversions.ts` | transporteGoogle | 🟢 |
| `lib/extensions/capacidades.ts` | `destinoDaCapacidade`, `PORTA_DA_CAPACIDADE` | 🟢 |
| `lib/extensions/manifest.ts` | `checkCompatibility`, `validateArtifact` | 🟢 |
| `lib/extensions/download.ts` | `resolvePublicAddresses`, `pinnedLookup` | 🟢 |
| `lib/extensions/service.ts` | `installExtension` | 🟢 |
