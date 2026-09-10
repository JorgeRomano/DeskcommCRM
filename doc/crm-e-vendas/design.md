# CRM e Vendas — Design Técnico

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Interface

### Funções principais 🟢

| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `garantirLeadDaConversa` | `(db, dados: DadosDoNascimento)` | `NascimentoDoLead` (`{criado,leadId,...}` \| `{criado:false,motivo}`) |
| `encerraDemanda` | `(supabase, ctx: HandlerCtx, input: EncerraDemandaInput)` | `DemandaEncerrada {lead, jaEstava}` |
| `criarEtapa` / `atualizarEtapa` / `arquivarEtapa` | `(deps: DepsDeEtapa, input)` | `EtapaCriada` / `{funil, updates}` / `EtapaArquivada` |
| `calculaScore` | `(sinais: SinaisDoLead)` | `ScoreCalculado {score:number\|null, reason, evidence, band, semSinal?}` |
| `classifyRisk` | `(input: RiskInput)` | `RiskResult {bucket, hoursSinceActivity, onRadar}` |
| `carregaRadarDeRisco` | `(admin, opts)` | `RadarDeRisco {items, counts, sem_proximo_passo}` |
| `sincronizaEstagioDoAgente` | `(admin, input)` | `ResultadoDaSincronizacao {moveu, motivo, ...}` |
| `resolveCardState` | `(input: CardInput, t?)` | `CardState {kind, border, slot, showStageAge}` |
| `resolveBand` | `(score, anterior: ScoreBand\|null)` | `ScoreBand` (`frio\|morno\|quente`) |
| `midpoint` | `(prev:number\|null, next:number\|null)` | `number` (NaN se prev===next) |

## Fluxo Principal (nascimento do lead) 🟢

1. Lê contato; `is_blocked` → `contato_bloqueado` (não cria).
2. Já há lead `open`? → `ja_existe`.
3. `funilDeEntrada`: `crm_pipelines.is_default` + `!is_archived`; etapa = menor `position` não-terminal.
4. Insere `crm_leads` (título via `rotuloDoContato`; `source` whatsapp/anúncio; tags de anúncio).
5. `emitLeadActivity` `lead_created` (fire-and-forget).

## Fluxo — score (fórmula) 🟢

`BASE 30` + `+12/compromisso` (teto 3) `−8/objeção` (teto 3) `+5/BANT` (teto 4) + ajuste de recência por bucket → clamp 0–100. Recusa: status≠open (`negocio_fechado`), <2 sinais (`sem_conteudo`), sem checkpoint OU nenhum fator com âncora (`sem_lastro_citavel`). `reason` derivado das parcelas; `band` via histerese.

## Fluxo — encerramento 🟢

Valida motivo (lost); lê lead (filtro org); idempotente se já no desfecho; busca estágio terminal (`is_won`/`is_lost`, menor position); UPDATE stage/position/lost_reason (status/closed_at vêm do trigger `fn_crm_lead_close_on_stage`); `emitLeadActivity` `demand_closed`; audit `lead.won`/`lead.lost`.

## Fluxos Alternativos 🟢

- **Sem funil/etapa:** `sem_funil_de_entrada`/`sem_etapa` (falha de configuração visível).
- **Move do agente:** `sem_mapeamento` (não inventa), `ja_esta_la`, `ambiguo` (2 leads → não move), `conflito_humano` (trava otimista), `fora_do_escopo`, `indisponivel` (banco fora ≠ sem_negocio).
- **Arquivar etapa/funil:** recusa se único/padrão/com webhook/com automação; com negócios exige destino/arquiva.
- **Conversão:** google_ads sem transporte → skip; `value_cents<=0` → `sem_valor`; evento >7d → permanente.

## Dependências 🟢

- `kanban` (`resolveCardState`, `score-band`, `fractional-indexing`).
- `pipelines` (edição de funil).
- `contacts` (`rotuloDoContato`, dedup).
- `agenda` (proteção de follow-up no radar).
- `conversoes` + `plataformas-de-anuncio` (reporte de venda).
- `event-log` (emite `lead.won`/`lead.stage_changed`), `audit`, `supabase`.

## Decisões de Design Identificadas 🟢

| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Um lead por demanda (não por mensagem) | `garantirLeadDaConversa` | 🟢 |
| Score é fórmula, não LLM (razão derivada) | `score-formula.ts` | 🟢 |
| Histerese na faixa (banda 5pt, degrau a degrau) | `score-band.ts` | 🟢 |
| Janela de esfriamento por estágio | `risk-radar.ts:resolveStageWindow` | 🟢 |
| Trava otimista no move do agente | `agent-stage-sync.ts` `.eq(stage_id).select()` | 🟢 |

## Estado Interno 🟢

`crm_leads` (status, stage_id, position_in_stage, score, tags, source_metadata), `crm_stages`/`crm_pipelines`, `crm_lead_activities` (timeline), `lead_checkpoints`/`lead_state` (fonte do score).

## Observabilidade 🟢

- `crm_lead_activities`: lead_created, stage_changed, demand_closed, handoff_triggered.
- `event_log`: lead.won/lost/stage_changed.
- Radar expõe contagens (critico/em_risco/em_voo) + demandas sem próximo passo.

## Riscos e Lacunas

- 🟡 `lib/contacts/*` (dedup, CPF, CSV) e `lib/conversoes/*` (leitura-da-atribuicao, registro-de-envio) não lidos em profundidade.
- 🔴 Triggers `fn_crm_lead_close_on_stage`, `fn_seed_default_pipeline_for_org` — SQL não lido.
