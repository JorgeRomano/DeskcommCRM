# ADRs Retroativos — DeskcommCRM

> Gerados pelo **Detetive** (Reversa) a partir de código, doutrina escrita e arqueologia Git.
> São decisões **inferidas do estado atual** do sistema — o projeto também mantém ADRs próprios em `docs/adr/`.

| # | Decisão | Status | Confiança |
|---|---|---|---|
| [0001](0001-dois-runtimes-de-ia.md) | Dois runtimes de IA convivendo (agent-engine + workers legados) | Aceito | 🟢 |
| [0002](0002-cadeia-de-guardrails-determinista.md) | Cadeia de guardrails de saída determinística e versionada | Aceito | 🟢 |
| [0003](0003-anti-morte-follow-up-e-radar.md) | Follow-up como anti-morte + Radar de Risco | Aceito | 🟢 |
| [0004](0004-multi-tenancy-rls-e-service-role.md) | Multi-tenancy por RLS com filtro manual sob service role | Aceito | 🟢 |
| [0005](0005-marca-branca-e-lgpd-controlador.md) | Marca branca do banco + PDF de LGPD nomeando o controlador | Aceito | 🟢 |
| [0006](0006-event-log-como-barramento.md) | `event_log` como barramento único de eventos | Aceito | 🟢 |

> **Nota:** o ADR oficial `docs/adr/0001-packaging-e-distribuicao.md` (packaging/distribuição self-host)
> já existe no repo e não foi reescrito aqui — é a fonte primária para o eixo de empacotamento.
