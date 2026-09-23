# Lacunas — DeskcommCRM

> Gerado pelo Revisor (Reversa) em 2026-09-23 · `doc_level = detalhado`
> Lacunas categorizadas por severidade. Uma lacuna aqui **não** é erro da spec: é honestidade sobre
> o que o código não deixa determinar sozinho. Perguntas ao usuário estão em [`questions.md`](questions.md).

## Legenda de severidade

- **Crítico** — bloqueia reimplementação fiel; precisa de resposta antes de codar a unit.
- **Moderado** — não bloqueia, mas a spec fica incompleta ou com número/valor a reconferir.
- **Cosmético** — imprecisão de rastreabilidade (âncora de linha, atribuição de arquivo) sem efeito no design.

---

## Crítico

Nenhuma lacuna **bloqueante** foi encontrada nas 14 units. As 6 lacunas 🔴 que exigiam decisão
humana (`questions.md`) foram **todas respondidas em 2026-09-23** e reclassificadas para 🟢:
voz/SIP (dependência externa), ponte WebRTC (esqueleto por desenho), transições de agendamento
(ator: IA + operador), enum de prospecção (`search_status` fechado pela migration 0369), política de
reconciliação (prevenir duplicata a todo custo) e catálogo de modelos (`gateway.ts` é canônico).

## Moderado

1. **Camada de banco inteira delegada ao Data Master.** Todas as units que tocam dados marcam 🔴 os
   corpos de RPCs SECURITY DEFINER, policies RLS, CHECKs e triggers. As specs cobrem quem **invoca**,
   não o corpo SQL. Afeta: `auth-tenancy-rbac`, `compliance`, `plataforma-operacao`,
   `integracoes-externas`, `infra-transversal-relatorios`, `superficie-http`, `agenda-financeiro`,
   `crm-funil`. **Ação:** rodar o agente **Data Master** (`/reversa` → agente independente) para
   documentar o schema e as funções, fechando estas lacunas de uma vez.
   - `fn_user_role_in_org`, `fn_is_platform_admin`, `fn_accept_team_invite`, `fn_support_context`
   - `fn_extensions_*`, `fn_activity_report`, `fn_atrito_metrics`
   - Tabelas/RLS: `webhook_sources`, `api_tokens`, `conversations`, `contacts`, `crm_leads`,
     `platform_branding`, `platform_config`, `platform_settings`, `idempotency_keys`, view `_safe`,
     policy `idempotency_tenant`, CHECK 0089/`crm_stages_hint_coerente`.

2. **~337 rotas de `app/api/v1/**` não lidas uma a uma** (`superficie-http`). A cobertura é por
   amostra representativa de superfície + contagens estruturais. **Ação:** ao aprofundar
   `superficie-http`, mapear cada endpoint e reconferir a contagem com
   `git ls-files 'app/api/**/route.ts' | wc -l` (não fixar o número na prosa).

3. **24 schemas de entidade em `lib/schemas/` vistos pelo registro/validação, não campo a campo**
   (`infra-transversal-relatorios`). **Ação:** conferência campo a campo ao escrever specs por entidade.

4. **Valores exatos de env/knobs de produção** (`nucleo-ia-agente`, janelas/caps/thresholds em
   `env.ts`/`turn-knobs.ts`). Corretamente **não fixados em prosa**. **Ação:** confirmar defaults na
   instalação antes de reimplementar.

5. **Contagens de `NAV_CATALOG` / `CATALOGO_DA_INSTALACAO`** (`plataforma-operacao`) a reconferir no
   disco com `git grep`/`wc -l` em vez de citar número.

6. **Comportamento por provedor externo sob erro** (`integracoes-externas`, rate limit / saldo zero /
   API desativada) inferido do mapeamento `classificaErro`/`classifica4xx` — 🟡, validar com casos reais.

## Cosmético

1. **[voz-telefonia] Atribuição do código `voice_estado_indeterminado`.** O terceiro "código honesto"
   (503 indeterminado) vive em `lib/voice/guarda.ts` (`exigirVozLigada`), não em `lib/voice/opt-in.ts`
   (`estadoDaVoz.motivo`, que tem só `ligada | instalacao_nao_oferece | organizacao_nao_ligou`). Ambos
   existem e conferem; é imprecisão de âncora, não erro de fato. **Ação sugerida:** deixar explícito no
   critério que "indeterminado" é do guard. Não corrigido in-place por não alterar o design.

2. **[agenda-financeiro ↔ automacao-roteamento] Semântica de `windows` vazio** ("zero horário" na
   agenda vs "24/7" no roteamento). **Não é contradição** — é divergência intencional por subsistema,
   as duas units se citam mutuamente e o código trata cada caso com operador diferente (`&&` vs `??`).
   Registrado como resolvido/consciente.
