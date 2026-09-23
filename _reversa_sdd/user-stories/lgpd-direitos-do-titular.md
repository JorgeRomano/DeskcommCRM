# User Stories — Direitos do Titular (LGPD)

> Acesso e anonimização (Art. 18). Confiança: 🟢 CONFIRMADO.
> Units relacionadas: `compliance`, `plataforma-operacao` (marca), `superficie-http`.

## US-01 — Titular pede seus dados (direito de acesso)
**Como** titular
**Quero** receber tudo o que a organização sabe sobre mim
**Para** exercer o direito de acesso

```gherkin
Dado um pedido de acesso dentro do SLA (dias úteis do país da org)
Quando collectExportData roda
Então agrega todo dado pessoal (o que se apaga é o que se entrega)
E gera um PDF que nomeia o CONTROLADOR (não a marca do revendedor)
E entrega por e-mail (só sha256 no log)
```

## US-02 — Titular pede para ser esquecido (anonimização)
**Como** titular
**Quero** que meus dados sejam anonimizados de forma irreversível
**Para** exercer o direito de exclusão

```gherkin
Dado um pedido de anonimização
Quando a cascata roda
Então a foto de perfil é enfileirada ANTES (nunca fica órfã no bucket)
E a régua de recuperação viva é cancelada (não ressuscita o vínculo)
E a operação é idempotente (reexecutar não corrompe títulos)
```

## US-03 — SLA sendo monitorado
**Como** encarregado (DPO)
**Quero** ser alertado antes de estourar o prazo
**Para** não perder o SLA legal

```gherkin
Dado um pedido chegando a D+5 ou D+10
Quando o cron de SLA avalia
Então dispara um alarme sem PII (só ids e contagens), dedup 24h
```

## US-04 — Cliente pede para parar de receber mensagens (opt-out)
**Como** cliente
**Quero** parar de receber mensagens da empresa
**Para** não ser mais contatado

```gherkin
Dado que envio "parar de me mandar mensagens"
Quando ehPedidoDeOptOut avalia (verbo de cessação + objeto de comunicação)
Então sou bloqueado (is_blocked)

Dado que envio "tem como parar a dor?"
Quando ehPedidoDeOptOut avalia
Então NÃO sou bloqueado (parar sem objeto de comunicação)
```
