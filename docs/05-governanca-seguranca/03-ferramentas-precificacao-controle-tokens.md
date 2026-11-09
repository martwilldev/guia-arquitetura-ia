# Ferramentas, Serviços e Produtos para Precificação e Controle de Tokens de IA

> Guia prático para quem **oferece/revende IA como serviço** para seus próprios usuários.
> Foco em: medir consumo por usuário, cobrar, aplicar limites, ter margem e escolher a ferramenta certa
> — seja um **produto SaaS externo**, algo **dentro do orquestrador** ou no **API Gateway**.

---

## Índice

1. [O problema de quem revende IA](#1-o-problema-de-quem-revende-ia)
2. [Conceitos: os 4 pilares](#2-conceitos-os-4-pilares)
3. [Onde o controle pode ficar (as 3 camadas)](#3-onde-o-controle-pode-ficar-as-3-camadas)
4. [Modelos de precificação para os SEUS usuários](#4-modelos-de-precificação-para-os-seus-usuários)
5. [Categoria A — Gateways de LLM (o coração da solução)](#5-categoria-a--gateways-de-llm-o-coração-da-solução)
6. [Categoria B — Observabilidade e FinOps de IA](#6-categoria-b--observabilidade-e-finops-de-ia)
7. [Categoria C — Billing e monetização](#7-categoria-c--billing-e-monetização)
8. [Categoria D — API Gateway e rate limiting](#8-categoria-d--api-gateway-e-rate-limiting)
9. [Categoria E — Nativo AWS](#9-categoria-e--nativo-aws)
10. [Categoria F — Construir você mesmo (DIY)](#10-categoria-f--construir-você-mesmo-diy)
11. [Tabela comparativa geral](#11-tabela-comparativa-geral)
12. [Comparação: complexidade vs custo](#12-comparação-complexidade-vs-custo)
13. [Arquiteturas de referência por estágio](#13-arquiteturas-de-referência-por-estágio)
14. [Como calcular preço e margem](#14-como-calcular-preço-e-margem)
15. [Árvore de decisão](#15-árvore-de-decisão)
16. [Boas práticas e armadilhas](#16-boas-práticas-e-armadilhas)
17. [Glossário](#17-glossário)

---

## 1. O problema de quem revende IA

Quando você **oferece IA para seus usuários**, você vira um intermediário:

```
Seus usuários  ──►  SEU produto  ──►  Provedor de LLM (Bedrock/OpenAI/...)
   (pagam você)      (você mede,        (você paga por token)
                      limita, cobra)
```

Isso cria necessidades que quem só consome IA para si **não** tem:

| Necessidade | Por quê |
|-------------|---------|
| **Medir consumo por usuário** | Saber quanto cada cliente gastou (tokens/custo) |
| **Atribuir custo** | Separar o custo do usuário A do usuário B |
| **Aplicar limites/quotas** | Impedir que um usuário estoure seu orçamento |
| **Cobrar (billing)** | Transformar consumo em fatura |
| **Ter margem** | Cobrar mais do que você paga ao provedor |
| **Evitar abuso** | Rate limiting, detecção de anomalia, fraude |
| **Multi-tenancy** | Isolar dados e custo entre clientes |

> **Regra de ouro:** você precisa saber, para cada request, **quem** fez, **quantos tokens** custou e **quanto** isso representa em dinheiro — em tempo real o suficiente para aplicar limites.

---

## 2. Conceitos: os 4 pilares

Qualquer solução (comprada ou construída) precisa cobrir estes 4 pilares:

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  1. METERING │  │ 2. LIMITING  │  │ 3. PRICING   │  │ 4. BILLING   │
│  (medir)     │─►│ (limitar)    │─►│ (precificar) │─►│ (cobrar)     │
│              │  │              │  │              │  │              │
│ tokens/user  │  │ quota/rate   │  │ markup/plano │  │ fatura/cartão│
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

1. **Metering (medir)** — contar tokens/requests por usuário/tenant.
2. **Limiting (limitar)** — aplicar quotas, rate limits, orçamentos; bloquear ou degradar ao exceder.
3. **Pricing (precificar)** — converter tokens em preço (com sua margem), planos, tiers.
4. **Billing (cobrar)** — gerar fatura, integrar com pagamento (Stripe, boleto, etc.).

Nem toda ferramenta faz os 4. Muitas fazem 1–2. A estratégia comum é **combinar**: um gateway para medir+limitar + um billing para cobrar.

---

## 3. Onde o controle pode ficar (as 3 camadas)

Você perguntou: serviço externo, dentro do orquestrador, ou no API Gateway? As três são válidas e podem coexistir. Cada uma pega o request num ponto diferente.

```
Usuário
   │
   ▼
┌───────────────────────────────────────────────────────┐
│  CAMADA 1: API GATEWAY / EDGE                           │  ◄── rate limit grosso,
│  (rate limiting, auth, quota por API key)               │      auth, quota por plano
└───────────────────────┬───────────────────────────────┘
                        ▼
┌───────────────────────────────────────────────────────┐
│  CAMADA 2: ORQUESTRADOR / GATEWAY DE LLM                │  ◄── medição de tokens,
│  (conta tokens, aplica orçamento, roteia, degrada)      │      orçamento por tenant,
└───────────────────────┬───────────────────────────────┘      markup, roteamento
                        ▼
┌───────────────────────────────────────────────────────┐
│  CAMADA 3: PROVEDOR (Bedrock/OpenAI)                    │  ◄── retorna uso real
│  (retorna usage: input/output tokens)                   │      (fonte da verdade)
└───────────────────────────────────────────────────────┘
```

| Camada | O que controla bem | O que NÃO controla |
|--------|--------------------|--------------------|
| **API Gateway** | Rate limit por chave, quota grossa (nº de requests), auth | Não sabe quantos *tokens* nem custo — só conta requests |
| **Orquestrador / Gateway LLM** | Tokens exatos, custo, orçamento por usuário, markup, roteamento, degradação | Precisa de código/produto dedicado |
| **Provedor** | Fonte da verdade do uso (retorna tokens usados) | Não conhece seus usuários nem sua precificação |

> **Chave:** o **API Gateway limita por REQUEST** (não sabe de token). O **controle real de TOKEN/custo** acontece na camada do orquestrador/gateway de LLM, que lê o `usage` retornado pelo provedor. Use as duas juntas: gateway barra abuso grosseiro; orquestrador controla custo fino.

---

## 4. Modelos de precificação para os SEUS usuários

Antes das ferramentas, decida **como você vai cobrar**. Isso define o que precisa medir.

| Modelo | Como funciona | Prós | Contras | Ferramenta ajuda |
|--------|---------------|------|---------|------------------|
| **Por token/uso (usage-based)** | Repassa tokens com markup | Justo, escala com uso | Receita imprevisível; precisa medir preciso | Metering + billing usage-based |
| **Créditos / pré-pago** | Usuário compra créditos, consome por token | Caixa antecipado, simples de limitar | UX de "acabou o crédito" | Wallet/saldo + metering |
| **Planos/tiers (assinatura)** | Free/Pro/Enterprise com cotas de tokens | Receita previsível | Risco de usuário pesado dar prejuízo | Quota por plano + rate limit |
| **Híbrido (assinatura + overage)** | Plano base + cobra o excedente | Previsível + captura uso alto | Mais complexo de faturar | Metering + billing com overage |
| **Flat por seat** | Preço fixo por usuário/mês | Simples de vender | Difícil garantir margem se uso varia muito | Quota por seat |

> **Recomendado para começar:** **créditos pré-pago** ou **planos com cota**. Ambos limitam seu risco de custo (o usuário não gasta além do que pagou/da cota), e são mais simples que usage-based puro com fatura no fim do mês.

---

## 5. Categoria A — Gateways de LLM (o coração da solução)

Um **gateway de LLM** (LLM gateway / AI gateway) fica **entre seu app e os provedores**. É a ferramenta que mais resolve o problema de metering + limiting + roteamento de uma vez. Muitos são open-source e rodam no seu ambiente (inclusive AWS).

| Produto | O que faz | Metering por usuário | Limites/quota | Billing | Open-source | Roda na AWS/Lambda |
|---------|-----------|:---:|:---:|:---:|:---:|:---:|
| **LiteLLM** | Proxy/gateway unificado p/ 100+ modelos; contabiliza tokens/custo por API key/usuário; orçamentos e rate limits | ✅ | ✅ (budgets, RPM/TPM) | Parcial (tracking) | ✅ | ✅ (proxy em ECS/EC2; SDK em Lambda) |
| **Portkey** | AI gateway com metering, budgets, guardrails, cache, observabilidade | ✅ | ✅ | Parcial | Parcial (SaaS + gateway) | ✅ (SaaS ou self-host) |
| **Helicone** | Observabilidade + gateway; custo por usuário, rate limit, alertas | ✅ | ✅ | Parcial | ✅ | ✅ |
| **OpenRouter** | Gateway SaaS que agrega provedores; você compra crédito e revende | ✅ (por chave) | ✅ | ✅ (créditos) | ❌ | ✅ (via API) |
| **Kong AI Gateway** | Plugins de IA sobre o Kong (rate limit por token, roteamento) | ✅ | ✅ | ❌ | ✅ (core) | ✅ (ECS/EC2) |
| **Cloudflare AI Gateway** | Gateway no edge: cache, rate limit, analytics de uso | ✅ (analytics) | ✅ | ❌ | ❌ | Edge (fora AWS) |
| **TrueFoundry / LangDB / Eden AI** | Gateways/agregadores com billing e metering | ✅ | ✅ | Parcial/✅ | Varia | Varia |

### Por que o gateway é central

Ele resolve de uma vez:
- **Medição precisa** de tokens/custo por chave/usuário (lê o `usage` do provedor).
- **Orçamento e rate limit** por usuário/tenant (bloqueia ou degrada ao exceder).
- **Roteamento** (manda request simples para modelo barato → protege sua margem).
- **Cache** (respostas repetidas não custam tokens → economia direta).
- **Multi-provedor** (troca de modelo sem mudar seu app).

### Destaque: LiteLLM (opção open-source popular)

O LiteLLM Proxy é frequentemente a escolha inicial para quem revende IA porque cobre metering + budgets + rate limit por "virtual key" (uma chave por usuário/tenant), e roda no seu ambiente:

```
Seus usuários ──► LiteLLM Proxy (virtual keys, budgets, tracking) ──► Bedrock/OpenAI
                        │
                        └──► Postgres (uso por chave) + logs de custo
```

Você cria uma **virtual key** por usuário com um **budget** (ex: US$ 5/mês) e **rate limit** (ex: 100 req/min). O proxy contabiliza tokens/custo e **bloqueia automaticamente** ao estourar. Depois você lê esses dados para faturar.

> Roda bem como **container no ECS/Fargate ou EC2**. Em Lambda, o mais comum é usar o **SDK** (não o proxy persistente), ou chamar um proxy LiteLLM hospedado à parte.

---

## 6. Categoria B — Observabilidade e FinOps de IA

Ferramentas focadas em **enxergar** o consumo/custo (dashboards, traces, custo por usuário). Não bloqueiam por si só, mas são essenciais para precificar certo e detectar anomalia.

| Produto | Foco | Custo por usuário/tenant | Alertas/anomalia | Open-source |
|---------|------|:---:|:---:|:---:|
| **Langfuse** | Tracing + custo por usuário/sessão/feature; muito usado | ✅ | ✅ | ✅ |
| **Helicone** | Observabilidade + custo + rate limit | ✅ | ✅ | ✅ |
| **Langsmith** | Tracing do ecossistema LangChain | ✅ | Parcial | ❌ |
| **Phoenix (Arize)** | Observabilidade de LLM open-source | ✅ | Parcial | ✅ |
| **Datadog LLM Observability** | APM + custo de LLM integrado | ✅ | ✅ | ❌ |
| **Portkey / Helicone** | Também fazem gateway (cat. A) | ✅ | ✅ | Parcial/✅ |

> **Padrão comum:** gateway (LiteLLM/Portkey) para **controlar** + Langfuse/Helicone para **visualizar e atribuir custo** por usuário/feature.

---

## 7. Categoria C — Billing e monetização

Estas transformam consumo medido em **dinheiro cobrado**. Você conecta o metering (do gateway) a uma dessas.

| Produto | Foco | Usage-based (por token) | Créditos/wallet | Assinatura | Nota |
|---------|------|:---:|:---:|:---:|------|
| **Stripe Billing** | Pagamentos + billing usage-based e assinatura | ✅ (metered billing) | Parcial | ✅ | Padrão de mercado; reporta uso via API |
| **Metronome** | Billing usage-based para IA/infra | ✅ | ✅ | ✅ | Feito para métricas de consumo em escala |
| **Orb** | Billing usage-based flexível | ✅ | ✅ | ✅ | Bom para modelos híbridos |
| **Lago** | Billing usage-based open-source | ✅ | ✅ | ✅ | Self-host, controla dados |
| **Togai / m3ter** | Metering + rating para usage-based | ✅ | ✅ | ✅ | Camada de metering→preço |
| **Paddle / Chargebee** | Assinaturas (merchant of record) | Parcial | ❌ | ✅ | Bom p/ SaaS por plano |
| **AWS Marketplace (metering)** | Vender via marketplace com metering | ✅ | ❌ | ✅ | Se distribui pela AWS |

### Como o fluxo de billing funciona

```
Gateway mede tokens/custo por usuário
        │
        ▼
Você aplica markup (ex: custo × 1.4)
        │
        ▼
Reporta "unidades de uso" para o billing (Stripe/Lago/Metronome)
        │
        ▼
Billing gera fatura / debita crédito / cobra cartão
```

> **Dica:** meça em **tokens** (ou "créditos" que você define), não em dólar bruto do provedor. Assim você desacopla seu preço do preço do fornecedor e protege a margem se o custo do provedor mudar.

---

## 8. Categoria D — API Gateway e rate limiting

Estas controlam **requests** (não tokens), mas são a primeira linha de defesa contra abuso e a base de quotas por plano.

| Produto | Rate limit | Quota por chave/plano | Auth | Nota |
|---------|:---:|:---:|:---:|------|
| **Amazon API Gateway** | ✅ | ✅ (Usage Plans + API Keys) | ✅ (Cognito/authorizer) | Nativo AWS; quota por requests, não por token |
| **Kong** | ✅ | ✅ | ✅ | Tem plugins de IA (cat. A) |
| **Tyk** | ✅ | ✅ | ✅ | Open-source |
| **Cloudflare** | ✅ | ✅ | ✅ | Edge, analytics |
| **Zuplo** | ✅ | ✅ | ✅ | API gateway "dev-first" |

### Limitação importante

O API Gateway conta **requisições**, não **tokens**. Um único request pode gastar 100 ou 100.000 tokens. Por isso:

```
API Gateway  →  "usuário fez 500 requests hoje"  (não sabe o custo)
Orquestrador →  "usuário gastou 1,2M tokens = US$ 3,60"  (sabe o custo)
```

Use o API Gateway para: **auth**, **rate limit por segundo/minuto** (anti-abuso) e **quota grossa por plano** (ex: Free = 1.000 req/mês). Use o orquestrador/gateway de LLM para o **controle de custo por token**.

---

## 9. Categoria E — Nativo AWS

Se você já está na AWS (Bedrock/Lambda), dá para montar boa parte sem produto externo.

| Serviço | Papel no controle de tokens |
|---------|------------------------------|
| **Amazon API Gateway (Usage Plans)** | Rate limit e quota por API key/plano |
| **Amazon Cognito** | Identidade dos seus usuários (quem é quem) |
| **Amazon DynamoDB** | Guardar orçamento/saldo/consumo por usuário (com incremento atômico) |
| **AWS Lambda (orquestrador)** | Ler o `usage` do Bedrock, contar tokens, aplicar orçamento, degradar modelo |
| **Amazon Bedrock** | Retorna `usage` (input/output tokens) — fonte da verdade |
| **CloudWatch (métricas custom)** | Emitir métrica de tokens por tenant; alarmes |
| **AWS Budgets + Cost Allocation Tags** | Orçamento e alerta de custo por tag/tenant |
| **AWS Marketplace Metering** | Se você vende seu produto pela AWS |

### Padrão AWS puro (sem SaaS externo)

```
Usuário ─► API Gateway (Cognito + Usage Plan)
                │
                ▼
        Lambda orquestrador
        1. identifica usuário (Cognito)
        2. lê saldo/orçamento no DynamoDB
        3. se ok, chama Bedrock
        4. lê usage (tokens) da resposta
        5. calcula custo × markup
        6. debita saldo no DynamoDB (atômico)
        7. emite métrica CloudWatch
                │
                ▼
        Bedrock retorna resposta + usage
```

> Esse padrão é detalhado (com código Python de orçamento por tenant em DynamoDB) no documento [`01-precificacao-tokens-credenciais-aws.md`](01-precificacao-tokens-credenciais-aws.md) deste mesmo módulo.

---

## 10. Categoria F — Construir você mesmo (DIY)

Você **pode** implementar metering + limiting sem produto externo. Faz sentido no início (poucos usuários) ou quando quer controle total.

### O núcleo do metering (exemplo conceitual em Python)

```python
import boto3, json
from decimal import Decimal

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
ddb = boto3.resource("dynamodb").Table("uso-por-usuario")

# Preço do provedor (ILUSTRATIVO — confirme na página de preços do Bedrock)
PRECO_ENTRADA = Decimal("0.000003")   # US$ por token de entrada
PRECO_SAIDA   = Decimal("0.000015")   # US$ por token de saída
MARKUP        = Decimal("1.4")        # sua margem (40%)

def custo_do_request(usage):
    entrada = usage["inputTokens"]  * PRECO_ENTRADA
    saida   = usage["outputTokens"] * PRECO_SAIDA
    return (entrada + saida) * MARKUP   # preço COM sua margem

def orcamento_ok(user_id):
    item = ddb.get_item(Key={"user_id": user_id}).get("Item", {})
    gasto = item.get("gasto_mes", Decimal("0"))
    limite = item.get("limite_mes", Decimal("5"))  # US$ 5/mês default
    return gasto < limite

def debitar(user_id, valor):
    # incremento atômico — seguro sob concorrência
    ddb.update_item(
        Key={"user_id": user_id},
        UpdateExpression="ADD gasto_mes :v",
        ExpressionAttributeValues={":v": valor},
    )

def handler(event, context):
    user_id = event["requestContext"]["authorizer"]["claims"]["sub"]  # Cognito
    if not orcamento_ok(user_id):
        return {"statusCode": 402, "body": json.dumps({"erro": "orcamento excedido"})}

    body = {"anthropic_version": "bedrock-2023-05-31", "max_tokens": 512,
            "messages": [{"role": "user", "content": json.loads(event["body"])["message"]}]}
    resp = bedrock.invoke_model(
        modelId="anthropic.claude-3-5-sonnet-20241022-v2:0", body=json.dumps(body))
    payload = json.loads(resp["body"].read())

    # usage = fonte da verdade do consumo
    usage = payload["usage"]  # {"inputTokens": N, "outputTokens": M}
    valor = custo_do_request(usage)
    debitar(user_id, valor)   # cobra do saldo do usuário

    return {"statusCode": 200, "body": json.dumps({
        "reply": payload["content"][0]["text"],
        "tokens": usage,
        "custo_usuario": float(valor),
    })}
```

### DIY: prós e contras

✅ **Prós:** controle total, sem custo de licença, dados 100% seus, simples no início.
❌ **Contras:** você reimplementa o que gateways já resolvem (rate limit, cache, roteamento, dashboards, faturas); mais código para manter; billing robusto (impostos, cartão, disputa) é complexo — melhor terceirizar (Stripe/Lago).

> **Recomendação híbrida realista:** DIY o **metering/orçamento** (é simples) + use **Stripe/Lago** para o **billing** (é o difícil). Ou adote um **gateway (LiteLLM)** que já entrega metering+limites e conecte ao billing.

---

## 11. Tabela comparativa geral

| Ferramenta | Categoria | Mede token/usuário | Limita | Precifica/markup | Billing | Open-source | Encaixe AWS |
|------------|-----------|:---:|:---:|:---:|:---:|:---:|:---:|
| **LiteLLM** | Gateway LLM | ✅ | ✅ | Parcial | Parcial | ✅ | ✅ ECS/EC2 |
| **Portkey** | Gateway LLM | ✅ | ✅ | Parcial | Parcial | Parcial | ✅ |
| **Helicone** | Gateway + Obs. | ✅ | ✅ | Parcial | ❌ | ✅ | ✅ |
| **OpenRouter** | Gateway SaaS | ✅ | ✅ | ✅ (crédito) | ✅ | ❌ | ✅ API |
| **Kong AI Gateway** | Gateway/API | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| **Langfuse** | Observabilidade | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Datadog LLM Obs.** | Observabilidade | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Stripe Billing** | Billing | ➖ (recebe uso) | ❌ | ✅ | ✅ | ❌ | ✅ |
| **Lago** | Billing | ➖ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Metronome / Orb** | Billing | ➖ | ❌ | ✅ | ✅ | ❌ | ✅ |
| **Amazon API Gateway** | API Gateway | ❌ (só request) | ✅ | ❌ | ❌ | ❌ | ✅ nativo |
| **DynamoDB + Lambda (DIY)** | Nativo/DIY | ✅ | ✅ | ✅ | ➖ | — | ✅ nativo |

Legenda: ✅ faz · ➖ complementar/parcial · ❌ não faz

---

## 12. Comparação: complexidade vs custo

```
  Complexidade de implementar
       ▲
 Alta  │                                   ● Stack completo
       │                                     (Gateway + Obs. + Billing
       │                                      + multi-tenant)
       │                     ● Gateway self-host
       │                       (LiteLLM/Kong em ECS)
 Média │          ● DIY metering
       │            (Lambda+DynamoDB)
       │   ● Gateway SaaS
       │     (OpenRouter/Portkey)
 Baixa │ ● API Gateway
       │   Usage Plans (só request)
       └──────────────────────────────────────────►
         Baixo        Custo mensal da ferramenta        Alto
```

| Abordagem | Esforço | Custo da ferramenta | Quando usar |
|-----------|---------|---------------------|-------------|
| **API Gateway Usage Plans** | Baixo | Baixo (nativo AWS) | MVP, limitar só nº de requests |
| **Gateway SaaS (OpenRouter/Portkey)** | Baixo | Médio (comissão/assinatura) | Validar rápido sem operar infra |
| **DIY metering (Lambda+DynamoDB)** | Médio | Baixo (só infra AWS) | Poucos usuários, controle total |
| **Gateway self-host (LiteLLM/Kong)** | Médio | Baixo (infra) + operação | Escala, multi-modelo, controle de custo |
| **Stack completo** | Alto | Médio-alto | Produto maduro, muitos tenants, billing sério |

> **Custo real geralmente NÃO é a ferramenta** — é o **token do LLM** que domina. A ferramenta certa se paga só evitando desperdício (cache, roteamento para modelo barato, bloqueio de abuso).

---

## 13. Arquiteturas de referência por estágio

### 13.1 MVP / validação (poucos usuários)

```
Usuário ─► API Gateway (Cognito + Usage Plan p/ rate limit)
              │
              ▼
        Lambda (metering DIY: lê usage do Bedrock, debita DynamoDB)
              │
              ▼
        Bedrock
```
- Billing: manual ou créditos simples. Barato e rápido. Sem produto externo.

### 13.2 Crescimento (dezenas/centenas de usuários)

```
Usuário ─► API Gateway ─► LiteLLM Proxy (ECS/Fargate)
                              │  (virtual key por usuário, budget, rate limit)
                              ├──► Bedrock / OpenAI
                              └──► Postgres (uso por usuário)
                                        │
                                        ▼
                              Stripe/Lago (fatura por uso)
              + Langfuse (dashboards de custo por usuário)
```

### 13.3 Produto maduro / multi-tenant sério

```
Usuário ─► API Gateway (auth, quota por plano)
              │
              ▼
        Orquestrador + Gateway LLM (LiteLLM/Portkey)
              │   ├─ roteamento (barato→caro) p/ margem
              │   ├─ cache (economia)
              │   ├─ orçamento e quota por tenant
              │   └─ guardrails
              ▼
        Bedrock (+ VPC endpoint) / múltiplos provedores
              │
              ▼
   Metronome/Orb/Lago (billing usage-based + overage)
   Langfuse/Datadog (observabilidade e anomalia)
   DynamoDB (saldo/estado)  ·  CloudWatch/Budgets (FinOps)
```

---

## 14. Como calcular preço e margem

O ponto central de "oferecer IA para seus usuários": **cobrar mais do que você paga**.

### 14.1 A conta básica

```
Custo do provedor por request = (tokens_entrada × preço_entrada)
                              + (tokens_saída × preço_saída)

Seu preço ao usuário = Custo do provedor × (1 + margem)
                     +  rateio de infra (Lambda/gateway/store)
                     +  rateio de ferramentas (billing/obs.)
```

### 14.2 Exemplo ilustrativo

> ⚠️ Valores **ilustrativos** para raciocínio. Confirme os preços reais na [página de preços do Bedrock](https://aws.amazon.com/bedrock/pricing/) e na [Calculadora AWS](https://calculator.aws).

```
Suponha custo do provedor ≈ US$ 3 / 1M tokens entrada, US$ 15 / 1M saída.
Um request típico: 2.000 entrada + 500 saída.
  custo provedor = 2.000×0,000003 + 500×0,000015 = US$ 0,0135

Com margem de 40%:
  seu preço = 0,0135 × 1,40 = US$ 0,0189 por request

Se o usuário faz 1.000 requests/mês:
  seu custo   = US$ 13,50
  você cobra  = US$ 18,90
  margem      = US$ 5,40 (antes de infra/ferramentas)
```

### 14.3 Como proteger a margem

- **Roteamento**: mande requests simples para modelo barato (pode cortar 50–80% do custo).
- **Cache**: respostas repetidas = custo zero de token.
- **Limite `max_tokens`**: teto de saída por request (a saída é a parte cara).
- **Truncar contexto**: menos tokens de entrada.
- **Cobrar em créditos**: desacopla seu preço do preço do fornecedor.
- **Cota por plano**: garante que usuário do plano Free não te dê prejuízo.

### 14.4 Cuidado com o "usuário tóxico"

Em plano **flat/assinatura**, um único usuário pesado pode consumir mais do que paga. Mitigações:
- Cota de tokens embutida no plano (ex: Pro = 2M tokens/mês, depois cobra overage).
- Rate limit por usuário.
- Degradação graciosa (ao atingir a cota, cai para modelo mais barato em vez de bloquear).

---

## 15. Árvore de decisão

```
Você está começando (MVP, quer validar)?
   ├── Sim, quer o mínimo de infra ─────► API Gateway Usage Plans + metering DIY (Lambda+DynamoDB)
   └── Sim, quer nem operar gateway ────► Gateway SaaS (OpenRouter/Portkey) + Stripe

Já tem tração e quer controlar custo/margem a sério?
   ├── Quer open-source e controle ─────► LiteLLM (self-host ECS) + Lago + Langfuse
   └── Prefere gerenciado ──────────────► Portkey/Helicone + Metronome/Orb

Precisa faturar de verdade (cartão, imposto, disputa)?
   └────────────────────────────────────► NÃO faça DIY do billing: use Stripe/Lago/Metronome

Só precisa impedir abuso por nº de requests?
   └────────────────────────────────────► API Gateway (rate limit) resolve — mas NÃO controla custo por token

Precisa saber custo por usuário/feature para decidir preço?
   └────────────────────────────────────► Langfuse ou Helicone (observabilidade)
```

### Recomendação prática por estágio

| Estágio | Metering/Limite | Billing | Observabilidade |
|---------|-----------------|---------|-----------------|
| **MVP** | Lambda+DynamoDB (DIY) ou Usage Plans | Créditos manuais / Stripe simples | CloudWatch |
| **Crescimento** | LiteLLM (self-host) | Stripe / Lago | Langfuse |
| **Maduro** | LiteLLM/Portkey + multi-tenant | Metronome / Orb / Lago | Langfuse / Datadog |

---

## 16. Boas práticas e armadilhas

### Boas práticas

- **Meça pelo `usage` do provedor** (fonte da verdade), nunca estime tokens "no olho".
- **Debite de forma atômica** (DynamoDB `ADD`) para não perder contagem sob concorrência.
- **Separe request-limit (gateway) de token-limit (orquestrador)** — são coisas diferentes.
- **Cobre em créditos/tokens seus**, não em dólar do fornecedor (protege margem e simplifica UX).
- **Cache agressivo** de respostas repetidas — economia direta.
- **Roteie por custo** — modelo barato para tarefa simples.
- **Alertas de anomalia** — um pico de consumo pode ser bug ou abuso.
- **Terceirize o billing** (Stripe/Lago) — é mais complexo do que parece (impostos, chargeback).

### Armadilhas

| Armadilha | Consequência | Mitigação |
|-----------|--------------|-----------|
| Limitar só no API Gateway | Usuário estoura custo com poucos requests grandes | Controle de token no orquestrador |
| Cobrar flat sem cota | Usuário pesado dá prejuízo | Cota + overage + rate limit |
| Estimar tokens em vez de medir | Cobrança errada, margem furada | Ler `usage` real |
| Guardar saldo sem atomicidade | Contagem errada sob concorrência | Incremento atômico |
| DIY do billing cedo demais | Bugs de cobrança, retrabalho | Stripe/Lago desde cedo |
| Ignorar custo de saída | Surpresa na fatura (saída é cara) | `max_tokens` + monitorar output |
| Sem observabilidade por usuário | Não sabe quem/onde gasta | Langfuse/Helicone |

---

## 17. Glossário

| Termo | Significado |
|-------|-------------|
| **Metering** | Medir o consumo (tokens/requests) por usuário/tenant |
| **Rate limiting** | Limitar requisições por unidade de tempo |
| **Quota** | Limite total no período (ex: 1M tokens/mês) |
| **Budget / orçamento** | Teto de gasto (em US$ ou créditos) por usuário |
| **Markup** | Margem que você adiciona sobre o custo do provedor |
| **Usage-based billing** | Cobrança proporcional ao consumo |
| **Overage** | Cobrança do excedente além da cota do plano |
| **LLM Gateway / AI Gateway** | Camada entre app e provedores que mede, limita, roteia |
| **Virtual key** | Chave por usuário no gateway, com budget/limite próprios |
| **Multi-tenancy** | Vários clientes isolados na mesma infraestrutura |
| **Degradação graciosa** | Cair para modelo mais barato ao atingir limite (em vez de bloquear) |
| **`usage`** | Campo retornado pelo provedor com tokens de entrada/saída |
| **Merchant of record** | Quem assume responsabilidade legal/fiscal da venda (ex: Paddle) |
| **Chargeback** | Contestação de cobrança pelo cliente |

---

## Resumo em uma frase

> **Para oferecer IA aos seus usuários, combine três camadas — API Gateway (anti-abuso por request), um gateway de LLM como LiteLLM ou orquestrador próprio no Lambda (medição de tokens, orçamento e roteamento por usuário) e um billing como Stripe/Lago (cobrança) — medindo sempre pelo `usage` real do provedor e cobrando em créditos com markup para proteger sua margem; comece simples (DIY no Lambda+DynamoDB) e adote ferramentas dedicadas conforme escala.**

---

### Documentos relacionados

- [`01-precificacao-tokens-credenciais-aws.md`](01-precificacao-tokens-credenciais-aws.md) — governança, IAM, arquitetura de referência e código de orçamento por tenant.
- [`../02-orquestracao/06-roteamento-de-llms.md`](../02-orquestracao/06-roteamento-de-llms.md) — roteamento por custo (protege margem).
- [`../03-rag-dados/01-bancos-vetoriais-e-documentos.md`](../03-rag-dados/01-bancos-vetoriais-e-documentos.md) — custo da camada de RAG.
