# Guia de Referência Técnica e Especificação As-Built (SDD)
> **Diretório Mapeado:** `D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc`  
> **Perfil do Leitor:** Engenheiro de Produto (Features, Limites Técnicos, Schemas, Contratos de API, Lógica de Código)  
> **Origem dos Dados:** Extração e engenharia reversa via pipeline Reversa (SDD As-Built — 2026-09-10)  
> **Total de Arquivos:** 62 arquivos em 14 diretórios

---

## 1. Visão Geral e Propósito para Engenharia de Produto

A pasta `D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc` representa o **Software Design Document (SDD)** vivo do sistema. Diferente de especificações conceituais que podem ficar desatualizadas, este diretório reflete a arquitetura real do código implementado no monólito Next.js 16 e no worker 24/7.

### Quando o Engenheiro de Produto deve consultar este guia:
1. **Modelagem de Novas Features:** Para checar o schema de tabelas existentes ([`erd-complete.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/erd-complete.md)) e tipos de dados ([`data-dictionary.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/data-dictionary.md)).
2. **Ciclo de Vida e Estados:** Para entender como conversas, leads e mensagens transitam entre estados ([`state-machines.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/state-machines.md)).
3. **Contratos de API e Integração:** Para inspecionar endpoints REST v1 ([`openapi/api-v1.yaml`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/openapi/api-v1.yaml)).
4. **Decisões Técnicas Fundamentais:** Para entender por que o sistema foi construído dessa forma ([`adrs/`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/adrs)).
5. **Detalhes dos Módulos Funcionais:** Para analisar requisitos, fluxos e tarefas de cada um dos 9 domínios (`requirements.md`, `design.md`, `tasks.md`).

---

## 2. Matriz Cruzada de Features (De-Para Rápido)

Consulte esta tabela para localizar imediatamente a documentação técnica de uma funcionalidade:

| Feature / Domínio | Módulo Técnico (`doc/`) | ADR Relevante | Diagrama de Fluxo |
| :--- | :--- | :--- | :--- |
| **WhatsApp / Mensageria** | [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\canais-e-mensageria`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/canais-e-mensageria) | ADR-0002 (Guardrails) | [`ingestao-mensagem.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/flowcharts/ingestao-mensagem.md) |
| **Agentes de IA & RAG** | [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\ia-e-agentes`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/ia-e-agentes) | ADR-0001 (Dois Runtimes) | [`turno-do-agente.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/flowcharts/turno-do-agente.md) |
| **Follow-up & Automações** | [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\automacao-e-followup`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/automacao-e-followup) | ADR-0003 (Anti-morte) | [`followup-e-eventos.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/flowcharts/followup-e-eventos.md) |
| **Funil de Vendas / CRM** | [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\crm-e-vendas`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/crm-e-vendas) | ADR-0003 (Radar de Risco) | [`state-machines.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/state-machines.md) |
| **Tenancy, Auth & RBAC** | [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\auth-e-multitenancy`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/auth-e-multitenancy) | ADR-0004 (RLS & Service Role) | [`auth-e-lgpd.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/flowcharts/auth-e-lgpd.md) |
| **LGPD & Termos Legais** | [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\lgpd-legal-e-auditoria`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/lgpd-legal-e-auditoria) | ADR-0005 (Marca Branca & LGPD) | [`auth-e-lgpd.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/flowcharts/auth-e-lgpd.md) |
| **Barramento & Workers** | [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\governanca-de-eventos`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/governanca-de-eventos) | ADR-0006 (Event-log Barramento)| [`c4-containers.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/c4-containers.md) |
| **E-commerce / Nuvemshop**| [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\integracoes-externas`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/integracoes-externas) | — | [`contracts.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/integracoes-externas/contracts.md) |
| **Setup & Onboarding** | [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\onboarding-e-instalacao`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/onboarding-e-instalacao)| ADR-0005 (White-label) | [`deployment.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/deployment.md) |

---

## 3. Arquivos Globais de Arquitetura e Engenharia (Raiz de `doc/`)

Os 16 arquivos localizados na raiz de `D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc` documentam os alicerces sistêmicos e dados transversais da plataforma:

### 1. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\architecture.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/architecture.md)
* **O que define:** Síntese arquitetural da plataforma. Detalha os dois planos de execução: **Plano Web** (Next.js 16, App Router, Server Actions, cliente `supabase-js`, autenticação JWT) e **Plano Worker** (`workers/agent-worker/main.ts`, processo Node contínuo 24/7 com `pg.Pool` puro e fila durável). Apresenta a topologia de contêineres e o princípio do barramento transacional desacoplado via Postgres (`event_log`).
* **Utilidade para Produto:** Leitura obrigatória antes de planejar qualquer funcionalidade assíncrona ou em tempo real, delimitando o que roda na requisição do usuário vs. no worker de fundo.

### 2. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\c4-context.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/c4-context.md)
* **O que define:** Diagrama C4 Nível 1 (Contexto do Sistema). Mapeia as fronteiras do DeskcommCRM com atores externos (Vendedor, Supervisor, Admin, Lead WhatsApp) e sistemas vizinhos: gateway WhatsApp (WAHA), Supabase (Auth, Postgres, Realtime, Storage), Gateways LLM (OpenAI, Anthropic, Google via Vercel AI SDK), Redis/Upstash e Nuvemshop.
* **Utilidade para Produto:** Visualização clara das dependências de terceiros e dos limites de responsabilidade do sistema.

### 3. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\c4-containers.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/c4-containers.md)
* **O que define:** Diagrama C4 Nível 2 (Contêineres). Documenta cada contêiner Docker do ecossistema de produção: `app` (Next.js), `worker` (ts-node loop), `scheduler` (disparador de crons via curl), `waha` (gateway WhatsApp NOWEB), `redis` + `srh` (rate limit REST), `caddy` (proxy reverso com SSL automático) e Supabase Postgres.
* **Utilidade para Produto:** Compreender onde as regras são executadas e quais portas e protocolos conectam cada serviço.

### 4. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\c4-components.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/c4-components.md)
* **O que define:** Diagrama C4 Nível 3 (Componentes de Domínio). Detalha a organização modular interna de `lib/`: `agent-engine`, `ai`, `auth`, `crm`, `followup`, `waha`, `events`, `security`, `branding` e `database`.
* **Utilidade para Produto:** Rastrear exatamente qual módulo de código é responsável por cada etapa do pipeline operacional.

### 5. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\domain.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/domain.md)
* **O que define:** Modelo de domínio conceitual (DDD). Define os agregados e entidades fundamentais: `Organization` (tenant raiz), `User` (membro da equipe), `Lead` (contato comercial), `Conversation` (sessão de atendimento), `Message` (unidade de diálogo), `Agent` (perfil de IA), `Pipeline` (funil), `Stage` (etapa), `Deal` (negócio) e `EventLog` (trilha de auditoria e barramento).
* **Utilidade para Produto:** Vocabulário ubíquo oficial. Garante que produto e engenharia usem a mesma terminologia para entidades e regras.

### 6. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\erd-complete.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/erd-complete.md)
* **O que define:** Diagrama Entidade-Relacionamento (ERD) completo em sintaxe Mermaid. Mostra todas as tabelas do Supabase, chaves primárias (UUIDs), chaves estrangeiras (`organization_id`), tipos de dados e cardinalidades (1:1, 1:N, N:N).
* **Utilidade para Produto:** Consulta rápida de relacionamentos estruturais (ex: como um `lead` se conecta a uma `conversation`, `pipeline` e `deal`).

### 7. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\data-dictionary.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/data-dictionary.md)
* **O que define:** Dicionário de dados consolidado com descrição coluna a coluna de cada tabela do banco de dados, incluindo restrições de nulabilidade, valores padrão (`DEFAULT`), enums e comentários de negócio.
* **Utilidade para Produto:** Essencial para entender os campos disponíveis para criação de filtros, relatórios, dashboards e integrações.

### 8. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\state-machines.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/state-machines.md)
* **O que define:** Máquinas de estado de todas as entidades reativas do sistema:
  * **Conversa:** `open` → `waiting_bot` → `human_takeover` → `paused` → `closed`.
  * **Lead:** `new` → `contacted` → `qualified` → `won` / `lost`.
  * **Fila de Mensagens Outbound:** `pending` → `sending` → `sent` / `failed`.
  * **Fila de Jobs Worker:** `enqueued` → `claimed` → `completed` → `failed` (com retry e DLQ).
* **Utilidade para Produto:** Regras de transição válidas e inválidas, impedindo especificações de funcionalidades que violem o ciclo de vida dos dados.

### 9. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\permissions.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/permissions.md)
* **O que define:** Matriz de permissões e controle de acesso baseado em papéis (RBAC). Documenta os níveis de privilégio: `owner` (dono da organização), `admin` (administrador do tenant), `supervisor` (gerente de equipe), `member` (vendedor/atendente) e `platform_admin` (super-admin cross-tenant de VPS). Detalha as políticas de RLS e guards de borda (`requireRole`).
* **Utilidade para Produto:** Definir quem pode ver, editar ou deletar dados em cada tela e endpoint.

### 10. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\dependencies.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\dependencies.md)
* **O que define:** Mapeamento técnico de dependências do projeto (Next.js 16, React 19, TypeScript 6, Tailwind 4, Zod 4, Supabase-js, Vercel AI SDK, Upstash, Sentry, Vitest, Playwright).
* **Utilidade para Produto:** Entender restrições de compatibilidade e ecossistema de bibliotecas disponíveis.

### 11. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\deployment.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\deployment.md)
* **O que define:** Topologia de infraestrutura para implantação em VPS self-host: mapeamento de portas, redes Docker internas, volumes duráveis de armazenamento e integração de proxy reverso.
* **Utilidade para Produto:** Requisitos mínimos de infraestrutura de clientes para dimensionamento de vendas e viabilidade técnica.

### 12. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\code-analysis.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\code-analysis.md)
* **O que define:** Análise estática do código-fonte realizada pela engenharia reversa. Métricas de complexidade, densidade de comentários, linhas de código por módulo e pontos de atenção de acoplamento.
* **Utilidade para Produto:** Identificar quais áreas do sistema têm maior complexidade e risco de regressão durante refatorações.

### 13. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\inventory.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\inventory.md)
* **O que define:** Inventário quantitativo e nominal completo de todos os componentes de código: 169 rotas API REST, Server Actions, Workers contínuos, Crons e Triggers de banco de dados.
* **Utilidade para Produto:** Mapa de inventário para auditoria de cobertura de funcionalidades existentes.

### 14. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\confidence-report.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\confidence-report.md)
* **O que define:** Relatório de confiabilidade da extração Reversa. Classifica cada informação extraída entre 🟢 **CONFIRMADO** (provado por testes e código ativo), 🟡 **INFERIDO** (derivado de padrões sem teste direto) e 🔴 **LACUNA** (comportamento não coberto).
* **Utilidade para Produto:** Nível de certeza sobre regras de negócio antes de assumir comportamentos em novas especificações.

### 15. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\questions.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\questions.md)
* **O que define:** Perguntas em aberto e pontos de ambiguidade identificados na arquitetura (ex: limites de timeout em LLM, concorrência de atendimento humano simultâneo).
* **Utilidade para Produto:** Backlog de refinamento e decisões de produto pendentes.

### 16. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\gaps.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\gaps.md)
* **O que define:** Lacunas e débitos técnicos conhecidos (endpoints que faltam idempotência, limites de taxa em crons, cobertura de testes de borda).
* **Utilidade para Produto:** Matriz de riscos operacionais e priorização de bugs de produto.

---

## 4. Decisões Arquiteturais Formais (`doc/adrs/`)

Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\adrs`  
Contém as ADRs (Architecture Decision Records) que estabelecem as regras fundamentais de engenharia:

| Arquivo | Título e Decisão Arquitetural | Impacto Operacional / Produto |
| :--- | :--- | :--- |
| [`README.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/adrs/README.md) | Índice e formato padrão de registro de decisões arquiteturais. | Padrão formal de documentação técnica. |
| [`0001-dois-runtimes-de-ia.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/adrs/0001-dois-runtimes-de-ia.md) | **Dois runtimes de IA:** Separação estrita entre o plano web (Next.js para UI e chat manual do operador) e o plano worker (Node 24/7 com `pg.Pool` para mensagens automáticas de WhatsApp). | O chat da web não bloqueia nem compete com a fila em lote de mensagens do WhatsApp. |
| [`0002-cadeia-de-guardrails-determinista.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/adrs/0002-cadeia-de-guardrails-determinista.md) | **Cadeia de guardrails determinista:** Nenhuma resposta de LLM é enviada diretamente ao cliente sem passar por checagens antes do envio (`before-send`: detecção de alucinação de dados bancários, termos proibidos e conformidade de canal). | Segurança de marca e proteção contra respostas prejudiciais de IA. |
| [`0003-anti-morte-follow-up-e-radar.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/adrs/0003-anti-morte-follow-up-e-radar.md) | **Mecanismo anti-morte de follow-up e radar de risco:** O sistema detecta leads inativos e recalcula score dinamicamente. Pausa o follow-up se um humano intervier e reativa com limite de tentativas. | Garante que o robô não fale por cima do vendedor humano nem envie mensagens a clientes que já converteram. |
| [`0004-multi-tenancy-rls-e-service-role.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/adrs/0004-multi-tenancy-rls-e-service-role.md) | **Isolamento por RLS e controle de Service Role:** Isolamento multi-tenant garantido pelo Postgres RLS em todas as tabelas. Uso de service-role restrito a rotas de background, exigindo filtro manual obrigatório de `organization_id`. | Impossibilita vazamento de dados entre empresas clientes na mesma base. |
| [`0005-marca-branca-e-lgpd-controlador.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/adrs/0005-marca-branca-e-lgpd-controlador.md) | **Marca branca dinâmica vs. Identificação LGPD:** A marca visual é totalmente customizável via banco (white-label para revendedores), mas o relatório/dossiê LGPD **nunca** leva marca comercial fictícia — identifica obrigatoriamente a razão social do controlador legal e do DPO. | Conformidade jurídica da LGPD protegendo revendedores e clientes finais. |
| [`0006-event-log-como-barramento.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/adrs/0006-event-log-como-barramento.md) | **Event-log como barramento transacional:** Em vez de filas externas (RabbitMQ/Kafka), o sistema utiliza a tabela `event_log` do Postgres com triggers e claims atômicos `FOR UPDATE SKIP LOCKED`. | Reduz dramaticamente o custo e a complexidade de infraestrutura para self-host. |

---

## 5. Diagramas e Fluxos Operacionais (`doc/flowcharts/`)

Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\flowcharts`  
Fluxogramas visuais em sintaxe Mermaid descrevendo processos de ponta a ponta:

* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\flowcharts\ingestao-mensagem.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/flowcharts/ingestao-mensagem.md):  
  **Fluxo de Ingestão de Mensagens:** Recebimento via webhook do WAHA → Validação de HMAC de segurança → Resolução de Tenant e Contato → Inserção no banco → Emissão do evento `message.received` no `event_log` → Disparo assíncrono para o worker.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\flowcharts\turno-do-agente.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\flowcharts\turno-do-agente.md):  
  **Fluxo do Turno do Agente de IA:** Worker retira job da fila → Recupera histórico recente da conversa → Realiza busca semântica RAG na base de conhecimento do tenant → Envia prompt ao gateway LLM → Avalia resposta na cadeia de guardrails determinísticos → Enfileira mensagem na tabela de envio outbound.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\flowcharts\followup-e-eventos.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\flowcharts\followup-e-eventos.md):  
  **Fluxo de Follow-up e Automação Reativa:** Agendamento de disparo temporal → Checagem de janelas de envio (não enviar de madrugada) → Validação de consentimento do titular → Envio da mensagem de cadência → Monitoramento de resposta.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\flowcharts\auth-e-lgpd.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\flowcharts\auth-e-lgpd.md):  
  **Fluxo de Autenticação e Consentimento LGPD:** Login via Supabase Auth → Injeção de `tenant_id` e claims no JWT → Middleware Next.js validando sessão → Termo de aceite do titular via WhatsApp → Registro imutável de consentimento.

