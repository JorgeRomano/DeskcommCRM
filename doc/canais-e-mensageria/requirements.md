# Canais e Mensageria

> Spec SDD (híbrida — módulo de topo) · Redator (Reversa) · Nível: Detalhado
> Módulos: `lib/waha/*`, `lib/channels/*`, `lib/messaging/*`, `lib/inbox/*`, `lib/atendimento/*`
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Visão Geral

Camada que conecta o WhatsApp (via WAHA) ao CRM: recebe webhooks, resolve identidade e persiste mensagens, aplica os efeitos de negócio pós-entrada (opt-out, nascimento do lead, despacho do agente), envia mensagens respeitando janela e restrição de canal, e mantém a fronteira de atendimento (continuidade). A restrição de canal é doutrina: nenhuma feature nomeia o provider fora de `lib/channels/`. 🟢

## Responsabilidades

- Verificar HMAC do webhook e parsear a identidade WhatsApp (phone/lid/group/unknown). 🟢
- Resolver contato/conversa atomicamente (RPC) e persistir a mensagem. 🟢
- Aplicar efeitos pós-entrada na ordem: opt-out → lead → despacho. 🟢
- Enviar mensagens ao WAHA com teto de relógio e endereço correto (lid antes de telefone). 🟢
- Expor o estado da janela de 24h a quem atende. 🟢
- Manter a `ServiceBoundary` (trava otimista de continuidade). 🟢
- Pausar a IA quando um humano responde por fora do CRM. 🟢

## Regras de Negócio

- Idempotência de webhook via `unique(org, external_id)` (23505 → 200 sem duplicar). 🟢 (W-05)
- Resolução atômica via RPC porque NOWEB emite `message` E `message.any` (corrida). 🟢
- Grupos (`@g.us`) não criam leads; unknown emite `whatsapp.chat_id_not_recognized`. 🟢 (W-09)
- `CONVERSAS_IGNORADAS` (status/broadcast/channels/groups) cortadas na fonte (WAHA). 🟢
- `lid:` vem antes do telefone no envio (não trocar canal de conversa @lid viva). 🟢
- Erro do WAHA expõe só o status HTTP, nunca o corpo (PII). 🟢
- A ordem opt-out→lead→despacho é regra de negócio, vigiada por teste; cada passo falha "para dentro". 🟢
- Janela de 24h: fora dela, só template aprovado (canais com hetero-restrição). 🟢 (W-04)
- Pausa por atendimento manual: 60min, renova a cada fala, nunca encurta silêncio maior. 🟢
- `service_revision` avança ao tocar estado terminal ou trocar de demanda. 🟢 (AT-01)

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Ingerir webhook WAHA (HMAC → parse → upsert → persist → efeitos) | Must | Dado webhook `message.any` válido, a mensagem é persistida e `message.received` emitido |
| RF-02 | Recusar webhook com HMAC inválido (fail-closed) | Must | Dado HMAC incorreto, retorna sem persistir |
| RF-03 | Aplicar efeitos pós-entrada na ordem definida | Must | Dado inbound de contato novo, opt-out roda antes do nascimento do lead |
| RF-04 | Enviar mensagem ao WAHA com endereço correto | Must | Dado contato @lid com telefone, o envio vai por `@lid` |
| RF-05 | Expor estado da janela de 24h | Should | Dada conversa em canal com janela, retorna aberta/fechada/sem_restricao |
| RF-06 | Pausar IA por atendimento manual pelo canal | Should | Dada resposta `fromMe` do operador, `bot_silenced_until` = agora+60min |
| RF-07 | Detectar eco de envio próprio vs digitação humana | Should | Dado eco na janela de 60s com mesmo corpo, não silencia a IA |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Performance | Teto de relógio nas chamadas WAHA (15s texto, 30s mídia) | `lib/waha/client.ts` | 🟢 |
| Segurança | HMAC-SHA512 com `timingSafeEqual`, fail-closed | `lib/waha/ingest.ts:verifyHmacSha512` | 🟢 |
| Segurança | Parse de identidade anti-ReDoS (não regex em campo de webhook) | `lib/waha/ingest.ts:semSufixoDeChat` | 🟢 |
| Disponibilidade | Ingestão nunca cai por efeito de negócio (falha para dentro) | `lib/channels/pos-entrada.ts` | 🟢 |
| Escalabilidade | Corta conversas ignoradas na fonte (economia medida ~376MB) | `lib/waha/client.ts:CONVERSAS_IGNORADAS` | 🟢 |

## Critérios de Aceitação

```gherkin
Dado um webhook WAHA message.any de um número 1-a-1 válido
Quando a ingestão processa
Então contato e conversa são resolvidos atomicamente, a mensagem é persistida e message.received é emitido

Dado um webhook com assinatura HMAC inválida e WAHA_WEBHOOK_REQUIRE_SIGNATURE=true
Quando a ingestão verifica
Então a requisição é recusada sem persistir (fail-closed)

Dado um contato @lid que ganhou telefone
Quando o agente envia
Então o endereço usado é <lid>@lid, não <telefone>@c.us
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Ingestão + persistência | Must | Sem ela nada entra no CRM |
| Ordem dos efeitos pós-entrada | Must | Regra de negócio; inverter cria card para quem pediu para sair |
| Endereçamento de envio | Must | Endereço errado para de responder o cliente |
| Janela de 24h | Should | Evita `failed 131047` da plataforma |
| Pausa por atendimento manual | Should | Impede IA responder por cima do humano |

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/waha/ingest.ts` | `parseChatId`, `verifyHmacSha512`, `handleInbound` | 🟢 |
| `lib/waha/client.ts` | `WahaClient`, `CONVERSAS_IGNORADAS` | 🟢 |
| `lib/waha/send.ts` | `resolveWahaChatId`, `sendWAHA` | 🟢 |
| `lib/channels/pos-entrada.ts` | `aplicarEfeitosPosEntrada` | 🟢 |
| `lib/channels/janela.ts` | `estadoDaJanela` | 🟢 |
| `lib/atendimento/fronteira.ts` | `assertCurrentServiceBoundary` | 🟢 |
| `lib/escalacao/atendimento-manual.ts` | `pausarIaPorAtendimentoManual` | 🟢 |
| `lib/messaging/*`, `lib/inbox/*` | — | 🟡 (não lidos em profundidade) |
