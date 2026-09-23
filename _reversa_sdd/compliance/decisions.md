# Compliance — Decisões

> Referências: `_reversa_sdd/adrs/`. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## D-01 — Audit fire-and-forget, mas reportado 🟢
Falha de auditoria nunca bloqueia a mutação primária (doutrina), mas é reportada ao Sentry — senão a trilha inteira poderia parar sem ninguém perceber. — `audit/index.ts`.

## D-02 — Append-only sem GRANT de UPDATE/DELETE 🟢
`api_audit_log` é append-only por schema (nem service role atualiza/apaga); por isso segredo nunca entra no metadata (ficaria os 5 anos da retenção). — `audit/actions.ts`.

## D-03 — O que se apaga é o que se entrega 🟢
Invariante export × cascata: o gate deriva a lista das duas pontas; campos obrigatórios fazem caminho novo não compilar. — `lgpd/export-collector.ts`.

## D-04 — Anonimização falha aberta 🟢
A cascata aborta antes de deixar PII órfão (foto enfileirada antes de zerar o ponteiro; falha fechada no enfileiramento). — `lgpd/redact-cascade.ts`.

## D-05 — PDF nomeia o controlador, não a marca 🟢
O revendedor é OPERADOR, não controlador; nomeá-lo inverteria papéis num documento jurídico. O PDF imprime `legal_name` e o Encarregado. Ver ADR-0008 (marca resolve do banco). — `lgpd/pdf-renderer.tsx`.

## D-06 — País entra com citação revisada ou não entra 🟢
Sem fallback para a LGPD: citar a lei errada é pior que nenhuma. `paisesOferecidos()` só inclui país com citação revisada. — `legal/perfil-do-pais.ts` (issue #1033).

## D-07 — Operador é quem responde pela instalação 🟢
MIT self-host: quem instala opera e responde; os documentos nomeiam o operador (client de sessão, nunca service role em `/legal/*` público). — `legal/operador.ts`.

## D-08 — Opt-out por intenção, dois níveis 🟢
Verbo de cessação + objeto de comunicação; inequívoco bloqueia (estado que só pessoa desfaz), ambíguo só escala. Deixar o ambíguo bloquear sozinho inverteria a política. — `opt-out/deteccao.ts`.

## D-09 — Piso de retenção no SQL 🟢
`greatest(..., piso)` dentro das funções vale para qualquer chamador (inclusive `psql`); a cópia no TS serve para o operador ver no log que o valor foi elevado. — `retencao/politica.ts`.

## D-10 — Retenção fora de `lib/env.ts` 🟢
`lib/env.ts` lança no import; um valor digitado errado derrubaria o produto (contêiner healthy com 500). `interpretarRetencao` degrada com aviso em vez de lançar. — `retencao/politica.ts`.
