# Fluxogramas — Unidade 4: Voz / Telefonia

> Gerado pelo Arqueólogo (Reversa) — nível **detalhado**
> Cobre os dois canais de voz: WaCalls (WhatsApp, humano/WebRTC) e SIP (Asterisk, IA/AudioSocket).
> Escala: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

---

## 4.0 Visão geral — dois canais, uma tabela 🟢

```mermaid
flowchart TB
  subgraph WACALLS["Canal WaCalls (WhatsApp · humano · WebRTC)"]
    direction TB
    W1[Navegador do atendente] -->|SDP relay| W2[Rotas app/api/v1/voice/*]
    W2 -->|Bearer| W3[Serviço WaCalls externo]
    W3 -.->|SSE /api/events| W4[Worker: runVoiceCallsBridgeLoop]
    W4 --> DB[(voice_calls provider=wacalls)]
  end
  subgraph SIP["Canal SIP (Asterisk · IA · AudioSocket)"]
    direction TB
    S1[POST /api/v1/calls] -->|originate| S2[Asterisk ARI]
    S2 -.->|StasisStart / WS| S3[Worker voice-agent]
    S2 ==>|AudioSocket TCP| S3
    S3 <-->|PCM16 / u-law| S4[OpenAI Realtime]
    S3 --> DB2[(voice_calls provider=sip)]
  end
  DB --> API[GET /api/v1/calls* e telas]
  DB2 --> API
```

---

## 4.1 Opt-in: a regra das duas perguntas 🟢 — `voice/opt-in.ts`

```mermaid
flowchart TD
  Start([estadoDaVoz]) --> Q1{instalacaoOferece?<br/>WACALLS_API_BASE_URL != ''}
  Q1 -->|não| M1[motivo = instalacao_nao_oferece<br/>ligada = false]
  Q1 -->|sim| Q2{escolhaDaOrg ?? false<br/>org_voice_calls.enabled}
  Q2 -->|null / false| M2[motivo = organizacao_nao_ligou<br/>ligada = false]
  Q2 -->|true| M3[motivo = ligada<br/>ligada = true]
  M1 --> End([EstadoDaVoz])
  M2 --> End
  M3 --> End
```

> A combinação é `instalacaoOferece && (escolhaDaOrg ?? false)` — nunca `??`. Ausência de linha = off.

---

## 4.2 Guarda de rota 🟢 — `voice/guarda.ts::exigirVozLigada`

```mermaid
flowchart TD
  Start([exigirVozLigada]) --> Read[lerEscolhaDaOrg]
  Read -->|throw leitura| E503A[fail 503 voice_estado_indeterminado]
  Read -->|ok| Est[estadoDaVoz escolha, instalacaoOferece]
  Est --> Lig{ligada?}
  Lig -->|sim| Pass[return null — segue]
  Lig -->|não instalacao_nao_oferece| E503B[fail 503 voice_indisponivel_na_instalacao]
  Lig -->|não organizacao_nao_ligou| E422[fail 422 voice_desligada_na_organizacao]
```

> Falha **fechada** (erro vira recusa). Rotas de **desligar** NÃO passam por esta guarda.

---

## 4.3 Desparear 🟢 — `voice/desparear.ts::despareaVoz` (ordem é regra)

```mermaid
flowchart TD
  Start([despareaVoz]) --> Q[SELECT channel_sessions<br/>provider=wacalls, archived_at null]
  Q -->|sem linha| Nada[return desapareado=false<br/>idempotente, sucesso]
  Q -->|linha| HasSid{wacalls_session_id?}
  HasSid -->|sim| L[wacalls.logoutSession]
  L --> D[wacalls.deleteSession]
  HasSid -->|não| Arq
  L -.->|erro != 404| Throw[[THROW — banco NÃO muda]]
  D -.->|erro != 404| Throw
  L -->|wacalls_404| D
  D -->|wacalls_404 tolerado| Arq
  D --> Arq[UPDATE archived_at=now, status=STOPPED,<br/>wacalls_session_id=null, wacalls_paired_at=null]
  Arq --> OK[return desapareado=true]
```

> `logout` → `delete` → banco. Banco só muda depois que o WaCalls confirmou. `404` é tolerado.

---

## 4.4 Número discável — grafia certa do celular 🟢 — `voice/numero-discavel.ts`

