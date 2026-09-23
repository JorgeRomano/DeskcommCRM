# User Stories — Onboarding e Marca Própria

> Primeira configuração e white-label. Confiança: 🟢 CONFIRMADO.
> Units relacionadas: `plataforma-operacao`, `crm-funil`, `auth-tenancy-rbac`.

## US-01 — Dono configura o CRM no wizard
**Como** dono da organização
**Quero** ser guiado pelos passos essenciais
**Para** começar a usar rápido

```gherkin
Dado que abri o CRM pela primeira vez
Quando o wizard mostra os passos
Então cada passo decide a própria existência (nuvemshop só com loja ligada)
E o passo de funil vem depois de configurar a IA
```

## US-02 — Sugestão de funil com plano B
**Como** dono
**Quero** um funil sugerido para o meu negócio
**Para** não montar do zero

```gherkin
Dado que descrevi meu negócio
Quando a IA sugere um funil
Então a sugestão é validada; se reprova (ou a IA está off)
Então caio num pacote curado por tipo de negócio (nunca quadro vazio)
```

## US-03 — Revendedor aplica a própria marca
**Como** revendedor (white-label)
**Quero** o CRM com o meu nome, logo e cor
**Para** vender como meu

```gherkin
Dado que configurei nome/logo/cor no banco
Quando a marca é resolvida em app/layout.tsx
Então a cor vira uma rampa com contraste WCAG (nunca lança no render)
E os documentos de LGPD continuam nomeando o CONTROLADOR (não a marca)
```

## US-04 — Rollback de versão mantém a marca
**Como** operador da VPS
**Quero** que um rollback de imagem não apague a marca
**Para** a tela continuar com a identidade certa

```gherkin
Dado que fiz rollback da imagem (não do schema)
Quando a marca é re-derivada neste build
Então usa a ENTRADA guardada (semente), nunca os stops derivados
E a tela continua pintada
```
