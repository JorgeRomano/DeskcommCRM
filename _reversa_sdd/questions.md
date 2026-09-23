# Perguntas para Validação — DeskcommCRM

> Gerado pelo Revisor (Reversa) em 2026-09-23 · `doc_level = detalhado` · `answer_mode = file`
> Preencha o campo **Resposta** de cada pergunta e me avise (digite `reversa` ou "respondi as perguntas").
> Estas são as lacunas 🔴 que **só você (ou quem administra a instalação) pode resolver**. As demais
> lacunas 🔴 são da camada de banco (RLS, RPCs SECURITY DEFINER, CHECKs, triggers) e estão delegadas
> ao agente **Data Master** — não são perguntas, são trabalho de escavação SQL pendente.

---

## Pergunta 1

**Contexto:** Unit `voz-telefonia` — topologia SIP/Asterisk. O código aplica `fn_resolve_inbound_number` e depende de `asterisk/pjsip.conf` e `extensions.conf` (migration 0349), que vivem **fora de `lib/`** e são aplicados manualmente por VPS.
**Spec afetada:** [`_reversa_sdd/voz-telefonia/design.md`](voz-telefonia/design.md) (Riscos e Lacunas) · [`_reversa_sdd/voz-telefonia/tasks.md`](voz-telefonia/tasks.md)
**Pergunta:** A topologia Asterisk (roteamento de número de entrada, dialplan, resolução `fn_resolve_inbound_number`) é parte do produto que o self-hoster recebe, ou é configuração manual de cada VPS? Existe um artefato versionado dessa configuração que eu deva referenciar?
**Impacto:** Define se a reimplementação da voz precisa reproduzir o dialplan (spec passa a 🟢/🟡 com fonte) ou se ele fica declarado como dependência de infraestrutura externa (permanece 🔴 documentado).

✅ Respondida — 🔴→🟢 (dependência externa; `voz-telefonia/design.md` e `tasks.md` atualizados)
**Resposta:** dependencia externa

---

## Pergunta 2

**Contexto:** Unit `voz-telefonia` — a ponte de áudio SIP (`workers/voice-agent/audioSocketBridge.ts`) foi lida só por referência nesta passagem, e a ponte WebRTC humano→navegador está descrita como "esqueleto" (`mode:'human'` grava o dono mas não abre áudio).
**Spec afetada:** [`_reversa_sdd/voz-telefonia/design.md`](voz-telefonia/design.md)
**Pergunta:** A ponte WebRTC humano→navegador do SIP está de fato incompleta (esqueleto) no legado, ou existe implementação que eu não localizei? Isso é comportamento pretendido ou dívida técnica conhecida?
**Impacto:** Se for esqueleto intencional, a spec registra como limitação 🟢; se houver implementação real, precisa de escavação linha a linha do `audioSocketBridge.ts`.

✅ Respondida — 🟡→🟢 (esqueleto por desenho; `voz-telefonia/design.md` e `tasks.md` atualizados)
**Resposta:** é esqueleto.

---

## Pergunta 3

**Contexto:** Unit `agenda-financeiro` / `state-machines.md` — as transições de `calendar_appointments` (confirmar/completar/no_show) foram **inferidas dos nomes das tools MCP**, não de uma máquina de estados explícita no código.
**Spec afetada:** [`_reversa_sdd/state-machines.md`](state-machines.md) (Lacunas 🔴)
**Pergunta:** Quem pode disparar cada transição de agendamento (agente IA, operador, cliente) e existe guarda de ordem temporal (ex.: não pode marcar `no_show` antes da hora do compromisso; não pode `completar` um agendamento futuro)?
**Impacto:** Define se as transições marcadas 🟡 sobem para 🟢 com regra de negócio e RBAC, ou permanecem inferência.

✅ Respondida — ator 🔴→🟢 (só agente IA e operador; cliente nunca); guarda temporal segue 🟡 → Data Master. `state-machines.md` atualizado.
**Resposta:** agente IA e operador.

---

## Pergunta 4

**Contexto:** Unit `crm-funil` / `state-machines.md` — `prospecting_campaigns.search_status` tem um conjunto de valores **não fechado** no dicionário de dados (o enum/CHECK não estava visível na leitura).
**Spec afetada:** [`_reversa_sdd/state-machines.md`](state-machines.md) (Lacunas 🔴)
**Pergunta:** Quais são os valores válidos de `prospecting_campaigns.search_status` e quais transições entre eles são permitidas? (Se preferir, aponte o CHECK/enum na migration que os define.)
**Impacto:** Fecha a máquina de estados de prospecção; hoje está aberta e 🔴.

✅ Respondida — 🔴→🟢. CHECK localizado em `supabase/migrations/20260921030100_0369_prospeccao_nativa.sql:16`: `search_status ∈ (starting, running, succeeded, failed, unknown)`, default `starting`. `state-machines.md` atualizado com o diagrama.
**Resposta:** vamos apontar na migration.

---

## Pergunta 5

**Contexto:** Unit `nucleo-ia-agente` — reconciliação de entrega sob **perda parcial de rede** (o provedor de canal recebe o envio mas a resposta HTTP se perde), além do que `reconcileAcceptedSend` já cobre.
**Spec afetada:** [`_reversa_sdd/nucleo-ia-agente/design.md`](nucleo-ia-agente/design.md) · [`_reversa_sdd/nucleo-ia-agente/tasks.md`](nucleo-ia-agente/tasks.md)
**Pergunta:** Há um comportamento esperado (documentado ou observado em produção) para o caso em que o WAHA aceita a mensagem mas a resposta se perde? A reconciliação deve evitar reenvio a todo custo, ou tolera duplicata em favor de entrega garantida?
**Impacto:** Determina se a spec pode fixar a política de reconciliação (🟢) ou se ela fica como decisão em aberto (🔴).

✅ Respondida — 🔴→🟢 (prevenir duplicata a todo custo; não reenvia em dúvida). `nucleo-ia-agente/design.md` e `tasks.md` atualizados.
**Resposta:** deve previnir duplicata a todo custo.

---

## Pergunta 6

**Contexto:** Unit `ia-suporte` — o catálogo canônico de strings de modelo (`AGENT_MODELS` × `DEFAULT_*`) precisa de confirmação, e o `lib/ai/README.md` está marcado como **obsoleto** (não deve ser usado como fonte).
**Spec afetada:** [`_reversa_sdd/ia-suporte/tasks.md`](ia-suporte/tasks.md)
**Pergunta:** A fonte de verdade das strings de modelo é `lib/ai/gateway.ts` (onde `DEFAULT_BOT_MODEL = "anthropic/claude-sonnet-5"` foi confirmado)? Confirma que o `README.md` de `lib/ai` está desatualizado e pode ser ignorado na reimplementação?
**Impacto:** Confirma qual arquivo é a fonte canônica do catálogo de modelos e evita reintroduzir informação obsoleta.

✅ Respondida — 🔴→🟢 (`lib/ai/gateway.ts` é a fonte canônica; `README.md` obsoleto, ignorar). `ia-suporte/tasks.md` atualizado.
**Resposta:** pode ser ignorado.
