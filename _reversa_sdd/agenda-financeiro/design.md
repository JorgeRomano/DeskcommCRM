# Agenda e Financeiro — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface (destaques)
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `horariosLivres` | `(entrada)` | `Slot[]` (puro) |
| `instanteDe` | `(parede, fuso)` | `Date` (duas passagens, nunca Invalid) |
| `horariosLivresDaOrg` | `(client, ...)` | `ResultadoDaConsulta` (recusa honesta) |
| `compare` | `(base, local, remote)` | `accept_remote \| publish \| converged \| conflict` |
| `reconcileAppointment` | `(...)` | máquina de estados por compromisso |
| `classificarErroDoGoogle` | `(erro, operacao)` | `DesfechoDoGoogle` |
| `percentualDaComissao` | `(regras, alvo)` | percentual (precedência) |
| `tokenizar` / `pontuar` | `(termo)` / `(...)` | tokens / nota\|null |

## Fluxo Principal — Horários livres
1. Grade nasce no início da janela publicada e não se move (de `intervaloMin` em `intervaloMin`). 🟢
2. Faixas sobrepostas unidas (`unirFaixas`, `<=` funde contíguas). 🟢
3. Remove ocupados → remove o que começa antes de `agora+avisoMinimo` → remove o que passa de `agora+janelaDias`. 🟢
4. Buffer infla o SLOT, não o compromisso. Dedup final por instante (protege DST). 🟢

## Fluxo Principal — Sync Google (merge de três pontas)
1. OAuth DB-first (`platform_google_oauth` cifrado) com fallback `.env`. 🟢
2. `compare(base, local, remote)`: outbound guardado como hashes SHA256. 🟢
3. `reconcileAppointment`: RPC `fn_google_appointment` (claim/commit/meet/idle/prepare/renew/error/release), sempre `release` no finally. 🟢
4. Erros por `classificarErroDoGoogle` → `estadoDaConexaoApos` grava só os 5 estados decididos pelo sistema. 🟢
5. Meet: `meetVideoUrl` valida estritamente; entrega passa por `fn_meet_delivery_policy`. 🟢

## Fluxo Principal — Financeiro/Catálogo
- Comanda: `percentualDaComissao` (precedência) e `totalDoItem` (piso zero) antes do insert; escrita via `fn_finalizar_comanda`. 🟢
- Busca: `tokenizar` separa palavra (difusa) de número (exato); número FILTRA (`pontuar` devolve null); `buscarComRelaxamento` marca ignorados. 🟢

## Dependências
- `agenda` → `schemas/routing`, `schemas/settings`, `routing/eligibility`, `tempo/fusos`, `atendimento/fronteira`, `ai/elegibilidade/gate`, `date-fns`. 🟢
- `agenda/google` → API Google Calendar v3 + OAuth2, `webhooks/secrets`, `crypto`. 🟢
- `financeiro` → `zod` (schemas puros); RPCs de escrita. 🟢
- `catalogo` → `contacts/csv` (`parseCsv`), `schemas/produtos`, `money`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Bloqueio = ocupado (não janela menor) | `agenda/horarios-livres.ts` (DECISÃO 12) | 🟢 |
| Conexão Google bloqueia salvo humano parar | `agenda/tipos.ts` (DECISÃO 3.2) | 🟢 |
| Merge de três pontas com hashes, não PII | `agenda/google/sync-model.ts` | 🟢 |
| `id`+`iCalUID` nunca juntos | `agenda/google/evento.ts` | 🟢 |
| PALAVRA difusa, NÚMERO exato que filtra | `catalogo/busca.ts` | 🟢 |

## Estado Interno
- `calendar_appointments`, `calendar_event_types`, `calendar_availability_exceptions`, `attendant_availability`, `calendar_connections`, `calendar_external_events`, `crm_lead_links` (vínculo polimórfico); financeiras `financial_accounts`, `payment_methods`, `account_plans`, `commission_rules`, `recurring_entries`, `sale_orders`/`sale_items`; `catalog_products`. 🟢

## Observabilidade
- Recusa em duas vozes (`motivoParaOperador` vs `motivoParaCliente`); sinais de defasagem (`fontesDefasadas`, `googleCoberturaParcial`). 🟢

## Riscos e Lacunas
- 🔴 RPCs security-definer (`fn_google_*`, `fn_meet_*`, `fn_finalizar_comanda`, `fn_agenda_ocupacao_google_do_dono`) vivem no baseline — lógica interna é lacuna para o Data Master.
- 🔴 Migration 0240 (invariantes da comanda) referenciada em prosa; CHECK/triggers exatos para o Data Master.
- 🟡 TTL de autorização de entrega do Meet lê env var cujo nome exato não aparece nestes arquivos.
