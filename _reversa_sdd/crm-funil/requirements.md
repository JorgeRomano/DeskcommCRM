# CRM e Funil (`leads`, `pipelines`, `kanban`, `contacts`, `tags`, `conversoes`, `prospecting`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 5).

## Visão Geral
O núcleo do "sistema vivo" (doutrina anti-morte): nenhuma demanda aberta fica sem próximo passo nem sem desfecho. Cobre o ciclo de vida do negócio (nascimento → funil → score/risco → encerramento → reativação), o Kanban, contatos (dedup, rótulo, CPF), etiquetas, reporte de conversão às plataformas de anúncio e a esteira de prospecção fria. Quase toda a lógica é pura e testável sem banco; as escritas ficam na borda. O vocabulário do funil (`lead/deal/won/lost`) é renomeável por org (multi-nicho). 🟢

Distinção central: no harness da IA "lead" = **contato** (pessoa); no CRM "lead" = **negócio** (`crm_leads`). Uma pessoa pode ter vários negócios; `active-lead.ts` faz a ponte. 🟢

## Responsabilidades
- Calcular score citável e classificar risco por estágio (Risk Radar). 🟢
- Gerir a máquina de estados do funil (edição de funil/etapa, movimentação de card). 🟢
- Gerir o ciclo de vida (nascimento, encerramento, motivo de perda, reativação, clone). 🟢
- Registrar atividade/timeline com vocabulário fechado. 🟢
- Gerir Kanban (card-state, dono, local echo, filtros). 🟢
- Deduplicar contatos (union-find), rótulo e CPF. 🟢
- Reportar conversão de venda ao anúncio (handler de evento idempotente). 🟢
- Rodar a esteira de prospecção fria com ritmo próprio. 🟢

## Regras de Negócio
- Score exige lastro citável (`checkpointId` + fator com âncora); sem lastro → `score=null`. — `leads/score-formula.ts` 🟢
- Score `null` apaga `crm_lead_scores` (null é melhor que número velho). — `leads/score-writer.ts` 🟢
- Janela de risco é por estágio (`expected_duration_hours`; fallback 24h/72h). — `leads/risk-radar.ts` 🟢
- Histerese de faixa: sobe/desce um degrau por vez, zona morta 5. — `kanban/score-band.ts` 🟢
- Funil sem etapa de ganho não fecha (`/win` → `pipeline_no_won_stage`). — `pipelines/pipeline-editing.ts` 🟢
- `pipeline_id` é imutável (P-01); mover cross-pipeline = clonar + encerrar origem. — `leads/clonar-para-funil.ts` 🟢
- Nascimento idempotente por contato (um lead por DEMANDA); advisory lock por org+contato. — `leads/nascimento-do-lead.ts` 🟢
- Encerramento: `motivo` obrigatório em lost; status/closed_at vêm do trigger. — `leads/encerramento.ts` 🟢
- `moved_to_another_pipeline` é canônico e excluído das métricas; nunca ofertado na tela. — `leads/motivo-da-perda.ts` 🟢
- Movimentadores de card usam trava otimista (`.eq("stage_id", origem)` → 0 linhas = `conflito_humano`). 🟢
- Conversão só grava com atribuição de anúncio; Meta fora do ar nunca bloqueia a venda. — `conversoes/envio.handler.ts` 🟢
- Prospecção fria ~4× mais lenta que a resposta; teto falha fechado (knobs incompletos → teto 1). — `prospecting/ritmo-da-esteira-fria.ts` 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Score citável ou nulo | Must | Sem lastro → `score=null`, `semSinal` preenchido |
| RF-02 | Classificar risco por estágio | Must | Janela por `expected_duration_hours`; buckets `critico/em_risco/em_voo/em_dia` |
| RF-03 | Máquina de estados do funil | Must | Transição inválida rejeitada; funil sem won não fecha |
| RF-04 | `pipeline_id` imutável | Must | Mover cross-pipeline clona + encerra origem |
| RF-05 | Nascimento idempotente | Must | 3 mensagens juntas geram 1 lead |
| RF-06 | Movimentação com trava otimista | Must | Conflito humano → `conflito_humano` |
| RF-07 | Conversão idempotente por atribuição | Should | Só grava com atribuição; `Purchase` exige valor+moeda |
| RF-08 | Prospecção fria com ritmo próprio | Should | ~4× mais lenta; teto fecha em falha; jitter só atrasa |
| RF-09 | Dedup de contatos union-find | Should | A~B, B~C → agrupa os 3; anonimizado sai |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | CPF por hash (sha256) para dedup sem plaintext | `contacts/cpf.ts` | 🟢 |
| Segurança | Prospecção: reserva cota antes do envio; falha fechada | `prospecting/worker.ts` | 🟢 |
| Escalabilidade | Radar em lotes (`SCAN_CAP=500`, `IDS_POR_CONSULTA=100`) | `leads/radar-de-risco.ts` | 🟢 |
| Disponibilidade | Workers de score/risco só escrevem quando muda (evita piscar o board) | `leads/risk-worker.ts`, `score-writer.ts` | 🟢 |
| Escalabilidade | Indexação fracionária no Kanban com rebalance | `kanban/fractional-indexing.ts` | 🟢 |
| Acessibilidade | Paleta de etiquetas testada contra réguas de contraste OKLab | `tags/cor-da-etiqueta.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado um lead novo sem checkpoint
Quando o score é calculado
Então score=null com semSinal=sem_lastro_citavel

Dado um card arrastado por um humano ao mesmo tempo que a IA
Quando a IA tenta mover com trava otimista
Então 0 linhas afetadas → conflito_humano

Dado uma venda com atribuição de anúncio
Quando lead.won é emitido
Então conversion_ledger grava Purchase idempotente (<leadId>:Purchase)
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Score/risco (RF-01/02) | Must | Coração do sistema vivo |
| Máquina do funil (RF-03/04) | Must | Integridade do negócio |
| Nascimento/movimentação (RF-05/06) | Must | Ciclo de vida sem lixo |
| Conversão (RF-07) | Should | Reporte externo, não bloqueia venda |
| Prospecção fria (RF-08) | Should | Canal proativo com anti-ban |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/leads/score-formula.ts` | `calculaScore` | 🟢 |
| `lib/leads/risk-radar.ts` | `classifyRisk`, `compareRisk` | 🟢 |
| `lib/pipelines/pipeline-editing.ts` | `validarArquivamento`, `updatesDeMarcaExclusiva` | 🟢 |
| `lib/leads/nascimento-do-lead.ts` | nascimento idempotente | 🟢 |
| `lib/leads/clonar-para-funil.ts` | clone cross-pipeline | 🟢 |
| `lib/kanban/card-state.ts` | `resolveCardState` | 🟢 |
| `lib/contacts/duplicados.ts` | union-find de dedup | 🟢 |
| `lib/conversoes/envio.handler.ts` | handler de conversão | 🟢 |
| `lib/prospecting/worker.ts` | `sendNextCandidate`, `tickProspecting` | 🟢 |
