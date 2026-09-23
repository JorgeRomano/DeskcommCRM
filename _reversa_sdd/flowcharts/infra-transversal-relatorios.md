# Fluxogramas — Infra transversal e relatórios

> Gerado pelo Arqueólogo (Reversa) — Unidade 13.
> Cobre `lib/api/`, `lib/supabase/`, `lib/crypto/`, `lib/env.ts`, `lib/i18n/`, `lib/reports/`, `lib/metrics/`.

---

## 1. Envelope de resposta — `ok`/`fail` (`api/wrappers.ts`)

```mermaid
flowchart LR
  A[Handler] --> B{sucesso?}
  B -- sim --> C[ok data, meta?<br/>status 200/201/204]
  B -- não --> D[fail code, message, status]
  C --> E[NextResponse.json data:...<br/>+ X-Request-Id]
  D --> F[NextResponse.json error:code,message<br/>+ X-Request-Id]
  D -. code aceita string & {} .-> G[⚠️ código inventado vira<br/>contrato de wire calado<br/>→ registrar em errors.ts]
```

---

## 2. Idempotência com fecho de corrida — `comIdempotencia` (`api/idempotency.ts`)

```mermaid
flowchart TD
  A[comIdempotencia entrada] --> B[hash = SHA-256 de ordenar corpo]
  B --> C[lê linha viva expires_at>now]
  C --> D{achou?}
  D -- sim --> E[classificar]
  E -->|hash difere| E1[conflito → 409]
  E -->|status_code null| E2[em_curso → 409 in_progress]
  E -->|terminal| E3[replay: devolve body gravado]
  D -- não --> F[INSERT reserva status_code=null<br/>expira 60s]
  F --> G{23505?}
  G -- sim --> H[re-lê crua; vencida?<br/>reescreve com guard expires_at]
  G -- não --> I[executar efeito]
  H --> I
  I --> J{lançou?}
  J -- sim --> J1[LIBERA reserva vence agora<br/>propaga erro]
  J -- não --> K[grava recibo terminal<br/>status+body+24h na mesma linha]
  K --> L[executou]
```

---

## 3. Os três clients Supabase e a decisão de RLS

```mermaid
flowchart TD
  A[Precisa falar com o banco] --> B{contexto}
  B -->|Server Component/Route/Action| C[server.createClient<br/>anon key + cookie<br/>getUser SEMPRE]
  B -->|Browser| D[browser.createClient singleton<br/>anon key + realtime token via /auth]
  B -->|Webhook/cron/worker/admin| E[admin.createAdminClient<br/>service role]
  C --> F[RLS isola a org]
  D --> F
  E --> G[⚠️ BYPASSA RLS<br/>filtrar organization_id manual<br/>de fonte confiável, NUNCA do body]
```

---

## 4. Cifra AES-GCM — `encryptKey`/`decryptKey` (`crypto/aes_gcm.ts`)

```mermaid
flowchart LR
  subgraph Encrypt
    A[encryptKey plaintext] --> B[getKey: AI_CRED_AES_KEY<br/>base64 → 32 bytes ou lança]
    B --> C[IV aleatório 12B + aes-256-gcm]
    C --> D[ciphertext + tag 16B + last4]
    D --> E[bufToBytea → \\xHEX para PostgREST]
  end
  subgraph Decrypt
    F[decryptKey ciphertext,iv,tag] --> G[setAuthTag + final]
    G --> H[plaintext — NUNCA logado/devolvido]
  end
```

---

## 5. Contrato de env — `lib/env.ts` (parse no import)

```mermaid
flowchart TD
  A[import lib/env] --> B{NEXT_PHASE<br/>= build?}
  B -- sim, parse falha --> B1[semeia placeholders<br/>next build segue]
  B -- não --> C[schema.safeParse process.env]
  C --> D{válido?}
  D -- não --> DX[THROW no boot real<br/>= 500 em toda tela]
  D -- sim --> E[env congelado]
  E --> F[knob que a pessoa digita = z.string<br/>NUNCA z.enum/coerce<br/>senão contêiner healthy + 100% 500]
  E --> G[CRON_SECRET da Vercel<br/>→ copia p/ INTERNAL_CRON_SECRET]
```

---

## 6. Índice de Atrito — `montarPares` (`metrics/atrito.ts`)

```mermaid
flowchart TD
  A[AtritoRaw de fn_atrito_metrics] --> B[Para cada eixo:<br/>Conversão, Automação, Custo humano, Contenção]
  B --> C[razao num, den]
  C --> D{den <= 0?}
  D -- sim --> D1[null → tela mostra —<br/>NUNCA 0 que leria como 0% falso]
  D -- não --> D2[valor calculado]
  D1 --> E[Par: eficiencia + danos não-vazio]
  D2 --> E
  E --> F[⚠️ tipo PROÍBE eficiência sozinha<br/>sempre pareada com o dano]
```