```mermaid
flowchart TD
  Start([resolverNumeroDiscavel]) --> Cad[doCadastro = digitos do cadastro, fonte=cadastro]
  Cad --> Sess[SELECT channel_sessions WORKING,<br/>PROVIDERS_DE_MENSAGEM, limit 10]
  Sess -->|vazio| RetCad[return doCadastro]
  Sess -->|>=1| Race{Promise.race}
  Race --> Proc[procurar: por sessão<br/>adapter.resolveRegisteredPhone]
  Race --> Timer[setTimeout PRAZO_DA_CONSULTA_MS = 4000]
  Proc -->|1a resposta != null| Wa[return digitos, fonte=whatsapp]
  Proc -->|todas null| RetCad
  Timer -->|estourou| RetCad
```

> Falha **ABERTA**: qualquer falha/timeout → disca o cadastro (comportamento anterior).

---

## 4.5 Ponte de eventos WaCalls — despacho SSE 🟢 — `wacalls/events-bridge.ts::despacharEventoWacalls`

```mermaid
flowchart TD
  Start([linha SSE data:]) --> Parse{JSON.parse ok?}
  Parse -->|não| Ret[[ignora]]
  Parse -->|sim| T{ev.type}
  T -->|call-list| CL[snapshot: por chamada != ended<br/>-> handleCallStatus sentido DECLARADO]
  T -->|session-list| Ret
  T -->|incoming| IN[INSERT voice_calls inbound/ringing<br/>on conflict só corrige direction]
  T -->|auth-state| AS[handleAuthState]
  T -->|call-status| CS[handleCallStatus]
  T -->|call-ended| CE[handleCallEnded]
  T -->|outros| Ret
  CL --> Sess[resolveSession por wacalls_session_id]
  IN --> Sess
  AS --> Sess
  CS --> Sess
  CE --> Sess
  Sess -->|sessão de outra instalação| Ret
```

---

## 4.6 `handleCallStatus` — inferência de sentido + upsert 🟢

```mermaid
flowchart TD
  Start([handleCallStatus]) --> Peer[peerPhone = peerToPhone]
  Peer --> Dono[dono = donoValido owner<br/>só se forma de UUID]
  Dono --> Dir{direction declarado?<br/>inbound/outbound no evento}
  Dir -->|sim| Dd[direction = declarado — declarada=true]
  Dir -->|não| Di{dono existe?}
  Di -->|sim| Do[direction = outbound]
  Di -->|não| Ib[direction = inbound]
  Dd --> Ins
  Do --> Ins
  Ib --> Ins
  Ins[INSERT voice_calls<br/>contact por 2 grafias<br/>answered_at se status=connected] --> Conf{conflito<br/>org+wacalls_call_id?}
  Conf -->|sim| Upd[UPDATE: status;<br/>direction só se declarada=true;<br/>contact_id/owner via coalesce;<br/>answered_at se connected]
  Conf -->|não| Novo[linha nova]
  Upd --> Cal
  Novo --> Cal
  Cal{status=connected<br/>e contato?}
  Cal -->|sim| Silencio[calarIaDuranteALigacao<br/>bot_silenced_until = now+2h<br/>NUNCA encurta]
  Cal -->|não| End([fim])
  Silencio --> End
```

---

## 4.7 `handleCallEnded` — fechamento e efeitos 🟢

```mermaid
flowchart TD
  Start([handleCallEnded]) --> Upd[UPDATE voice_calls status=ended,<br/>end_reason, ended_at, duration_ms,<br/>owner via coalesce]
  Upd -->|sem linha| Warn[[log warn — retorna]]
  Upd -->|linha| Voz{tem contato?}
  Voz -->|sim| Dev[devolverAVozDaIa<br/>só onde last_handoff_reason = MOTIVO_DO_SILENCIO]
  Voz -->|não| Perd
  Dev --> Perd{!atendida e recebida?}
  Perd -->|sim| Inbox[INSERT agent_inbox_items<br/>voice_call_missed / warn]
  Perd -->|não| Log
  Inbox --> Log[INSERT event_log voice_call.ended status=done]
  Log --> Ativ{tem contato?}
  Ativ -->|sim| Emit[emitAgentActivityForContact<br/>tipo = atendida? voice_call :<br/>recebida? voice_call_missed :<br/>voice_call_unanswered<br/>usuarioId = owner_user_id]
  Ativ -->|não| End([fim])
  Emit --> End
```

> `voice_call` quebra o silêncio do negócio; `voice_call_missed`/`voice_call_unanswered` **não**.

---

## 4.8 `handleAuthState` — pareamento / desvínculo 🟢

