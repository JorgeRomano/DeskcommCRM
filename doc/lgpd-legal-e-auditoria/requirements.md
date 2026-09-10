# LGPD, Legal e Auditoria

> Spec SDD (híbrida — módulo de topo) · Redator (Reversa) · Nível: Detalhado
> Módulos: `lib/lgpd/*`, `lib/legal/*`, `lib/audit/*`, `lib/branding/*`, `workers/lgpd-*`
> 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## Visão Geral

Conformidade LGPD by-design (export do Art. 18 II e redação/anonimização por contato ou tenant), auditoria append-only de operações sensíveis, resolução do responsável legal (controlador) e marca branca white-label. O PDF de LGPD nomeia o CONTROLADOR (operador da VPS), nunca a marca do revendedor. 🟢

## Responsabilidades

- Processar export LGPD (coleta PII-safe → PDF + JSON → Storage → email). 🟢
- Processar redação por contato (cascata) e por tenant (lotes). 🟢
- Escrever auditoria append-only fire-and-forget (`api_audit_log`). 🟢
- Resolver o operador/controlador desta instalação (documentos legais). 🟢
- Resolver a marca white-label por camadas, sem nunca lançar. 🟢

## Regras de Negócio

- Anonimização preferida sobre delete físico; irreversível (403 `lgpd_anonymization_irreversible`). 🟢 (L-01/L-04)
- SLA em dias úteis BR: export D+7, redact D+15. 🟡 (L-02/L-03; dias inferidos)
- PDF de LGPD nomeia CONTROLADOR (`organizations.legal_name`) + DPO, sem marca. 🟢
- Redação enfileira o avatar em `storage_redaction_queue` ANTES de zerar o ponteiro (fail-closed). 🟢
- Redação de tenant marca `organizations.status='redacted'` (lotes de 100). 🟢
- Auditoria nunca bloqueia a mutação (fire-and-forget), mas é barulhenta (Sentry). 🟢
- `AUDIT_ACTIONS` é fonte única (painel deriva do array). 🟢
- Resolvedor de marca NUNCA lança (roda em `app/layout.tsx`); fonte é o banco, `.env` é semente/piso. 🟢
- CPF/PII nunca em log (`beforeSend` + logger). 🟢 (L-08)
- `resolverOperador` usa client de SESSÃO (nunca service role — `/legal/*` é pública). 🟢

## Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Processar export LGPD com PDF + JSON e entrega por email | Must | Dado `data_request`, gera PDF (controlador), sobe ao Storage, entrega com marca da org |
| RF-02 | Processar redação por contato (cascata via RPC) | Must | Dado `redact` de contato, enfileira avatar antes e anonimiza 5 tabelas |
| RF-03 | Processar redação por tenant (lotes) | Should | Dado `store_redact`, processa 100/lote e marca org redacted |
| RF-04 | Escrever auditoria append-only fire-and-forget | Must | Dada mutação sensível, grava `api_audit_log`; falha não bloqueia |
| RF-05 | Resolver operador/controlador da instalação | Must | Dado `/legal/*`, nomeia `legal_name`/DPO sem service role |
| RF-06 | Resolver marca por camadas sem lançar | Must | Dado layout, marca resolve org→instalação→env→padrão; erro → padrão |
| RF-07 | Marca de saída para email/MFA (tema claro) | Should | Dado email de LGPD, usa `marcaDaSaida` da org com accent+contraste |

## Requisitos Não Funcionais

| Tipo | Requisito inferido | Evidência no código | Confiança |
|------|--------------------|---------------------|-----------|
| Segurança/Privacidade | PII-safe: logs só com shortId/sha256/hashEmail | `workers/lgpd-export-worker.ts` | 🟢 |
| Disponibilidade | Resolvedor de marca e `marcaDaSaida` nunca lançam | `lib/branding/resolve.ts`, `saida.ts` | 🟢 |
| Correção | Cascata via RPC SECURITY DEFINER em TX única | `lib/lgpd/redact-cascade.ts` | 🟢 |
| Auditabilidade | Audit append-only (sem GRANT de UPDATE/DELETE) | `lib/audit`, L-10 | 🟢 |
| Robustez | attempts cap 3 → failed; sem email → pending_review | `workers/lgpd-*` | 🟢 |

## Critérios de Aceitação

```gherkin
Dado um data_request recebido
Quando o export worker processa
Então gera PDF nomeando o controlador (legal_name) e o DPO, sobe PDF+JSON e entrega por email

Dado um redact de contato
Quando cascadeRedactContact roda
Então o avatar é enfileirado em storage_redaction_queue ANTES de zerar o ponteiro (fail-closed)

Dado erro ao gravar auditoria de uma mutação sensível
Quando audit() falha
Então a mutação primária NÃO é bloqueada e o erro vai ao Sentry
```

## Prioridade (MoSCoW)

| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Export + redação LGPD | Must | Obrigação legal (Art. 18) |
| Auditoria append-only | Must | Conformidade + rastreabilidade |
| Resolvedor de operador/marca | Must | Layout não pode quebrar; PDF jurídico correto |
| Redação de tenant | Should | Uninstall de loja |
| Marca de saída | Should | Email/MFA com identidade da org |

## Rastreabilidade de Código

| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `workers/lgpd-export-worker.ts` | `processLgpdExport` | 🟢 |
| `workers/lgpd-redact-worker.ts` | `processLgpdRedact` | 🟢 |
| `lib/lgpd/redact-cascade.ts` | `cascadeRedactContact` | 🟢 |
| `lib/lgpd/pdf-renderer.tsx` | `renderLgpdPdf` | 🟢 |
| `lib/lgpd/types.ts`, `repository.ts` | `LgpdRequest`, `createLgpdRequest` | 🟢 |
| `lib/audit/index.ts`, `actions.ts` | `audit`, `AUDIT_ACTIONS` | 🟢 |
| `lib/legal/operador.ts` | `resolverOperador` | 🟢 |
| `lib/branding/resolve.ts`, `saida.ts`, `branding.ts` | `resolverMarca`, `marcaDaSaida`, `branding` | 🟢 |
