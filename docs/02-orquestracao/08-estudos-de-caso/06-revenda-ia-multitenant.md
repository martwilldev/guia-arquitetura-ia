# Estudo de Caso 6 — Plataforma que Revende IA (Multi-Tenant)

> **Cenário:** você constrói um SaaS que **revende IA** para muitos clientes (tenants). Cada tenant consome tokens, e você precisa **medir**, **limitar** (orçamento/quota) e **cobrar** — protegendo sua **margem** e evitando que um tenant "estoure" o custo de todos. A decisão central é **construir esse controle (DIY) ou comprar** (gateway/SaaS).

> 🔙 Volte ao índice dos estudos de caso: [README.md](./README.md)
>
> 📎 Leitura complementar: [Ferramentas para Precificação e Controle de Tokens](../../05-governanca-seguranca/03-ferramentas-precificacao-controle-tokens.md)

---

## Índice

1. [Problema, produto e requisitos](#1-problema-produto-e-requisitos)
2. [Arquitetura de referência](#2-arquitetura-de-referência)
3. [Anatomia do "controle": medir, limitar, cobrar](#3-anatomia-do-controle-medir-limitar-cobrar)
4. [Opção A — DIY: Lambda + DynamoDB + API Gateway usage plans + Bedrock 🟡](#4-opção-a--diy-lambda--dynamodb--api-gateway-usage-plans--bedrock-)
5. [Opção B — Gateway LiteLLM self-host (ECS) + Stripe billing + roteamento 🔴](#5-opção-b--gateway-litellm-self-host-ecs--stripe-billing--roteamento-)
6. [Opção C — Gateway SaaS (Portkey/OpenRouter) + billing gerenciado 🟢](#6-opção-c--gateway-saas-portkeyopenrouter--billing-gerenciado-)
7. [Tabela comparativa](#7-tabela-comparativa)
8. [Recomendação final por perfil](#8-recomendação-final-por-perfil)
9. [Armadilhas](#9-armadilhas)
10. [Glossário](#10-glossário)
11. [Resumo em uma frase](#11-resumo-em-uma-frase)

---

## 1. Problema, produto e requisitos

Você oferece um produto onde cada cliente final (tenant) faz chamadas de IA. Precisa: atribuir **cada token ao tenant certo**, impor **orçamento/quota por tenant**, **cobrar** conforme o consumo (ou plano), e **não perder dinheiro** — o custo do provedor (Bedrock) deve ficar abaixo do que você cobra.

| Requisito | Alvo |
|-----------|------|
| **Tenants** | ~500 tenants ativos |
| **Volume** | ~2.000.000 requests/mês (~média 4k requests/tenant/mês; distribuição desigual) |
| **Tokens/request** | ~1.500 in + 400 out (média) |
| **Isolamento** | Consumo e limites **por tenant**; um tenant não afeta o outro |
| **Cobrança** | Por uso (metered) e/ou planos com quota; faturas/relatórios |
| **Margem** | Preço de venda > custo Bedrock + custo de controle |
| **Governança** | Auditoria por tenant, rate limit, kill switch por orçamento |

> 💡 **Insight central:** o "controle" (medir/limitar/cobrar) **também custa** — em infra, em taxa de gateway SaaS, ou em engenharia. A decisão é: onde esse custo dói menos **para o seu estágio** e **quanta margem** ele consome.

---

## 2. Arquitetura de referência

```
   Tenant App ──► API (sua) ──► [ CAMADA DE CONTROLE ] ──► Provedor LLM (Bedrock)
                                     │  autentica tenant
                                     │  checa orçamento/quota
                                     │  roteia modelo
                                     │  MEDE tokens/custo
                                     ▼
                              ┌──────────────────┐
                              │ Uso por tenant   │ (DynamoDB / gateway / SaaS)
                              │ → Billing        │ (Stripe / Lago / faturamento)
                              │ → Dashboards     │
                              └──────────────────┘
```

O que muda entre as opções é **onde vive a CAMADA DE CONTROLE**: você mesmo escreve (A), um gateway open-source que você hospeda (B), ou um gateway SaaS que alguém opera para você (C). Veja o mapa de ferramentas em [03-ferramentas-precificacao-controle-tokens.md](../../05-governanca-seguranca/03-ferramentas-precificacao-controle-tokens.md).

Todos os valores abaixo são **ILUSTRATIVOS**. Confirme na [Calculadora AWS](https://calculator.aws) e nos [preços do Bedrock](https://aws.amazon.com/bedrock/pricing/).

---

## 3. Anatomia do "controle": medir, limitar, cobrar

```
   MEDIR   → capturar tokens in/out por request e atribuir ao tenant
   LIMITAR → orçamento mensal, quota, rate limit, kill switch por tenant
   COBRAR  → converter uso em fatura (metered billing) ou aplicar plano
```

Base de custo do LLM (comum às três opções), ILUSTRATIVO com modelo barato (Nova Lite/Haiku):

```
2.000.000 requests × (1.500 in + 400 out) tokens
= 3,0 B tokens in + 0,8 B tokens out
in:  3,0 B @ ~US$0,06/M ≈ US$ 180
out: 0,8 B @ ~US$0,24/M ≈ US$ 192
CUSTO LLM BASE ≈ ~US$ 372/mês  (ILUSTRATIVO — verifique)
```

> A diferença entre as opções é o **overhead de controle** somado a esses ~US$ 372.

---

## 4. Opção A — DIY: Lambda + DynamoDB + API Gateway usage plans + Bedrock 🟡

### Como funciona

Você constrói a camada de controle: **API Gateway** com **usage plans + API keys por tenant** (rate limit e quota nativos), um **Lambda authorizer** que valida o tenant e checa o **orçamento em DynamoDB** (com contador atômico de tokens/custo), e um Lambda que chama o **Bedrock** e **grava o consumo**. Cobrança você resolve depois (exporta uso → Stripe/planilha).

```
API GW (usage plan/key por tenant) ──► Lambda authorizer (checa orçamento DDB)
   └► Lambda proxy ──► Bedrock ──► grava tokens/custo em DynamoDB (atômico)
                                    └► kill switch se estourar orçamento
```

### Exemplo de código (Lambda-ready)

```python
# proxy.py — clientes FORA do handler
import json, boto3
from decimal import Decimal

bedrock = boto3.client("bedrock-runtime")
ddb = boto3.resource("dynamodb")
uso = ddb.Table("uso_por_tenant")
MODEL = "us.amazon.nova-lite-v1:0"
PRECO_IN, PRECO_OUT = Decimal("0.06") / 1_000_000, Decimal("0.24") / 1_000_000

def _dentro_do_orcamento(tenant_id: str) -> bool:
    item = uso.get_item(Key={"tenant_id": tenant_id, "mes": "2026-09"}).get("Item", {})
    return Decimal(str(item.get("custo", 0))) < Decimal(str(item.get("orcamento", 50)))

def handler(event, _ctx):
    tenant_id = event["requestContext"]["authorizer"]["tenant_id"]
    if not _dentro_do_orcamento(tenant_id):
        return {"statusCode": 402, "body": "Orçamento do mês esgotado"}

    body = json.loads(event["body"])
    resp = bedrock.converse(
        modelId=MODEL,
        messages=body["messages"],
        inferenceConfig={"maxTokens": 400},
    )
    u = resp["usage"]
    custo = u["inputTokens"] * PRECO_IN + u["outputTokens"] * PRECO_OUT
    # contador atômico: soma consumo do tenant
    uso.update_item(
        Key={"tenant_id": tenant_id, "mes": "2026-09"},
        UpdateExpression="ADD custo :c, tokens_in :i, tokens_out :o",
        ExpressionAttributeValues={":c": custo, ":i": u["inputTokens"], ":o": u["outputTokens"]},
    )
    texto = resp["output"]["message"]["content"][0]["text"]
    return {"statusCode": 200, "body": json.dumps({"resposta": texto})}
```

### Custo estimado mensal (ILUSTRATIVO)

| Item | Conta | Custo/mês |
|------|-------|-----------|
| LLM base (Bedrock) | ~US$ 372 (ver seção 3) | ~US$ 372 |
| Lambda (proxy + authorizer) | ~4M invocações × ~1,5 s × 256MB | ~US$ 60 |
| API Gateway | 2M requests | ~US$ 7 |
| DynamoDB (contadores) | writes atômicos on-demand | ~US$ 30 |
| **Overhead de controle** | (Lambda+APIGW+DDB) | **~US$ 97** |
| **Total** | | **~US$ 469/mês** |

> Overhead de controle ≈ **26%** sobre o custo de LLM. Billing "de verdade" (faturas, cartões) ainda precisa ser adicionado (ex.: Stripe).

### Complexidade: 🟡 média

### Trade-offs

- ✅ **Só AWS**, sem novos fornecedores; custo de infra baixo e previsível.
- ✅ Controle total dos dados de uso e das regras de orçamento.
- ✅ Usage plans do API Gateway dão rate limit/quota "de graça".
- ⚠️ Você constrói/mantém billing, dashboards e roteamento você mesmo.
- ⚠️ Contadores atômicos e concorrência exigem cuidado (hot partition por tenant grande).
- ⚠️ Multi-provedor (trocar Bedrock por outro) dá trabalho — não há abstração pronta.

---

## 5. Opção B — Gateway LiteLLM self-host (ECS) + Stripe billing + roteamento 🔴

### Como funciona

Você hospeda um **gateway LiteLLM** (proxy open-source compatível com OpenAI) em **ECS/Fargate**. Ele centraliza: **chaves virtuais por tenant**, **budgets/rate limits**, **roteamento** entre modelos/provedores e **logs de uso**. O uso é exportado para o **Stripe** (metered billing) que gera faturas. É o "meio-termo poderoso": open-source (sem taxa por token de terceiro) mas você opera a infra.

```
Tenant ──► LiteLLM Gateway (ECS) ──► Bedrock / outros provedores
                │  virtual keys, budgets, roteamento, logs
                ▼
        uso → Stripe (metered) → faturas
```

### Exemplo de config (LiteLLM — esboço)

```yaml
# litellm_config.yaml — carregado no boot do container
model_list:
  - model_name: barato
    litellm_params:
      model: bedrock/amazon.nova-lite-v1:0
  - model_name: forte
    litellm_params:
      model: bedrock/anthropic.claude-3-5-sonnet-20241022-v2:0

router_settings:
  routing_strategy: cost-based-routing   # manda simples p/ barato

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL  # guarda keys/budgets/uso
```

```python
# provisiona_tenant.py — cria chave virtual com orçamento (via API do gateway)
import requests, os
GW = os.environ["LITELLM_URL"]; ADMIN = os.environ["LITELLM_MASTER_KEY"]

def cria_tenant(tenant_id: str, orcamento_usd: float):
    r = requests.post(f"{GW}/key/generate",
        headers={"Authorization": f"Bearer {ADMIN}"},
        json={"key_alias": tenant_id, "max_budget": orcamento_usd,
              "budget_duration": "30d", "rpm_limit": 60})
    return r.json()["key"]   # chave virtual entregue ao tenant
```

### Custo estimado mensal (ILUSTRATIVO)

| Item | Conta | Custo/mês |
|------|-------|-----------|
| LLM base (Bedrock) | ~US$ 372 | ~US$ 372 |
| ECS/Fargate (gateway sempre-ligado) | 2 tasks × 1 vCPU/2GB × 730h (HA) | ~US$ 70 |
| ALB | 1 balanceador | ~US$ 20 |
| RDS/Postgres (estado do gateway) | pequeno | ~US$ 30 |
| Stripe (billing) | ~2,9% + US$0,30 por fatura (sobre o que você fatura) | variável |
| **Overhead de controle** | (ECS+ALB+RDS) | **~US$ 120** + taxa Stripe |
| **Total AWS** | | **~US$ 492/mês** + Stripe |

> Overhead de infra ≈ **32%** sobre o LLM, **fixo** (não escala a zero). Some a taxa do Stripe sobre a receita e o custo de engenharia para operar o gateway.

### Complexidade: 🔴 alta

### Trade-offs

- ✅ **Multi-provedor e roteamento** prontos (Bedrock, OpenAI, etc. atrás de uma API).
- ✅ Virtual keys, budgets e rate limits nativos por tenant.
- ✅ Sem taxa por token de um SaaS de gateway (open-source).
- ❌ Infra sempre-ligada (ECS/ALB/DB) = **custo fixo** + operação.
- ❌ Você é responsável por uptime, upgrades, escala e segurança do gateway.
- ❌ Integrar Stripe metered corretamente dá trabalho (idempotência, reconciliação).

---

## 6. Opção C — Gateway SaaS (Portkey/OpenRouter) + billing gerenciado 🟢

### Como funciona

Você usa um **gateway SaaS** (ex.: Portkey, OpenRouter) que já entrega **medição por chave/tenant**, **budgets/limites**, **roteamento/fallback**, **observabilidade** e, em alguns casos, **billing**. Você aponta suas chamadas para o SaaS e configura tudo no painel. **Menor esforço operacional** — em troca de uma **taxa** (por request/token ou % ) e de enviar metadados de uso a um terceiro.

```
Tenant ──► sua API fina ──► Gateway SaaS (Portkey/OpenRouter)
                                │  medição, budgets, roteamento, obs.
                                ▼
                          Bedrock / provedores
                          (billing/relatórios gerenciados)
```

### Exemplo de config (esboço)

```python
# proxy_saas.py — cliente FORA do handler; SaaS compatível com OpenAI
import os
from openai import OpenAI

# gateway SaaS expõe endpoint OpenAI-compatible; metadados identificam o tenant
client = OpenAI(
    base_url=os.environ["GATEWAY_BASE_URL"],   # ex.: https://gateway.saas/v1
    api_key=os.environ["GATEWAY_API_KEY"],
)

def handler(event, _ctx):
    tenant_id = event["requestContext"]["authorizer"]["tenant_id"]
    resp = client.chat.completions.create(
        model="bedrock/nova-lite",
        messages=event["messages"],
        extra_headers={"x-tenant-id": tenant_id},  # SaaS atribui uso ao tenant
        max_tokens=400,
    )
    return {"statusCode": 200, "body": resp.choices[0].message.content}
```

### Custo estimado mensal (ILUSTRATIVO)

| Item | Conta | Custo/mês |
|------|-------|-----------|
| LLM base (via SaaS, repassado) | ~US$ 372 | ~US$ 372 |
| Taxa do gateway SaaS | ex.: ~5% sobre o consumo **ou** US$ fixo por seat/volume | ~US$ 20–100 |
| Sua API fina (Lambda) | 2M invocações leves | ~US$ 20 |
| **Overhead de controle** | (taxa SaaS + Lambda fina) | **~US$ 40–120** |
| **Total** | | **~US$ 412–492/mês** |

> Overhead ≈ **10–30%**, mas **quase sem operação**. A taxa cresce com o volume — em escala muito grande, a % do SaaS pode superar o custo fixo do self-host (Opção B).

### Complexidade: 🟢 baixa (para operar); a plataforma carrega o trabalho

### Trade-offs

- ✅ **Time-to-market mínimo**: medição, budgets, roteamento e obs. prontos.
- ✅ Sem infra para manter; escala automática.
- ✅ Billing/relatórios muitas vezes inclusos.
- ⚠️ **Taxa** corrói margem; em volume alto pode ficar caro.
- ⚠️ Metadados de uso (e às vezes prompts) passam por terceiro — avalie LGPD/contrato.
- ⚠️ Dependência de fornecedor (lock-in, disponibilidade, roadmap).

---

## 7. Tabela comparativa

| Critério | A — DIY (AWS nativo) 🟡 | B — LiteLLM self-host (ECS) 🔴 | C — Gateway SaaS 🟢 |
|----------|-------------------------|--------------------------------|---------------------|
| **Custo/mês (ILUSTRATIVO)** | ~US$ 469 (US$372 LLM + US$97 controle) | ~US$ 492 + Stripe (US$120 fixo) | ~US$ 412–492 (US$40–120 controle) |
| **Overhead de controle** | ~26% (infra baixa) | ~32% fixo + operação | ~10–30% (taxa, cresce c/ volume) |
| **Complexidade** | 🟡 média | 🔴 alta | 🟢 baixa |
| **Multi-provedor / roteamento** | 🔴 você constrói | 🟢 pronto | 🟢 pronto |
| **Billing (faturas/cartão)** | 🔴 integrar depois | 🟡 via Stripe (você integra) | 🟢 muitas vezes incluso |
| **Controle de dados / LGPD** | 🟢 tudo na sua conta | 🟢 na sua conta | ⚠️ passa por terceiro |
| **Margem em escala** | 🟢 melhor (custo fixo dilui) | 🟢 boa em volume alto | 🟡 taxa % pode doer |
| **Quando escolher** | Só-AWS, quer controle e custo baixo | Multi-provedor + volume alto + time p/ operar | Lançar rápido, poucos tenants, sem time de infra |

---

## 8. Recomendação final por perfil

- **MVP / validação:** **Opção C** (Gateway SaaS). Lance em dias com medição, limites e billing prontos. Aceite a taxa enquanto o volume é pequeno — o que importa é validar o produto.
- **Crescimento:** **Opção A** (DIY na AWS). Quando os tenants crescerem e a taxa do SaaS começar a corroer a margem, traga o controle para casa com Lambda + DynamoDB + usage plans. Overhead baixo e previsível, dados na sua conta.
- **Escala / multi-provedor:** **Opção B** (LiteLLM self-host). Quando você precisar de roteamento entre vários provedores, budgets sofisticados e volume alto o bastante para diluir o custo fixo do ECS — e tiver time para operar. O custo fixo do gateway se paga em volume.

> Regra de margem: seu **preço de venda** por 1k tokens deve cobrir **custo do provedor + overhead de controle + margem-alvo**. Reavalie a opção sempre que o **overhead %** mudar de faixa com o volume. Veja o comparativo detalhado de ferramentas em [03-ferramentas-precificacao-controle-tokens.md](../../05-governanca-seguranca/03-ferramentas-precificacao-controle-tokens.md).

---

## 9. Armadilhas

- **Não medir por tenant desde o dia 1:** sem atribuição de tokens por tenant, você não sabe quem dá lucro nem quem dá prejuízo. Meça antes de crescer.
- **Sem kill switch:** um tenant com bug/abuso pode gerar milhares de dólares numa noite. Orçamento + corte automático são obrigatórios.
- **Hot partition (DIY):** um tenant enorme concentrando writes no DynamoDB gera throttling. Distribua a chave (ex.: `tenant#shard`).
- **Taxa do SaaS invisível na margem:** 5% parece pouco, mas em volume alto vira o maior custo depois do LLM. Modele a margem com o volume real.
- **Billing sem idempotência:** eventos de uso duplicados no Stripe = cobrança errada. Use chaves de idempotência.
- **LGPD no gateway SaaS:** prompts/uso passando por terceiro exigem base legal e contrato (DPA). Verifique antes.
- **Custo fixo que não escala a zero (Opção B):** ECS/ALB rodam mesmo sem tráfego. Em baixo volume, o overhead % explode.
- **Preço abaixo do custo:** revender IA sem contar o overhead de controle na precificação = prejuízo silencioso.

---

## 10. Glossário

| Termo | Significado |
|-------|-------------|
| **Tenant** | Cliente da sua plataforma (isolado dos demais em consumo e limites). |
| **Multi-tenancy** | Arquitetura que atende muitos tenants na mesma plataforma com isolamento. |
| **Usage plan / API key** | Recurso do API Gateway para rate limit e quota por chave/tenant. |
| **Gateway de LLM** | Camada que centraliza chamadas ao modelo (medição, limites, roteamento). |
| **LiteLLM** | Gateway open-source compatível com OpenAI (virtual keys, budgets, roteamento). |
| **Metered billing** | Cobrança baseada no consumo medido (ex.: Stripe metered). |
| **Kill switch** | Corte automático quando o orçamento do tenant é atingido. |
| **Overhead de controle** | Custo extra (infra/taxa/engenharia) para medir, limitar e cobrar. |
| **Margem** | Diferença entre o que você cobra e o custo total (LLM + controle). |

---

## 11. Resumo em uma frase

> **Para revender IA multi-tenant, comece com um gateway SaaS para lançar rápido, traga o controle para a AWS (Lambda + DynamoDB + usage plans) quando a taxa corroer a margem, e adote um LiteLLM self-hosted quando precisar de multi-provedor e volume alto — sempre medindo o consumo por tenant e precificando acima do custo de LLM + overhead de controle.**
