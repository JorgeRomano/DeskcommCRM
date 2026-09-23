# ADR-0001 — Multi-tenancy com RLS no banco desde o dia 1

> ADR **retroativo** reconstruído pelo Detetive (Reversa) a partir do código e do histórico.
> Confiança: 🟢 CONFIRMADO (imposto por código, doutrina e testes de invariante).
> Data inferida: fundacional (anterior ao histórico Git disponível).

## Status
Aceito (vigente).

## Contexto
O DeskcommCRM é multi-tenant e **self-host**: cada instalação numa VPS pode hospedar várias
organizações, e quem instala é o próprio usuário. Um vazamento cross-tenant não é bug de tela, é
vazamento de dado de cliente de terceiros. A pergunta fundadora foi: **onde mora o isolamento?**

## Decisão
O isolamento vive **no banco**, via Row Level Security (RLS) em toda tabela tenant-aware
(`organization_id uuid not null references organizations(id) on delete cascade`), usando os helpers
`SECURITY DEFINER` `fn_user_org_ids()` / `fn_user_role_in_org()` — as **mesmas** funções que o RBAC
de aplicação consulta. A aplicação nunca é a única barreira. Onde o service role
(`lib/supabase/admin.ts`) bypassa RLS, o filtro de `organization_id` é **manual e obrigatório**,
resolvido de fonte confiável (cookie/JWT/webhook secret/path token), **nunca do body** (fix da
issue #236). O job `invariants` do CI roda `test:db` provando o isolamento cross-tenant.

## Alternativas consideradas
1. **Isolamento só na aplicação (filtro `WHERE organization_id`)** — rejeitado: um único handler que
   esquece o filtro vaza tudo, e não há gate automático que pegue todos.
2. **Um banco/schema por tenant** — rejeitado: inviável para self-host com N orgs numa VPS modesta;
   custo operacional de migração por tenant; contradiz "bump de versão não edita arquivo à mão".
3. **RLS + filtro manual no service role (escolhida)** — a RLS cobre o caminho comum; o filtro
   manual cobre o caminho de service role, com a responsabilidade explicitada na doutrina.

## Consequências
- **Positivas:** defesa em profundidade; o banco barra mesmo se a aplicação falhar; testável por
  invariante determinístico.
- **Negativas / custo:** o service role é um furo por construção — não há gate automático para o
  filtro de `organization_id`, então cada handler novo carrega essa responsabilidade (RN-02). Boa
  parte de `app/api/**` usa service role, ampliando a superfície de atenção.
- **Derivada:** `ai_operator` jamais entra em `user_organizations` para não furar as FKs
  `_by_user_id` nem o gate `pre_go_live` (ver ADR-0004).