```mermaid
flowchart TD
  Start([handleAuthState]) --> P{ev.paired?}
  P -->|false| S{state = logged_out?}
  S -->|não ex: qr| Noop1[[no-op — QR em curso]]
  S -->|sim| Unlink[UPDATE wacalls_paired_at=null,<br/>status=STOPPED<br/>WHERE wacalls_paired_at is not null]
  Unlink -->|rowCount>0| WarnLog[log warn: aparelho desvinculado]
  P -->|true| G[UPDATE status=WORKING,<br/>wacalls_paired_at = coalesce antes, now<br/>WHERE antes is null OR status != WORKING]
  G -->|primeiro=true| L1[log: sessão pareada]
  G -->|rows>0| L2[log: voltou ao ar]
  G -->|0 linhas heartbeat| Noop2[[no-op]]
```

---

## 4.9 Canal SIP — chamada de SAÍDA 🟢 — `POST /api/v1/calls` → `originateCall`

```mermaid
flowchart TD
  Start([POST /api/v1/calls]) --> Sup[requireSupportWrite<br/>acompanhamento só-leitura não disca]
  Sup -->|negado| E403[[Response de negação]]
  Sup -->|ok| Role[requireRole manager]
  Role -->|falha| E401[[authz.response]]
  Role -->|ok| Val[valida createCallSchema]
  Val --> Trunk[SELECT voip_trunk_settings ativo<br/>fallback VOIP_TRUNK_ENDPOINT]
  Trunk -->|nenhum| E422[fail 422 trunk_not_configured]
  Trunk -->|ok| Cid[channelId = randomUUID]
  Cid --> Ins[INSERT voice_calls sip/ringing<br/>asterisk_channel_id = channelId ANTES do originate]
  Ins -->|erro| E500[fail 500]
  Ins --> Orig[originateCall<br/>endpoint PJSIP/dest@endpoint, context voice-agent-out]
  Orig -->|ok| A[audit call.created] --> R201[ok 201 callId, channelId]
  Orig -->|erro| Fail[UPDATE status=ended/failed] --> E502[fail 502 originate_failed]
```

> `channelId` gerado ANTES fecha a race com o StasisStart (processo separado, via WebSocket).

---

## 4.10 Canal SIP — chamada de ENTRADA + AudioSocket 🟢 — `workers/voice-agent/index.ts`

```mermaid
flowchart TD
  subgraph STASIS["Stasis (só passagem, entrada)"]
    SS([StasisStart]) --> RN[resolveInboundNumber<br/>exten 's' -> único número ativo,<br/>senão fn_resolve_inbound_number]
    RN -->|não mapeado| HU[hangupChannel]
    RN -->|ok| Cont[resolveOrCreateCallerContact<br/>+ garantirLeadDaConversa best-effort]
    Cont --> UUID[gera UUID separado p/ AudioSocket<br/>channel.id nativo não é UUID]
    UUID --> InsIn[INSERT voice_calls sip/inbound/ringing]
    InsIn --> SetVar[setChannelVariable AUDIOSOCKET_UUID]
    SetVar --> ContD[continueDialplan from-trunk-audiosocket]
  end
  subgraph AS["AudioSocket TCP porta 9092 — entrada E saída convergem"]
    Conn([conexão TCP]) --> Frame{1o frame tipo 0x01 UUID?}
    Frame -->|não| Endc[socket.end]
    Frame -->|sim| Find[SELECT voice_calls por asterisk_channel_id]
    Find -->|não achou| Endc
    Find --> Agent[getActiveVoiceAgent]
    Agent -->|nenhum| Endc
    Agent --> RAG[resolverAcervoDoAgente em paralelo]
    RAG --> Bridge[AudioSocketCallBridge -> OpenAI Realtime<br/>u-law <-> PCM16]
    Bridge --> UpdConn[UPDATE status=connected, answered_at, handled_by=ai]
    UpdConn -.->|fim| Fin[finalizeAudioSocketCall<br/>status=ended, duration_ms, transcript]
  end
  ContD -.->|dialplan chama AudioSocket| Conn
  Dest([ChannelDestroyed]) --> HCD[handleChannelDestroyed<br/>fecha ringing preso -> ended/timeout<br/>só linha ainda ringing]
```

---

## 4.11 Loop de vida da ponte SSE 🟢 — `runVoiceCallsBridgeLoop`

```mermaid
flowchart TD
  Start([runVoiceCallsBridgeLoop]) --> Loop{signal abortado?}
  Loop -->|sim| Exit([sai])
  Loop -->|não| Fetch[fetch /api/events Bearer]
  Fetch -->|ok+body| Reset[backoff = 1000] --> Pump[pumpSse: linhas data:<br/>-> despacharEventoWacalls]
  Fetch -->|erro/!ok| Log[log erro]
  Pump -->|stream caiu| Wait
  Log --> Wait[espera backoff ou abort]
  Wait --> Back[backoff = min backoff*2, maxBackoffMs] --> Loop
```