---

## 6. Especificações por Módulo de Domínio (`doc/<modulo>/`)

Cada um dos 9 módulos possui a tríade:
* `requirements.md`: Requisitos funcionais (FR), não-funcionais (NFR) e regras de negócio.
* `design.md`: Funções de código, assinaturas TypeScript, fluxos principais e dependências.
* `tasks.md`: Lista de tarefas técnicas de implementação e validação.

### 6.1. Auth e Multi-tenancy
* **Caminho:** [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\auth-e-multitenancy`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/auth-e-multitenancy)
* **Arquivos:** [`requirements.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/auth-e-multitenancy/requirements.md), [`design.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/auth-e-multitenancy/design.md), [`tasks.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/auth-e-multitenancy/tasks.md)
* **O que define:** Gerenciamento de organizações, convites por e-mail com token criptográfico HMAC, autenticação MFA, resolução de papéis (`roles`), middleware de proteção de rotas privadas e políticas de isolamento Postgres RLS.
* **Destaques para Produto:** Como usuários são vinculados a empresas, permissões de cada tela e fluxos de convite para novos vendedores.

### 6.2. Automação e Follow-up
* **Caminho:** [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\automacao-e-followup`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/automacao-e-followup)
* **Arquivos:** [`requirements.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/automacao-e-followup/requirements.md), [`design.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/automacao-e-followup/design.md), [`tasks.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/automacao-e-followup/tasks.md)
* **O que define:** Motor de automação reativo (gatilhos `lead.created`, `stage.changed`, `tag.added`) e motor de follow-up por nós (`trigger`, `wait`, `condition`, `ai_classify`, `action`, `end`). Funções-chave: `runAutomationForEvent`, `processNode`, `runFollowupTick`, `pausarIaPorAtendimentoManual`.
* **Destaques para Produto:** Regras de ouro para evitar disparo de mensagens fora de horário comercial e cancelamento automático de cadências quando o lead responde.

