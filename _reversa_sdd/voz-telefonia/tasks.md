# Voz / Telefonia — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas: `voice_calls` (unificada), `org_voice_calls`, `voip_trunk_settings`, `phone_numbers`
- [ ] Serviço externo WaCalls (binário Go) acessível; env `WACALLS_API_BASE_URL`, `WACALLS_API_TOKEN`
- [ ] Asterisk com ARI e dialplan (`voice-agent-out`, `from-trunk-audiosocket`); env `ARI_*`, `AUDIOSOCKET_PORT`
- [ ] OpenAI Realtime configurada (agente de voz); RPC `fn_resolve_inbound_number`

## Tarefas
- [ ] T-01, Implementar opt-in e guarda de rota de voz
  - Origem no legado: `lib/voice/opt-in.ts`, `voice/guarda.ts`
  - Critério de pronto: `&&` de duas perguntas; três códigos; falha fechada; desligar isento
  - Confiança: 🟢
- [ ] T-02, Implementar cliente WaCalls e pareamento
  - Origem no legado: `lib/wacalls/client.ts`, `nome-da-sessao.ts`, `session.ts`
  - Critério de pronto: sempre Bearer; `createSession` pareia; nunca `/pair`; nome = uuid inteiro
  - Confiança: 🟢
- [ ] T-03, Implementar desparear e encerramento
  - Origem no legado: `lib/voice/desparear.ts`, `wacalls/calls.ts`
  - Critério de pronto: logout→delete→banco; tolera 404; idempotente; `podeEncerrar` só quem está na linha
  - Confiança: 🟢
- [ ] T-04, Implementar número discável e conversão de áudio WebRTC
  - Origem no legado: `lib/voice/numero-discavel.ts`, `wacalls/pcm.ts`
  - Critério de pronto: consulta diretório com Promise.race 4s; falha aberta; PCM16 LE ↔ float32
  - Confiança: 🟢
- [ ] T-05, Implementar a ponte de eventos WaCalls SSE → Postgres
  - Origem no legado: `lib/wacalls/events-bridge.ts`, `motivo-da-chamada.ts`
  - Critério de pronto: backoff; inferência de sentido; upsert idempotente; silenciar/devolver IA; 3 tipos de atividade
  - Confiança: 🟢
- [ ] T-06, Implementar canal SIP: ARI, trunk, caller, μ-law
  - Origem no legado: `lib/voip/ariClient.ts`, `guardar-trunk.ts`, `resolve-caller.ts`, `ulaw.ts`, `call-vocabulary.ts`
  - Critério de pronto: originate ao dialplan; trunk cifrado; caller anti-duplicação; μ-law bit-exato
  - Confiança: 🟢
- [ ] T-07, Implementar a superfície HTTP e o worker de voz SIP
  - Origem no legado: `app/api/v1/calls/route.ts`, `workers/voice-agent/index.ts`
  - Critério de pronto: `requireSupportWrite` antes do RBAC; channelId antes do originate; AudioSocket + Realtime; `mapStatusParaApi`
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Opt-in: motivo correto quando instalação não oferece
- [ ] TT-02, Desparear falha → linha não arquivada; 404 tolerado
- [ ] TT-03, Inferência de sentido no call-status
- [ ] TT-04, Chamada perdida vira inbox; feita não atendida não vira
- [ ] TT-05, μ-law ↔ PCM16 bit-exato

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, Migrar/unificar `voice_calls` (wacalls+sip) discriminado por `provider` (migration 0348)

## Ordem Sugerida
1. T-01 (opt-in/guarda) → T-02/T-03 (parear/desparear) → T-04.
2. T-05 (ponte de eventos) depende de T-02.
3. T-06/T-07 (SIP) são independentes do WaCalls e podem ir em paralelo.

## Lacunas Pendentes (🔴)
- Topologia Asterisk (`pjsip.conf`/`extensions.conf`, `fn_resolve_inbound_number`) — validar com quem administra a VPS.
- `audioSocketBridge.ts` precisa de escavação linha a linha antes de reimplementar o áudio SIP.
- Ponte WebRTC humano→navegador do SIP está ausente ("esqueleto").
