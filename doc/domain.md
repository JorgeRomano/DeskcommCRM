# Domínio e Regras de Negócio — DeskcommCRM

> Gerado pelo **Detetive** (Reversa) em 2026-09-10 · Nível: **Detalhado**
> Escala de confiança: 🟢 CONFIRMADO (código) · 🟡 INFERIDO · 🔴 LACUNA
>
> **Fontes cruzadas:** código-fonte (`lib/`, `workers/`), doutrina escrita do projeto
> (`docs/doctrine/`, `docs/business-rules/00-business-rules-catalog.md`) e arqueologia Git
> (3834 commits, abr–set/2026; 1392 `fix`, 891 `feat`, 106 `revert`).
> Quando a regra já está escrita no repo, a referência é citada — **nada aqui foi inventado**.

---

## 1. Propósito e princípio-raiz 🟢

DeskcommCRM se define como um **sistema vivo**, não um CRUD com telas (`docs/doctrine/sistema-vivo.md`):

> Chegou uma demanda — um lead interessado ou um usuário com um problema — e o sistema é responsável pela **linha do tempo inteira** dessa demanda até a resolução ou o encerramento declarado pelo próprio lead. **Nada pode morrer por falta de resolução, resposta ou visibilidade.**

### Os 7 invariantes do Sistema Vivo (lei, cobrada pelo DoD/CI)

1. **Regra das 2 conexões** — toda peça tem ≥1 aresta de entrada e ≥1 de saída (nada é ilha).
2. **Continuidade IA↔humano** nas duas direções (payload de handoff, não conversa crua).
3. **Log universal e visível** — toda mutação vira atividade no banco E na tela.
4. **Nenhuma demanda sem próximo passo** — follow-up é o mecanismo anti-morte.
5. **Informação com propósito** — todo dado responde "e daí?".
6. **Toda configuração tem superfície** — tela de ver, tela de mudar, caminho visível de falha.
7. **Todo laço se fecha** — toda decisão automatizada tem retorno mensurável (parcial hoje; o flywheel fecha o laço do agente).

### A regra do tempo

> Observação em **tempo real** (direito do observador). Ação no **tempo apropriado** ao humano do outro lado.

Corolário da **interruptibilidade**: o sistema nunca deve ser mais rápido do que o humano consegue interromper. Enviar mensagem é irreversível — throttle, janela horária e warm-up são atrasos deliberados, não bugs.

---

## 2. Glossário de domínio 🟢

| Termo | Significado no sistema |
|---|---|
| **Organização (tenant)** | Cliente self-host isolado por RLS (`organization_id`). |
| **Contato** | A pessoa (WhatsApp/e-mail). Chave de reencontro por telefone/e-mail. |
| **Lead / Negócio / Demanda** | Uma oportunidade num pipeline. **Um por demanda, não por mensagem.** |
| **Pipeline (funil)** | Coleção ordenada de estágios. Um `is_default` por org. |
| **Estágio (etapa)** | Coluna do kanban; pode ser `is_won`/`is_lost`/`requires_human`; tem `agent_stage_hint`. |
| **Conversa** | Fio de mensagens de um contato num canal. Tem status e silêncio de bot. |
| **Atendimento (ServiceBoundary)** | Trava de continuidade: identidade capturada na origem, revalidada antes de efeito. |
| **Agente de IA** | Assistente publicado (`ai_agent_versions`) que atende conversas via tools. |
| **Playbook** | System prompt + regras do agente, resolvido por ponteiro. |
| **Follow-up (enrollment)** | Percurso de um contato por um grafo de reengajamento. |
| **Handoff** | Transição bot→humano (7 razões). |
| **Caso humano (human case)** | Pendência de retaguarda aberta pelo agente sem silenciar a conversa. |
| **Guardrail** | Gate determinístico que veta um envio e ensina o modelo. |
| **Pacing / Spinning** | Anti-ban: ritmo de envio / anti-template-idêntico. |
| **Flywheel** | Laço de aprendizado (judge + distiller) com gate humano. |
| **Marca (white-label)** | Identidade visual resolvida por camadas (org → instalação → env → padrão). |
| **Controlador (LGPD)** | O operador da VPS (`organizations.legal_name`), nomeado no PDF de LGPD. |
| **Platform admin** | Único papel cross-tenant (dono da instalação). |

