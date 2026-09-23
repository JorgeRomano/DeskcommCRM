# Voz / Telefonia (`voice`, `voip`, `wacalls`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 4).

## Visão Geral
Dois canais de voz independentes que gravam na mesma tabela `voice_calls` (unificada na migration 0348, discriminada pela coluna `provider`), com pilhas técnicas distintas: 🟢
1. **`wacalls`** — chamada de voz por WhatsApp (segundo aparelho vinculado via serviço externo WaCalls, áudio por WebRTC direto navegador↔WaCalls, CRM só faz relay de SDP). Atendimento humano.
2. **`sip`** — telefonia PSTN/SIP via Asterisk (trunk SIP por org, áudio por AudioSocket TCP entre Asterisk e worker Node que faz ponte com OpenAI Realtime). Atendimento por IA.

`lib/voice/` (opt-in, guarda de rota, número discável, desparear) serve apenas o canal WaCalls; `lib/voip/` serve apenas o canal SIP.

## Responsabilidades
- Opt-in de voz WhatsApp (regra das duas perguntas: instalação oferece E org ligou). 🟢
- Guarda de rota de voz com três códigos honestos. 🟢
- Cliente REST autenticado do WaCalls e ponte de eventos SSE → Postgres. 🟢
- Resolver número discável na grafia que o WhatsApp reconhece. 🟢
- Canal SIP: cliente ARI, trunk cifrado, caller, conversão μ-law, worker de voz. 🟢
- Silenciar/devolver a voz da IA durante ligação. 🟢

## Regras de Negócio
- `chamadaDeVozLigada = instalacaoOferece && (escolhaDaOrg ?? false)` — dois eixos por `&&`, nunca `??` (senão ligaria para todas as orgs). — `voice/opt-in.ts` 🟢
- `escolhaDaOrg` null = nunca escolheu = desligado. 🟢
- Guarda de voz falha FECHADA (custo de errar = vincular aparelho sem consentimento); rotas de DESLIGAR não passam pela guarda. — `voice/guarda.ts` 🟢
- Nunca chamar `/pair` separado no WaCalls (quebra `s.calls`); re-parear = apagar e criar. — `wacalls/client.ts` 🟢
- Desparear: ordem `logout → delete → banco`; banco só muda após WaCalls confirmar; arquiva, não apaga; idempotente. — `voice/desparear.ts` 🟢
- `record` NUNCA `true` no `startCall` (LGPD). 🟢
- `podeEncerrar`: só quem está na linha desliga (dono, ou quem discou se ninguém assumiu). — `wacalls/calls.ts` 🟢
- Número discável falha ABERTA (disca o cadastro se a consulta falhar). — `voice/numero-discavel.ts` 🟢
- `calarIaDuranteALigacao`: silêncio de 2h (anti-morte), nunca encurta handoff durável. `devolverAVozDaIa` só desfaz o que esta ponte fez. 🟢
- Trunk SIP: senha AES-GCM, só `password_last4` em claro; obrigatória só na criação. — `voip/guardar-trunk.ts` 🟢
- SIP `originateCall` entrega direto ao dialplan (não Stasis app) por limitação do `externalMedia`. — `voip/ariClient.ts` 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Opt-in por duas perguntas | Must | Voz só liga se instalação oferece E org ligou |
| RF-02 | Guarda de rota com três códigos | Must | 422 org, 503 instalação, 503 indeterminado; rotas de desligar isentas |
| RF-03 | Parear via criar sessão, nunca `/pair` | Must | Re-parear passa por apagar+criar |
| RF-04 | Desparear de verdade (logout→delete→banco) | Must | Banco só muda após WaCalls confirmar; idempotente; tolera 404 |
| RF-05 | Número discável na grafia certa | Should | Consulta diretório do canal; falha aberta disca o cadastro |
| RF-06 | Ponte de eventos WaCalls SSE → Postgres | Must | Reconexão com backoff; inferência de sentido; upsert idempotente |
| RF-07 | Autorização de encerramento | Must | Só quem está na linha desliga |
| RF-08 | Canal SIP: originate + AudioSocket + Realtime | Must | Chamada SIP nasce `ringing`, conecta via AudioSocket, IA atende |
| RF-09 | Silenciar/devolver voz da IA | Must | Silêncio 2h anti-morte; devolver só desfaz o próprio |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | Cliente WaCalls sempre com Bearer; sem token não funciona | `wacalls/client.ts` | 🟢 |
| Segurança | Senha do trunk SIP cifrada AES-GCM | `voip/guardar-trunk.ts` | 🟢 |
| Segurança | `record` nunca true (LGPD) | `wacalls/client.ts` | 🟢 |
| Segurança | `donoValido` só aceita UUID (evita quebrar FK auth.users) | `wacalls/events-bridge.ts` | 🟢 |
| Disponibilidade | Backoff exponencial na SSE; erro em evento não derruba stream | `wacalls/events-bridge.ts` | 🟢 |
| Disponibilidade | Silêncio de IA com teto 2h (anti-morte) | `wacalls/events-bridge.ts` | 🟢 |
| Performance | `PRAZO_DA_CONSULTA_MS=4000` via Promise.race (evita >30s) | `voice/numero-discavel.ts` | 🟢 |
| Performance | RAG em paralelo com WS OpenAI no worker SIP | `workers/voice-agent/index.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado que a instalação não oferece voz
Quando estadoDaVoz avalia
Então motivo = instalacao_nao_oferece (escolha da org é irrelevante)

Dado um desparear com WaCalls indisponível
Quando desparear executa
Então lança e a linha NÃO é arquivada (a tela não mente)

Dado uma chamada recebida não atendida
Quando handleCallEnded fecha
Então cria agent_inbox_items kind=voice_call_missed

Dado uma chamada SIP de entrada
Quando o AudioSocket recebe o UUID
Então resolve voice_calls, sobe OpenAI Realtime e grava handled_by=ai
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Opt-in e guarda (RF-01/02) | Must | Consentimento e capacidade |
| Parear/desparear corretos (RF-03/04) | Must | Vínculo de aparelho sem lixo |
| Ponte de eventos (RF-06) | Must | Coração do canal WaCalls |
| Canal SIP (RF-08) | Must | Atendimento por IA |
| Número discável (RF-05) | Should | Confiabilidade da discagem |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/voice/opt-in.ts` | `chamadaDeVozLigada`, `estadoDaVoz` | 🟢 |
| `lib/voice/guarda.ts` | `exigirVozLigada`, `lerEscolhaDaOrg` | 🟢 |
| `lib/voice/desparear.ts` | desparear (logout→delete→banco) | 🟢 |
| `lib/voice/numero-discavel.ts` | `resolverNumeroDiscavel` | 🟢 |
| `lib/wacalls/client.ts` | `WacallsClient`, `getWacallsClient` | 🟢 |
| `lib/wacalls/events-bridge.ts` | `runVoiceCallsBridgeLoop`, `handleCallStatus`, `handleCallEnded` | 🟢 |
| `lib/wacalls/calls.ts` | `podeEncerrar`, `resolveVoiceCall` | 🟢 |
| `lib/voip/ariClient.ts` | `originateCall`, `connectAriEvents` | 🟢 |
| `lib/voip/guardar-trunk.ts` | `guardarTrunk` | 🟢 |
| `lib/voip/ulaw.ts` | `pcm16ToUlaw`, `ulawToPcm16` | 🟢 |
| `workers/voice-agent/index.ts` | worker de voz SIP | 🟢 |
