# Plataforma e Operação — Design Técnico

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Interface (destaques)
| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `resolverMarca` | `(camadas, regua)` | `MarcaResolvida` (nunca lança) |
| `derivarMarca` | `(semente, regua)` | `Marca` (rampa + tokens WCAG) |
| `resolver` (config) | `(linha, doAmbiente)` | `ValorResolvido` (`banco\|ambiente\|ausente`) |
| `apagarDadosOperacionaisDaOrg` | `(client, organizationId)` | `ResultadoDoApagamento` |
| `canSee` | `(d, platform, role)` | `boolean` (única função de autorização) |
| `combinarInterfaces` | `(daEmpresa, doVinculo)` | interseção nos dois eixos |
| `canTransition` | `(from, to)` | `boolean` (só dispatched→terminal) |
| `sugerirFunil` | `(ctx, gerar)` | proposta ou pacote com `porque` |

## Fluxo Principal — Marca (white-label)
1. `resolverMarca` compõe camadas org→instalação→env por CAMPO (`primeiroDefinido`); cor inválida anota e continua descendo. 🟢
2. `derivarMarca`/`rampaDeSemente` derivam 11 stops (OKLab, semente ancorada no stop 600); `escolherAccent` desloca a rampa e escolhe accent com contraste; `reconciliarSemanticas` rotaciona as semânticas sob dicromacia. 🟢
3. `cssDaMarca` serializa fail-closed (allowlist de forma, `:root:root` para vencer globals.css). 🟢
4. Cache de derivação em `WeakMap<Regua, Map<seed, Marca>>`, FIFO teto 64. 🟢

## Fluxo Principal — Config da instalação
`resolver(linha, doAmbiente)`: banco vence se a linha existir; `valorDaInstalacao` lê `platform_config` (segredos cifrados AES-GCM, só last4 na tela); `gravarPelaTela` fail-closed sem chave de cifra. `comportamento.ts` (kill-switches) tem leitura STICKY. 🟢

## Fluxo Principal — Reset de dados operacionais
DELETE em ordem (filhos antes de `contacts`, que tem FKs `ON DELETE RESTRICT`); não-atômico mas a ordem mantém o banco íntegro; contagens parciais voltam na falha. 🟢

## Fluxo Principal — Navegação e onboarding
- `NAV_CATALOG` único; `sidebarGroups`/`hubSections`/`searchable` são projeções; `canSee` decide por ROLE_RANK; `combinarInterfaces` faz interseção (org estreita, vínculo estreita dentro). 🟢
- Onboarding: `passosVisiveis`/`proximoPasso`; `sugerirFunil` → `escolherPacotePorTexto` → `gerar` (rede injetada) → `normalizarProposta` → `validarProposta`; qualquer falha → pacote curado. 🟢

## Dependências
- `branding` → `schema`/`contraste`/`css`/`rampa`/`resolve`, `lib/env`, `lib/supabase/admin`, `lib/instalacao/config`. 🟢
- `instalacao` → `lib/crypto/aes_gcm`, `platform_config`/`platform_settings`, `pg` (worker). 🟢
- `navigation` → `lib/auth/types` (ROLE_RANK), Phosphor icons. 🟢
- `operacao` → `webhook_sources`, `automation_rules`, `message_templates`, `lib/audit`. 🟢

## Decisões de Design Identificadas
| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Marca resolve do banco, nunca do env (guarda entrada) | `branding/resolve.ts`, `schema.ts` | 🟢 (ADR-0008) |
| Nunca lançar no render; degradar + carregar motivo | `resolve.ts`, `instalacao.ts`, `saida.ts` | 🟢 |
| Memo em globalThis com geração (Next instancia 2×) | `branding/instalacao.ts` | 🟢 |
| Interface é apresentação, não autorização | `navigation/interface.ts` | 🟢 |
| Reset na ordem de FK (contacts por último) | `settings/apagar-dados-operacionais.ts` | 🟢 |
| Onboarding recusa e cai em pacote (nunca auto-completa) | `onboarding/proposta-de-funil.ts` | 🟢 |

## Estado Interno
- `platform_branding`, `platform_config`, `platform_settings`, `webhook_sources`, `automation_rules`, `message_templates`, colunas `last_change_actor_kind`/`last_change_at`. 🟢

## Observabilidade
- `autoriaDaMudanca` + `audit()` nas escritas de operação; selo de autoria só para ai/system (humano não emite selo). 🟢

## Riscos e Lacunas
- 🔴 Tabelas/CHECKs/RLS (`platform_branding`, `platform_config`, `platform_settings`, `webhook_sources`, CHECK 0089/`crm_stages_hint_coerente`) — Data Master.
- 🟡 Conta exata de `NAV_CATALOG` e `CATALOGO_DA_INSTALACAO` lida do array; reconferir com `git grep`/`wc -l`.
- 🟡 `provarSaldo`/`sugerirFunil` fazem chamada de rede real; comportamento por provedor sob erro inferido do mapeamento.