---

## 3. Catálogo de regras de negócio (destiladas)

> O repo já mantém um catálogo formal em `docs/business-rules/00-business-rules-catalog.md`
> (notação GIVEN/WHEN/THEN, enforcement layer, override). Abaixo, as regras que a
> **escavação confirmou no código**, com o ID do catálogo quando existe.

### 3.1 Tenancy e isolamento (T) 🟢

- **T-01/T-02** — Toda tabela tenant-aware tem `organization_id` + RLS; service role (admin client) bypassa RLS e exige filtro manual de `organization_id` de fonte confiável (cookie/JWT/webhook secret/path token), **nunca do body**. Confirmado em `lib/supabase/admin.ts` e em todos os handlers/workers.
- **T-04** — `platform_admin` é o único papel cross-tenant; `is_platform_admin` bypassa o rank do tenant em `requireRole` (`lib/auth/require-role.ts:92`).
- **T-06** — Org nova nasce com pipeline default semeado (`fn_seed_default_pipeline_for_org`).
- **Role efetivo vem do banco**, nunca do snapshot do cookie (`fn_user_role_in_org`, a mesma função das RLS). 🟢

### 3.2 LGPD (L) 🟢

- **L-01** — Anonimização preferida sobre delete físico (`cascadeRedactContact`).
- **L-02/L-03** — SLA em dias úteis BR: data_request D+7, redact D+15 (`lib/lgpd/repository.ts:computeDueAt`). 🟡 (dias exatos inferidos do catálogo/comentários).
- **L-04** — Anonimização é irreversível (`403 lgpd_anonymization_irreversible`).
- **L-05** — Consentimento granular; `transactional` em resposta a inbound é dispensado (janela 24h).
- **L-08** — CPF/PII nunca em log (mascarado; `beforeSend` do Sentry + `lib/logger.ts`).
- **PDF de LGPD nomeia o CONTROLADOR (`legal_name`) e o DPO, nunca a marca** — decisão jurídica (`lib/lgpd/pdf-renderer.tsx`). 🟢
- **Redação enfileira o avatar em `storage_redaction_queue` ANTES de zerar o ponteiro** (fail-closed) — senão a imagem fica órfã. 🟢

### 3.3 WhatsApp / anti-ban (W) 🟢

- **W-01** — Throttle 1msg/1.2s + jitter ≤800ms por sessão (`PACING_DEFAULTS`).
- **W-02** — Pedido de descadastro INEQUÍVOCO bloqueia (`is_blocked`); ambíguo só escala. PT + ES (`lib/opt-out/deteccao.ts`). Deixou de ser regex de palavra solta em 2026-08-21 (12 falsos positivos medidos).
- **W-03** — Contato bloqueado nunca recebe outbound automatizado (gate `stopGate`, irrevogável).
- **W-04/IA-01** — Janela de 24h respeitada por automações; fora dela, só template aprovado (`messagingWindowGate`).
- **W-05** — Idempotência de webhook inbound via `unique(org, external_id)` (código 23505).
- **W-06** — Limite diário de número novo (warm-up por idade: 20→50→100→200→sem cap).
- **W-07** — Janela horária default 7h–22h, **domingo LIBERADO** (revisto pelo dono em 2026-08-20: janela é cortesia, não anti-ban).
- **W-09** — Mensagens em grupos (`@g.us`) não criam leads.
- **W-10** — Multi-device sync exige `message.any`; `fromMe` sem duplicar (dedup atômico via RPC).
- **W-12** — Cron marca `sending` >5min como `failed`; `queued` tem dono (agent-engine) e não entra.

### 3.4 Pipeline e lead (P) 🟢

- **P-01** — Lead vive em UM pipeline; mover entre pipelines não é suportado (clonar).
- **P-02** — `status` won/lost derivado de estágio com flag (trigger `fn_crm_lead_close_on_stage`); status/closed_at nunca escritos à mão.
- **P-03** — `lost_reason` obrigatório na perda (`encerraDemanda` recusa 422).
- **P-05** — Reorder por fractional indexing (`midpoint`, STEP=1000; NaN dispara rebalance).
- **Um lead por DEMANDA** (open) por contato; nova mensagem alimenta o existente (`garantirLeadDaConversa`).
- **Score é fórmula, não LLM**, com lastro citável obrigatório (`calculaScore`); faixa com histerese (`resolveBand`).
- **Radar de risco** classifica por janela de estágio (`classifyRisk`: critico/em_risco/em_voo/em_dia).

