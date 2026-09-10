# Onboarding e Instalação

> Spec SDD (híbrida — módulo de topo) · Redator (Reversa) · Nível: Detalhado
> Módulos: `lib/onboarding/*`, `lib/instalacao/*`
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Visão Geral

Wizard de configuração inicial (passos que existem conforme a instalação) e detecção do que o instalador já trouxe pronto no ambiente. Um passo só aparece se se aplica à instalação (loja só se `lojaLigada`), evitando "passo fantasma". 🟢

## Responsabilidades

- Definir os passos do wizard como fonte única (roteador + indicador + resumo). 🟢
- Decidir o próximo passo e o resumo final a partir do estado. 🟢
- Detectar o ambiente da instalação (chaves de provedor, gateway, email, transporte WhatsApp). 🟢
- Sugerir funil por IA no setup. 🟡
- Detectar nome placeholder da instalação. 🟢

## Regras de Negócio

- Passo só existe se `existe(ctx)`; `connect-nuvemshop` só se `lojaLigada`. 🟢
- Fonte única `PASSOS` alimenta roteador, indicador e resumo (evita listas que discordam). 🟢
- Ordem: welcome → connect-whatsapp → connect-nuvemshop → setup-ai → funil → testar → invite-team. 🟢
- Valor vazio de env é ausente (contrato do `.env`: template gera `CHAVE=`). 🟢
- Google não tem chave de plataforma (sem variável; declarar nome inventado enganaria). 🟢
- `NOME_PLACEHOLDER_DA_INSTALACAO = "Minha Empresa"` (detecção de org não configurada). 🟢
- Sugestão de funil nunca devolve funil vazio → cai em `PACOTE_PADRAO`. 🟡

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Resolver passos visíveis conforme a instalação | Must | Dada loja desligada, `connect-nuvemshop` não aparece |
| RF-02 | Decidir próximo passo e resumo | Must | Dado estado parcial, `proximoPasso` retorna o 1º não cumprido |
| RF-03 | Detectar ambiente da instalação | Should | Dado `.env` sem `RESEND_API_KEY`, `email=false` |
| RF-04 | Detectar nome placeholder | Should | Dada org "Minha Empresa", `nomeAindaEhPlaceholder=true` |
| RF-05 | Sugerir funil por IA no setup | Could | Dado nicho clínica, sugere pacote; falha → `PACOTE_PADRAO` |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Robustez | Sugestão de funil nunca vazia (fallback) | `lib/onboarding/sugerir-funil.ts` | 🟡 |
| Correção | Passo fantasma eliminado (fonte única) | `lib/onboarding/passos.ts` | 🟢 |
| Segurança | Detecção lê env, não expõe valores | `lib/instalacao/ambiente.ts` | 🟢 |

## Critérios de Aceitação

```gherkin
Dada uma instalação por kit com a integração de loja desligada
Quando passosVisiveis roda
Então o passo connect-nuvemshop não aparece (nem como pendência nem no resumo)

Dado um .env com ANTHROPIC_API_KEY preenchida e RESEND_API_KEY vazia
Quando lerAmbiente roda
Então chavesDeProvedor.anthropic=true e email=false

Dada uma org ainda chamada "Minha Empresa"
Quando nomeAindaEhPlaceholder roda
Então retorna true (o produto sabe que falta configurar)
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Passos visíveis + próximo passo | Must | Estrutura do wizard; sem ela o onboarding não flui |
| Detecção de ambiente | Should | Adapta o wizard ao que o instalador trouxe |
| Nome placeholder | Should | Sinaliza configuração pendente |
| Sugestão de funil | Could | Acelera setup; degrada para pacote padrão |

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/onboarding/passos.ts` | `PASSOS`, `passosVisiveis`, `proximoPasso`, `resumoDoOnboarding` | 🟢 |
| `lib/onboarding/sugerir-funil.ts` | `sugerirFunil` | 🟡 |
| `lib/instalacao/ambiente.ts` | `lerAmbiente`, `nomeAindaEhPlaceholder` | 🟢 |
