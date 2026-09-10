# IA e Agentes — Design Técnico

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Interface

### Tools do agente (`AGENT_TOOL_DEFS`, `lib/agent-engine/agent/inbound-turn.ts`) 🟢

| Tool | Entrada (Zod) | Efeito |
|------|---------------|--------|
| `get_lead_context` | `{}` | Relê contato + últimas mensagens |
| `send_message` | `{ body: string }` | Envia (passa pela cadeia before-send) |
| `update_lead_state` | `{ stage?, qualification?, next_action?, reason? }` | Marca avanço no funil (só próximo estágio válido) |
| `schedule_followup` | `{ reason, promised_at (ISO), promise, context_snapshot? }` | Agenda retorno |
| `save_lead_note` / `get_lead_note` | `{ headline, body, supersedes? }` / `{ note_id }` | Memória durável |
| `search_knowledge` | `{ query }` | RAG |
| `request_human_handoff` | `{ reason? }` | Passa a atendente |
| `open_human_case` / `provide_case_update` | `{ title, summary, blocker }` / `{ case_id, info }` | Caso de retaguarda |
| `send_template` | `{ template_name, language, values }` | Template Meta fora da janela 24h |
| `read_skill_reference` | `{ skill_name, ref_path }` | Lê reference de skill ativa |

> Validação real é whitelist `.strict()` — campo forjado vira erro de ensino, nunca exceção do SDK.

### Gates da cadeia `before-send` (v6) 🟢

| Símbolo | Veredito | Observação |
|---------|----------|------------|
| `stopGate` → `disclosureGate` | `GateVerdict = {pass:true, waitMs?, amendBody?, skipped?} | {pass:false, code, reason, nextAllowedAt?, detail?}` | Ordem constante versionada |
| `decidePacing(input)` | `{allow, waitMs} | {allow:false, code, nextAllowedAt, reason}` | Puro, sem I/O |
| `decideSpinning(input)` | `{allow} | {allow:false, code:'mass_identical', matchCount, reason}` | Jaccard de tokens |

### Handoff (`triggerHandoff`) 🟢

`triggerHandoff(input: TriggerHandoffInput): Promise<TriggerHandoffResult>` — `reason ∈ {requested_human, low_sentiment, low_confidence, critical_stage, legal_mention, refund_mention, orcamento_de_ia}`. Nunca lança.

## Fluxo Principal (turno do agente) 🟢

1. Job `inbound_turn` reivindicado da fila (`agent-engine/queue`).
2. `comHandoffSeOrcamentoAcabar` envolve o turno inteiro.
3. Lê playbook (system, por ponteiro) + `lead_checkpoints` + `lead_state` + últimas N mensagens (`get_lead_context`).
4. Loop de tools do modelo; cada `send_message` passa pela cadeia before-send (advisory lock por número).
5. Gate vetou → razão volta ao modelo → reescreve.
6. Fecha com 2ª chamada `purpose='checkpoint'` → JSON validado por Zod → `lead_checkpoints`.

## Fluxos Alternativos 🟢

- **Triagem síncrona:** G1 (regex humano), G4 legal, G4 stage disparam handoff ANTES de invocar o modelo (custo zero, determinístico).
- **G3 baixa confiança:** persiste rascunho mas NÃO despacha; dispara handoff `low_confidence`.
- **Sem agente publicado:** cai nos workers legados (`ai-response-worker`).
- **Orçamento estourado:** devolve à fila humana; `operator_turn` não tem handoff (perde só o `registrarDesfecho` daquele turno).

## Dependências 🟢

- `event-log` (drain → jobs; emissão de eventos).
- `channels`/`waha` (envio via `ChannelAdapter`/`WahaChannelAdapter`).
- `leads` (score, stage-sync, active-lead, checkpoint-diff).
- `atendimento` (ServiceBoundary — revalida antes de efeito).
- `supabase` (admin/pg — service role).
- Provedores de IA via Vercel AI SDK (resolvido por painel de provedores, `gateway-binding`).

## Decisões de Design Identificadas 🟢

| Decisão | Evidência no código | Confiança |
|---------|---------------------|-----------|
| Enviar é sempre tool call | `AGENT_TOOL_DEFS.send_message` | 🟢 |
| Cadeia versionada + teste de shape | `BEFORE_SEND_GATES`, `before-send-chain-shape.test.ts` | 🟢 |
| Modelo resolvido por provider (não id literal) | `resolverModeloDoPonto` (sentiment + response) | 🟢 |
| Advisory lock por número | `pg_advisory_xact_lock(hashtext(channel_session_id))` | 🟢 |
| Números anti-ban fonte única | `pacing/defaults.ts` + `lint-pacing.ts` | 🟢 |

## Estado Interno 🟢

- `job_queue`: status/lease/attempts por job.
- `lead_checkpoints` (rolling summary + compromissos/objeções/next_action), `lead_state` (BANT), `lead_notes` (durável).
- `ai_chunks` (RAG versionado por `ai_knowledge_sources`).
- `flywheel_judge_verdicts` / `flywheel_distiller_proposals`.

## Observabilidade 🟢

- `before_send_traces` (gate + veredito + código por tentativa).
- `llm_calls` (purpose, provider, model, tokens, cost_cents, latency).
- `agent_inbox_items` (Central: handoff, budget, jailbreak, conhecimento não indexado).
- `/healthz` e `/metrics` no worker.

## Riscos e Lacunas

- 🟡 `WahaChannelAdapter` (implementação concreta do envio) não lida em detalhe.
- 🔴 Sequência exata do laço de tools e fail-safes em `inbound-turn.ts` (>600 linhas) só parcialmente lida.
- 🟡 `lib/notifications/` (push) referenciado, não aprofundado.
