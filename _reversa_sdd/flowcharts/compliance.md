# Fluxogramas — Compliance (LGPD, legal, opt-out, retenção, audit)

> Gerado pelo Arqueólogo (Reversa) — Unidade 9.
> Cobre `lib/lgpd/`, `lib/legal/`, `lib/opt-out/`, `lib/retencao/`, `lib/audit/`.
> Nível `detalhado`: fluxograma por módulo + por função de lógica não-trivial.

---

## 1. Trilha de auditoria — `audit` (`audit/index.ts`)

```mermaid
flowchart TD
  A[audit entry] --> B{isServiceRoleConfigured?}
  B -- sim --> C[admin client bypassa RLS]
  B -- não --> C2[client de sessão: policy audit_log_insert_tenant_member]
  C --> D{resolve contexto de suporte?}
  C2 --> D
  D -- state OAuth assinado --> D1[platform_support_sessions por actor+auth_session]
  D -- sessão cookie --> D2[readSupportContext]
  D1 --> E[insert api_audit_log append-only]
  D2 --> E
  E --> F{erro?}
  F -- sim --> G[reportAuditFailure: console.error + Sentry]
  F -- não --> H[fim: NUNCA bloqueia a mutação primária]
  G --> H
```

> Fire-and-forget: a falha do audit não bloqueia a mutação (doutrina), mas é barulhenta (Sentry) —
> senão a trilha inteira poderia parar sem ninguém ver.

---

## 2. SLA em dias úteis — `computeDueAt` (`lgpd/sla.ts`)

```mermaid
flowchart TD
  A[computeDueAt receivedAt, businessDays, holidays] --> B[cursor = UTC midnight do dia recebido]
  B --> C{cursor é dia útil?<br/>não sáb/dom, não feriado}
  C -- não --> D[avança até o próximo dia útil]
  C -- sim --> E[remaining = businessDays]
  D --> E
  E --> F{remaining > 0?}
  F -- sim --> G[avança 1 dia]
  G --> H{novo dia é útil?}
  H -- sim --> I[remaining--]
  H -- não --> F
  I --> F
  F -- não --> J[return cursor = fim do N-ésimo dia útil]
```

---

## 3. Cascata de anonimização — `cascadeRedactContact` (`lgpd/redact-cascade.ts`)

```mermaid
flowchart TD
  A[cascadeRedactContact] --> B[lê avatar_storage_path do contato]
  B --> C{tem avatar?}
  C -- sim --> D[upsert storage_redaction_queue<br/>onConflict bucket,object_path]
  D --> E{enfileirou?}
  E -- não --> EX[THROW: falha FECHADA<br/>não zera o ponteiro, aborta]
  E -- sim --> F[zera avatar_storage_path]
  C -- não --> G
  F --> G[rpc fn_lgpd_cascade_redact_contact<br/>transação: contacts irreversível, conversas,<br/>mensagens, atividades, leads, orders, fila de mídia, audit]
  G --> H{erro?}
  H -- sim --> HX[THROW rpc failed]
  H -- não --> I[return alreadyAnonymized, counts, mediaPaths]
```

> A foto é enfileirada ANTES da RPC de propósito: a RPC zera o ponteiro; enfileirar depois deixaria o
> arquivo órfão no bucket (rosto guardado de quem foi "anonimizado").

---

## 4. Passos 2-4 idempotentes — `completarRedacaoDoContato` (`lgpd/cascata.ts`)

```mermaid
flowchart TD
  A[completarRedacaoDoContato contato] --> B[SELECT leads do contato]
  B --> C{para cada lead: jaRedigida title?}
  C -- sim --> C1[pula: guarda do sufixo]
  C -- não --> C2[UPDATE title = slice20 + sufixo]
  C1 --> D
  C2 --> D[SELECT atividades: filtra payload.redacted != true]
  D --> E{há pendentes?}
  E -- sim --> E1[UPDATE payload = redacted:true]
  E -- não --> F
  E1 --> F[SELECT followup_enrollments em STATUS_DA_REGUA_VIVA]
  F --> G{há régua viva?}
  G -- sim --> G1[UPDATE status=cancelled, cancel_reason,<br/>next_eval_at=null, claimed_until=null]
  G -- não --> H
  G1 --> H[retorna tabelas REALMENTE tocadas + falhas]
```

> SELECT antes de UPDATE em todos os passos: sem isso a varredura diária reescreveria dado já certo e a
> auditoria registraria efeito todo dia. `cancelled` não está em STATUS_DA_REGUA_VIVA → a 2ª passada não
> acha nada (idempotência).

---

## 5. Varredura sem clique — `varrerRedacoesIncompletas` (`lgpd/cascata.ts`)

