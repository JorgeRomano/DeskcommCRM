# Relatório de Confiança — DeskcommCRM

> Gerado pelo Revisor (Reversa) em 2026-09-23 · `doc_level = detalhado`
> **Atualizado após as respostas do usuário** (`questions.md`, 6/6 respondidas).
> Contagem de marcadores de confiança (🟢/🟡/🔴) por unit, medida no disco (linhas de legenda
> excluídas). Perguntas resolvidas em [`questions.md`](questions.md); lacunas em [`gaps.md`](gaps.md).

---

## Resumo Geral (units de domínio)

| Nível | Quantidade | Percentual |
|-------|-----------|------------|
| 🟢 CONFIRMADO | 1558 | 94,2% |
| 🟡 INFERIDO   | 51   | 3,1%  |
| 🔴 LACUNA     | 45   | 2,7%  |
| **Total**     | 1654 | 100%  |

**Confiança geral:** **95,7%** — `(🟢 + 🟡×0,5) / total = (1558 + 25,5) / 1654`.

> Artefatos globais do Detetive (`state-machines.md`, `permissions.md`, `domain.md`) são contados
> à parte (🟢 170 · 🟡 19 · 🔴 20) e receberam 2 das reclassificações desta rodada
> (transições de agendamento e `search_status`).

Leitura: a extração é fortemente ancorada no código. As 🔴 restantes estão concentradas na camada
de banco (delegada ao Data Master), não em regra de negócio da aplicação. Após as respostas, não há
mais 🔴 que dependa de decisão humana.

---

## Por Unit (pós-respostas)

| Unit | 🟢 | 🟡 | 🔴 | Confiança | Impressão |
|------|----|----|-----|-----------|-----------|
| `nucleo-ia-agente/` | 213 | 6 | 5 | 96% | solid (corrigida 13→12 tools; reconciliação fechada) |
| `ia-suporte/` | 176 | 8 | 5 | 95% | solid (corrigida 9→11 conferências; catálogo de modelos fechado) |
| `canais-mensageria/` | 147 | 7 | 4 | 95% | solid |
| `voz-telefonia/` | 93 | 4 | 1 | 97% | solid (SIP externo + esqueleto WebRTC confirmados) |
| `crm-funil/` | 117 | 4 | 5 | 94% | solid |
| `agenda-financeiro/` | 75 | 1 | 3 | 96% | solid |
| `automacao-roteamento/` | 77 | 4 | 1 | 96% | solid |
| `auth-tenancy-rbac/` | 162 | 4 | 7 | 95% | solid |
| `compliance/` | 82 | 3 | 2 | 96% | solid |
| `plataforma-operacao/` | 132 | 4 | 4 | 96% | solid |
| `integracoes-externas/` | 71 | 1 | 2 | 97% | solid |
| `eventos-tempo-real/` | 82 | 1 | 2 | 97% | solid |
| `infra-transversal-relatorios/` | 65 | 1 | 2 | 96% | solid |
| `superficie-http/` | 66 | 3 | 2 | 95% | solid |

> Os números por unit somam todos os arquivos da pasta (canônicos + opcionais + sub-units de caso de uso).

---

## Reclassificações desta revisão

### Contagens frágeis (revisão cruzada de código)

| De | Para | Afirmação | Evidência |
|----|------|-----------|-----------|
| 🟢 (frágil) | 🟢 (corrigido) | `nucleo-ia-agente`: "superfície estática de **13** tools" → **12** | `lib/agent-engine/agent/inbound-turn.ts:191` — `AGENT_TOOL_DEFS` tem 12 chaves |
| 🟢 (frágil) | 🟢 (corrigido) | `ia-suporte`: "**9** conferências de saída" → **11** | `lib/ai/guardrails/lista-de-conferencia.ts:71` — `CONFERENCIAS_DE_SAIDA` = 11 |

### Lacunas fechadas pelas respostas do usuário (2026-09-23)

