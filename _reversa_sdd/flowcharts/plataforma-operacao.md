# Fluxogramas — Plataforma e operação

> Unidade 10 do Arqueólogo. Módulos: `settings`, `onboarding`, `instalacao`, `branding`, `navigation`, `system`, `operacao`.
> Diagramas Mermaid dos fluxos não-triviais. 🟢 CONFIRMADO salvo indicação.

## 10.1 Resolução da marca (white-label) no render

```mermaid
flowchart TD
  A[app/layout.tsx] --> B[resolverMarca camadas, regua]
  B --> C{camada da organizacao tem cor valida?}
  C -->|sim| D[usa cor da org]
  C -->|nao / invalida| E{camada da instalacao?}
  E -->|sim| F[usa cor da instalacao]
  E -->|nao / invalida| G{camada do ambiente .env?}
  G -->|sim| H[usa semente do .env]
  G -->|nao| I[padrao do produto REGUA_DO_PRODUTO]
  D --> J[derivarMarca semente, regua]
  F --> J
  H --> J
  I --> J
  J --> K{semente acromatica? C < LIMIAR}
  K -->|sim| L[mantem rampa do produto + hex vai p/ --color-brand + motivo marca_acromatica]
  K -->|nao| M[rampaDeSemente: 11 stops via OKLab]
  M --> N[escolherAccent: desloca rampa p/ passar contraste]
  N --> O[reconciliarSemanticas sob dicromacia]
  O --> P[cssDaMarca: serializa allowlist + fail-closed]
  P --> Q[injeta CSS + carrega motivos]
  B -. nunca lanca; excecao aqui = 500 em toda tela .-> Q
```

## 10.2 Deslocamento de accent por contraste (escolherAccent)

```mermaid
flowchart TD
  A[rampa base + tema] --> B[candidatos de offset: 0, depois direcao afastando da superficie]
  B --> C{offset passa PISOS texto 4.5 e componente 3.0?}
  C -->|sim, primeiro que passa| D[aplica deslocamento na rampa INTEIRA]
  C -->|nenhum passa| E[fallback menos-ruim]
  E --> F[ranqueia por: menos falhas, depois maior folga relativa razao/piso]
  F --> D
  D --> G[EscolhaDeAccent com deslocamento + motivo se houve shift]
  G -. nunca lanca .-> H[retorna]
```

## 10.3 Reset de dados operacionais (ordem carga por FK RESTRICT)

```mermaid
flowchart TD
  A[apagarDadosOperacionaisDaOrg] --> B[para cada Raiz na ORDEM fixa]
  B --> C[messages .eq organization_id]
  C --> D[conversations]
  D --> E[calendar_appointments]
  E --> F[orders]
  F --> G[crm_leads]
  G --> H[contacts SEMPRE por ultimo]
  C & D & E & F & G & H --> I{erro 23503 ou outro?}
  I -->|sim| J[retorna ok:false + falha + counts parciais]
  I -->|nao, ate o fim| K[retorna ok:true + counts]
  A -. NAO atomico; ordem filhos-antes-de-pais deixa banco integro se falhar .-> J
```

## 10.4 Máquina de estado da rodada de update (update-run.ts)

```mermaid
stateDiagram-v2
  [*] --> dispatched
  dispatched --> success: canTransition ok
  dispatched --> failed: canTransition ok
  dispatched --> failed_rolled_back: canTransition ok
  success --> [*]: terminal imutavel
  failed --> [*]: terminal imutavel
  failed_rolled_back --> [*]: terminal imutavel
  note right of dispatched
    isRunStale apos RUN_STALE_AFTER_MS 15min
    (unknown derivado na leitura)
  end note
  note right of success
    sucessoJaInstalado suprime "Atualizar agora"
    ate o heartbeat de 5min escrever current_version
  end note
  note right of failed_rolled_back
    rollbackDesmentidoPeloApp:
    se APP_VERSION rodando == to_version,
    a imagem nova esta de pe
  end note
```

## 10.5 Sugestão de funil no onboarding (sugerirFunil)

```mermaid
flowchart TD
  A[ContextoDoNegocio nome + oQueFaz] --> B[escolherPacotePorTexto via PISTAS regex]
  B --> C[gerar - UNICO ponto de rede, injetado, usa cerebro do agente publicado]
  C --> D[extrairJson: fatia primeiro brace ao ultimo]
  D --> E[normalizarProposta: dedup por nome e por passo repetido]
  E --> F[validarProposta]
  F --> G{valida? tem nome, >=4 etapas, tem won, tem lost}
  G -->|sim| H[Sugestao origem: ia + proposta]
  G -->|nao / qualquer falha acima| I[Sugestao origem: pacote + porque]
  C -->|erro de rede| I
  D -->|json invalido| I
  I -. nunca quadro vazio; cai no pacote curado .-> J[retorna]
  H --> J
```

## 10.6 Interseção de interface (empresa × vínculo)

```mermaid
flowchart TD
  A[combinarInterfaces daEmpresa, doVinculo] --> B{ambas ausentes?}
  B -->|sim| C[COMPLETA]
  B -->|nao| D[org estreita o universo de destinos]
  D --> E[vinculo estreita DENTRO do universo da org]
  E --> F{intersecao vazia?}
  F -->|sim| G[SO_O_ESSENCIAL]
  F -->|nao| H[destinos = intersecao dos dois eixos]
  H --> I[canSee por ROLE_RANK - unica funcao de autorizacao]
  C --> I
  G --> I
  I -. gate de APRESENTACAO, nunca de autorizacao .-> J[projecoes: sidebar / hub / busca]
```

## 10.7 Leitura sticky de comportamento da instalação

```mermaid
flowchart TD
  A[comportamentoEmVigor / carregarComportamento] --> B{leu do banco com sucesso?}
  B -->|sim| C[grava em globalThis __ultimoComportamentoConhecido + geracao + TTL 30s]
  C --> D[retorna valor lido]
  B -->|erro de transporte| E{ja houve leitura bem-sucedida antes?}
  E -->|sim| F[STICKY: mantem ultimo conhecido - erro NAO promove piso]
  E -->|nao| G[piso por CAMPO: cada campo ruim cai no proprio piso]
  G -. lixo editado a mao nao desliga protecao de gasto .-> H[retorna]
  F --> H
  D --> H
```