### 6.3. Canais e Mensageria
* **Caminho:** [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\canais-e-mensageria`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/canais-e-mensageria)
* **Arquivos:** [`contracts.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/canais-e-mensageria/contracts.md), [`requirements.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/canais-e-mensageria/requirements.md), [`design.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/canais-e-mensageria/design.md), [`tasks.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/canais-e-mensageria/tasks.md)
* **O que define:** Conexão direta com WAHA (WhatsApp HTTP API), gerenciamento de sessões (QR Code), recepção de mídia (áudio, imagem, documentos), fila de envio outbound, warm-up (aquecimento) de novos números e proteção contra bloqueio por spam.
* **Destaques para Produto:** Formatos de mídia suportados, latência de envio e debounce para mensagens picadas do WhatsApp.

### 6.4. CRM e Vendas
* **Caminho:** [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\crm-e-vendas`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/crm-e-vendas)
* **Arquivos:** [`requirements.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/crm-e-vendas/requirements.md), [`design.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/crm-e-vendas/design.md), [`tasks.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/crm-e-vendas/tasks.md)
* **O que define:** Ciclo comercial de vendas: nascimento automático do lead via conversa (`garantirLeadDaConversa`), funis e etapas (`crm_pipelines`, `crm_pipeline_stages`), fórmula de cálculo do score do lead (`BASE 30 + 12/compromisso - 8/objeção + 5/BANT`), radar de risco de negócios estagnados e encerramento de demandas (`won` / `lost`).
* **Destaques para Produto:** Lógica matemática de pontuação de leads e regras de movimentação automática de cards no Kanban quando a IA interage.

### 6.5. Governança de Eventos
* **Caminho:** [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\governanca-de-eventos`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/governanca-de-eventos)
* **Arquivos:** [`requirements.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/governanca-de-eventos/requirements.md), [`design.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/governanca-de-eventos/design.md), [`tasks.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/governanca-de-eventos/tasks.md)
* **O que define:** Funcionamento do barramento `event_log`. Mecanismos de deduplicação por chave idempotente, claim atômico via `FOR UPDATE SKIP LOCKED`, estratégias de retry com backoff exponencial e descarte seguro em Dead Letter Queue (DLQ).
* **Destaques para Produto:** Garantia de que nenhuma mensagem, conversão ou disparo de evento seja perdido caso um serviço reinicie.

### 6.6. IA e Agentes
* **Caminho:** [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\ia-e-agentes`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/ia-e-agentes)
* **Arquivos:** [`requirements.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/ia-e-agentes/requirements.md), [`design.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/ia-e-agentes/design.md), [`tasks.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/ia-e-agentes/tasks.md), [`edge-cases.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/ia-e-agentes/edge-cases.md)
* **O que define:** Arquitetura do motor conversacional inteligente: injeção de personalidade, RAG com embeddings pgvector por tenant, execução de ferramentas/tools internas pelo modelo, limite de teto de custos (`teto-de-orcamento`) e regras de transição para intervenção humana (handoff). O arquivo `edge-cases.md` cataloga situações anômalas (mensagens com emojis repetitivos, áudios inaudíveis, prompt injections).
* **Destaques para Produto:** Regras de handoff da IA para atendente humano e proteção orçamentária para o cliente não estourar a conta de LLM.

