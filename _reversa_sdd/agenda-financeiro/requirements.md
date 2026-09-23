# Agenda e Financeiro (`agenda`, `financeiro`, `catalogo`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 6).

## Visão Geral
Três domínios que fecham o laço comercial: **quando** o atendimento acontece (agenda), **o que** se vende (catálogo de produtos) e **quanto** entra e sai (financeiro/comanda). A agenda é a maior peça: motor de horários livres 100% puro, sincronização de duas vias com o Google Calendar (merge de três pontas), e vocabulário espelhado no banco por migration. Princípio transversal: quase toda regra é pura e testável sem banco nem relógio (`agora` é sempre parâmetro injetado). 🟢

## Responsabilidades
- Calcular horários livres (motor puro, sem banco/relógio). 🟢
- Sincronizar duas vias com Google Calendar (OAuth, reconciliação, Meet, lembretes). 🟢
- Decidir o que ocupa vs o que só parece ocupar. 🟢
- Gerir comanda (conta do atendimento) e catálogo financeiro (contas, formas, planos, comissão, recorrências). 🟢
- Buscar produtos com busca difusa (palavra aproximada, número exato) e importar por planilha. 🟢
- Proteger follow-up quando há compromisso vivo. 🟢

## Regras de Negócio
- Bloqueio por exceção é um OCUPADO, não uma janela menor (subtrair reparte a grade). — `agenda/horarios-livres.ts` 🟢
- `windows` vazio = zero horário na agenda (≠ 24/7 do roteamento). 🟢
- Na dúvida, OCUPA (oferecer de menos se recupera; marcar em dobro não). — `agenda/ocupados.ts` 🟢
- Conexão do Google BLOQUEIA a menos que um humano tenha mandado parar (só `disconnected`/`connecting` não contam). — `agenda/tipos.ts` 🟢
- `SITUACAO_SEGURA_O_LEAD`: pending/confirmed seguram o lead; resto não. 🟢
- Google: `id` e `iCalUID` nunca enviados juntos (HTTP 400). — `agenda/google/evento.ts` 🟢
- Comissão: precedência pessoa+serviço > pessoa > serviço > zero; empate resolve pelo MAIOR percentual. — `financeiro/comanda.ts` 🟢
- Comanda: nada apagado, cancelar é status; total do item com piso zero; desconto da comanda não entra no item. 🟢
- Catálogo: PALAVRA é difusa, NÚMERO é exato (número FILTRA, não ranqueia). — `catalogo/busca.ts` 🟢
- Moeda da org: função única, fallback BRL com rastro; nunca aceita moeda de quem chamou. — `catalogo/moeda-da-org.ts` 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Calcular horários livres puro | Must | Grade não se move; faixas unidas; ocupados/aviso/janela removidos |
| RF-02 | Bloqueio como ocupado, não janela menor | Must | Reunião 10:30-11:30 não desliza a tarde |
| RF-03 | Sincronizar duas vias com Google | Must | Merge de três pontas classifica accept_remote/publish/converged/conflict |
| RF-04 | Ocupação conservadora | Must | Na dúvida ocupa; só cancelled/no_show liberam |
| RF-05 | Comissão com precedência | Must | pessoa+serviço vence; empate pelo maior |
| RF-06 | Busca difusa palavra/número | Should | 128GB não aparece para quem pediu 256GB |
| RF-07 | Proteger follow-up com compromisso vivo | Should | Follow-up adia quando há agenda viva |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | OAuth Google DB-first cifrado; state HMAC-SHA256 | `agenda/google/config.ts`, `estado.ts` | 🟢 |
| Segurança | Outbound de sync guardado como hashes SHA256, não PII | `agenda/google/sync-model.ts` | 🟢 |
| Segurança | Toda query filtra `organization_id` (client injetado rota/MCP) | `agenda/consulta.ts` | 🟢 |
| Disponibilidade | Ocupação do Google por dono em paralelo (um falhando não derruba) | `agenda/ocupacao-externa.ts` | 🟢 |
| Disponibilidade | Recusa conhecida nunca vira 5xx (Meet) | `agenda/google/motivo-do-meet.ts` | 🟢 |
| Corretude | Fuso com DST de duas passagens; nunca Invalid Date | `agenda/fuso.ts` | 🟢 |
| Corretude | Custo em centavos; comissão congelada na linha | `financeiro/comanda.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado uma reunião 10:30-11:30 e um tipo com slots de 1h
Quando horariosLivres calcula
Então a tarde não desliza e o 17:00 livre permanece

Dado uma conexão Google com token_expired
Quando ocupadosDoDono avalia
Então o compromisso continua ocupando (só humano manda parar)

Dado regras de comissão pessoa+serviço e pessoa
Quando percentualDaComissao resolve
Então usa pessoa+serviço (mais específica); empate pelo maior
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Motor de horários livres (RF-01/02) | Must | Base de todo agendamento |
| Ocupação conservadora (RF-04) | Must | Evita marcação em dobro |
| Sync Google (RF-03) | Should | Integração de duas vias, isolada |
| Comissão (RF-05) | Must | Dinheiro no bolso de quem trabalhou |
| Busca difusa (RF-06) | Should | Qualidade da resposta de preço |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/agenda/horarios-livres.ts` | `horariosLivres`, `unirFaixas` | 🟢 |
| `lib/agenda/fuso.ts` | `instanteDe` | 🟢 |
| `lib/agenda/consulta.ts` | `horariosLivresDaOrg`, `coletaOQueOcupa`, `listaAgendamentos` | 🟢 |
| `lib/agenda/ocupados.ts` | `ocupadosDoDono` | 🟢 |
| `lib/agenda/google/sync-model.ts` | `compare`, `reconcileAppointment` | 🟢 |
| `lib/agenda/google/erros.ts` | `classificarErroDoGoogle` | 🟢 |
| `lib/financeiro/comanda.ts` | `percentualDaComissao`, `totalDoItem` | 🟢 |
| `lib/catalogo/busca.ts` | `tokenizar`, `pontuar`, `buscarComRelaxamento` | 🟢 |
| `lib/catalogo/moeda-da-org.ts` | `moedaDaOrganizacao` | 🟢 |
