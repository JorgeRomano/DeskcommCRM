# Fluxogramas — Integrações externas

> Gerado pelo Arqueólogo (Reversa) — Unidade 11.
> Cobre `lib/external-db/`, `lib/nuvemshop/`, `lib/plataformas-de-anuncio/`, `lib/extensions/`.

---

## 1. Abrir acesso a banco externo — `abrirAcesso` (`external-db/acesso.ts`)

```mermaid
flowchart TD
  A[abrirAcesso admin, orgId, connectionId] --> B[carregarConexao<br/>.eq organization_id .eq id]
  B --> C{linha existe?}
  C -- não --> CX[motivo: nao_encontrada]
  C -- sim --> D{enabled?}
  D -- não --> DX[motivo: desativada]
  D -- sim --> E[decryptKey da senha]
  E --> F{decifra ok?}
  F -- não --> FX[motivo: cifra_indisponivel<br/>falta AI_CRED_AES_KEY]
  F -- sim --> G[validarHostDeBanco host<br/>RE-VALIDADO a cada leitura]
  G --> H{host é IP literal?}
  H -- sim --> H1{ipDeBancoProibido?}
  H -- não --> H2[dns.lookup all:true]
  H2 --> H3{algum endereço<br/>em faixa proibida?}
  H1 -- sim --> GX[motivo: host_bloqueado]
  H3 -- sim --> GX
  H1 -- não --> I[obterPool cria/reusa<br/>por chave id..sslMode]
  H3 -- não --> I
  I --> J[Acesso pronto: pool + limites]
```

---

## 2. Leitura só-leitura montada no servidor — `lerTabela` (`external-db/leitura.ts`)

```mermaid
flowchart TD
  A[lerTabela pool, pedido, permitidas] --> B[montarConsulta]
  B --> C{schema e tabela<br/>presentes?}
  C -- não --> CX[LeituraInvalidaError: tabela_obrigatoria]
  C -- sim --> D[exigirColuna de cada<br/>projeção/filtro/ordem<br/>contra catálogo]
  D --> E{coluna fora<br/>do catálogo?}
  E -- sim --> EX[coluna_inexistente:c]
  E -- não --> F[clausulaDeFiltro:<br/>switch fechado, valores viram $n]
  F --> G[teto = min LIMITE.maximo,<br/>limiteMax válido senão absoluto]
  G --> H[consultar: BEGIN READ ONLY<br/>+ SET LOCAL timeouts]
  H --> I{INSERT/UPDATE/DDL?}
  I -- sim --> IX[Postgres RECUSA<br/>read-only imposto pelo banco]
  I -- não --> J[serializarLinha:<br/>bigint→string, Date→ISO,<br/>texto>20k truncado]
  J --> K[COMMIT + release]
```

---

## 3. OAuth Nuvemshop + verificação de webhook HMAC (`nuvemshop/oauth.ts`)

```mermaid
flowchart LR
  subgraph Consentimento
    A[buildAuthorizeUrl appId, state] --> B[issueState orgId, actor?<br/>HMAC INTERNAL_SECRET, TTL 10min]
  end
  subgraph Callback
    C[callback com code + state] --> D[verifyState<br/>timingSafeEqual + expiração]
    D --> E[exchangeCodeForToken<br/>POST JSON cache no-store]
    E --> F{access_token E user_id?}
    F -- não --> FX[invalid_token_response]
    F -- sim --> G[storeId = user_id<br/>token não expira]
  end
  subgraph Webhook
    H[webhook x-linkedstore-hmac-sha256] --> I[verifyHmac rawBody, sig, clientSecret]
    I --> J{timingSafeEqual bate?}
    J -- não --> JX[recusa 401]
    J -- sim --> K[processa evento]
  end
```

---

## 4. Envio de conversão offline — transporte por plataforma (`plataformas-de-anuncio/`)

```mermaid
flowchart TD
  A[ConversaoOffline<br/>eventoId = leadId:evento] --> B[transporteDe plataforma<br/>via registry]
  B --> C{plataforma}
  C -- meta_ads --> D[lerCredencial: dataset_id +<br/>access_token decifrado]
  C -- google_ads --> E[lerCredencial: refresh_token +<br/>customer/action id]
  D --> F{evento > 7 dias?}
  F -- sim --> FX[permanente: recusa por idade]
  F -- não --> G[hash telefone SHA-256<br/>action_source business_messaging<br/>event_time em segundos]
  G --> H[POST graph.facebook.com/events<br/>Bearer em header]
  E --> I[renovarToken refresh→access<br/>derivado a cada envio]
  I --> J[POST googleads uploadClickConversions<br/>orderId = eventoId dedup]
  H --> K{resposta}
  J --> K
  K -->|ok| K1[ok: grava enviado]
  K -->|5xx / 429 / throttle| K2[transitorio: reagenda sem contar tentativa]
  K -->|token / config| K3[permanente: vira linha de erro]
```

---

## 5. Instalar extensão declarativa — `installExtension` (`extensions/service.ts`)

```mermaid
flowchart TD
  A[installExtension actorId, operationId, input] --> B[requireExtensionPlatform<br/>is_platform_admin + MFA aal2]
  B --> C[RPC fn_extensions_prepare_install<br/>devolve recibo com applied_now]
  C --> D[validateCatalogSnapshot<br/>anti prototype-pollution]
  D --> E[checkCompatibility<br/>host_api min≤2≤max + cobertura permissão]
  E --> F[downloadArtifact<br/>DNS pinning, https, ≤64KB]
  F --> G[validateArtifact:<br/>SHA-256 bate? mirrorsCatalog?]
  G --> H{tudo válido?}
  H -- não --> HX[RPC fn_extensions_fail_install<br/>+ audit]
  H -- sim --> I[RPC fn_extensions_finish_install<br/>+ audit]
  I --> J[capacidade→destino LITERAL<br/>via PORTA_DA_CAPACIDADE]
```

---

## 6. Modelo de capacidade — tradução nome→destino (`extensions/capacidades.ts`)

```mermaid
flowchart LR
  A[Cartão pede capacidade<br/>ex tasks.open] --> B[destinoDaCapacidade cap]
  B --> C{ehCapacidade?<br/>lista fechada}
  C -- não --> CX[null → chamador RECUSA<br/>nunca destino de reserva]
  C -- sim --> D[PORTA_DA_CAPACIDADE cap<br/>Record exaustivo]
  D --> E[destino = /app/tasks<br/>constante de código]
  E --> F{destino em<br/>DESTINOS_PERMITIDOS?}
  F -- sim --> G[host devolve destino<br/>tela reconfere]
```
