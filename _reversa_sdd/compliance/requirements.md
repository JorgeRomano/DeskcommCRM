# Compliance (`lgpd`, `legal`, `opt-out`, `retencao`, `audit`)

> `requirements.md` — foca no QUE a unit faz. Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.
> Fonte: `_reversa_sdd/code-analysis.md` (Unidade 9).

## Visão Geral
A camada que faz o produto cumprir a lei e deixar rastro: direitos do titular (LGPD Art. 18 — acesso e anonimização), trilha de auditoria append-only, política de retenção (poda/expurgo com piso), detecção de opt-out por intenção, e o perfil legal por país. Multi-tenant: quase toda escrita usa admin client (bypassa RLS), então cada query filtra `organization_id` à mão de fonte confiável. Padrão transversal: falha na direção segura + sucesso nunca declarado sobre trabalho não feito. 🟢

## Responsabilidades
- Gravar a trilha de auditoria append-only (`audit()`), com vocabulário canônico. 🟢
- Calcular SLA LGPD em dias úteis do país da org. 🟢
- Executar a cascata de anonimização idempotente com retomada. 🟢
- Coletar e entregar o export de acesso (PDF, assinatura PAdES stub, e-mail). 🟢
- Drenar a fila de redação de storage e alarmar SLA. 🟢
- Resolver o perfil legal do país e o operador da instalação. 🟢
- Detectar opt-out por intenção (verbo de cessação + objeto de comunicação). 🟢
- Aplicar política de retenção pura com piso no SQL. 🟢

## Regras de Negócio
- `audit()` é fire-and-forget: falha nunca bloqueia a mutação, mas grita no Sentry. — `audit/index.ts` 🟢
- `isServiceRoleConfigured` nunca infere validade pelo comprimento (vazio/PLACEHOLDER = não; erra para "tenho a chave"). 🟢
- `AUDIT_ACTIONS` é o array canônico; o painel deriva dele; proibido importar qualquer coisa nesse arquivo; nunca renomear código. 🟢
- O que se APAGA a pedido do titular é o que se ENTREGA (invariante export × cascata). — `lgpd/export-collector.ts` 🟢
- Foto de perfil enfileirada ANTES da cascata (senão fica órfã no bucket); falha fechada. — `lgpd/redact-cascade.ts` 🟢
- Cascata idempotente: `jaRedigida()` guarda o sufixo "(anonimizado)"; passo 4 cancela a régua viva. — `lgpd/cascata.ts` 🟢
- PDF de acesso NÃO leva marca: nomeia o CONTROLADOR (`legal_name`) e o Encarregado. — `lgpd/pdf-renderer.tsx` 🟢
- País entra com citação revisada ou não entra (sem fallback para a LGPD). — `legal/perfil-do-pais.ts` 🟢
- Opt-out: verbo de cessação + objeto de comunicação; inequívoco bloqueia, ambíguo só escala. — `opt-out/deteccao.ts` 🟢
- Retenção: piso de verdade no SQL (`greatest(..., piso)`); nunca vira apagador de rastro recente. — `retencao/politica.ts` 🟢

## Requisitos Funcionais
| ID | Requisito | Prioridade | Critério de Aceite |
|----|-----------|-----------|-------------------|
| RF-01 | Auditoria append-only fire-and-forget | Must | Falha de audit reporta ao Sentry, não bloqueia a mutação |
| RF-02 | SLA em dias úteis do país | Must | `computeDueAt` pula fim de semana/feriado |
| RF-03 | Cascata de anonimização idempotente | Must | Reexecução não duplica sufixo; retomada por cron sem starvation |
| RF-04 | Export = o que se redige | Must | Gate deriva a lista das duas pontas (cascata × coletor) |
| RF-05 | Fila de storage idempotente | Should | Claim `pending→processing`; "not found" = skipped |
| RF-06 | Perfil de país com citação revisada | Must | País sem citação não cita lei (sem fallback LGPD) |
| RF-07 | Opt-out por intenção | Must | "tem como parar a dor?" não bloqueia; "parar de me mandar" bloqueia |
| RF-08 | Retenção com piso | Must | Piso no SQL vale para qualquer chamador |

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|------|--------------------|-----------|-----------|
| Segurança/Privacidade | PII nunca vai ao log (só ids e contagens) | `lgpd/export-collector.ts`, `sla-alarm.ts` | 🟢 |
| Segurança | Segredos nunca no metadata (append-only, retenção 5 anos) | `audit/actions.ts` | 🟢 |
| Segurança | `urlDePoliticaSegura` guarda de saída (só https/http) | `legal/operador.ts` | 🟢 |
| Segurança | e-mail só como sha256 no log (L-08) | `lgpd/email-delivery.ts`, `audit` (`hashEmail`) | 🟢 |
| Disponibilidade | Cascata falha aberta (aborta antes de deixar PII órfão) | `lgpd/redact-cascade.ts` | 🟢 |
| Corretude | Retenção não deriva de `lib/env.ts` (que lança no import) | `retencao/politica.ts` | 🟢 |

## Critérios de Aceitação
```gherkin
Dado um pedido de anonimização com foto de perfil
Quando cascadeRedactContact roda
Então a foto é enfileirada ANTES de a RPC zerar o ponteiro (falha fechada)

Dado uma cascata rodando de novo sobre título já redigido
Quando cascata.ts executa
Então jaRedigida() evita duplicar o sufixo "(anonimizado)"

Dado "tem como parar a dor?"
Quando ehPedidoDeOptOut avalia
Então não é opt-out (parar sem objeto de comunicação)
```

## Prioridade (MoSCoW)
| Requisito | MoSCoW | Justificativa |
|-----------|--------|---------------|
| Auditoria (RF-01) | Must | Trilha legal e de segurança |
| Cascata/export (RF-03/04) | Must | Direitos do titular (Art. 18) |
| SLA (RF-02) | Must | Prazo legal |
| Perfil de país (RF-06) | Must | Documento jurídico correto |
| Opt-out (RF-07) | Must | Não bloquear quem não pediu |
| Retenção (RF-08) | Must | Não apagar rastro recente |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---------|-----------------|-----------|
| `lib/audit/index.ts` | `audit`, `auditForOrganizations`, `isServiceRoleConfigured`, `hashEmail` | 🟢 |
| `lib/audit/actions.ts` | `AUDIT_ACTIONS`, `AuditAction` | 🟢 |
| `lib/lgpd/sla.ts` | `computeDueAt` | 🟢 |
| `lib/lgpd/redact-cascade.ts` | `cascadeRedactContact` | 🟢 |
| `lib/lgpd/cascata.ts` | passos 2-4, `varrerRedacoesIncompletas` | 🟢 |
| `lib/lgpd/export-collector.ts` | `collectExportData` | 🟢 |
| `lib/lgpd/storage-redaction-queue.ts` | `drainStorageRedactionQueue` | 🟢 |
| `lib/legal/perfil-do-pais.ts` | `perfilDaOrganizacao` | 🟢 |
| `lib/opt-out/deteccao.ts` | `ehPedidoDeOptOut`, `ehOptOutProvavel` | 🟢 |
| `lib/retencao/politica.ts` | `interpretarRetencao` | 🟢 |
