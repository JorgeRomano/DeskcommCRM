# ADR 0005 — Marca branca resolvida do banco e PDF de LGPD nomeando o controlador

> ADR retroativo · Status: **Aceito** · Confiança: 🟢 CONFIRMADO (`lib/branding/`, `lib/lgpd/pdf-renderer.tsx`, `docs/white-label.md`)

## Contexto

O produto é open source e **revendido** (white-label): quem instala numa VPS é o usuário, e o nome não pertence aos mantenedores. Ao mesmo tempo, o produto tem obrigações de LGPD (relatório do Art. 18 II), onde o revendedor é **operador**, não controlador.

## Decisão

1. **Marca resolvida por camadas** (`resolverMarca`): organização → instalação (banco: `platform_branding`, `organizations.settings.branding`) → `.env` (semente e piso de rollback) → padrão do produto. Precedência por campo (ausente numa camada não apaga a de baixo).
2. **O resolvedor NUNCA lança** — roda em `app/layout.tsx`, e um throw ali é 500 em todas as telas; degrada para o padrão.
3. **"Deskcomm"/"DeskcommCRM" proibido em código que alcança o usuário** (`tests/unit/branding.test.ts`; allowlist só encolhe).
4. **O PDF de LGPD NÃO leva marca** — nomeia o CONTROLADOR (`organizations.legal_name`) e o DPO. Documento jurídico não pode inverter papéis (revendedor é operador).
5. **`marcaDaSaida`** para saídas sem DOM (e-mail, MFA issuer): sempre tema claro, accent + contraste, degrada para padrão (o e-mail de LGPD tem SLA legal — falhar por cor trocaria estética por descumprimento).

## Alternativas consideradas

1. **Marca só do `.env` (build-time).** Rejeitada: exigiria rebuild por cliente; o banco permite trocar sem redeploy. `.env` fica como semente/piso.
2. **Envelope de cor gravando os 11 stops derivados.** Rejeitada: congelaria a instalação no gerador do dia; grava só `{semente_hex, papel}`.
3. **PDF com a marca do revendedor.** Rejeitada: inverteria controlador↔operador num documento legal.

## Consequências

- **Positivas:** revenda sem rebuild; conformidade jurídica no PDF; nenhuma tela quebra por falha de marca.
- **Negativas:** resolução por camadas é complexa (cache WeakMap, versões de FORMA/ALGORITMO); dois caminhos de marca (DOM via layout, saída via `marcaDaSaida`).