```mermaid
flowchart TD
  A[cron diário] --> B[SELECT contacts is_anonymized=true<br/>limit MAX_CONTATOS_EXAMINADOS=5000]
  B --> C[para cada bloco de 100: detecta resíduo<br/>2 consultas por bloco]
  C --> D[pendentes = contatos com lead/atividade não redigida]
  D --> E[completarRedacaoDoContato nos primeiros<br/>MAX_CONTATOS_POR_VARREDURA=200]
  E --> F[temResto = pendentes > 200 OU examinados >= 5000]
  F --> G[return examinados, comResiduo, completados, temResto]
```

> Dois tetos DIFERENTES (5000 leitura, 200 conserto) evitam starvation: um só faria a rodada olhar
> sempre os mesmos 200 e nunca alcançar o contato 201.

---

## 6. Export de acesso — invariante redige ⇔ exporta (`lgpd/export-collector.ts`)

```mermaid
flowchart LR
  A[Cascata de redação<br/>apaga PII do titular] --> C{gate deriva a lista<br/>das DUAS pontas}
  B[collectExportData<br/>entrega dado do titular] --> C
  C --> D[tests/unit/lgpd-exporta-o-que-redige.test.ts]
  D -->|desalinhou| E[CI reprova: bloco novo na cascata<br/>sem bloco no export]
  D -->|alinhado| F[Art. 18 II: o que se apaga = o que se entrega]
```

```mermaid
flowchart TD
  A[collectExportData] --> B[lerControlador: legal_name, dpo, country<br/>NUNCA lança: vazio -> rodapé com traço]
  B --> C{tem contactId ou externalCustomerId?}
  C -- não --> CX[emptyPayload: relatório vazio<br/>ainda nomeia o controlador]
  C -- sim --> D[resolve contactId por external se preciso]
  D --> E[agrega blocos: contact, consents, conversations,<br/>messages, leads, orders, appointments, sales, tasks,<br/>voice_calls, cases, passagens, ... PII só no payload, nunca no log]
  E --> F[campos obrigatórios: case_chat_messages, passagens<br/>-> caminho novo não COMPILA se esquecer]
  F --> G[return ExportPayload]
```

---

## 7. Perfil do país e citação da lei — `perfilDaOrganizacao` (`legal/perfil-do-pais.ts`)

```mermaid
flowchart TD
  A[perfilDaOrganizacao orgId] --> B[SELECT organizations.country]
  B --> C{country vazio/null?}
  C -- sim --> D[PERFIL_BR: Brasil é o padrão]
  C -- não --> E{PERFIS_DO_PAIS tem o código?}
  E -- não --> F[degrada para Brasil + RASTRO console+Sentry]
  E -- sim --> G[perfil do país]
  D --> H
  G --> H[citacaoDaLei]
  F --> H
  H --> I{lei.revisada === true?}
  I -- sim --> I1[cita: LGPD Art. 18, II Lei nº 13.709/2018]
  I -- não --> I2[null: documento NÃO cita lei<br/>sem fallback para a LGPD]
```

---

## 8. Opt-out por intenção — `ehPedidoDeOptOut` / `ehOptOutProvavel` (`opt-out/deteccao.ts`)

```mermaid
flowchart TD
  A[texto] --> B[normalizarTexto: minúsculas, sem acento]
  B --> C{mensagem inteira = palavra isolada?<br/>stop/parar/sair/baja/salir...}
  C -- sim --> INEQ[INEQUÍVOCO]
  C -- não --> D{casa FRASES_DE_OPT_OUT?<br/>verbo de cessação + OBJETO de comunicação<br/>lookahead exclui pedido/boleto/fatura}
  D -- sim --> INEQ
  D -- não --> E{casa FRASES_AMBIGUAS?<br/>me deixa em paz / ya basta}
  E -- sim --> AMB[AMBÍGUO]
  E -- não --> NEG[não é opt-out]
  INEQ --> R1[ehPedidoDeOptOut = true<br/>autoriza gravar is_blocked]
  AMB --> R2[ehOptOutProvavel = true<br/>só para de responder + escala ao humano]
```

> A palavra solta no meio da frase NÃO conta ("tem como parar a dor?" não bloqueia). O bloqueio
> definitivo só vem do inequívoco; o ambíguo escala para uma pessoa confirmar.

---

## 9. Retenção — `interpretarRetencao` (`retencao/politica.ts`)

```mermaid
flowchart TD
  A[interpretarRetencao bruto, chave/padrao/piso] --> B{texto vazio?}
  B -- sim --> BX[dias = padrão, sem aviso<br/>caminho de quem nunca editou .env]
  B -- não --> C[numero = Number texto, não parseInt]
  C --> D{não é inteiro > 0?}
  D -- sim --> DX[dias = padrão, COM aviso]
  D -- não --> E{numero < piso?}
  E -- sim --> EX[dias = piso, COM aviso]
  E -- não --> F[dias = numero, sem aviso]
```

> O piso de VERDADE mora no SQL (`greatest(..., piso)`) e vale até para `psql` na mão; esta cópia serve
> para o operador ver no log que o valor dele foi elevado. Exceção: a captação (DELETE do admin client)
> tem o piso só aqui no TS.
