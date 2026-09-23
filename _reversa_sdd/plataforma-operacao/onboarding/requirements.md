# Caso de Uso: Onboarding

> Sub-unit de `plataforma-operacao`. Passos do wizard e sugestão de funil (IA com pacote curado como plano B).
> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Visão Geral
Conduz o wizard de primeira configuração e propõe um quadro de funil, usando a IA quando disponível e caindo num pacote curado por tipo de negócio quando não. 🟢

## Responsabilidades
- Definir e ordenar os passos visíveis do wizard. 🟢
- Sugerir um funil (validar/normalizar; recusar → pacote curado). 🟢
- Descobrir o que mais existe após o wizard. 🟢

## Regras de Negócio
- Um passo decide a própria existência (`existe`); `funil` vem depois de `setup-ai`. 🟢
- `MIN_ETAPAS=4`, `MAX_ETAPAS=8`; etapa é NOME+PASSO; `is_won`/`is_lost` derivados do passo. 🟢
- Reprovar (sem won, sem lost, sem nome, < MIN) cai num pacote curado — nunca auto-completa. 🟢
- `PACOTE_PADRAO` busca por id e lança no load se ausente (nunca `PACOTES[0]`). 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Passos visíveis por contexto | Must | nuvemshop só quando `lojaLigada` (nunca fantasma) |
| RF-02 | Sugestão com plano B | Must | Falha do pipeline → pacote com `porque` |
| RF-03 | Validação de funil | Must | Sem etapa won → reprovado (evita `pipeline_no_won_stage`) |

## Critérios de Aceitação
```gherkin
Dado que a IA está indisponível
Quando sugerirFunil roda
Então devolve um pacote curado por tipo de negócio com porque

Dado uma proposta sem etapa de ganho
Quando validarProposta avalia
Então reprova (cai no pacote curado)
```

## Rastreabilidade de Código
| Arquivo | Função | Cobertura |
|---------|--------|-----------|
| `lib/onboarding/passos.ts` | `passosVisiveis`, `proximoPasso` | 🟢 |
| `lib/onboarding/proposta-de-funil.ts` | `validarProposta`, `etapasParaGravar` | 🟢 |
| `lib/onboarding/pacotes-de-funil.ts` | `PACOTES`, `PACOTE_PADRAO` | 🟢 |
| `lib/onboarding/sugerir-funil.ts` | `sugerirFunil`, `escolherPacotePorTexto` | 🟢 |
