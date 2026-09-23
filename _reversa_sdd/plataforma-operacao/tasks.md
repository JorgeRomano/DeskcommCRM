# Plataforma e Operação — Tarefas de Implementação

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## Pré-requisitos
- [ ] Tabelas: `platform_branding`, `platform_config`, `platform_settings`, `webhook_sources`, `automation_rules`, `message_templates`
- [ ] Storage bucket público `brand-logos`; `lib/crypto/aes_gcm`
- [ ] Env: `NEXT_PUBLIC_SUPABASE_URL`, `APP_NAME`/`APP_LOGO_URL`/`APP_ACCENT_HEX`, `SUPPORT_EMAIL`, `LGPD_DPO_EMAIL`

## Tarefas
- [ ] T-01, Implementar o motor de marca própria
  - Origem no legado: `lib/branding/*` (`rampa`, `contraste`, `resolve`, `schema`, `instalacao`, `logo`, `css`, `saida`, `contexto`, `icone`, `linguagem`)
  - Critério de pronto: nunca lança; guarda entrada; rampa OKLab com contraste; CSS fail-closed; logo por assinatura de byte
  - Confiança: 🟢
- [ ] T-02, Implementar config da instalação (banco > env)
  - Origem no legado: `lib/instalacao/*` (`config-resolve`, `config`, `catalogo`, `comportamento`, `ambiente`, `prova-de-credito`, `retrato`)
  - Critério de pronto: banco vence; segredos cifrados; `gravarPelaTela` fail-closed; comportamento sticky
  - Confiança: 🟢
- [ ] T-03, Implementar reset de dados operacionais
  - Origem no legado: `lib/settings/apagar-dados-operacionais.ts`
  - Critério de pronto: ordem de FK (contacts por último); filtro de org; contagens parciais na falha
  - Confiança: 🟢
- [ ] T-04, Implementar navegação e gate de apresentação
  - Origem no legado: `lib/navigation/*` (`catalogo`, `interface`, `registry`)
  - Critério de pronto: `NAV_CATALOG` único; `canSee` por ROLE_RANK; `combinarInterfaces` interseção; portas essenciais
  - Confiança: 🟢
- [ ] T-05, Implementar changelog e máquina de estado do update
  - Origem no legado: `lib/system/changelog.ts`, `update-run.ts`
  - Critério de pronto: parser Keep-a-Changelog; `canTransition` só dispatched→terminal; rollback pela versão reportada
  - Confiança: 🟢
- [ ] T-06, Implementar superfícies de operação
  - Origem no legado: `lib/operacao/*` (`autoria`, `entradas-automaticas`, `regras-automaticas`, `marcadores-e-time`, `modelos-de-mensagem`)
  - Critério de pronto: org-scoped; agente só alterna regra; escrita externa `critico`; preencher ≠ enviar; secret nunca sai
  - Confiança: 🟢
- [ ] T-07, Implementar onboarding
  - Origem no legado: `lib/onboarding/*` (`passos`, `proposta-de-funil`, `pacotes-de-funil`, `sugerir-funil`, `o-que-mais-existe`)
  - Critério de pronto: passo decide a própria existência; validar/recusar → pacote curado; won/lost derivados do passo
  - Confiança: 🟢

## Tarefas de Teste
- [ ] TT-01, Marca com cor inválida não lança (motivo anotado)
- [ ] TT-02, Config: linha do banco vence o env
- [ ] TT-03, Reset apaga filhos antes de contacts
- [ ] TT-04, `canSee` esconde por role; portas essenciais sempre visíveis
- [ ] TT-05, Sugestão inválida → pacote curado com porque
- [ ] TT-06, Update: transição para não-terminal recusada

## Tarefas de Migração de Dados (se aplicável)
- [ ] TM-01, `platform_config`/`platform_settings` com segredos cifrados; `interface_settings` por org e por vínculo (0367)

## Ordem Sugerida
1. T-02 (config) e T-01 (marca) primeiro (base de plataforma).
2. T-04 (navegação) e T-07 (onboarding) dependem de auth/types e funil.
3. T-03/T-05/T-06 por último.

## Lacunas Pendentes (🔴)
- Tabelas/CHECKs/RLS de plataforma (Data Master).
- Reconferir contagens de `NAV_CATALOG`/`CATALOGO_DA_INSTALACAO` no disco.