### 6.7. Integrações Externas
* **Caminho:** [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\integracoes-externas`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/integracoes-externas)
* **Arquivos:** [`contracts.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/integracoes-externas/contracts.md), [`requirements.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/integracoes-externas/requirements.md), [`design.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/integracoes-externas/design.md), [`tasks.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/integracoes-externas/tasks.md)
* **O que define:** Contratos e sincronização de dados com terceiros: e-commerce (Nuvemshop: pedidos, carrinhos abandonados, catálogo de produtos), webhooks de captação de leads (Facebook Ads, landing pages) e serviços de mensageria complementar.
* **Destaques para Produto:** Disparadores de recuperação de carrinho e atualização em tempo real de status de pedidos no WhatsApp.

### 6.8. LGPD, Legal e Auditoria
* **Caminho:** [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\lgpd-legal-e-auditoria`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/lgpd-legal-e-auditoria)
* **Arquivos:** [`requirements.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/lgpd-legal-e-auditoria/requirements.md), [`design.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/lgpd-legal-e-auditoria/design.md), [`tasks.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/lgpd-legal-e-auditoria/tasks.md)
* **O que define:** Mecanismos de compliance com a LGPD (Lei 13.709/2018): consentimento explícito, revogação pelo titular ("parar", "descadastrar"), auditoria imutável em `audit_log`, anonimização de dados pessoais (`data_retention_policy`) e geração de Dossiê do Titular em PDF com hash SHA256.
* **Destaques para Produto:** Como a plataforma protege legalmente a empresa que opera o CRM de multas e processos de privacidade.

### 6.9. Onboarding e Instalação
* **Caminho:** [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\onboarding-e-instalacao`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/onboarding-e-instalacao)
* **Arquivos:** [`requirements.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/onboarding-e-instalacao/requirements.md), [`design.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/onboarding-e-instalacao/design.md), [`tasks.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/doc/onboarding-e-instalacao/tasks.md)
* **O que define:** Jornada de primeira instalação do tenant e scripts de self-host: aplicação do `baseline.sql`, criação da primeira organização, provisionamento do usuário inicial, assistente de configuração (wizard de boas-vindas) e pareamento do WhatsApp via QR Code.
* **Destaques para Produto:** Medição de atrito e tempo para atingir o "Time to Value" (primeira mensagem enviada no WhatsApp).

---

## 7. Contratos de API REST (`doc/openapi/`)

* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\openapi\api-v1.yaml`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\openapi\api-v1.yaml)  
  * **O que define:** Especificação formal OpenAPI 3.0 descrevendo todas as rotas de `app/api/v1/`. Inclui esquemas de payload JSON (em formato `snake_case`), parâmetros de query, cabeçalhos de autenticação (Bearer JWT / API Key), códigos de resposta HTTP (sucessos via `ok()` e falhas padronizadas via `fail()`) e schemas de erro.
  * **Utilidade para Produto:** Contrato definitivo para especificação de integrações com apps móveis, frontends ou clientes externos.

