# CRM e Funil — Fluxos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Fluxo 1 — Nascimento do lead até o score
```mermaid
flowchart TD
  A[conversa recebe mensagem] --> B[nascimento-do-lead: idempotente por contato]
  B --> C{cliente pela agenda?}
  C -->|sim| D[funil is_client_pipeline]
  C -->|não| E[funil is_default]
  D & E --> F[fn_nascer_lead_da_conversa: advisory lock org+contato]
  F --> G[emit lead_created]
  G --> H[score-writer: lê sinais, classifyRisk, calculaScore]
  H --> I{score null?}
  I -->|sim| J[apaga crm_lead_scores]
  I -->|não| K[grava score citável + reason]
```

## Fluxo 2 — Movimentação de card com trava otimista
```mermaid
flowchart TD
  A[IA/humano move card] --> B[update .eq stage_id=origem .select id]
  B --> C{0 linhas?}
  C -->|sim| D[conflito_humano]
  C -->|não| E{negócio open?}
  E -->|não| F[lead_fechado]
  E -->|sim| G[stage_changed + emit_event lead.stage_changed]
```

## Fluxo 3 — Esfriamento e reativação (risk-worker)
```mermaid
flowchart TD
  A[risk-worker tick] --> B[classifyRisk por estágio]
  B --> C{bucket mudou?}
  C -->|não| D[não escreve, board não pisca]
  C -->|sim| E[narra de→para na timeline]
  E --> F[entrou em em_risco/critico → lead_cooled]
  F --> G[cria proposta de reativação no mesmo tick]
```

## Fluxo 4 — Reporte de conversão ao anúncio
```mermaid
flowchart TD
  A[lead.won OU lead.stage_changed] --> B[envio.handler: re-lê crm_leads]
  B --> C{tem atribuição de anúncio?}
  C -->|não| D[venda orgânica: não reporta]
  C -->|sim| E{Purchase tem valor+moeda?}
  E -->|não| F[pendência: tela existe para isso]
  E -->|sim| G[conversion_ledger idempotente <leadId>:Purchase]
```

## Fluxo 5 — Prospecção fria (a única linha que fala primeiro)
```mermaid
flowchart TD
  A[tickProspecting] --> B[guardas: janela, pacing, warmup frio, daily_limit, teto 50/24h]
  B --> C[pré-go-live, serviceBoundary, autorização IA]
  C --> D[recordSend: reserva cota ANTES do envio]
  D --> E[gerarAbordagemDeFormulario + rodapé de saída no idioma da org]
  E --> F[envio via messages/_handler]
  F --> G[audita prospecting.approach_sent sem PII]
```
🟢
