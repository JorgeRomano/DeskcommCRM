# Dependências — DeskcommCRM

> Gerado pelo **Scout** (Reversa) em 2026-09-22 · fonte: `package.json` 🟢
> Regra do projeto: declarar apenas a **major** em prosa; a versão exata é o `package.json`.
> Gerenciador: **pnpm 9.15.9** · Node **≥ 22** · `frozen-lockfile` no CI.

## Runtime (`dependencies`)

### IA / Agentes
| Pacote | Range | Papel |
|---|---|---|
| `ai` | ^7.0.103 | Vercel AI SDK — orquestração de LLM |
| `@ai-sdk/anthropic` | ^4.0.16 | provider Anthropic (primário) |
| `@ai-sdk/openai` | ^4.0.68 | provider OpenAI (embeddings) |
| `@ai-sdk/google` | ^4.0.18 | provider Google |
| `@modelcontextprotocol/sdk` | ^1.30.0 | CRM exposto por MCP |
| `gpt-tokenizer` | ^4.0.0 | contagem de tokens / custo |

### Framework / UI
| Pacote | Range | Papel |
|---|---|---|
| `next` | ^16.3.5 | Next.js App Router (Turbopack) |
| `react` / `react-dom` | ^19.3.0 | React 19 |
| `next-themes` | ^0.4.6 | tema claro/escuro |
| `@radix-ui/*` | 1.x–2.x | primitivas shadcn/ui (dialog, select, tabs, tooltip, dropdown, popover, etc.) |
| `@phosphor-icons/react` / `lucide-react` | 2.x / 1.x | ícones |
| `class-variance-authority` / `clsx` / `tailwind-merge` | — | composição de classes Tailwind |
| `sonner` | ^2.0.8 | toasts |
| `@hello-pangea/dnd` | ^18.0.1 | drag-and-drop (kanban) |
| `@xyflow/react` | ^12.11.6 | editor de fluxos |
| `recharts` | ^3.10.1 | gráficos |
| `@tanstack/react-query` (+ devtools, virtual) | 5.x / 3.x | data fetching / virtualização |
| `react-hook-form` + `@hookform/resolvers` | 7.x / 5.x | formulários |
| `react-hotkeys-hook` | ^5.3.3 | atalhos de teclado |
| `@emoji-mart/data` + `@emoji-mart/react` | 1.x | seletor de emoji |

### Dados / Backend
| Pacote | Range | Papel |
|---|---|---|
| `@supabase/supabase-js` | ^2.116.0 | cliente Supabase |
| `@supabase/ssr` | ^0.12.7 | auth SSR (cookies) |
| `pg` | ^8.23.0 | node-postgres (acesso direto/workers) |
| `@upstash/redis` | ^1.38.4 | Redis (rate limit, cache) |
| `zod` | ^4.6.5 | validação de schema |

### Comunicação / Mídia
| Pacote | Range | Papel |
|---|---|---|
| `nodemailer` | 10.0.10 | envio de e-mail (pinado exato) |
| `resend` | ^6.28.1 | e-mail transacional |
| `web-push` | ^3.6.7 | push notifications |
| `ws` | ^8.21.3 | WebSocket |
| `qrcode` | ^1.5.4 | QR de sessão WhatsApp |
| `@react-pdf/renderer` | ^4.9.0 | PDF (LGPD) — patch em `@react-pdf/hyphenate` |
| `pdfjs-dist` | ^6.3.289 | leitura de PDF |
| `fflate` | ^0.8.3 | compressão |

### Observabilidade / Infra
| Pacote | Range | Papel |
|---|---|---|
| `@sentry/nextjs` | ^10 | monitoramento (`beforeSend` higieniza PII) |
| `import-in-the-middle` / `require-in-the-middle` | 3.x / 8.x | instrumentação |
| `ipaddr.js` | ^2.5.0 | parsing de IP (rate limit por IP) |
| `jsonc-parser` | ^3.3.1 | parsing de JSONC |
| `date-fns` | ^4.4.0 | datas |

## Desenvolvimento (`devDependencies`)

### Testes
| Pacote | Range |
|---|---|
| `vitest` + `@vitest/coverage-v8` | 4.x |
| `@playwright/test` + `@axe-core/playwright` | 1.x / 4.x |
| `@testing-library/react` + `/jest-dom` + `/user-event` | 16.x / 7.x / 14.x |
| `jsdom` | ^30.0.1 |

### Build / Lint / Tipos
| Pacote | Range |
|---|---|
| `typescript` | ^6.0.3 |
| `eslint` + `eslint-config-next` + `typescript-eslint` | 9.x / 16.x / 8.x |
| `prettier` + `prettier-plugin-tailwindcss` | 3.x / 0.8.x |
| `tailwindcss` + `@tailwindcss/postcss` + `postcss` | 4.x / 4.x / 8.x |
| `tsx` | ^4.23.13 |
| `vite` | ^8.3.0 |
| `@types/*` | node 22.x, react 19.x, pg, ws, qrcode, nodemailer, web-push, jest |

## Overrides (pisos de versão para transitivas com advisory) 🟢

`postcss ^8.5.28`, `sharp ^0.35.0`, `hono ^4.12.34`, `js-yaml ^4.3.1`,
`brace-expansion@1 ^1.1.18`, `path-to-regexp@6 ^6.3.0`, `fast-uri ^3.1.7`,
`qs ^6.16.0`, `browserslist ^4.28.8`.

**Patch aplicado:** `@react-pdf/hyphenate` → `patches/@react-pdf__hyphenate.patch`.

## Notas

- 🟡 As versões acima são os **ranges declarados**; a versão resolvida está em `pnpm-lock.yaml` (não lido nesta fase).
- 🟢 `nodemailer` está **pinado exato** (`10.0.10`), diferente das demais.
- 🟢 Os seletores `@<major>` nos overrides são intencionais (duas árvores de `brace-expansion` e de `path-to-regexp` no lock).