---

## 8. Matrizes de Rastreabilidade e Impacto (`doc/traceability/`)

Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\traceability`

* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\traceability\code-spec-matrix.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\traceability\code-spec-matrix.md):  
  **Matriz Código ↔ Especificação:** Mapeia arquivos de código-fonte reais (`lib/`, `app/`, `workers/`) para seus respectivos documentos de requisitos e design técnico.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\traceability\spec-impact-matrix.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\traceability\spec-impact-matrix.md):  
  **Matriz de Impacto de Mudanças:** Indica quais tabelas, rotas e módulos são afetados caso uma regra de negócio ou requisito de um módulo específico seja alterado.

---

## 9. Histórias de Usuário Canônicas (`doc/user-stories/`)

Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\user-stories`

* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\user-stories\atendimento-com-ia.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\user-stories\atendimento-com-ia.md):  
  Jornadas de usuário detalhadas, critérios de aceite Gherkin e cenários de borda para o atendimento automatizado do lead pelo agente de IA.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\doc\user-stories\captacao-e-conversao-de-lead.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\doc\user-stories\captacao-e-conversao-de-lead.md):  
  Jornada completa de captação (entrada do lead via WhatsApp ou anúncio), qualificação automática de dados, criação de oportunidade no funil e distribuição para o vendedor.
