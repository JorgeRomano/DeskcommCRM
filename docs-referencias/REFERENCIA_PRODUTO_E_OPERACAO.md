# Guia de Referência de Produto, Regras de Negócio e Operação
> **Diretório Mapeado:** `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs`  
> **Perfil do Leitor:** Engenheiro de Produto (Regras de Negócio, PRDs, Fluxos de Usuário, Operação VPS, Runbooks, Design System)  
> **Total de Arquivos:** 226 arquivos em 25 diretórios

---

## 1. Visão Geral e Propósito para Engenharia de Produto

Enquanto a pasta `doc/` registra a especificação técnica do código (as-built), a pasta `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs` representa o **repositório de conhecimento estratégico, operacional e de produto**. Nela residem os requisitos de negócio que originaram o software, o design system, a doutrina de confiabilidade de engenharia (*Sistema Vivo*), os manuais de gestão e os runbooks de sobrevivência em servidores VPS de produção.

### Quando o Engenheiro de Produto deve consultar este guia:
1. **Definição de Novas Histórias e Critérios de Aceite:** Consultar os PRDs ([`docs/prd/`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/prd)) e o Catálogo de Regras de Negócio ([`docs/business-rules/`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/business-rules)).
2. **Desenho de Novas Telas e Fluxos:** Consultar o Design System ([`docs/design-system/`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system)) e o inventário de telas e jornadas ([`screen-flow/`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow)).
3. **Operação e Resolução de Incidentes:** Consultar os Runbooks de produção ([`docs/runbooks/`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/runbooks)), limites de cota e rotinas de deploy/atualização.
4. **Comercialização White-Label:** Consultar os contratos e regras de marca própria ([`white-label.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/white-label.md)).
5. **Garantia de Não-Regressão e QA:** Consultar o mapa de jornadas vivas ([`user-journey-map.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/testing/user-journey-map.md)) e a auditoria de invariantes ([`doctrine/sistema-vivo.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/doctrine/sistema-vivo.md)).

---

## 2. Matriz Cruzada de Features (De-Para Produto & Operação)

| Feature / Capacidade | PRD & Regras de Negócio | Specs Técnicas Originais | Telas & Design System | Operação & Runbooks |
| :--- | :--- | :--- | :--- | :--- |
| **WhatsApp (WAHA)** | PRD-03 · Regras `W-01..06` | Spec-03 · `pre-go-live` | DS-06 (Components) | `runbooks/waha-hostgator.md` |
| **Agente de IA & RAG** | PRD-05 · Regras `IA-01..07` | Spec-05 · Spec-10 · Spec-16 | `architecture/teto-de-orcamento` | `runbooks/ai-credentials-rotation.md` |
| **Pipeline & Kanban** | PRD-04 · Regras `P-01..06` | Spec-04 · Spec-17 (Lead) | `screen-flow/03-screen-inventory` | `design-system/screen-flow/` |
| **Customer 360** | PRD-02 · Regras `AT-01..06` | Spec-02 · Spec-15 (Casos) | `screen-flow/02-journeys` | `architecture/crm-vivo` |
| **Multi-Tenancy & RBAC** | PRD-01 · Regras `T-01..04` | Spec-01 | `docs/interface-por-vinculo.md` | `docs/support-sessions.md` |
| **LGPD & Conformidade** | PRD-06 · Regras `L-01..04` | Spec-06 | `doc/flowcharts/auth-e-lgpd` | `docs/threat-model.md` |
| **Deploy & Manutenção VPS**| — | Spec-08 | — | `runbooks/deploy.md` · `ATUALIZANDO.md` |
| **Marca Própria (White-Label)**| PRD-01 | — | DS-02 (Palette) | `docs/white-label.md` |

---

## 3. Documentos Mestres e Operacionais da Raiz (`docs/`)

Os 15 arquivos localizados na raiz de `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs` definem a governança operacional e a documentação transversal do sistema:

### 1. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\index.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/index.md)
* **O que define:** Índice mestre original do repositório documental. Contém a regra de ouro de precedência documental: `CLAUDE.md` (doutrina) > `docs/specs/` (contratos técnicos) > `docs/prd/` (intenção de produto) > `HANDOFF-*.md` (estado de sessão) > README.
* **Utilidade para Produto:** Tabela de consulta rápida e resolução de conflitos entre documentações de épocas distintas.

### 2. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\SETUP.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/SETUP.md)
* **O que define:** Manual completo de configuração local e em servidor. Detalha a finalidade de todas as variáveis de ambiente (`.env`), portas utilizadas (3000, 5432, 3001 para WAHA, 6379/8079 para Redis) e procedimentos para rodar o banco local via Docker.
* **Utilidade para Produto:** Guia rápido para reproduzir bugs localmente com exatamente as mesmas variáveis e chaves de produção.

### 3. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\DEPLOY-CHECKLIST.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/DEPLOY-CHECKLIST.md)
* **O que define:** Lista de checagem obrigatória antes e depois de subir uma nova versão em produção na VPS. Passos de validação de conectividade de banco, webhook do WhatsApp, certificados SSL e logs de boot.
* **Utilidade para Produto:** Checklist de "go/no-go" para lançamentos de novas versões e validações operacionais.

### 4. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\ATUALIZANDO.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/ATUALIZANDO.md)
* **O que define:** Procedimentos de upgrade do sistema na VPS. Explica o funcionamento dos scripts automatizados: `update.sh` (atualização sem downtime percebido), `restore.sh` (recuperação de desastres via backup) e `healthcheck.sh`.
* **Utilidade para Produto:** Compreender a experiência do cliente operador ao aplicar patches e atualizações de novas features.

### 5. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\current-state.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/current-state.md)
* **O que define:** Diagnóstico honesto da saúde do sistema: inventaria o que está 100% pronto, o que está parcialmente implementado e quais dívidas técnicas ou limitações existem em cada módulo.
* **Utilidade para Produto:** Entender a maturidade real de cada funcionalidade antes de prometer prazos ou melhorias a clientes.

### 6. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\harness-audit.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/harness-audit.md)
* **O que define:** Auditoria detalhada do harness de engenharia: cobertura de testes automatizados (unitários, banco/RLS e E2E Playwright), maturidade de CI/CD e controles de branch protection.
* **Utilidade para Produto:** Saber quais partes do sistema possuem proteção automática de testes contra quebras acidentais.

### 7. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\threat-model.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/threat-model.md)
* **O que define:** Modelagem de ameaças e superfícies de ataque em ambiente self-host. Análise de riscos de RLS bypass, interceptação de webhooks, injeção de prompt em IA e vazamento de tokens.
* **Utilidade para Produto:** Requisitos não-funcionais de segurança para blindar novas features contra abusos.

### 8. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\alertas-de-seguranca-triados.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/alertas-de-seguranca-triados.md)
* **O que define:** Histórico de triagem e justificativa técnica para alertas de segurança de scanners automáticos (GitHub Dependabot / Snyk) que foram descartados por serem falsos positivos ou mitigados.
* **Utilidade para Produto:** Respaldo técnico em conversas com clientes corporativos que exigem relatórios de vulnerabilidades.

### 9. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\white-label.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/white-label.md) (e traduções `.en.md`, `.es.md`)
* **O que define:** A bíblia do modelo White-Label do DeskcommCRM. Explica como revendedores mudam nome, logos, favicon e cores via banco de dados (`platform_branding`), como o domínio é resolvido e quais telas preservam a razão social do controlador por exigência jurídica da LGPD.
* **Utilidade para Produto:** Diretrizes completas para funcionalidades de personalização visual e modelo de revenda do software.

### 10. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\interface-por-vinculo.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/interface-por-vinculo.md)
* **O que define:** Mapeamento visual das variações de interface dependendo do papel do usuário logado (Admin da Plataforma, Dono do Tenant, Supervisor e Vendedor/Membro). Demonstra quais botões, menus e colunas são ocultados ou desativados para cada perfil.
* **Utilidade para Produto:** Referência essencial para especificação de layouts e comportamento de permissões em novas telas.

### 11. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\support-sessions.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/support-sessions.md)
* **O que define:** Mecanismo de sessão temporária de suporte técnico. Como um administrador de plataforma ganha acesso de leitura assistida a uma organização com autorização explícita, log de auditoria e expiração automática de token.
* **Utilidade para Produto:** Compreender como funciona o suporte ao cliente final sem violar a privacidade e o isolamento de dados.

### 12. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\vendaval-fusion-plan.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/vendaval-fusion-plan.md)
* **O que define:** Plano de fusão e integração com o projeto Vendaval (módulo legado/irmão de captação e automação).
* **Utilidade para Produto:** Contexto histórico sobre módulos que foram unificados ao DeskcommCRM.

### 13. [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\vendaval-vps-deploy-comandos.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/vendaval-vps-deploy-comandos.md)
* **O que define:** Roteiro técnico de comandos bash para migração de instâncias de clientes oriundas do ecossistema Vendaval.
* **Utilidade para Produto:** Suporte operacional para clientes em processo de migração de plataforma.

---

## 4. Regras de Negócio Oficiais & PRDs

### 4.1. Catálogo de Regras de Negócio (`docs/business-rules/`)
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\business-rules\00-business-rules-catalog.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/business-rules/00-business-rules-catalog.md)  
  **O arquivo mais importante para o Engenheiro de Produto.** Catálogo estruturado com todas as regras de negócio normalizadas, formato GIVEN/WHEN/THEN/EXCEPT e camada de aplicação (`DB`, `API`, `UI`, `Worker`, `LLM`):
  * **Regras `T-01` a `T-04` (Tenancy):** RLS mandatório em toda tabela, proibição de leitura cross-tenant, claims de JWT e regras do super-admin.
  * **Regras `L-01` a `L-04` (LGPD):** Consentimento obrigatório antes de mensagens ativas, tratamento automático de opt-out ("sair/parar"), prazo de expurgo de dados e inviolabilidade de logs de auditoria.
  * **Regras `W-01` a `W-06` (WhatsApp):** Anti-banimento (debounce de 45s, spinning de variações de texto), respeito à janela de atendimento de 24h da Meta e controle de instâncias WAHA.
  * **Regras `P-01` a `P-06` (Pipeline):** Exigência de funil padrão ativo, restrições para exclusão de etapas com leads associados, cálculo de recência e transição de negócios.
  * **Regras `AT-01` a `AT-06` (Atendimento):** Handoff bot→humano imediato sob pedido explícito, travamento de envio do bot durante digitação do operador humano e SLA de resposta.
  * **Regras `IA-01` a `IA-07` (Inteligência Artificial):** Verificação de teto de gastos antes de chamada LLM, passagem por guardrails determinísticos, proibição de respostas financeiras inventadas e fallback seguro.
  * **Regras `B-01` a `B-04` (Billing & VPS):** Limitações operacionais por plano/licença no modelo self-host.

### 4.2. PRDs — Product Requirement Documents (`docs/prd/`)
Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\prd`
* [`00-prd-master.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/prd/00-prd-master.md): Visão de produto, proposta de valor (CRM open source self-host sem taxas por usuário), personas e KPIs de sucesso.
* [`01-prd-platform-base.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/prd/01-prd-platform-base.md): Requisitos de autenticação, multi-tenancy e fundação LGPD.
* [`02-prd-customer-360.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/prd/02-prd-customer-360.md): Requisitos da tela de perfil unificado do cliente (dados, notas, compras, tags e histórico de chats).
* [`03-prd-whatsapp-waha.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/prd/03-prd-whatsapp-waha.md): Requisitos de integração com WhatsApp via WAHA e proteção de reputação de número.
* [`04-prd-pipeline-attendance.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/prd/04-prd-pipeline-attendance.md): Requisitos do funil visual Kanban e inbox de atendimento compartilhado.
* [`05-prd-ai-rag-handoff.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/prd/05-prd-ai-rag-handoff.md): Requisitos do agente de IA inteligente, base de conhecimento RAG e transferência para humanos.
* [`06-prd-nuvemshop-lgpd.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/prd/06-prd-nuvemshop-lgpd.md): Requisitos de integração com lojas virtuais Nuvemshop e conformidade de e-commerce.

### 4.3. Épicos e Histórias de Negócio (`docs/stories/epics/`)
Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\stories\epics`
* [`MASTER.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/stories/epics/MASTER.md) e [`TEMPLATE.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/stories/epics/TEMPLATE.md): Estrutura mestra do roadmap de engenharia.
* [`EPIC-00-foundation.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/stories/epics/EPIC-00-foundation.md) a [`EPIC-13-ai-agents-module.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/stories/epics/EPIC-13-ai-agents-module.md): Detalhamento em histórias de usuário de cada grande entrega do sistema (Auth, Onboarding, Inbox, Kanban, Customer 360, IA/RAG, Nuvemshop, LGPD, Equipes, Auditoria, Admin da Plataforma, Hardening de Segurança e Módulo de Agentes de IA).

---

## 5. Especificações Técnicas de Origem (`docs/specs/`)

Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\specs` (20 arquivos)  
Especificações técnicas detalhadas contendo contratos, tabelas e payloads originais:

| Arquivo | Domínio Técnico | Destaque para o Engenheiro de Produto |
| :--- | :--- | :--- |
| [`01-spec-platform-base.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/01-spec-platform-base.md) | Plataforma Base | Esquema de tabelas `organizations`, `users`, `audit_logs` e tokens de autenticação. |
| [`02-spec-customer-360.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/02-spec-customer-360.md) | Customer 360 | Estrutura de `contacts`, `contact_identities` e unificação determinística de telefones. |
| [`03-spec-whatsapp-waha.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/03-spec-whatsapp-waha.md) | WhatsApp WAHA | Payload de webhooks de entrada e filas de saída com controle de delay anti-ban. |
| [`04-spec-pipeline-attendance.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/04-spec-pipeline-attendance.md) | Funil e Atendimento | Schema de `crm_pipelines`, `crm_leads`, `conversations` e `messages`. |
| [`05-spec-ai-rag-handoff.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/05-spec-ai-rag-handoff.md) | IA e RAG | Tabelas de `knowledge_docs`, embeddings em pgvector e gatilhos de transição. |
| [`06-spec-nuvemshop-lgpd.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/06-spec-nuvemshop-lgpd.md) | E-commerce Nuvemshop | Mapeamento de produtos, pedidos, carrinhos abandonados e webhooks de privacidade. |
| [`07-spec-events-workers.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/07-spec-events-workers.md) | Workers e Eventos | Mecanismo de claim com `FOR UPDATE SKIP LOCKED`, tabelas de fila durável e métricas. |
| [`08-spec-deploy-observability.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/08-spec-deploy-observability.md) | Observabilidade | Configuração de Sentry, OpenTelemetry, logs estruturados Pino e métricas de saúde. |
| [`09-spec-frontend-backend-integration.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/09-spec-frontend-backend-integration.md) | Integração Front/Back | Padrão de respostas com wrappers `ok()` e `fail()`, tratamento de erros e Zod. |
| [`10-spec-ai-agents-runtime.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/10-spec-ai-agents-runtime.md) | Runtime de IA | Ciclo de inferência com timeouts, fallback de modelos e stream de respostas. |
| [`11-spec-mcp-server-internal.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/11-spec-mcp-server-internal.md) | MCP Server Interno | Catálogo de tools do Model Context Protocol expostas para o agente de IA. |
| [`12-spec-ai-agents-ui.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/12-spec-ai-agents-ui.md) | UI dos Agentes de IA | Componentes visuais do painel de configuração de agentes, prompts e testes. |
| [`13-spec-governanca-atendimento.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/13-spec-governanca-atendimento.md) | Governança G1–G6 | Portais de qualidade e regras de negócio para transição de atendimento. |
| [`14-contrato-governanca-agentes-externos.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/14-contrato-governanca-agentes-externos.md) | Agentes Externos | Contrato de API para que agentes de IA externos ou webhooks operem no CRM. |
| [`15-spec-casos-humanos.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/15-spec-casos-humanos.md) | Casos Humanos | Especificação da fila de transbordo e alarme quando a IA solicita ajuda humana. |
| [`16-spec-tres-papeis-do-agente.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/16-spec-tres-papeis-do-agente.md) | Três Papéis do Agente | Divisão comportamental da IA: Conversador (empático), Operador (ações) e Segurança (guardrails). |
| [`17-spec-conversa-vira-lead.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/17-spec-conversa-vira-lead.md) | Conversa Vira Lead | Automação que detecta intenção de compra no chat e abre card automaticamente no Kanban. |
| [`17-spec-indice-de-atrito.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/17-spec-indice-de-atrito.md) | Índice de Atrito | Métrica proprietária que quantifica a fricção e atrasos na jornada do cliente. |
| [`pre-go-live-whatsapp.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/pre-go-live-whatsapp.md) | Modo Pré-Go-Live | Ambiente de testes seguro no WhatsApp antes de liberar o número para clientes reais. |
| [`RECONCILIATION-LOG.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/specs/RECONCILIATION-LOG.md) | Log de Reconciliação | Histórico de ajustes de divergências encontradas entre specs e implementação. |

---

## 6. Operação Prática, VPS, Deploy e Incidentes (`docs/runbooks/`)

Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\runbooks`  
Guias passo a passo voltados a operadores e engenheiros para resolver problemas reais em produção:

* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\runbooks\deploy.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\docs\runbooks\deploy.md):  
  **Deploy em Produção (VPS):** O runbook mais crítico do sistema. Explica a obrigatoriedade dos **dois arquivos `-f`** no comando Docker (`-f docker-compose.prod.yml -f docker-compose.traefik.yml`). Omitir o segundo `-f` faz o Traefik perder o contêiner e derruba o domínio inteiro em erro 404 genérico com contêiner saudável.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\runbooks\remediar-worker-congelado.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\docs\runbooks\remediar-worker-congelado.md):  
  **Remediação de Worker Travado:** Diagnóstico com `diagnostico.sh`, identificação de locks órfãos na tabela de jobs e roteiro seguro de reinicialização do contêiner worker sem perder mensagens.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\runbooks\custo-e-cota-do-supabase.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\docs\runbooks\custo-e-cota-do-supabase.md):  
  **Custo e Cota do Supabase:** Como investigar crescimento excessivo de disco, tabelas que mais consom espaço (`event_log` e `messages`) e configuração de retenção e expurgo automático.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\runbooks\cloudpanel.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\docs\runbooks\cloudpanel.md):  
  **VPS com CloudPanel ou Nginx:** Como instalar o DeskcommCRM em servidores que já utilizam as portas 80/443 para hospedar outros sites.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\runbooks\waha-hostgator.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\docs\runbooks\waha-hostgator.md):  
  **Operação do WAHA na HostGator:** Monitoramento da estabilidade do contêiner WhatsApp, resolução de desconexões de sessão e atualização do engine NOWEB.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\runbooks\ai-credentials-rotation.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\docs\runbooks\ai-credentials-rotation.md):  
  **Rotação de Chaves de IA:** Procedimento seguro de substituição de chaves da OpenAI, Anthropic e Google sem downtime e sem falhas em mensagens em voo.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\runbooks\ativar-packaging.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\docs\runbooks\ativar-packaging.md):  
  **Ativação da Doutrina de Packaging:** Passos para publicação de imagens oficiais no GitHub Container Registry (GHCR).
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\runbooks\vercel-hobby-relogio.md`](file:///D:/Users/Jorge/Projetos-atu\DeskcommCRM\docs\runbooks\vercel-hobby-relogio.md):  
  **Limitações de Cron na Vercel Hobby:** Como configurar crons alternativos quando o plano serverless possui restrições de tempo de execução.

### Guias Complementares de Deploy
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\deploy-selfhost\README.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/deploy-selfhost/README.md): Passo a passo para subir o sistema em qualquer servidor Linux limpo.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\deploy-hostgator\README.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/deploy-hostgator/README.md): Especificidades, limites de memória e comandos otimizados para VPS da HostGator.

---

## 7. Doutrina de Engenharia e Sistema Vivo (`docs/doctrine/`)

Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\doctrine` (14 arquivos)  
A doutrina representa os mandamentos não-negociáveis de estabilidade do DeskcommCRM:

* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\doctrine\sistema-vivo.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/doctrine/sistema-vivo.md):  
  **Doutrina do Sistema Vivo (A LEI):** Os 7 invariantes fundamentais:
  1. *O sistema não mente sobre seu estado.*
  2. *Toda transição crítica emite evento durável.*
  3. *Toda ação destrutiva requer confirmação de autoridade.*
  4. *A unidade de demanda tem dono e prazo.*
  5. *O relógio do sistema é soberano.*
  6. *O sistema tolera falhas parciais sem derrubar a operação.*
  7. *Toda trava de emergência possui caminho de desativação.*
* [`docs/doctrine/sistema-vivo/`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/doctrine/sistema-vivo):  
  8 capítulos aprofundados (`01-fundamentos.md` a `08-aplicacao.md`) detalhando como aplicar os princípios a novas features.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\doctrine\packaging.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/doctrine/packaging.md):  
  **Doutrina de Packaging:** Regra de ouro: a VPS do cliente nunca constrói código (`docker compose build` é proibido em produção; a máquina apenas baixa imagens seladas do GHCR).
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\doctrine\restricao-de-canal.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/doctrine/restricao-de-canal.md):  
  Tratamento de políticas de canais externos (como Meta/WhatsApp). O sistema deve respeitar limites de taxa e janelas de 24 horas por design.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\doctrine\separacao-fala-e-operacao.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/doctrine/separacao-fala-e-operacao.md):  
  Separação de vocabulário: IDs de banco, payloads e jargões internos do CRM nunca podem ser expostos ao cliente final no WhatsApp.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\doctrine\versionamento.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/doctrine/versionamento.md):  
  Regras de SemVer e alinhamento de versões entre tags Git e `CHANGELOG.md`.
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\adr\0001-packaging-e-distribuicao.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/adr/0001-packaging-e-distribuicao.md):  
  Decisão de arquitetura sobre a criação das três imagens Docker publicadas no GHCR (`app`, `worker`, `scheduler`).

