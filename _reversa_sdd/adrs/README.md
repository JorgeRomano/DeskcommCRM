# ADRs Retroativos — DeskcommCRM

> Gerados pelo Detetive (Reversa) — fase de Interpretação · nível **detalhado**.
> São **retroativos**: reconstruídos do código, da doutrina e do histórico Git, não escritos no
> momento da decisão. Cada um traz **Alternativas consideradas** e **Consequências** (nível detalhado).
> Confiança: todos 🟢 CONFIRMADO (impostos por código/doutrina/testes), salvo notas internas.

| # | Título | Tema |
|---|---|---|
| [0001](0001-multi-tenancy-com-rls-desde-o-dia-1.md) | Multi-tenancy com RLS desde o dia 1 | Isolamento / segurança |
| [0002](0002-event-sourcing-leve-trigger-nunca-faz-http.md) | Event sourcing leve; trigger nunca faz HTTP | Arquitetura / fila |
| [0003](0003-ritual-do-turno-e-fechamento-por-checkpoint.md) | Ritual do turno e fechamento por checkpoint | Agente de IA |
| [0004](0004-guardrails-deterministicos-before-send.md) | Cadeia determinística `before_send` | Guardrails |
| [0005](0005-restricao-de-canal-feature-nunca-nomeia-provedor.md) | Restrição de canal (capabilities) | Canais |
| [0006](0006-webhooks-fail-closed-e-efeitos-pos-entrada-ordenados.md) | Webhooks fail-closed; efeitos pós-entrada ordenados | Ingestão / segurança |
| [0007](0007-pipeline-imutavel-mover-cross-pipeline-e-clonar.md) | Pipeline imutável; mover é clonar | CRM / funil |
| [0008](0008-marca-propria-resolve-do-banco-nunca-do-env.md) | Marca própria resolve do banco | White-label |
| [0009](0009-packaging-imagem-publicada-bump-sem-editar-vps.md) | Packaging: só imagem publicada | Distribuição / self-host |
| [0010](0010-crm-como-servidor-mcp-com-rbac-e-recusa-para-o-modelo.md) | CRM como servidor MCP com RBAC | Capacidades do agente |

> Estes 10 ADRs cobrem as decisões fundadoras. Decisões mais finas (ex.: fuso da organização na
> agenda, ausência de citação não trava conversa, falha de classificador não cala o agente) estão
> documentadas como **regras de negócio** em `domain.md` (§2), com o commit de origem quando há.
