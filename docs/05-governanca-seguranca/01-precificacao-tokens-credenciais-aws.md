# Precificação, Controle de Tokens, Credenciais e Controle de Acesso — Arquitetura de LLM na AWS

> **Camada de Governança, FinOps e Segurança** de uma arquitetura de LLM em produção na AWS.
> Os outros documentos deste estudo tratam de orquestradores de IA, roteamento, RAG e frameworks (Bedrock/Lambda). Este documento é a camada que garante que tudo isso rode **barato, seguro e auditável**.

> ⚠️ **AVISO SOBRE PREÇOS:** Todos os valores monetários neste documento são **ILUSTRATIVOS**, servem apenas para ensinar o raciocínio de cálculo. Preços mudam com frequência, variam por região e por modelo. **SEMPRE confirme na [página de preços do Amazon Bedrock](https://aws.amazon.com/bedrock/pricing/) e na [AWS Pricing Calculator](https://calculator.aws/) antes de tomar decisões.**

---

## Índice

1. [Anatomia de custo de uma arquitetura LLM](#1-anatomia-de-custo-de-uma-arquitetura-llm)
2. [Precificação detalhada do Bedrock](#2-precificação-detalhada-do-bedrock)
3. [Controle de tokens](#3-controle-de-tokens)
4. [Credenciais e segredos](#4-credenciais-e-segredos)
5. [Controle de acesso](#5-controle-de-acesso)
6. [Governança e FinOps](#6-governança-e-finops)
7. [Guardrails e compliance](#7-guardrails-e-compliance)
8. [Arquitetura de referência completa](#8-arquitetura-de-referência-completa)
9. [Multi-tenant em detalhe](#9-multi-tenant-em-detalhe)
10. [Checklist de produção, riscos e glossário](#10-checklist-de-produção-riscos-e-glossário)

---

## 1. Anatomia de custo de uma arquitetura LLM

Antes de otimizar, você precisa saber **para onde o dinheiro vai**. Uma arquitetura de LLM em produção tem seis grandes baldes de custo:

| Balde | O que é | Peso típico | Como cobra |
|-------|---------|-------------|------------|
| **Tokens do modelo** | Inferência LLM (entrada + saída) | 🔴 **60–90%** | Por 1.000 tokens (on-demand) ou por hora (provisioned) |
| **Compute** | Lambda / ECS / EC2 que orquestra | 🟡 5–15% | Por GB-segundo (Lambda) ou por hora (ECS/EC2) |
| **Armazenamento vetorial / RAG** | OpenSearch, Aurora pgvector, Kendra | 🟡 5–20% | Por hora de instância / OCU / storage |
| **Rede** | Data transfer, NAT Gateway, VPC endpoints | 🟢 1–5% | Por GB transferido |
| **Orquestração** | Step Functions, Bedrock Agents | 🟢 1–5% | Por transição de estado / por request |
| **Observabilidade** | CloudWatch Logs, métricas, tracing | 🟢 1–10% | Por GB ingerido / por métrica |

### Diagrama: para onde o dinheiro vai

```
                        CUSTO TOTAL DA ARQUITETURA LLM
                                    │
      ┌─────────────────┬──────────┼──────────┬─────────────┬────────────┐
      │                 │          │          │             │            │
      ▼                 ▼          ▼          ▼             ▼            ▼
 ┌─────────┐      ┌──────────┐ ┌────────┐ ┌────────┐  ┌──────────┐ ┌──────────┐
 │ TOKENS  │      │ COMPUTE  │ │  RAG   │ │  REDE  │  │ ORQUESTR.│ │  OBSERV. │
 │ do LLM  │      │ Lambda/  │ │ Vector │ │ NAT /  │  │  Step Fn │ │ CloudW.  │
 │         │      │ ECS/EC2  │ │ Store  │ │ VPC EP │  │ / Agents │ │ Logs     │
 └────┬────┘      └──────────┘ └────────┘ └────────┘  └──────────┘ └──────────┘
      │
      │  ████████████████████████████████████████████████  60-90%
      │  (entrada + saída; SAÍDA costuma ser 3-5x mais cara)
      │
   OUTROS ██████  10-40%

   ┌────────────────────────────────────────────────────────────┐
   │  REGRA DE OURO: em 9 de 10 arquiteturas LLM, os TOKENS       │
   │  dominam a conta. Otimize tokens PRIMEIRO. Compute e infra   │
   │  são ruído perto do custo de inferência em escala.           │
   └────────────────────────────────────────────────────────────┘
```

### Por que tokens dominam

Um único request de um agente RAG pode consumir:
- **Prompt de sistema:** 500–2.000 tokens
- **Contexto RAG (documentos recuperados):** 2.000–8.000 tokens
- **Histórico de conversa:** 1.000–4.000 tokens
- **Pergunta do usuário:** 50–500 tokens
- **Resposta gerada (saída):** 200–2.000 tokens

Isso é facilmente **5.000–15.000 tokens por request**. Multiplique por milhões de requests/mês e você entende por que a conta de tokens engole tudo. Um loop de agente (ReAct) que faz 5 chamadas ao LLM por tarefa multiplica esse custo por 5.

---

## 2. Precificação detalhada do Bedrock

O Bedrock tem **três modos de cobrança** principais. Entender os três é o que separa uma arquitetura barata de uma cara.

### 2.1 On-Demand (por token)

- Você paga **por token processado**, sem compromisso.
- **Entrada (input)** e **saída (output)** têm preços diferentes — **saída é tipicamente 3 a 5x mais cara** que entrada.
- Ideal para: cargas variáveis, MVP, tráfego imprevisível.

**Tabela ILUSTRATIVA (valores fictícios para ensinar o cálculo — confirme na página oficial):**

| Modelo (exemplo) | Entrada (US$/1M tokens) | Saída (US$/1M tokens) | Perfil |
|------------------|------------------------:|----------------------:|--------|
| Modelo "nano/haiku-like" | ~US$ 0,25 | ~US$ 1,25 | 🟢 Barato, rápido |
| Modelo "médio/sonnet-like" | ~US$ 3,00 | ~US$ 15,00 | 🟡 Equilibrado |
| Modelo "grande/opus-like" | ~US$ 15,00 | ~US$ 75,00 | 🔴 Caro, poderoso |
| Embeddings (ex. Titan Embed) | ~US$ 0,02 | — (sem saída) | 🟢 Muito barato |

> Repare: o modelo "grande" pode custar **60x mais** que o "nano". Escolher o modelo certo para cada tarefa é a maior alavanca de FinOps que existe.

### 2.2 Provisioned Throughput (por model unit/hora)

- Você **reserva capacidade** (Model Units) e paga **por hora**, independentemente do uso.
- Compromissos de 1 mês ou 6 meses dão desconto.
- Ideal para: tráfego alto e **previsível**, latência garantida, modelos customizados/fine-tuned (que exigem PT).

**Regra de decisão:**

```
Se (custo_on_demand_mensal > custo_provisioned_mensal):
    → considere Provisioned Throughput
Senão:
    → fique no On-Demand

Ponto de equilíbrio aproximado: quando a utilização da capacidade
reservada passa de ~50-70% de forma sustentada.
```

### 2.3 Batch (processamento assíncrono com desconto)

- Envia um lote de requests, recebe as respostas depois (não é tempo real).
- Oferece **desconto** (tipicamente ~50%) sobre o preço on-demand.
- Ideal para: classificação em massa, geração de embeddings de um corpus inteiro, análises noturnas, backfill.

### 2.4 Prompt Caching (redução de custo poderosa)

Muitos requests **repetem o mesmo prefixo** (prompt de sistema grande, few-shot examples, documento fixo). O **prompt caching** guarda esse prefixo processado e cobra **muito menos** (ex.: ~10% do preço de entrada) pelos tokens em cache nas chamadas seguintes.

```
SEM cache:
  Request 1: [SYSTEM 2000 tk] + [pergunta 100 tk] → paga 2100 tk de entrada
  Request 2: [SYSTEM 2000 tk] + [pergunta 120 tk] → paga 2120 tk de entrada
  Request 3: [SYSTEM 2000 tk] + [pergunta  90 tk] → paga 2090 tk de entrada

COM cache (prefixo SYSTEM em cache):
  Request 1: 2000 tk (escrita cache) + 100 tk normais
  Request 2: 2000 tk @ ~10% (leitura cache) + 120 tk normais  ← economia enorme
  Request 3: 2000 tk @ ~10% (leitura cache) +  90 tk normais

  Economia típica em prompts com prefixo grande e repetido: 50-90%.
```

### 2.5 Exemplos ILUSTRATIVOS de cálculo

> ⚠️ Valores fictícios. Objetivo: ensinar o raciocínio. Confirme sempre os preços reais.

**Premissas do exemplo:**
- Modelo "médio": entrada US$ 3,00 / 1M tk; saída US$ 15,00 / 1M tk.
- Request típico de chatbot RAG: **6.000 tokens de entrada + 800 tokens de saída**.

**Custo por request:**

```
Entrada: 6.000 tk × (US$ 3,00 / 1.000.000) = US$ 0,018
Saída:     800 tk × (US$ 15,00 / 1.000.000) = US$ 0,012
                                     TOTAL = US$ 0,030 por request
```

**Custo mensal em 3 volumes:**

| Volume mensal | Cálculo | Custo mensal aproximado |
|---------------|---------|------------------------:|
| 🟢 **50.000 req/mês** (piloto) | 50.000 × US$ 0,030 | **~US$ 1.500** |
| 🟡 **1.000.000 req/mês** (produção média) | 1.000.000 × US$ 0,030 | **~US$ 30.000** |
| 🔴 **10.000.000 req/mês** (larga escala) | 10.000.000 × US$ 0,030 | **~US$ 300.000** |

**Efeito de otimizações (sobre o volume de 1M req/mês = US$ 30.000 base):**

| Otimização | Efeito | Novo custo aprox. |
|------------|--------|------------------:|
| Trocar para modelo "nano" nas tarefas simples (roteamento) | ~60% dos requests 12x mais baratos | **~US$ 12.000** |
| Prompt caching no system prompt (2.000 tk fixos) | −40% na entrada | **~US$ 20.000** |
| Truncar contexto RAG de 6k→3k tokens | −50% na entrada | **~US$ 18.000** |
| **Combinando as três** | efeito multiplicativo | **~US$ 6.000–8.000** |

> Uma boa camada de governança de tokens pode reduzir a conta em **3 a 5x** sem perder qualidade percebida.

---

## 3. Controle de tokens

### 3.1 Por que controlar

| Motivo | Explicação |
|--------|------------|
| 💰 **Custo** | Um único usuário/bug pode gerar milhares de dólares em horas (loop de agente descontrolado). |
| 🛡️ **Abuso** | Prompt injection, scraping, uso automatizado indevido da sua API. |
| ⚖️ **Fairness** | Um tenant não pode consumir toda a capacidade e degradar os outros. |
| 📊 **Previsibilidade** | Orçamento por tenant torna a conta previsível e permite chargeback. |

### 3.2 Técnicas (do mais simples ao mais sofisticado)

1. **Limitar `max_tokens` da saída** — teto rígido por request. A defesa mais barata e imediata.
2. **Truncar/resumir contexto** — limitar quantos documentos RAG e quanto histórico entram no prompt.
3. **Contagem de tokens (tokenizers)** — medir antes de enviar; rejeitar requests gigantes.
4. **Orçamento por usuário/tenant/app** — teto de tokens (ou US$) por período, armazenado em DynamoDB.
5. **Quotas e throttling** — limitar requests por segundo/minuto.
6. **Rate limiting por API key/usuário** — API Gateway usage plans ou lógica na aplicação.
7. **Degradação graciosa (fallback)** — ao estourar o orçamento, cair para um modelo mais barato em vez de bloquear.
8. **Detecção de anomalia** — alertar quando o consumo de um tenant foge do padrão histórico.

### 3.3 Exemplo de código: contar tokens, orçamento em DynamoDB, bloquear/rebaixar

```python
"""
Controle de tokens e orçamento por usuário/tenant.
- Conta tokens antes de enviar
- Consulta e atualiza orçamento no DynamoDB (atômico)
- Bloqueia OU rebaixa para modelo barato ao exceder
"""
import os
import time
import boto3
from botocore.exceptions import ClientError

# Contagem de tokens.
# IMPORTANTE: o tokenizer ideal depende do modelo. Para modelos Bedrock,
# a contagem exata vem no response (usage). Aqui usamos uma aproximação
# para PRÉ-validação (barrar antes de gastar). Ajuste a lib ao seu modelo.
try:
    import tiktoken
    _enc = tiktoken.get_encoding("cl100k_base")
    def contar_tokens(texto: str) -> int:
        return len(_enc.encode(texto))
except ImportError:
    # Aproximação grosseira de fallback: ~4 caracteres por token.
    def contar_tokens(texto: str) -> int:
        return max(1, len(texto) // 4)

dynamodb = boto3.resource("dynamodb")
tabela_orcamento = dynamodb.Table(os.environ["BUDGET_TABLE"])  # ex.: "llm-budget"
bedrock = boto3.client("bedrock-runtime")

# Orçamento mensal de tokens por tenant (poderia vir de config/tenant).
ORCAMENTO_PADRAO_TOKENS = 5_000_000

MODELO_PRIMARIO = "anthropic.claude-3-5-sonnet-xxxx"   # bom e mais caro
MODELO_FALLBACK = "anthropic.claude-3-haiku-xxxx"      # barato para degradação
LIMITE_ENTRADA_TOKENS = 12_000   # rejeita prompts absurdamente grandes


def _periodo_atual() -> str:
    # Chave de partição por tenant + período (YYYY-MM) para reset mensal automático.
    return time.strftime("%Y-%m")


def consumo_atual(tenant_id: str) -> int:
    resp = tabela_orcamento.get_item(
        Key={"tenant_id": tenant_id, "periodo": _periodo_atual()}
    )
    return int(resp.get("Item", {}).get("tokens_usados", 0))


def registrar_consumo(tenant_id: str, tokens: int) -> int:
    """Incremento ATÔMICO — evita condição de corrida entre invocações concorrentes."""
    resp = tabela_orcamento.update_item(
        Key={"tenant_id": tenant_id, "periodo": _periodo_atual()},
        UpdateExpression="ADD tokens_usados :t",
        ExpressionAttributeValues={":t": tokens},
        ReturnValues="UPDATED_NEW",
    )
    return int(resp["Attributes"]["tokens_usados"])


def escolher_modelo(tenant_id: str, orcamento: int = ORCAMENTO_PADRAO_TOKENS) -> str:
    """Degradação graciosa: se passou de 90% do orçamento, rebaixa para o modelo barato."""
    usados = consumo_atual(tenant_id)
    if usados >= orcamento:
        raise PermissionError(
            f"Orçamento de tokens esgotado para {tenant_id} "
            f"({usados}/{orcamento}). Bloqueando request."
        )
    if usados >= 0.9 * orcamento:
        return MODELO_FALLBACK  # rebaixa em vez de bloquear
    return MODELO_PRIMARIO


def invocar_com_governanca(tenant_id: str, prompt: str, max_tokens_saida: int = 800):
    # 1) Pré-validação de tamanho (barrar ANTES de gastar dinheiro)
    tokens_entrada_est = contar_tokens(prompt)
    if tokens_entrada_est > LIMITE_ENTRADA_TOKENS:
        raise ValueError(
            f"Prompt muito grande: {tokens_entrada_est} > {LIMITE_ENTRADA_TOKENS}. "
            f"Trunque ou resuma o contexto."
        )

    # 2) Verifica orçamento e escolhe o modelo (bloqueia ou rebaixa)
    modelo = escolher_modelo(tenant_id)

    # 3) Invoca com teto rígido de saída (max_tokens)
    resposta = bedrock.converse(
        modelId=modelo,
        messages=[{"role": "user", "content": [{"text": prompt}]}],
        inferenceConfig={"maxTokens": max_tokens_saida},  # teto de saída
    )

    # 4) Contabiliza o consumo REAL retornado pelo modelo (fonte da verdade)
    uso = resposta["usage"]
    tokens_reais = uso["inputTokens"] + uso["outputTokens"]
    total = registrar_consumo(tenant_id, tokens_reais)

    return {
        "modelo": modelo,
        "texto": resposta["output"]["message"]["content"][0]["text"],
        "tokens_request": tokens_reais,
        "tokens_no_periodo": total,
    }
```

### 3.4 Rate limiting e detecção de anomalia (esboço)

```python
# Rate limiting simples por janela deslizante (token bucket) em DynamoDB/ElastiCache.
# Em produção, prefira API Gateway usage plans (por API key) para throttling na borda.

def excedeu_taxa(tenant_id: str, limite_por_minuto: int = 60) -> bool:
    chave = f"{tenant_id}:{int(time.time() // 60)}"
    resp = tabela_orcamento.update_item(
        Key={"tenant_id": chave, "periodo": "rate"},
        UpdateExpression="ADD contador :um",
        ExpressionAttributeValues={":um": 1},
        ReturnValues="UPDATED_NEW",
    )
    return int(resp["Attributes"]["contador"]) > limite_por_minuto


# Detecção de anomalia: compare o consumo da última hora com a média histórica.
# Se consumo_hora > media_movel * 3 (3 desvios), dispare alerta via SNS/CloudWatch.
```

---

## 4. Credenciais e segredos

### 4.1 Regra número um: NUNCA hardcode

```python
# ❌ NUNCA FAÇA ISSO
client = boto3.client(
    "bedrock-runtime",
    aws_access_key_id="AKIA...",       # vazamento garantido no git
    aws_secret_access_key="wJalr...",  # aparece em logs, em PRs, em prints
)
openai_key = "sk-proj-abc123..."       # PIOR AINDA

# ✅ FAÇA ISSO — sem credenciais no código
client = boto3.client("bedrock-runtime")  # usa a IAM Role automaticamente
```

### 4.2 IAM Roles (não access keys) para Lambda / ECS / EC2

O padrão correto na AWS é **identidade baseada em role**, não chaves estáticas:

- **Lambda:** tem uma *execution role*. O SDK (boto3) pega credenciais **temporárias e rotacionadas automaticamente** dessa role. Você nunca vê nem gerencia chaves.
- **ECS:** *Task Role* (via container credentials endpoint).
- **EC2:** *Instance Profile* (via IMDSv2).
- **On-premises / fora da AWS:** **IAM Roles Anywhere** — usa certificados X.509 para obter credenciais temporárias, eliminando access keys de longa duração em servidores próprios.

**Como o Lambda acessa o Bedrock sem credenciais estáticas:**

```
┌──────────┐   assume automaticamente    ┌─────────────────────┐
│  Lambda  │ ──────────────────────────► │  Execution Role     │
│  (código)│                             │  (IAM)              │
└────┬─────┘                             └──────────┬──────────┘
     │  boto3.client("bedrock-runtime")             │
     │                                              │ STS entrega
     │                                              │ credenciais
     │                                              │ TEMPORÁRIAS
     │                                              ▼
     │                                    ┌─────────────────────┐
     └───────── InvokeModel ────────────►│  Amazon Bedrock     │
        (assinado com cred. temporária)  │  (valida via IAM)   │
                                         └─────────────────────┘

  Zero chaves estáticas. Rotação automática. Nada para vazar.
```

### 4.3 Secrets Manager e Parameter Store para chaves de terceiros

Quando você **precisa** de segredos externos (chave da OpenAI, token de um SaaS, senha de banco), guarde-os em serviço gerenciado, nunca em variável de ambiente em texto puro nem no código:

| Serviço | Quando usar | Rotação | Custo |
|---------|-------------|---------|-------|
| **AWS Secrets Manager** | Segredos que precisam rotação automática (DB, API keys de terceiros) | ✅ Automática (via Lambda) | 🟡 Por segredo + por API call |
| **SSM Parameter Store (SecureString)** | Config e segredos simples, KMS-encrypted | ⚠️ Manual | 🟢 Standard gratuito |

```python
import boto3
import json
from functools import lru_cache

secrets = boto3.client("secretsmanager")

@lru_cache(maxsize=8)  # cache em memória para evitar chamar a cada request (custo/latência)
def obter_segredo(nome: str) -> dict:
    resp = secrets.get_secret_value(SecretId=nome)
    return json.loads(resp["SecretString"])

# Uso:
openai_key = obter_segredo("prod/llm/openai")["api_key"]
```

> **Atenção ao cache:** cachear segredos reduz custo/latência, mas você precisa **invalidar o cache na rotação** (ex.: TTL curto ou reagir a evento de rotação). Nunca cacheie por tempo indefinido.

### 4.4 Rotação, KMS e evitar vazamento em logs

- **Rotação:** Secrets Manager pode rotacionar automaticamente via Lambda. Sempre habilite para segredos sensíveis.
- **KMS:** criptografe tudo em repouso (segredos, DynamoDB, logs, vector store) com **customer-managed keys (CMK)** quando precisar de controle fino de acesso e auditoria de uso da chave.
- **Vazamento em logs — o erro mais comum:**

```python
# ❌ NUNCA logue o objeto inteiro do request/segredo
logger.info(f"request: {evento}")          # pode conter PII, tokens, headers de auth
logger.info(f"segredo: {openai_key}")      # segredo em texto puro no CloudWatch

# ✅ Logue apenas o necessário e mascare
logger.info("request recebido", extra={"tenant": tenant_id, "tokens": n_tokens})
logger.info(f"segredo carregado: {openai_key[:3]}***{openai_key[-2:]}")  # se precisar
```

Use filtros de dados sensíveis do CloudWatch Logs (data protection policies) para mascarar PII/segredos automaticamente na ingestão.

### 4.5 Policy IAM de menor privilégio — invocar SÓ modelos específicos

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InvocarApenasModelosPermitidos",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-*",
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-haiku-*",
        "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-*"
      ]
    },
    {
      "Sid": "NegarModelosCaros",
      "Effect": "Deny",
      "Action": "bedrock:InvokeModel*",
      "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-3-opus-*"
    }
  ]
}
```

> A combinação `Allow` restrito + `Deny` explícito em modelos caros/regiões erradas dá controle preciso de **o quê** e **onde** pode ser invocado. `Deny` sempre vence sobre `Allow`.

---

## 5. Controle de acesso

### 5.1 IAM com menor privilégio e condições

Use *condition keys* para restringir por região, por presença de tag, por VPC, etc.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BedrockSomenteNaRegiaoPermitida",
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:sa-east-1::foundation-model/*",
      "Condition": {
        "StringEquals": { "aws:RequestedRegion": "sa-east-1" }
      }
    },
    {
      "Sid": "ApenasDeDentroDaVPC",
      "Effect": "Deny",
      "Action": "bedrock:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:SourceVpce": "vpce-0abc123def456" }
      }
    }
  ]
}
```

### 5.2 Controle por modelo e por região

- Restrinja o `Resource` da policy aos ARNs dos modelos permitidos (visto acima).
- Use `aws:RequestedRegion` para impedir invocação em regiões fora da sua zona de compliance (ex.: manter dados no Brasil → apenas `sa-east-1`).

### 5.3 API Gateway com autorizadores

A borda da sua API é onde o controle de acesso do **usuário final** acontece:

| Autorizador | Quando usar |
|-------------|-------------|
| **Amazon Cognito** | Usuários de app (login/senha, social, MFA), JWT gerenciado |
| **JWT authorizer** | Você já tem um Identity Provider (Auth0, Okta, Keycloak) |
| **Lambda authorizer** | Lógica custom (validar tenant, checar orçamento, ACL própria) |
| **API keys + usage plans** | Parceiros B2B, throttling e quota por chave |

```
Cliente ──► API Gateway ──► [Autorizador] ──► Lambda orquestrador
                               │
                               ├─ Cognito/JWT: valida token, extrai tenant_id/claims
                               ├─ Usage Plan: aplica rate limit + quota por API key
                               └─ Lambda authorizer: checa orçamento no DynamoDB
                                  e injeta contexto (tenant, tier) na request
```

### 5.4 SCPs em nível de organização

Service Control Policies (AWS Organizations) são **guardrails que nenhuma conta pode furar**, nem o admin da conta:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProibirBedrockForaDoBrasil",
      "Effect": "Deny",
      "Action": "bedrock:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": ["sa-east-1", "us-east-1"] }
      }
    }
  ]
}
```

### 5.5 VPC Endpoints para Bedrock (PrivateLink)

Por padrão, chamar o Bedrock sai pela internet (via NAT Gateway). Com um **VPC Interface Endpoint (PrivateLink)** o tráfego fica **dentro da rede da AWS**, sem tocar a internet:

```
SEM VPC Endpoint:
  Lambda (VPC) ──► NAT Gateway ──► Internet ──► Bedrock
                   (custo NAT + superfície de risco)

COM VPC Endpoint (PrivateLink):
  Lambda (VPC) ──► VPC Endpoint (bedrock-runtime) ──► Bedrock
                   (tráfego privado, sem internet, sem NAT para isso)
```

Benefícios: **segurança** (dados não saem para a internet), **compliance** (LGPD/dados sensíveis) e você pode exigir o endpoint via `aws:SourceVpce` na policy (visto em 5.1).

---

## 6. Governança e FinOps

### 6.1 Cost Allocation Tags

Marque **todos** os recursos com tags consistentes para poder fatiar o custo depois:

| Tag | Exemplo | Serve para |
|-----|---------|-----------|
| `tenant` | `acme-corp` | Chargeback por cliente |
| `feature` | `chatbot-suporte` | Custo por funcionalidade |
| `env` | `prod` / `staging` | Separar produção de testes |
| `team` | `plataforma-ia` | Showback interno |

Ative as tags como **cost allocation tags** no Billing para elas aparecerem no Cost Explorer.

### 6.2 AWS Budgets com alertas

```
Budget "LLM-prod-mensal": limite US$ 30.000
  ├─ Alerta em 50%  → e-mail para o time
  ├─ Alerta em 80%  → Slack + on-call
  └─ Alerta em 100% → SNS → Lambda que ATIVA modo econômico
                      (força fallback para modelo barato em todos os tenants)
```

Crie budgets **por tenant** usando filtro de tag para detectar quem está estourando.

### 6.3 Cost Explorer e Anomaly Detection

- **Cost Explorer:** visualize custo por tag (`tenant`, `feature`), por serviço, por período.
- **Cost Anomaly Detection:** ML da AWS que aprende seu padrão e alerta em picos anômalos (ex.: um bug que triplicou as chamadas ao modelo caro).

### 6.4 Métricas customizadas de tokens (CloudWatch)

O Cost Explorer mostra dólares, mas para governança fina você quer **tokens por tenant em tempo quase real**:

```python
import boto3
cloudwatch = boto3.client("cloudwatch")

def emitir_metrica_tokens(tenant_id: str, feature: str, tokens_in: int, tokens_out: int):
    cloudwatch.put_metric_data(
        Namespace="LLM/Consumo",
        MetricData=[
            {
                "MetricName": "TokensEntrada",
                "Dimensions": [
                    {"Name": "Tenant", "Value": tenant_id},
                    {"Name": "Feature", "Value": feature},
                ],
                "Value": tokens_in, "Unit": "Count",
            },
            {
                "MetricName": "TokensSaida",
                "Dimensions": [
                    {"Name": "Tenant", "Value": tenant_id},
                    {"Name": "Feature", "Value": feature},
                ],
                "Value": tokens_out, "Unit": "Count",
            },
        ],
    )
```

Com isso você monta **dashboards por tenant/feature** e cria **alarmes** (ex.: tenant X passou de N tokens/hora).

### 6.5 Chargeback vs Showback

- **Showback:** você *mostra* a cada time/tenant quanto ele custou (transparência, sem cobrar).
- **Chargeback:** você *cobra* de fato o custo (rateio real). Depende de tags e métricas confiáveis.

### 6.6 Bedrock Model Invocation Logging

O Bedrock pode logar **todos os prompts e respostas** (input/output) para S3 e/ou CloudWatch — essencial para auditoria, debug e compliance.

```
Bedrock Invocation Logging ──► S3 (retenção longa, criptografado com KMS)
                            └─► CloudWatch Logs (análise rápida)

  ⚠️ Esses logs contêm prompts e respostas — podem ter PII.
     Trate-os como dados sensíveis: KMS, retenção limitada, acesso restrito.
```

---

## 7. Guardrails e compliance

### 7.1 Bedrock Guardrails

Guardrails aplicam políticas **na entrada e na saída** do modelo, independentemente do modelo escolhido:

| Política | O que faz |
|----------|-----------|
| **PII** | Detecta e **mascara/bloqueia** CPF, e-mail, cartão, telefone |
| **Tópicos negados** | Bloqueia assuntos proibidos (ex.: aconselhamento jurídico/médico) |
| **Filtros de conteúdo** | Toxicidade, ódio, violência, conteúdo sexual |
| **Palavras/regex** | Bloqueia termos específicos ou padrões |
| **Contextual grounding** | Reduz alucinação verificando aderência ao contexto/RAG |

### 7.2 LGPD / GDPR — considerações

- **Minimização:** não envie mais dados pessoais ao modelo do que o necessário.
- **Mascaramento de PII:** anonimize antes de enviar (Guardrails ou Amazon Comprehend PII).
- **Localização de dados:** mantenha inferência e logs na região de compliance (ex.: `sa-east-1`); reforce com SCP.
- **Retenção:** defina retenção de logs (invocation logging, CloudWatch) e apague o que não precisa.
- **Direito ao esquecimento:** tenha processo para localizar e apagar dados de um titular (logs, vector store, DynamoDB).

### 7.3 Mascaramento de PII (esboço)

```python
comprehend = boto3.client("comprehend")

def mascarar_pii(texto: str) -> str:
    entidades = comprehend.detect_pii_entities(Text=texto, LanguageCode="pt")
    resultado = texto
    for e in sorted(entidades["Entities"], key=lambda x: x["BeginOffset"], reverse=True):
        resultado = resultado[:e["BeginOffset"]] + f"[{e['Type']}]" + resultado[e["EndOffset"]:]
    return resultado
```

### 7.4 Auditoria com CloudTrail

- **CloudTrail** registra **quem** chamou **qual** API do Bedrock, **quando** e **de onde** (identidade, IP, VPC endpoint).
- Combine CloudTrail (chamadas de controle/gerência) + Invocation Logging (conteúdo de prompt/resposta) para trilha de auditoria completa.

---

## 8. Arquitetura de referência completa

```
┌──────────┐
│ USUÁRIO  │
└────┬─────┘
     │ HTTPS
     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY                                       │
│  • Autorizador (Cognito / JWT / Lambda authorizer)  ← CONTROLE DE ACESSO   │
│  • Usage Plans + API Keys (rate limit / quota)      ← CONTROLE DE TOKENS   │
│  • WAF opcional                                     ← SEGURANÇA            │
└──────────────────────────────┬────────────────────────────────────────────┘
                               │ (injeta tenant_id, tier nos claims)
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                LAMBDA ORQUESTRADOR (IAM Execution Role)                    │
│                        ┌──────────────────────────┐                        │
│                        │  Sem credenciais estáticas│  ← CREDENCIAIS        │
│                        │  (STS entrega temporárias)│                        │
│                        └──────────────────────────┘                        │
│  1. Lê orçamento/estado ──────────► DynamoDB (tokens_usados por tenant)    │
│  2. Escolhe modelo (rebaixa se >90% orçamento)      ← CONTROLE DE TOKENS   │
│  3. Aplica Guardrails / mascara PII                 ← COMPLIANCE           │
│  4. Lê segredos externos ─────────► Secrets Manager (OpenAI etc., KMS)     │
│  5. Emite métricas de tokens ─────► CloudWatch (por tenant/feature)        │
└──────┬───────────────────────────────────────────────┬───────────────────┘
       │ ROTEADOR                                        │
       ▼                                                 ▼
┌─────────────────────────────┐              ┌──────────────────────────────┐
│  BEDROCK (via VPC Endpoint) │              │  VECTOR STORE (RAG)           │
│  • Só modelos permitidos    │◄─ IAM policy │  OpenSearch / Aurora pgvector │
│  • Só região permitida      │              │  isolado por tenant           │
│  • Tráfego PRIVADO (não sai  │              └──────────────────────────────┘
│    para a internet)          │  ← CONTROLE DE ACESSO + SEGURANÇA
└──────────────┬──────────────┘
               │ Invocation Logging (prompts/respostas)
               ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  OBSERVABILIDADE E AUDITORIA                                                 │
│  • CloudWatch Logs/Metrics (tokens, latência, erros) ← FINOPS               │
│  • CloudTrail (quem chamou o quê)                    ← AUDITORIA            │
│  • S3 + KMS (invocation logs, retenção)              ← COMPLIANCE          │
│  • AWS Budgets → SNS → Lambda "modo econômico"       ← ALERTAS DE CUSTO    │
│  • Cost Explorer + Anomaly Detection (por tag/tenant)                       │
└───────────────────────────────────────────────────────────────────────────┘

  Onde cada controle entra:
   ACESSO   → API Gateway (auth) + IAM (menor privilégio) + VPC Endpoint + SCP
   TOKENS   → Usage Plans + DynamoDB orçamento + max_tokens + fallback
   CRED.    → IAM Role (Lambda) + Secrets Manager + KMS + sem hardcode
   FINOPS   → Tags + Budgets + CloudWatch métricas + Cost Explorer
   COMPLI.  → Guardrails + CloudTrail + Invocation Logging + mascaramento PII
```

---

## 9. Multi-tenant em detalhe

Multi-tenancy significa vários clientes/organizações compartilhando a mesma infraestrutura. Você precisa isolar **três coisas**: dados, custo e capacidade.

### 9.1 Isolamento de dados

| Estratégia | Descrição | Isolamento | Custo |
|------------|-----------|------------|-------|
| **Silo** | Recursos dedicados por tenant (índice/DB próprio) | 🟢 Máximo | 🔴 Alto |
| **Pool + partição** | Recursos compartilhados, dados particionados por `tenant_id` | 🟡 Lógico | 🟢 Baixo |
| **Bridge** | Híbrido: pool para pequenos, silo para grandes | 🟡 | 🟡 |

**No vector store (RAG):** use `tenant_id` como filtro obrigatório em toda query e/ou índices separados. Nunca deixe a busca vetorial cruzar dados entre tenants:

```python
# Todo filtro de busca DEVE conter o tenant — imposto no código, não opcional.
def buscar_contexto(tenant_id: str, query_embedding, k: int = 5):
    return vector_store.search(
        vector=query_embedding,
        k=k,
        filter={"tenant_id": tenant_id},  # OBRIGATÓRIO — isolamento lógico
    )
```

### 9.2 Atribuição de custo por tenant

- **Cost allocation tags** (`tenant`) em recursos compartilhados só resolvem parte — inferência on-demand não é taggeada por request.
- Para custo **por tenant** de tokens: contabilize os tokens de cada request (do `usage` da resposta) e some por `tenant_id` (DynamoDB + CloudWatch, como na seção 3.3 e 6.4). É a sua fonte de verdade para chargeback.

### 9.3 Quotas por tenant

```python
# Orçamento e limites POR TENANT (poderiam vir de uma tabela de configuração).
LIMITES_TENANT = {
    "acme-corp":  {"orcamento_tokens": 20_000_000, "rpm": 300, "tier": "enterprise"},
    "startup-x":  {"orcamento_tokens":  2_000_000, "rpm":  60, "tier": "pro"},
    "free-user":  {"orcamento_tokens":    100_000, "rpm":  10, "tier": "free"},
}

def limites_do_tenant(tenant_id: str) -> dict:
    return LIMITES_TENANT.get(tenant_id, LIMITES_TENANT["free-user"])
```

Aplique: quota mensal de tokens (bloqueia/rebaixa), rate limit por minuto (throttling), e escolha de modelo por tier (free → só modelo barato; enterprise → modelo premium).

---

## 10. Checklist de produção, riscos e glossário

### 10.1 Checklist de produção

**Segurança / Credenciais**
- [ ] Nenhuma access key ou segredo hardcoded (verificado com scanner de segredos no CI).
- [ ] Lambda/ECS usam IAM Roles; on-prem usa Roles Anywhere.
- [ ] Segredos de terceiros no Secrets Manager com rotação habilitada.
- [ ] KMS (CMK) criptografando segredos, DynamoDB, logs e vector store.
- [ ] Logs não vazam PII/segredos (data protection policies ativas).

**Controle de acesso**
- [ ] IAM de menor privilégio: só modelos e regiões permitidos.
- [ ] `Deny` explícito para modelos caros e regiões fora de compliance.
- [ ] API Gateway com autorizador (Cognito/JWT/Lambda) em toda rota.
- [ ] VPC Endpoint (PrivateLink) para Bedrock; policy exige `aws:SourceVpce`.
- [ ] SCPs na organização como guardrail final.

**Custo / Tokens**
- [ ] `max_tokens` de saída definido em todo request.
- [ ] Orçamento por tenant em DynamoDB com incremento atômico.
- [ ] Degradação graciosa (fallback para modelo barato) implementada.
- [ ] Rate limiting por API key / usuário.
- [ ] Cost allocation tags em todos os recursos.
- [ ] AWS Budgets com alertas (50/80/100%) e ação automática no 100%.
- [ ] Cost Anomaly Detection ativo.
- [ ] Métricas de tokens por tenant/feature no CloudWatch.
- [ ] Prompt caching avaliado para prompts com prefixo grande e repetido.

**Compliance / Auditoria**
- [ ] Bedrock Guardrails (PII, tópicos, toxicidade) configurados.
- [ ] Model Invocation Logging para S3 (KMS, retenção definida).
- [ ] CloudTrail habilitado e monitorado.
- [ ] Mascaramento de PII antes da inferência quando aplicável.
- [ ] Processo de retenção e direito ao esquecimento documentado.

### 10.2 Tabela de riscos e mitigações

| # | Risco | Impacto | Mitigação |
|---|-------|---------|-----------|
| 1 | Loop de agente descontrolado gera milhares de US$ | 🔴 Alto | `max_tokens`, limite de iterações, orçamento por tenant, Budgets com ação automática |
| 2 | Credencial vazada em git/logs | 🔴 Alto | IAM Roles (sem chaves), Secrets Manager, scanner de segredos, mascaramento de logs |
| 3 | Um tenant consome capacidade e degrada os outros | 🟡 Médio | Quotas + rate limiting por tenant, throttling na borda |
| 4 | Vazamento de dados entre tenants no RAG | 🔴 Alto | Filtro `tenant_id` obrigatório, índices separados, testes de isolamento |
| 5 | PII enviada ao modelo / exposta em logs | 🔴 Alto | Guardrails, mascaramento (Comprehend), data protection policies |
| 6 | Uso de modelo caro/errado ou em região indevida | 🟡 Médio | IAM `Deny` por modelo/região, SCPs, roteamento por tier |
| 7 | Tráfego sensível saindo para a internet | 🟡 Médio | VPC Endpoint (PrivateLink), `aws:SourceVpce` na policy |
| 8 | Conta estoura sem ninguém perceber | 🟡 Médio | Budgets, Anomaly Detection, dashboards por tenant |
| 9 | Segredo comprometido sem rotação | 🟡 Médio | Rotação automática no Secrets Manager, TTL de cache curto |
| 10 | Ausência de trilha de auditoria | 🟡 Médio | CloudTrail + Invocation Logging + retenção em S3/KMS |

### 10.3 Glossário

| Termo | Significado |
|-------|-------------|
| **Token** | Unidade de texto que o modelo processa (~4 caracteres em inglês). Base da cobrança. |
| **On-Demand** | Cobrança por token, sem compromisso. |
| **Provisioned Throughput** | Capacidade reservada, cobrada por model unit/hora. |
| **Batch** | Processamento assíncrono com desconto. |
| **Prompt Caching** | Reuso do prefixo do prompt já processado, cobrado bem mais barato. |
| **IAM Role** | Identidade com permissões e credenciais temporárias — sem chaves estáticas. |
| **Roles Anywhere** | IAM Roles para cargas fora da AWS via certificado X.509. |
| **Secrets Manager** | Serviço gerenciado para segredos, com rotação automática. |
| **Parameter Store** | Armazenamento de config/segredos (SecureString com KMS). |
| **KMS** | Serviço de gerenciamento de chaves de criptografia. |
| **SCP** | Service Control Policy — guardrail no nível da organização. |
| **VPC Endpoint / PrivateLink** | Acesso privado a serviços AWS sem passar pela internet. |
| **Cost Allocation Tag** | Tag usada para fatiar custo no Billing/Cost Explorer. |
| **Chargeback / Showback** | Cobrar / apenas mostrar o custo por time/tenant. |
| **Guardrails** | Políticas do Bedrock (PII, tópicos, toxicidade) na entrada/saída. |
| **Invocation Logging** | Log de prompts e respostas do Bedrock para auditoria. |
| **Multi-tenancy** | Vários clientes compartilhando a mesma infraestrutura. |
| **Degradação graciosa** | Cair para modelo mais barato ao estourar orçamento, em vez de bloquear. |

### 10.4 Resumo em uma frase

> **Em uma arquitetura de LLM na AWS, tokens dominam o custo — então controle tokens com orçamentos e fallback, elimine credenciais estáticas com IAM Roles e Secrets Manager, restrinja acesso por IAM/API Gateway/VPC Endpoint, e torne tudo auditável e previsível com tags, Budgets, Guardrails e CloudTrail.**

---

*Documento didático — camada de Governança, FinOps e Segurança. Valores monetários são ilustrativos; confirme sempre na página de preços do Bedrock e na AWS Pricing Calculator.*