---

## 8. Design System, UX e Fluxos de Navegação (`docs/design-system/`)

Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\design-system` (21 arquivos)

### 8.1. Guias Visuais e Tokens
* [`00-overview.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/00-overview.md): Visão geral dos 5 pilares visuais lockados da plataforma.
* [`01-foundation-tokens.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/01-foundation-tokens.md): Variáveis CSS, espaçamentos e raios de borda.
* [`02-palette-sage.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/02-palette-sage.md): Paleta de cores oficial (Sage suave) para redução de fadiga visual do vendedor.
* [`03-typography.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/03-typography.md): Tipografia oficial (*Atkinson Hyperlegible* para legibilidade máxima e *IBM Plex Mono* para dados).
* [`04-density-aerada.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/04-density-aerada.md): Escala de densidade de tela aerada para evitar sensação de aperto em listas e Kanbans.
* [`05-iconography-phosphor.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/05-iconography-phosphor.md): Biblioteca oficial de ícones Phosphor (estilo Duotone).
* [`06-components.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/06-components.md): Catálogo de componentes de interface (Cards de Lead, Bolhas de Chat, Modais, Badges de Status).
* [`07-motion-language.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/07-motion-language.md): Micro-animações e transições sutis de tela.
* [`08-voice-and-tone.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/08-voice-and-tone.md): Tom de voz dos textos da interface (direto, profissional e prestativo).
* [`09-anti-patterns.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/09-anti-patterns.md): O que **NUNCA** fazer na interface (cores berrantes, botões confusos, modais aninhados).

### 8.2. Jornadas e Arquitetura de Telas (`screen-flow/`)
Subdiretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\design-system\screen-flow`
* [`00-personas.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow/00-personas.md): Personas de usuário (Vendedor ágil, Supervisor focado em métricas, Operador de TI).
* [`01-sitemap.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow/01-sitemap.md): Mapa completo de rotas e navegação da interface do usuário.
* [`02-journeys.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow/02-journeys.md): Jornadas de ponta a ponta do usuário no aplicativo.
* [`03-screen-inventory.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow/03-screen-inventory.md): Inventário detalhado de cada tela com seus respectivos componentes visuais.
* [`04-clickflows.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow/04-clickflows.md): Fluxo de cliques detalhado para ações comuns (arrastar card, enviar áudio, aplicar tag).
* [`05-state-machines.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow/05-state-machines.md): Como a UI se comporta durante estados assíncronos (carregando, sucesso, erro).
* [`06-empty-states-and-errors.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow/06-empty-states-and-errors.md): Telas de lista vazia e ilustrações explicativas com botões de ação ("Call to Action").
* [`07-responsive-strategy.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow/07-responsive-strategy.md): Estratégia de responsividade e adaptação para tablets e smartphones.
* [`08-accessibility.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design-system/screen-flow/08-accessibility.md): Padrões de acessibilidade WCAG 2.1 AA (contraste de cores, navegação por teclado e leitor de tela).

### 8.3. Especificações Específicas de Design (`docs/design/`)
* [`D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\design\teto-de-orcamento.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design/teto-de-orcamento.md) e [`onda-7-alarme-de-orcamento.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/design/onda-7-alarme-de-orcamento.md): Especificações visuais para o componente de alarme e barra de progresso de consumo de orçamento de IA.

---

## 9. Mapas de Arquitetura Viva (`docs/architecture/`)

Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\architecture` (23 arquivos)  
Mapas formais de relações de runtime testados por testes unitários (`mapas-de-arquitetura.test.ts`):

* [`agent-turn.html`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/agent-turn.html) e [`agent-turn.workflow.json`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/agent-turn.workflow.json): Diagrama interativo navegável em navegador do turno completo do agente de IA.
* [`teto-de-orcamento.architecture.json`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/teto-de-orcamento.architecture.json): Mapa vivo de proteção orçamentária: quem alimenta a medição de tokens, como a trava de parada é acionada e como o operador reativa o serviço.
* [`pre-go-live-whatsapp.architecture.json`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/pre-go-live-whatsapp.architecture.json): Mapa do modo de testes do WhatsApp com whitelist de números autorizados.
* [`marca-propria.architecture.json`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/marca-propria.architecture.json): Mapa de resolução de marcas e temas por tenant sem quebra de páginas.
* [`crm-vivo.architecture.json`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/crm-vivo.architecture.json): Mapeamento de sinais vitais, cálculo de score e reatividade comercial.
* [`ponte-agendamento-followup.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/ponte-agendamento-followup.md): Arquitetura de integração entre a marcação de compromissos na agenda e o motor de follow-up.
* Demais mapas: [`acervo-de-conhecimento`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/acervo-de-conhecimento.architecture.json), [`agenda-google-sync`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/agenda-google-sync.architecture.json), [`atualizacao-self-service`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/atualizacao-self-service.architecture.json), [`central-avisos`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/central-avisos.architecture.json), [`encerramento-atendimento`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/encerramento-atendimento.architecture.json), [`escalacao-ciclo-humano`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/escalacao-ciclo-humano.architecture.json), [`followup-dossie`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/followup-dossie.architecture.json), [`gestao-funis`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/gestao-funis.architecture.json), [`ia-360-organizar`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/ia-360-organizar.architecture.json), [`ia-360-retencao`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/ia-360-retencao.architecture.json), [`indice-de-atrito`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/indice-de-atrito.architecture.json), [`interface-por-vinculo`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/interface-por-vinculo.architecture.json), [`organizacoes-e-acesso`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/organizacoes-e-acesso.architecture.json), [`retencao-de-historico`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/retencao-de-historico.architecture.json), [`roteamento-por-canal`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/architecture/roteamento-por-canal.architecture.json).

---

## 10. Histórico de Ondas e Handoffs de Engenharia

### 10.1. Planos e Specs de Ondas (`docs/superpowers/`)
Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\superpowers` (39 arquivos)  
Planos datados de entrega de funcionalidades que documentam o racional original de cada feature:
* **Planos de Ondas de Mídia e WhatsApp (`plans/`):**
  * `2026-07-21-onda0-fundacao-midia.md` e `onda1-midia-na-ui.md`: Infraestrutura de envio e visualização de mídias (fotos/áudios).
  * `2026-07-21-onda2-composer-whatsapp.md`: Caixa de composição de mensagens estilo WhatsApp Web.
  * `2026-07-22-onda3-agente-multimodal.md`: Capacidade do agente de IA processar imagens e áudios enviados pelo lead.
  * `2026-07-22-onda4-split-mensagens.md`: Algoritmo que quebra respostas longas da IA em mensagens menores simulando digitação humana natural.
  * `2026-07-22-onda5-templates-vendedor.md`, `onda51-rascunho-ia.md`, `onda52-notas-internas.md` e `onda53-snooze-lembrete.md`: Superpoderes do vendedor humano (respostas prontas, notas internas invisíveis para o cliente e adiamento de cards com lembretes).
* **Planos de Canais e Funil (`plans/`):**
  * `2026-07-27-canais-seam-fases-0-2.md`, `2026-07-28-canais-fase-3a-templates.md` e `2026-07-28-canais-fase-4-janela.md`: Arquitetura de canais de comunicação.
  * `2026-07-27-gerenciar-etapas-do-funil.md` e `2026-08-03-gestao-funis.md`: Gestão dinâmica de etapas do pipeline comercial.
* **Harness e Evolução do Agente (`plans/`):**
  * `2026-07-23-harness-fase0-convergencia.md` a `fase4-painel-evolucao.md`: Evolução da memória da organização e painel de aprendizado da IA.
* **Specs de Implementação (`specs/`):**
  * Specs datadas de implementação técnica correspondentes às ondas listadas acima.

### 10.2. Arquivo de Handoffs de Ciclos (`docs/handoffs/` e `docs/handoff/`)
Diretórios: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\handoffs` e `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\handoff` (18 arquivos)  
Relatórios formais de encerramento de frentes com inventário do que foi entregue, testes executados e débitos remanescentes:
* `HANDOFF-crm-vivo.md` e `BRIEFING-crm-vivo.md`: Entrega do CRM reativo e sinais vitais do cliente.
* `HANDOFF-ia-360.md`, `BRIEFING-ia-360.md` e `PR-ia-360.md`: Entrega da visão consolidada de IA 360.
* `HANDOFF-casos-humanos.md`: Entrega da fila de transbordo para atendentes.
* `HANDOFF-inbox-multimodal.md`: Entrega do suporte a áudios, imagens e mídias no chat.
* `HANDOFF-indice-de-atrito.md`: Entrega das métricas de fricção de jornada.
* `HANDOFF-lgpd.md`: Entrega dos controles de privacidade e relatórios de titulares.
* `HANDOFF-canais-oficial.md`: Oficialização da integração de canais.
* `HANDOFF-provedores-de-ia.md`: Suporte multi-provedor (OpenAI, Anthropic, Google).
* `waves/W1-painel-do-humano.md` a `W4-organizar-a-operacao.md`: Marcos de entrega das 4 ondas de experiência do usuário.
* `agente-pausado-assume-conversa.md`: Regra operacional de retomada de conversa pelo operador humano.

---

## 11. Garantia de Qualidade, Testes e Auditorias

### 11.1. Estratégia de Testes (`docs/testing/`)
Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\testing`
* [`user-journey-map.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/testing/user-journey-map.md):  
  **Mapa de Jornadas Vivo:** Cataloga todas as jornadas de usuário do produto, categorizadas por prioridade crítica (`[P0]`, `[P1]`, `[P2]`), descrevendo o comportamento esperado e as falhas a serem vigiadas em testes E2E.
* [`HANDOFF-vps-qa.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/testing/HANDOFF-vps-qa.md):  
  Instruções para provisionamento de uma VPS espelho de homologação para testes manuais e exploratórios com banco semeado idêntico a produção.
* [`followup-reactivity-experimento-pareado.csv`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/testing/followup-reactivity-experimento-pareado.csv):  
  Dados brutos de experimentos pareados de tempo de reação e cancelamento de follow-ups automáticos.

### 11.2. Auditorias Históricas (`docs/audits/`)
Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\audits`
* [`2026-08-14-afirmacoes-de-estado.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/audits/2026-08-14-afirmacoes-de-estado.md):  
  Auditoria rigorosa de 393 declarações documentais confrontadas diretamente contra comandos de código, identificando discrepâncias entre o que a documentação antiga afirmava e o que o código faz.
* [`2026-08-14-alinhamento-stable-v1.3.0.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/audits/2026-08-14-alinhamento-stable-v1.3.0.md):  
  Auditoria do conteúdo exato entregue na release de produção `v1.3.0`.

### 11.3. Evidências Visuais (`docs/evidence/`)
Diretório: `D:\Users\Jorge\Projetos-atu\DeskcommCRM\docs\evidence` (18 arquivos de imagem PNG)  
Capturas de tela oficiais atestando a entrega visual de fluxos críticos:
* `casos-humanos/` (6 imagens): Painel de alerta, modal de assunção de conversa e visualização de fila de espera.
* `inbox-multimodal/` (6 imagens): Player de áudio integrado, visualizador de imagens e cards de anexos.
* Raiz de `evidence/` (6 imagens): Telas de onboarding, Kanban e configurações gerais.

---

## 12. Materiais de Apoio Institucional, Growth e Pesquisa

* **Marca e Identidade (`docs/brand/`):** [`README.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/brand/README.md), [`og-card.html`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/brand/og-card.html) e assets visuais de pré-visualização social de links.
* **Aquisição e Growth (`docs/growth/`):** [`lp-plano.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/growth/lp-plano.md) (estratégia de landing page), [`lp-prompts-imagens.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/growth/lp-prompts-imagens.md), [`submissoes-diretorios.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/growth/submissoes-diretorios.md) e metadados para o repositório [`awesome-selfhosted-deskcommcrm.yml`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/growth/awesome-selfhosted-deskcommcrm.yml).
* **Apresentações Comerciais (`docs/presentation/`):** [`pitch-deck.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/presentation/pitch-deck.md) (Pitch institucional para clientes e parceiros) e [`HANDOFF.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/presentation/HANDOFF.md).
* **Pesquisa Tecnológica (`docs/research/`):** [`architecture-diagrams.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/research/architecture-diagrams.md), [`followup-reference-mining.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/research/followup-reference-mining.md) (benchmarking de motores de cadência) e [`reference-synthesis.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/research/reference-synthesis.md) (aprendizados do ecossistema WAHA).
* **Decola AI (`docs/decola-ai/`):** [`prompt-sdr-inicial.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/decola-ai/prompt-sdr-inicial.md) (Engenharia de prompt base para qualificação de vendas).
* **Diagramas Visuais de Autenticação (`docs/diagrams/`):** [`auth-flows.html`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/diagrams/auth-flows.html) e [`auth-flows.sequence.json`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/diagrams/auth-flows.sequence.json).
* **Notas de Lançamento (`docs/release/`):** [`teto-de-orcamento.md`](file:///D:/Users/Jorge/Projetos-atu/DeskcommCRM/docs/release/teto-de-orcamento.md) (Nota orientativa para donos de VPS sobre como ativar o limite financeiro de IA).