| De | Para | Afirmação | Evidência / Decisão |
|----|------|-----------|---------------------|
| 🔴 | 🟢 | `voz-telefonia`: topologia Asterisk/SIP | Dependência externa de infraestrutura por decisão — não se reimplementa |
| 🟡 | 🟢 | `voz-telefonia`: ponte WebRTC humano | Esqueleto por desenho (dívida conhecida); preservar |
| 🔴 | 🟢 | `state-machines`: ator das transições de `calendar_appointments` | Só agente IA e operador; cliente nunca |
| 🔴 | 🟢 | `state-machines`: valores de `prospecting_campaigns.search_status` | CHECK em `migrations/20260921030100_0369_prospeccao_nativa.sql:16`: `starting, running, succeeded, failed, unknown` |
| 🔴 | 🟢 | `nucleo-ia-agente`: política de reconciliação sob perda de rede | Prevenir duplicata a todo custo (não reenvia em dúvida) |
| 🔴 | 🟢 | `ia-suporte`: fonte canônica do catálogo de modelos | `lib/ai/gateway.ts`; `README.md` obsoleto, ignorar |

Arquivos atualizados nesta revisão:
`nucleo-ia-agente/contracts.md`, `nucleo-ia-agente/design.md`, `nucleo-ia-agente/tasks.md`,
`ia-suporte/requirements.md`, `ia-suporte/tasks.md`, `voz-telefonia/design.md`,
`voz-telefonia/tasks.md`, `state-machines.md`.

---

## Verificações que sustentaram os 🟢 (amostra da revisão de código)

Confirmadas contra a fonte, sem reclassificação:
- `nucleo`: `BEFORE_SEND_CHAIN_VERSION = 7`; 11 gates before-send; `DEFAULT_MAX_SENDS_PER_TURN = 3`;
  purposes isentos de orçamento; grafo lead-state `new→…→won|lost` (won/lost terminais).
- `auth`: `getUser()` na borda e no `requireRole`; cookie `sameSite:"strict"`+`httpOnly`; MFA soma
  plataforma+org, default não exige.
- `ia-suporte`: `DEFAULT_BOT_MODEL = "anthropic/claude-sonnet-5"`.
- `canais`: ordem pós-entrada opt-out→origem→demanda→campanha→acelerar→despacho.
- `crm`: score `null` sem lastro; `SCAN_CAP=500`/`IDS_POR_CONSULTA=100`.
- `agenda`: precedência de comissão pessoa+serviço > pessoa > serviço, empate pelo maior.
- `voz`: `instalacaoOferece && (escolhaDaOrg ?? false)`; guard de 3 códigos 422/503/503.
- `eventos`: `MAX_ATTEMPTS=5`, backoff `2^attempts` em minutos.
- `infra`: `begin read only` imposto pelo Postgres; `MAX_POOLS=32`; idempotência reserva-antes-do-efeito.
- `plataforma`: "11 stops" = rampa de cor (guarda de entrada, não de saída).

---

## Lacunas Pendentes 🔴 (síntese pós-respostas)

Detalhe e ação em [`gaps.md`](gaps.md). Após as respostas do usuário, **nenhuma 🔴 depende mais de
decisão humana**. As 🔴 restantes são todas da camada SQL, delegadas ao **Data Master**:

- Corpos de RPCs SECURITY DEFINER, policies RLS, CHECKs e triggers das tabelas tenant-aware.
- Guarda de ordem temporal das transições de `calendar_appointments` (🟡): confirmar se existe
  CHECK/trigger; o ator já está fechado (🟢).
- Convite de time (`StatusConvite`): máquina derivada de timestamps, documentada como nota.

---

## Recomendações

- [ ] **Rodar o Data Master** para fechar o bloco restante de 🔴/🟡 (camada de banco). Maior retorno
      de confiança agora que as lacunas humanas estão resolvidas.
- [ ] Ao aprofundar `superficie-http`, mapear as ~337 rotas uma a uma e reconferir contagens no disco.
- [ ] Conferir os 24 schemas de `lib/schemas/` campo a campo antes de gerar specs por entidade.

---

## Revisão Cruzada

- Engine externa consultada: **nenhuma** — o plugin do Codex não estava ativo nesta sessão
  (`doc_level = detalhado` exige a cruzada **quando disponível**; ausente, a etapa é ignorada).
- A revisão foi feita pelo Revisor com leitura direta do código-fonte legado (amostra de ~40
  afirmações 🟢 verificadas contra `arquivo:linha`).