### 3.5 Atendimento e roteamento (AT) 🟢/🟡

- **AT-01** — Conversa e demanda têm ciclos distintos revisionados (`service_revision` avança ao tocar estado terminal ou trocar demanda). 🟢 (`lib/atendimento/fronteira.ts`).
- **AT-02** — "Eu cuido" é claim atômico (`UPDATE ... WHERE assigned_to IS NULL`). 🟡 (regra no catálogo; a UI não foi lida em profundidade).
- **AT-04** — Supervisor (manager+) lê conversa não atribuída em modo somente-leitura. 🟡.
- **AT-05** — Notas internas nunca vão pro WhatsApp. 🟡.
- **Pausa por atendimento manual pelo canal**: 60min, renova a cada fala, nunca encurta silêncio maior (`pausarIaPorAtendimentoManual`). 🟢.

### 3.6 IA e bot (IA) 🟢

- Cadeia `before-send` versionada v6 com 10 gates; veto volta ao modelo como erro instrutivo.
- Enviar é sempre tool call; texto direto do modelo é descartado.
- Handoff idempotente 5s; avisa o lead antes de silenciar; `bot_silenced_until='infinity'`.
- Promessa fora da tabela versionada é vetada (preço/desconto/parcela + texto livre semântico).
- Promessa-de-humano sem caso aberto é vetada (invariante sagrada).
- Orçamento de IA estourado devolve a conversa à fila humana (não descarta o job).

### 3.7 Restrição de canal (doutrina) 🟢

Fonte: `docs/doctrine/restricao-de-canal.md`. Dois eixos que não se generalizam:
- **Auto-restrição** (anti-ban): posso falar quando quiser, mas o canal me bane se abusar → pacing/spinning.
- **Hetero-restrição** (janela 24h da API oficial): a plataforma me proíbe e cobra → template fora da janela.
- **Invariante 1:** nenhuma feature nomeia o provider fora de `lib/channels/`.
- **Invariante 4:** restrição não aplicável é REGISTRADA (`skipped: not_applicable`), não omitida.

---

## 4. Máquinas de estado 🟢

Detalhadas em `state-machines.md`. Entidades com estado central:
- **`crm_leads.status`**: `open → won | lost` (reabertura volta a `open`).
- **`conversations.status`**: `open → pending/ai_handling/claimed → closed/resolved/archived`.
- **`followup_enrollments.status`**: `active/waiting_reply → paused_handoff/paused_manual → completed/cancelled/dead`.
- **`lgpd_requests.status`**: `received → processing → completed | failed | pending_review`.
- **`event_log.status`**: `pending → processing → done | dead`.
- **`lead_state.stage`** (funil do agente): `new → contacted → qualifying → qualified → negotiating → won | lost`.

---

## 5. Permissões (RBAC) 🟢

Detalhado em `permissions.md`. Papéis: `viewer(1) < agent(2) < ai_operator(3) < manager(4) < admin(5)` + `platform_admin` (cross-tenant). `ai_operator` só existe em token efêmero do agente.

---

## 6. Lacunas e pontos de validação humana 🔴

- 🔴 **SQL das RPCs e policies RLS** não lidos (só call sites) — o Data Master deve documentá-los a partir de `supabase/migrations/`.
- 🟡 **AT-02/AT-04/AT-05/AT-08** (claim atômico, supervisor read-only, notas internas, idle→offline): regras no catálogo, não confirmadas no código nesta escavação (UI/handlers de mensagens não lidos em profundidade).
- 🟡 **Dias exatos do SLA LGPD** (7/15): inferidos do catálogo; confirmar em `lib/lgpd/sla.ts`.
- 🔴 **Retenção de audit (5 anos, piso 90 dias)**: regra L-10 no catálogo; `fn_expurgar_auditoria_vencida` não lida.
- 🟡 **Notifications** (push VAPID): pipeline não lido em profundidade.
- 🔴 **Divergências catálogo↔código**: o próprio catálogo registra correções recentes (ex.: opt-out em ES já coberto; domingo liberado). Ao gerar specs, o código vigente vence a prosa.
