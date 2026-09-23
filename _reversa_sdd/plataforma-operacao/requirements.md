# Plataforma e Operação (`settings`, `onboarding`, `instalacao`, `branding`, `navigation`, `system`, `operacao`, `release`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 10).

## Visão Geral
A camada que faz o produto ser self-host e revendável: como a marca é resolvida sem tocar no `.env`, como a instalação lê configuração do banco acima do ambiente, como o onboarding propõe um funil, como a navegação sai de um registro único, como o update se narra na tela e como as superfícies de operação são compartilhadas entre a tela e o agente de IA. Um fio atravessa tudo: **banco acima do `.env`, `.env` é semente + piso de rollback**, e **nunca lançar no caminho de render**. 🟢

## Responsabilidades
- Resolver marca própria (white-label) do banco, com rampa de cor + contraste WCAG, sem lançar. 🟢
- Ler configuração da instalação do banco acima do `.env` (segredos cifrados). 🟢
- Conduzir o onboarding (passos do wizard + sugestão/pacotes de funil). 🟢
- Prover o registro único de navegação (`NAV_CATALOG`) e o gate de apresentação. 🟢
- Narrar o changelog e a máquina de estado do update. 🟢
- Expor superfícies de operação compartilhadas (tela + agente): entradas automáticas, regras, marcadores, modelos. 🟢
- Apagar dados operacionais de uma org (zona de perigo, na ordem certa). 🟢

## Regras de Negócio
- Marca resolve do banco (org → instalação → `.env`), precedência por CAMPO; nunca lança (roda em `app/layout.tsx`). — `branding/resolve.ts` 🟢
- Marca guarda ENTRADA, nunca SAÍDA (nunca os 11 stops), para correções alcançarem instalações existentes. — `branding/schema.ts` 🟢
- Configuração da instalação: banco vence o `.env`; `.env` é semente + piso de rollback (`agent.sh` reverte só a imagem, não o schema). — `instalacao/config-resolve.ts` 🟢
- Reset de dados operacionais: `contacts` é sempre o último (FKs `ON DELETE RESTRICT`); filtro de org é o único separador de tenant. — `settings/apagar-dados-operacionais.ts` 🟢
- Navegação: `NAV_CATALOG` é a única lista; interface é APRESENTAÇÃO, nunca autorização (`canSee` compara ROLE_RANK). — `navigation/interface.ts` 🟢
- Onboarding: RECUSAR proposta ruim é o desfecho certo (cai num pacote curado), nunca auto-completar; funil sem won → `pipeline_no_won_stage`. — `onboarding/proposta-de-funil.ts` 🟢
- Update: só `dispatched→terminal`; terminal é imutável; a VERSÃO reportada decide rollback, tempo sozinho não basta. — `system/update-run.ts` 🟢
- Operação: agente só ALTERNA regra (humano escreve/revisa), nunca cria/edita/apaga; escrita ao mundo externo é `critico`. — `operacao/regras-automaticas.ts` 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Resolver marca sem lançar | Must | Recusa volta como `MotivoDaMarca`, nunca 500 no render |
| RF-02 | Config da instalação banco > env | Must | Linha no banco vence o `.env` (mesmo `valor:null` = branco intencional) |
| RF-03 | Reset de dados na ordem correta | Must | `contacts` por último; DELETE filtra org; contagens parciais voltam na falha |
| RF-04 | Registro único de navegação | Must | Sidebar/hubs/⌘K são projeções de `NAV_CATALOG` |
| RF-05 | Interface é apresentação | Must | `canSee` decide visibilidade por ROLE_RANK; `PORTAS_ESSENCIAIS` não removíveis |
| RF-06 | Onboarding com pacote como plano B | Must | Sugestão de IA falha → pacote curado com `porque` |
| RF-07 | Máquina de estado do update | Should | `canTransition` só dispatched→terminal; rollback decidido pela versão reportada |
| RF-08 | Operação compartilhada tela+agente | Must | Agente alterna regra, nunca cria; escrita externa é `critico` |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança | Segredos cifrados AES-GCM; só last4 na tela; fail-closed sem chave | `instalacao/config.ts` | 🟢 |
| Segurança | Logo por assinatura de byte, SVG banido; PATH nunca URL | `branding/logo.ts` | 🟢 |
| Segurança | CSS da marca fail-closed (allowlist de forma; rejeita `<`, `;}`) | `branding/css.ts` | 🟢 |
| Segurança | Prefixo reafirmado no DELETE de logo (service-role bypassa RLS) | `branding/logo-arquivo.ts` | 🟢 |
| Acessibilidade | Rampa com contraste WCAG e correção de dicromacia | `branding/contraste.ts` | 🟢 |
| Disponibilidade | Memo em globalThis com geração (Next instancia módulo 2×) | `branding/instalacao.ts`, `instalacao/*` | 🟢 |
| Corretude | Reset não-atômico com ordem que mantém o banco íntegro | `settings/apagar-dados-operacionais.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado uma cor de marca inválida numa camada superior
Quando resolverMarca avalia
Então anota o motivo e continua descendo (nunca lança no render)

Dado um reset de dados operacionais
Quando apagarDadosOperacionaisDaOrg roda
Então apaga filhos antes de contacts (evita 23503) e filtra org

Dado uma sugestão de funil inválida da IA
Quando validarProposta reprova
Então cai num pacote curado com porque (nunca quadro vazio)
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Marca sem lançar (RF-01) | Must | Roda em toda tela |
| Config banco>env (RF-02) | Must | Doutrina de packaging |
| Reset na ordem (RF-03) | Must | Integridade sob FKs RESTRICT |
| Navegação única (RF-04/05) | Must | Anti-drift + apresentação ≠ autorização |
| Onboarding (RF-06) | Must | Primeira experiência sem quadro vazio |
| Update (RF-07) | Should | Narra a rodada na UI |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/branding/resolve.ts` | `resolverMarca` | 🟢 |
| `lib/branding/rampa.ts` | `rampaDeSemente`, `hexParaOklch` | 🟢 |
| `lib/branding/contraste.ts` | `derivarMarca`, `escolherAccent` | 🟢 |
| `lib/instalacao/config-resolve.ts` | `resolver` | 🟢 |
| `lib/instalacao/config.ts` | `valorDaInstalacao`, `gravarPelaTela` | 🟢 |
| `lib/settings/apagar-dados-operacionais.ts` | `apagarDadosOperacionaisDaOrg` | 🟢 |
| `lib/navigation/catalogo.ts` | `NAV_CATALOG`, `NAV_GROUPS` | 🟢 |
| `lib/navigation/interface.ts` | `canSee`, `combinarInterfaces` | 🟢 |
| `lib/system/update-run.ts` | `canTransition`, `rollbackFoiSuperado` | 🟢 |
| `lib/onboarding/sugerir-funil.ts` | `sugerirFunil` | 🟢 |
| `lib/operacao/regras-automaticas.ts` | `definirRegraAtiva` | 🟢 |
