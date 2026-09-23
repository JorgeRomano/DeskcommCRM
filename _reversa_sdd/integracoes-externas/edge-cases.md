# Integrações Externas — Casos Extremos

> Confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA.

## EC-01 — INSERT no banco externo 🟢
`consultar()` abre `begin read only`; qualquer INSERT/UPDATE/DDL é recusado pelo POSTGRES, não por análise de string.

## EC-02 — DNS-rebinding no banco externo 🟢
`resolvePublicAddresses` recusa se QUALQUER endereço cair em faixa proibida (rebinding devolve um público e um privado); host RE-VALIDADO a cada leitura. DNS falho → fail-closed.

## EC-03 — Metadata de nuvem via link-local 🟢
`169.254.169.254` está na faixa `169.254/16` sempre bloqueada; mas RFC1918 (banco na LAN) é permitida de propósito (caso real do dono da VPS).

## EC-04 — ILIKE estrito devolvendo zero 🟢
Regra C-008: `contem`/`comeca_com` são tolerantes a espaço e caixa (`replace(lower(...),' ','')`), porque o modelo manda "cb250" e a base tem "CB 250 F Twister"; estrito fazia a IA concluir "não temos".

## EC-05 — `limiteMax` inválido 🟢
`montarConsulta` cai no teto absoluto quando `limiteMax` é NaN/negativo, nunca "ilimitado".

## EC-06 — Re-stringificar o webhook Nuvemshop 🟢
`verifyHmac` opera sobre rawBody; re-stringificar quebraria o HMAC. false em qualquer erro de parse.

## EC-07 — Access token do Google armazenado 🟢
Nunca armazenado: derivado a cada envio via `renovarToken(app, refreshToken)`; `lerCredencial` devolve `accessToken:""` de propósito.

## EC-08 — Evento de conversão antigo 🟢
Meta rejeita evento > 7 dias como PERMANENTE (com contagem de dias); `event_time` em segundos (ms daria ano ~55000 em silêncio).

## EC-09 — Plataforma no vocabulário sem transporte 🟢
`TRANSPORTES` com entrada `null` deixa a plataforma entrar no vocabulário sem virar `undefined`; o chamador loga `plataforma_sem_transporte` (registrado, não omitido).

## EC-10 — Hierarquia de anúncio na ingestão 🟢
`resolverHierarquiaDoContato` é cache-first, resolvido preguiçosamente ao abrir a ficha (não na ingestão: rede em hot path → tempestade de reentrada + custo de cota); devolve cache velho quando a plataforma recusa (nome velho > erro).

## EC-11 — Prototype pollution no JSON de extensão 🟢
`parseStrictJson` rejeita `__proto__/prototype/constructor`, BOM, comentários, chaves duplicadas, strings não-JSONB-safe.

## EC-12 — SHA-256 do artefato divergente 🟢
`validateArtifact` recusa com `extension_digest_mismatch`; `mirrorsCatalog` exige manifesto == entrada do catálogo.

## EC-13 — Capacidade de extensão sem porta 🟢
`PORTA_DA_CAPACIDADE` é Record exaustivo: capacidade nova sem porta NÃO COMPILA (em vez de virar `undefined`); só telas de trabalho qualificam (config/credencial/webhook excluídas).

## EC-14 — Extensão removida ainda referenciada 🟢
`SQL_ERRORS` mapeia `extension_removed` para HTTP 410; código de banco desconhecido → 503 genérico (nunca vaza detalhe do banco).
