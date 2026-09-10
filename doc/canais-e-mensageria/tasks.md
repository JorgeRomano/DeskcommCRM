# Canais e Mensageria — Tarefas de Implementação

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Pré-requisitos
- [ ] WAHA rodando (engine NOWEB) com `WHATSAPP_HOOK_EVENTS` incluindo `message.any`
- [ ] `WAHA_API_KEY` (plaintext no app; `sha512:` no container) + `WAHA_HMAC_SECRET`
- [ ] RPCs `fn_upsert_wa_contact`, `fn_upsert_wa_conversation`, `fn_mark_conversation_message`
- [ ] Trigger `trg_messages_emit_event` → `message.received`

## Tarefas

- [ ] T-01, Implementar verificação HMAC-SHA512 do webhook (fail-closed)
  - Origem no legado: `lib/waha/ingest.ts:verifyHmacSha512`
  - Critério de pronto: HMAC inválido não persiste; `timingSafeEqual`
  - Confiança: 🟢

- [ ] T-02, Implementar parse de identidade WhatsApp (anti-ReDoS)
  - Origem no legado: `lib/waha/ingest.ts:parseChatId`, `semSufixoDeChat`, `telefoneAlternativoDe`
  - Critério de pronto: phone/lid/group/unknown corretos; sem corte por regex
  - Confiança: 🟢

- [ ] T-03, Implementar resolução atômica + persistência de mensagem
  - Origem no legado: `lib/waha/ingest.ts:handleInbound`, `WA_TYPE_MAP`
  - Critício de pronto: `message` e `message.any` não duplicam (unique org+external_id)
  - Confiança: 🟢

- [ ] T-04, Implementar efeitos pós-entrada na ordem opt-out→lead→despacho
  - Origem no legado: `lib/channels/pos-entrada.ts`
  - Critério de pronto: `tests/unit/pos-entrada-*` verdes; cada passo falha para dentro
  - Confiança: 🟢

- [ ] T-05, Implementar cliente WAHA (envio texto/mídia/presença) com teto de relógio
  - Origem no legado: `lib/waha/client.ts`, `lib/waha/send.ts`
  - Critério de pronto: erro expõe só status; `lid:` antes do telefone; teto 15s/30s
  - Confiança: 🟢

- [ ] T-06, Implementar estado da janela de 24h
  - Origem no legado: `lib/channels/janela.ts`, `guardrails/messaging-window.ts`
  - Critério de pronto: derivado a cada leitura; sem_restricao/aberta/fechada
  - Confiança: 🟢

- [ ] T-07, Implementar fronteira de atendimento (trava otimista)
  - Origem no legado: `lib/atendimento/fronteira.ts`
  - Critério de pronto: `service_revision` avança só ao tocar terminal/trocar demanda
  - Confiança: 🟢

- [ ] T-08, Implementar pausa de IA por atendimento manual pelo canal
  - Origem no legado: `lib/escalacao/atendimento-manual.ts`
  - Critério de pronto: 60min, renova, nunca encurta silêncio maior
  - Confiança: 🟢

## Tarefas de Teste

- [ ] TT-01, Happy path: webhook válido → mensagem persistida + `message.received`
- [ ] TT-02, HMAC inválido não persiste
- [ ] TT-03, Ordem dos efeitos pós-entrada (opt-out antes do lead)
- [ ] TT-04, Endereçamento @lid vence telefone
- [ ] TT-05, Eco de envio próprio não silencia a IA

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Modelo de identidade canônica `wa_identity`/`wa_lid` (migrations 0027/0122)

## Ordem Sugerida
1. T-01/T-02/T-03 (ingestão) são base.
2. T-04 (efeitos) depende de `leads` e `opt-out`.
3. T-05/T-06 (envio/janela) alimentam o agente e o atendente.
4. T-07/T-08 (fronteira/pausa) transversais.

## Lacunas Pendentes (🔴)
- SQL das RPCs de upsert.
- `lib/messaging/*` e `lib/inbox/*` — ler antes de reimplementar mídia e comandos da conversa.
