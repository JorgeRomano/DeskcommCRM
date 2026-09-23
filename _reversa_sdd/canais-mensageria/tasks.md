# Canais e Mensageria — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas: `channel_sessions`, `channel_session_health`, `conversations`, `messages`, `contacts`
- [ ] RPCs: `fn_upsert_wa_contact`, `fn_upsert_wa_conversation`; trigger `trg_messages_emit_event`
- [ ] Env: `WAHA_*`, `WAHA_WEBHOOK_REQUIRE_SIGNATURE`, `VAPID_PUBLIC_KEY/PRIVATE_KEY`, `SMTP_*`
- [ ] Gate `pnpm lint:channels` funcionando

## Tarefas
- [ ] T-01, Implementar a abstração de canal (tipos, capabilities, adapters)
  - Origem no legado: `lib/channels/types.ts`, `capabilities.ts`, `index.ts`, `adapters/*`
  - Critério de pronto: fail-closed em provider desconhecido; matriz de capabilities por provider; adapter puro
  - Confiança: 🟢
- [ ] T-02, Implementar autenticação e ingest de webhook WAHA
  - Origem no legado: `lib/waha/webhook-auth.ts`, `ingest.ts`, `envelope.ts`, `client.ts`, `send.ts`, `message-id.ts`
  - Critério de pronto: fail-closed; guarda ReDoS/timestamp; upsert atômico; idempotência 23505; não emite evento
  - Confiança: 🟢
- [ ] T-03, Implementar efeitos pós-entrada em ordem
  - Origem no legado: `lib/channels/pos-entrada.ts`
  - Critério de pronto: opt-out primeiro; nenhum efeito lança; ordem guardada por teste
  - Confiança: 🟢
- [ ] T-04, Implementar janela 24h, saúde e estado
  - Origem no legado: `lib/channels/janela.ts`, `health.ts`, `estado.ts`, `canal-mudo.ts`
  - Critério de pronto: janela derivada por leitura; alertas por status; só quem observou fecha episódio
  - Confiança: 🟢
- [ ] T-05, Implementar inbound Zernio/social e conexão/pairing
  - Origem no legado: `lib/channels/inbound.ts`, `pairing-code.ts`, `nome-da-sessao.ts`
  - Critério de pronto: assinatura por canal fail-closed; pairing rate-limited; renomear só sessão órfã
  - Confiança: 🟢
- [ ] T-06, Implementar comando da conversa e fronteira de atendimento
  - Origem no legado: `lib/inbox/comando-da-conversa.ts`, `lib/atendimento/fronteira-server.ts`
  - Critério de pronto: `comandoDaConversa` com precedência de motivo; `assertCurrentServiceBoundary` (CAS/revision)
  - Confiança: 🟢
- [ ] T-07, Implementar notificações e e-mail
  - Origem no legado: `lib/notifications/*`, `lib/email/roteador.ts`, `config.ts`
  - Critério de pronto: push 404/410 deleta subscription; roteador por configuração; erro não achatado
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Provider desconhecido lança fail-closed
- [ ] TT-02, Webhook com assinatura errada → bad_signature
- [ ] TT-03, message + message.any não duplicam (idempotência)
- [ ] TT-04, Guarda ReDoS: 1MB de chatId processa linear
- [ ] TT-05, Efeitos pós-entrada: opt-out antes de abrirDemanda
- [ ] TT-06, comandoDaConversa: bloqueado > travado > silêncio

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Migrar identidade canônica `wa_identity`/`wa_lid` (migrations 0027/0122)

## Ordem Sugerida
1. T-01 (abstração) → T-02 (ingest) → T-03 (pós-entrada).
2. T-04/T-05 (janela/saúde/conexão) em paralelo.
3. T-06 (comando/fronteira) e T-07 (notificações/e-mail) por último.

## Lacunas Pendentes (🔴)
- Confirmar defesa de rede (Caddy) para o default de assinatura WAHA.
- Ler linha a linha `channels/{meta,social,zernio}/` e `messaging/media/*` antes de reimplementar SSRF/mídia.
