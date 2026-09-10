# Dependências — DeskcommCRM

> Gerado pelo **Scout** (Reversa) em 2026-09-10 · Fonte: `package.json` 🟢
> Gerenciador: **pnpm 9.15.9** · Runtime: **Node ≥ 22**

## Dependências de produção

### Framework e UI
| Pacote | Versão | Papel |
|---|---|---|
| `next` | ^16.3.4 | Framework web (App Router) |
| `react` / `react-dom` | ^19.2.8 | UI runtime |
| `next-themes` | ^0.4.6 | Alternância de tema |
| `tailwind-merge` | ^3.6.0 | Merge de classes Tailwind |
| `class-variance-authority` | ^0.7.0 | Variantes de componentes |
| `clsx` | ^2.1.1 | Composição de className |
| `@radix-ui/react-*` | 1.x–2.x | Primitivos acessíveis (shadcn/ui) |
| `@phosphor-icons/react` | ^2.1.10 | Ícones |
| `lucide-react` | ^1.39.0 | Ícones |
| `@hello-pangea/dnd` | ^18.0.1 | Drag-and-drop (Kanban) |
| `@xyflow/react` | ^12.11.6 | Editor de fluxos (automação) |
| `recharts` | ^3.10.1 | Gráficos |
| `sonner` | ^2.0.8 | Toasts |
| `@emoji-mart/*` | 1.x | Seletor de emoji |
| `react-hotkeys-hook` | ^5.3.3 | Atalhos de teclado |

### Formulários e validação
| Pacote | Versão | Papel |
|---|---|---|
| `react-hook-form` | ^7.87.0 | Formulários |
| `@hookform/resolvers` | ^5.9.1 | Integração de resolvers |
| `zod` | ^4.5.4 | Validação de schema (entrada de API) |

### Dados e infraestrutura
| Pacote | Versão | Papel |
|---|---|---|
| `@supabase/supabase-js` | ^2.114.0 | Cliente Supabase |
| `@supabase/ssr` | ^0.12.5 | SSR/cookies Supabase |
| `pg` | ^8.23.0 | Driver Postgres direto |
| `@upstash/redis` | ^1.38.3 | Redis (rate limit, cache, fila) |
| `@tanstack/react-query` | ^5.102.8 | Estado de servidor no cliente |
| `@tanstack/react-virtual` | ^3.14.10 | Virtualização de listas |

### IA e agentes
| Pacote | Versão | Papel |
|---|---|---|
| `ai` | ^7.0.90 | Vercel AI SDK |
| `@ai-sdk/anthropic` | ^4.0.16 | Provider Anthropic |
| `@ai-sdk/openai` | ^4.0.56 | Provider OpenAI |
| `@ai-sdk/google` | ^4.0.18 | Provider Google |
| `@modelcontextprotocol/sdk` | ^1.30.0 | MCP (Model Context Protocol) |
| `gpt-tokenizer` | ^4.0.0 | Contagem de tokens |

### Integrações e utilitários
| Pacote | Versão | Papel |
|---|---|---|
| `resend` | ^6.25.0 | Envio de e-mail |
| `web-push` | ^3.6.7 | Notificações push |
| `qrcode` | ^1.5.4 | QR (pareamento de canal WhatsApp) |
| `@react-pdf/renderer` | ^4.9.0 | Geração de PDF (LGPD) |
| `pdfjs-dist` | ^6.3.289 | Leitura de PDF |
| `date-fns` | ^4.4.0 | Datas |
| `fflate` | ^0.8.3 | Compressão |
| `@sentry/nextjs` | ^10 | Observabilidade/erros |
| `import-in-the-middle` / `require-in-the-middle` | 3.x / 8.x | Instrumentação |

## Dependências de desenvolvimento

| Pacote | Versão | Papel |
|---|---|---|
| `typescript` | ^6.0.3 | Compilador (estrito) |
| `typescript-eslint` | ^8.69.0 | Lint TS |
| `eslint` | ^9.12.0 | Lint |
| `eslint-config-next` | ^16.2.10 | Regras Next |
| `prettier` (+ `prettier-plugin-tailwindcss`) | ^3.9.6 | Formatação |
| `vitest` (+ `@vitest/coverage-v8`) | ^4.1.11 | Testes unitários |
| `@playwright/test` | ^1.62.1 | Testes E2E |
| `@axe-core/playwright` | ^4.13.0 | Acessibilidade E2E |
| `@testing-library/{react,user-event,jest-dom}` | 16.x / 14.x / 7.x | Testes de componente |
| `jsdom` | ^30.0.1 | DOM para testes |
| `tailwindcss` (+ `@tailwindcss/postcss`) | ^4 | CSS |
| `postcss` | ^8.5.26 | Pipeline CSS |
| `tsx` | ^4.23.13 | Runner TS (workers, scripts) |
| `vite` | ^8.2.2 | Build de testes |
| `@types/*` | — | Tipos (node, react, pg, qrcode, web-push) |
| `@vercel/config` | ^0.7.0 | Config Vercel |

## Overrides pnpm (pisos de segurança em transitivas) 🟢

Aplicados para elevar versões de dependências transitivas com advisory, sem arrastar o resto:
`postcss ^8.5.26`, `sharp ^0.35.0`, `hono ^4.12.34`, `js-yaml ^4.3.1`,
`brace-expansion@1 ^1.1.18`, `path-to-regexp@6 ^6.3.0`, `fast-uri ^3.1.7`,
`qs ^6.16.0`, `browserslist ^4.28.8`.

> Os seletores com `@<major>` são intencionais (LOAD-BEARING): o lock tem árvores
> duplas de `brace-expansion` (1.x/5.x) e `path-to-regexp` (6.x/8.x); sem o escopo,
> o override rebaixaria a árvore nova para a antiga silenciosamente.

## Dependências externas de runtime (não-npm) 🟡

- **WAHA 2026.7.2** (engine NOWEB) — serviço WhatsApp, referenciado por tag fixa (licenciado).
- **Supabase** (Postgres + Auth + Realtime + Storage).
- **Upstash Redis** — fallback em memória por processo quando ausente.
- **Resend** — e-mail transacional.
- **Nuvemshop** — integração de e-commerce (`lib/nuvemshop/`).
- **Provedores de IA** — Anthropic / OpenAI / Google (via AI Gateway).
