# Plataforma e Operação — Decisões

> Referências: `_reversa_sdd/adrs/`. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## D-01 — Banco acima do `.env`, `.env` é semente + piso de rollback 🟢
Em branding, instalação e saída: `agent.sh` reverte só a imagem, não o schema, então a verdade viva mora no banco. — `instalacao/config-resolve.ts`, `branding/instalacao.ts`. Ver ADR-0009 (packaging) e ADR-0008 (marca do banco).

## D-02 — Nunca lançar no caminho de render 🟢
`resolve.ts`, `instalacao.ts`, `saida.ts`, `contexto.tsx`, `comportamento.ts`, `config.ts` degradam e carregam um motivo em vez de lançar (um throw em `app/layout.tsx` é 500 em toda tela).

## D-03 — Marca guarda ENTRADA, nunca SAÍDA 🟢
O envelope guarda a semente/eixos de versão, nunca os 11 stops derivados, para que correções de contraste alcancem instalações existentes e o rollback continue pintado. — `branding/schema.ts`.

## D-04 — Memo em globalThis com geração 🟢
O Next instancia módulos 2× por processo (route vs page); memo `let` zerado pela rota de escrita nunca era lido pela página. `globalThis` + contador de geração contra lost-update. — `branding/instalacao.ts`.

## D-05 — Reset delega ao app, não a `security definer` 🟢
A RPC teria a org como único seletor de linha (vetor de adulteração, doutrina 0167); a exclusão vive no app com filtro explícito de org. — `settings/apagar-dados-operacionais.ts`.

## D-06 — Interface é apresentação, não autorização 🟢
`canSee` compara ROLE_RANK (única função de autorização); `combinarInterfaces` só estreita (nunca alarga); portas essenciais não removíveis. — `navigation/interface.ts`.

## D-07 — Onboarding recusa e cai em pacote curado 🟢
Recusar proposta ruim é o desfecho certo; auto-completar produziria funil incapaz de fechar. Pacotes são plano B (IA off) e a régua contra a qual a sugestão é comparada. — `onboarding/proposta-de-funil.ts`.

## D-08 — Update: a versão reportada decide, não o tempo 🟢
`rollbackFoiSuperado`/`rollbackDesmentidoPeloApp` leem `APP_VERSION` (assado na imagem); só dispatched→terminal, terminal imutável. — `system/update-run.ts`.

## D-09 — Agente alterna, humano escreve a regra 🟢
Agente só liga/desliga `automation_rules` (superfície mais perigosa: dispara para sempre sem vigilância); criar/editar/apagar é do humano; escrita ao mundo externo é `critico`. — `operacao/regras-automaticas.ts`.
