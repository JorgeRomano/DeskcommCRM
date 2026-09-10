# Onboarding e Instalação — Design Técnico

> Redator (Reversa) · Nível: Detalhado · 🟢 CONFIRMADO · 🟡 INFERIDO

## Interface 🟢

| Símbolo | Assinatura | Retorno |
|---------|-----------|---------|
| `passosVisiveis` | `(ctx: ContextoDoPasso)` | `PassoDoOnboarding[]` |
| `proximoPasso` | `(state: OnboardingState, ctx)` | `PassoDoOnboarding \| null` |
| `resumoDoOnboarding` | `(state, ctx)` | `ItemDoResumo[]` |
| `lerAmbiente` | `(source?: FonteDeAmbiente)` | `AmbienteDaInstalacao` |
| `nomeAindaEhPlaceholder` | `(org)` | `boolean` |
| `sugerirFunil` | `(ctx, gerar)` | `Promise<Sugestao>` 🟡 |

### DTOs 🟢

- `PassoDoOnboarding { segmento, rotulo, existe(ctx), cumprido(state), pulado(state) }`
- `ContextoDoPasso { lojaLigada: boolean }`
- `AmbienteDaInstalacao { chavesDeProvedor: Record<string,boolean>, gateway, email, transporteDeWhatsapp }`

## Fluxo Principal 🟢

1. `ContextoDoPasso` resolve `lojaLigada` (integração ligada?).
2. `passosVisiveis` filtra `PASSOS` por `existe(ctx)`.
3. `proximoPasso` = primeiro visível não `cumprido`.
4. `resumoDoOnboarding` lista visíveis com `feito`/`pulado`.
5. `lerAmbiente` inspeciona o `.env` (chaves por provedor, gateway, email, transporte).

## Fluxos Alternativos 🟢

- **Loja desligada:** `connect-nuvemshop` não vira passo (nem pendência nem culpa no resumo).
- **Env vazio:** tratado como ausente.
- **Sugestão de funil falha:** `PACOTE_PADRAO` por nicho (regex `PISTAS`).

## Dependências 🟢

- `schemas/onboarding` (`OnboardingState`), `channels/transporte` (transporte WhatsApp), `env`.
- Sugestão de funil usa o mesmo cérebro/chave que atende (IA).

## Decisões de Design Identificadas 🟢

| Decisão | Evidência | Confiança |
|---------|-----------|-----------|
| Fonte única de passos (roteador+indicador+resumo) | `passos.ts` | 🟢 |
| Passo existe se aplicável (sem fantasma) | `passos.ts:existe` | 🟢 |
| Google sem variável de chave (não inventar) | `ambiente.ts:VARIAVEL_DA_CHAVE` | 🟢 |
| Sugestão nunca vazia (fallback) | `sugerir-funil.ts` | 🟡 |

## Estado Interno 🟢

`OnboardingState` (welcome/whatsapp/nuvemshop/ai/funil/teste/team, cada um com `skipped?`). `organizations.slug/display_name` (placeholder).

## Observabilidade 🟡

- Progresso do wizard refletido na UI (indicador). Sem log específico lido.

## Riscos e Lacunas

- 🟡 `lib/onboarding/sugerir-funil.ts`, `pacotes-de-funil.ts`, `proposta-de-funil.ts` não lidos em profundidade.
- 🟡 `lib/instalacao/{prova-de-credito,retrato}` não lidos.
- 🔴 `hostgator-setup-kit/` (kit de instalação assistida) não analisado.
