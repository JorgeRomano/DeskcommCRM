# CRM e Funil — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — Score sem lastro citável 🟢
Score exige `checkpointId` + ao menos um fator com âncora. Sem lastro → `score=null`, `semSinal ∈ {sem_lastro_citavel, sem_conteudo, negocio_fechado}`. Número velho é apagado (null é melhor que velho).

## EC-02 — "Sem resposta há 2 dias" 🟢
É normal num contrato e abandono num agendamento. A janela de risco é por estágio (`expected_duration_hours`), não global.

## EC-03 — `since` no futuro 🟢
`risk-since.ts` clampa em `agora`: os atalhos de agenda atribuem bucket sem cruzar limiar, e futuro violaria `check (since <= detected_at)` derrubando o worker.

## EC-04 — 3 mensagens juntas 🟢
Nascimento com advisory lock por org+contato; sem ele, 3 mensagens criaram 3 cards em produção.

## EC-05 — Vários leads abertos para o contato 🟢
`resolveActiveLeadForContact`: empate no topo → `ambiguous_open_leads` (não escolhe; mover card errado é o único bug visível ao cliente).

## EC-06 — Perda que não é perda real 🟢
`moved_to_another_pipeline` é canônico e excluído das métricas; nunca ofertado na tela (seria 1 clique para tirar perda real da métrica).

## EC-07 — Card arrastado por humano e IA ao mesmo tempo 🟢
Trava otimista `.eq("stage_id", origem)` → 0 linhas = `conflito_humano`. Erro de SELECT é lido (supabase-js não lança em rede) → `indisponivel`.

## EC-08 — Meta fora do ar 🟢
Conversão é handler de evento, não chamada em `encerramento.ts`: a Meta fora do ar nunca bloqueia uma venda.

## EC-09 — Knobs de prospecção incompletos 🟢
`tetoDiarioDaEsteiraFria` falha fechada (teto 1, nunca "sem teto"); o número que morre é o do cliente. Jitter só atrasa (cadência exata é assinatura de robô).

## EC-10 — Dono com agente desligado 🟢
Dono agente resolvido sem filtrar `is_active` (exibir dono é obrigatório mesmo com agente desligado); nome null → rótulo genérico, nunca "Sem responsável".

## EC-11 — Nomes que colidem por acento 🟢
`chaveDeNome` normaliza NFD sem acento ("Pos venda" colide com "Pós-venda").

## EC-12 — Local echo (minha própria mudança) 🟢
O que eu mudei não pulsa na minha tela: marca segue o ciclo de vida da mutação; `ehEcoLocal` não consome a marca (defeito 12.c).

## EC-13 — CPF sem cifra at-rest 🔴
`encrypt_cpf` RPC não provisionada; degrada para `cpf_hash` only com warn. Lacuna de segurança a resolver.
