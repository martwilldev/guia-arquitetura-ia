# Estudo de Caso 1 — Chatbot de FAQ / Atendimento Simples

> Um cenário end-to-end de **atendimento por FAQ**: como a **técnica** (FAQ no prompt vs. RAG simples vs. Knowledge Base gerenciada) e a **escolha do modelo** mudam drasticamente o **custo** e a **complexidade** — para o mesmo produto.

📍 Volte ao índice dos estudos de caso: [README.md](README.md)

---

## Índice

1. [O problema e os requisitos](#1-o-problema-e-os-requisitos)
2. [Arquitetura geral](#2-arquitetura-geral)
3. [Opção A — SDK puro + modelo barato + FAQ no prompt 🟢](#3-opção-a--sdk-puro--modelo-barato--faq-no-prompt-)
4. [Opção B — LangChain + modelo médio + RAG simples 🟡](#4-opção-b--langchain--modelo-médio--rag-simples-)
5. [Opção C — Bedrock Agents + Knowledge Base gerenciada 🟡](#5-opção-c--bedrock-agents--knowledge-base-gerenciada-)
6. [Tabela comparativa](#6-tabela-comparativa)
7. [Recomendação por perfil](#7-recomendação-por-perfil)
8. [Armadilhas](#8-armadilhas)
9. [Glossário](#9-glossário)
10. [Resumo em uma frase](#10-resumo-em-uma-frase)

---

## 1. O problema e os requisitos

Uma empresa quer um **chatbot de atendimento** que responda perguntas frequentes: horários, políticas de troca, status de conta, "como faço X". O conhecimento cabe em um conjunto **pequeno e estável** de FAQs (dezenas a poucas centenas de itens).

| Requisito | Alvo |
|-----------|------|
| **Volume** | ~100.000 mensagens/mês (~3.300/dia) |
| **Latência** | < 3 s por resposta (idealmente < 1,5 s) |
| **Qualidade** | Respostas corretas e no tom da marca; recusar o que não sabe |
| **Base de conhecimento** | FAQ pequena, muda poucas vezes por mês |
| **Orçamento** | O mais enxuto possível — é atendimento de alto volume e baixa margem |

**Tamanho típico da requisição (para as contas):**

- Entrada: ~600 tokens (system prompt + FAQ no contexto ou trechos recuperados + pergunta).
- Saída: ~150 tokens.
- Nas opções sem FAQ inteiro no prompt, a entrada cai para ~300 tokens.

> ⚠️ **Todos os custos abaixo são ILUSTRATIVOS**, em ordem de grandeza, para raciocínio comparativo. Confirme sempre na [Calculadora de Preços da AWS](https://calculator.aws) e na página de [preços do Amazon Bedrock](https://aws.amazon.com/bedrock/pricing/).

---

## 2. Arquitetura geral

O esqueleto é o mesmo nas três opções; muda o **miolo** (como o conhecimento chega ao modelo).

```
   Usuário
      │  (mensagem)
      ▼
 ┌──────────────┐     ┌─────────────────────────────────────┐
 │ API Gateway  │────►│           AWS Lambda                 │
 └──────────────┘     │  (orquestração da resposta)          │
                      │                                      │
                      │   ┌──────── MIOLO (varia A/B/C) ───┐ │
                      │   │ A: FAQ inteira no prompt        │ │
                      │   │ B: RAG simples (embeddings+top-k)│ │
                      │   │ C: Bedrock Agents + KB           │ │
                      │   └──────────────────────────────────┘ │
                      └───────────────┬──────────────────────┘
                                      ▼
                            ┌──────────────────┐
                            │  Amazon Bedrock  │  (modelo)
                            └──────────────────┘
```

---

## 3. Opção A — SDK puro + modelo barato + FAQ no prompt 🟢

**Framework:** nenhum (boto3 puro) · **Modelo:** Claude 3 Haiku / Amazon Nova Micro · **Técnica:** FAQ inteira injetada no prompt (context stuffing) · **Compute:** Lambda.

### Como funciona

A FAQ inteira (ou o bloco relevante) vive num arquivo/variável. A cada mensagem, o system prompt já traz a FAQ e a instrução "responda apenas com base nisto; se não souber, diga que vai transferir". Sem banco vetorial, sem ingestão, sem infraestrutura extra.

Funciona **enquanto a FAQ couber confortavelmente na janela de contexto** (dezenas a poucas centenas de Q&As).

### Exemplo de código (Lambda-ready)

```python
import json
import boto3

# Clientes fora do handler: reaproveitados entre invocações (evita cold start repetido)
bedrock = boto3.client("bedrock-runtime")

MODEL_ID = "amazon.nova-micro-v1:0"  # ou "anthropic.claude-3-haiku-20240307-v1:0"

# FAQ estática carregada uma vez (poderia vir de S3 no cold start)
FAQ = """
P: Qual o horário de atendimento?
R: De segunda a sexta, das 8h às 18h.
P: Como troco um produto?
R: Solicite a troca em até 7 dias no menu Pedidos > Trocar.
... (dezenas de itens) ...
"""

SYSTEM = (
    "Você é o atendente virtual da Loja X. Responda SOMENTE com base na FAQ abaixo. "
    "Se a resposta não estiver na FAQ, diga que vai transferir para um humano.\n\n"
    f"FAQ:\n{FAQ}"
)

def handler(event, context):
    pergunta = json.loads(event["body"])["mensagem"]

    resp = bedrock.converse(
        modelId=MODEL_ID,
        system=[{"text": SYSTEM}],
        messages=[{"role": "user", "content": [{"text": pergunta}]}],
        inferenceConfig={"maxTokens": 200, "temperature": 0.2},
    )
    texto = resp["output"]["message"]["content"][0]["text"]
    return {"statusCode": 200, "body": json.dumps({"resposta": texto})}
```

### Custo estimado mensal (ILUSTRATIVO)

Preços de ordem de grandeza para modelo barato (~US$ 0,25 / 1M tokens entrada, ~US$ 1,25 / 1M saída). Como a FAQ vai no prompt, a entrada é maior (~600 tokens).

| Item | Conta | Custo |
|------|-------|-------|
| Tokens entrada | 100k msg × 600 tok = 60M tok × US$ 0,25/1M | ~US$ 15 |
| Tokens saída | 100k msg × 150 tok = 15M tok × US$ 1,25/1M | ~US$ 19 |
| Lambda | 100k inv × ~1,5 s × 256 MB | ~US$ 1 |
| API Gateway | 100k req | ~US$ 0,35 |
| Storage | — (FAQ em arquivo/S3, centavos) | ~US$ 0 |
| **Total** | | **~US$ 35/mês** |

### Complexidade: 🟢 baixa

### Trade-offs

- ✅ Baratíssimo, simples de manter, latência baixa (modelo pequeno).
- ✅ Zero infraestrutura de dados.
- ❌ Não escala com base grande: FAQ grande infla a entrada (custo por token sobe) e um dia estoura o contexto.
- ❌ Atualizar a FAQ = editar/redeploy (ou ler de S3).

---

## 4. Opção B — LangChain + modelo médio + RAG simples 🟡

**Framework:** LangChain · **Modelo:** Claude 3.5 Sonnet / Nova Lite (médio) · **Técnica:** RAG simples (embeddings + top-k) · **Compute:** Lambda + vetor em memória/pgvector leve.

### Como funciona

Em vez de mandar a FAQ inteira, você **indexa** os itens como embeddings uma vez. Na consulta, busca os **3–5 trechos mais parecidos** e injeta só eles no prompt. A entrada fica menor e a base pode crescer para centenas/milhares de itens sem estourar contexto.

Para uma FAQ pequena, o índice cabe até num arquivo carregado em memória (FAISS/em-memória) — sem servidor de banco dedicado.

### Exemplo de código (Lambda-ready)

```python
import json
import boto3
from langchain_aws import ChatBedrock, BedrockEmbeddings
from langchain_community.vectorstores import FAISS

# Clientes/objetos pesados fora do handler
llm = ChatBedrock(model_id="anthropic.claude-3-5-sonnet-20240620-v1:0",
                  model_kwargs={"max_tokens": 250, "temperature": 0.2})
embeddings = BedrockEmbeddings(model_id="amazon.titan-embed-text-v2:0")

# Índice carregado do S3 no cold start (gerado offline a partir da FAQ)
vectorstore = FAISS.load_local("/tmp/faq_index", embeddings,
                               allow_dangerous_deserialization=True)

PROMPT = ("Responda a pergunta usando apenas o CONTEXTO. "
          "Se não houver resposta no contexto, diga que vai transferir.\n\n"
          "CONTEXTO:\n{ctx}\n\nPERGUNTA: {q}")

def handler(event, context):
    q = json.loads(event["body"])["mensagem"]
    docs = vectorstore.similarity_search(q, k=4)          # top-k
    ctx = "\n---\n".join(d.page_content for d in docs)
    resp = llm.invoke(PROMPT.format(ctx=ctx, q=q))
    return {"statusCode": 200, "body": json.dumps({"resposta": resp.content})}
```

### Custo estimado mensal (ILUSTRATIVO)

Modelo médio é mais caro por token (~US$ 3 / 1M entrada, ~US$ 15 / 1M saída), mas a entrada cai (~300 tokens porque só trechos relevantes vão no prompt). Some o custo de embeddings da consulta.

| Item | Conta | Custo |
|------|-------|-------|
| Tokens entrada | 100k × 300 tok = 30M × US$ 3/1M | ~US$ 90 |
| Tokens saída | 100k × 150 tok = 15M × US$ 15/1M | ~US$ 225 |
| Embeddings (query) | 100k × ~20 tok = 2M × ~US$ 0,02/1M | ~US$ 0,04 |
| Lambda | 100k inv × ~2 s × 512 MB (LangChain + FAISS) | ~US$ 3 |
| API Gateway | 100k req | ~US$ 0,35 |
| Storage | índice FAISS no S3 (poucos MB) | ~US$ 0 |
| **Total** | | **~US$ 320/mês** |

> 💡 O salto de custo vem quase todo do **modelo médio**, não do RAG. Trocar o modelo por um barato aqui derrubaria o total para a faixa da Opção A.

### Complexidade: 🟡 média

### Trade-offs

- ✅ Escala para bases maiores sem estourar contexto; qualidade de resposta melhor (modelo médio + trechos focados).
- ✅ Atualizar a base = reindexar (pipeline offline), sem mexer no prompt.
- ❌ Dependência do LangChain (versionamento, tamanho do pacote no Lambda).
- ❌ Você opera o índice (gerar, versionar, carregar do S3).
- ❌ Custo domina pelo modelo — dimensione o modelo ao valor real da tarefa.

---

## 5. Opção C — Bedrock Agents + Knowledge Base gerenciada 🟡

**Framework:** Bedrock Agents (gerenciado) · **Modelo:** modelo médio associado ao agente · **Técnica:** Knowledge Base gerenciada (ingestão + retrieval automáticos) · **Compute:** gerenciado pela AWS (você só chama a API).

### Como funciona

Você aponta uma **Knowledge Base** para um bucket S3 com os documentos de FAQ. A AWS faz **chunking, embeddings, indexação (em OpenSearch Serverless por padrão) e retrieval** automaticamente. Seu Lambda só chama `retrieve_and_generate` (ou o agente) e recebe a resposta com citações. Menos código, mais serviço gerenciado.

### Exemplo de código / config (Lambda-ready)

```python
import json
import boto3

bedrock_agent = boto3.client("bedrock-agent-runtime")

KB_ID = "XXXXXXXXXX"
MODEL_ARN = "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20240620-v1:0"

def handler(event, context):
    q = json.loads(event["body"])["mensagem"]
    resp = bedrock_agent.retrieve_and_generate(
        input={"text": q},
        retrieveAndGenerateConfiguration={
            "type": "KNOWLEDGE_BASE",
            "knowledgeBaseConfiguration": {
                "knowledgeBaseId": KB_ID,
                "modelArn": MODEL_ARN,
            },
        },
    )
    return {"statusCode": 200,
            "body": json.dumps({"resposta": resp["output"]["text"]})}
```

Ingestão (config, não código do request): sincronizar o data source aponta para o S3 e dispara a indexação.

### Custo estimado mensal (ILUSTRATIVO)

Some tokens do modelo (semelhante à B) **+ o custo do índice gerenciado**. O ponto de atenção é o **OpenSearch Serverless**, que tem um **piso de OCUs** cobrado por hora, independente do volume.

| Item | Conta | Custo |
|------|-------|-------|
| Tokens entrada | 100k × ~350 tok = 35M × US$ 3/1M | ~US$ 105 |
| Tokens saída | 100k × 150 tok = 15M × US$ 15/1M | ~US$ 225 |
| Embeddings ingestão | pequena, one-time por atualização | ~US$ 1 |
| OpenSearch Serverless | piso ~2 OCU (search+index) × ~US$ 0,24/OCU-h × 730h | ~US$ 350 |
| Lambda + API GW | 100k req | ~US$ 3 |
| **Total** | | **~US$ 685/mês** |

> ⚠️ Para FAQ **pequena**, o **piso do OpenSearch Serverless** domina e torna a opção cara por mensagem. Ela compensa quando a base cresce (aí o mesmo piso serve muito mais conteúdo/consultas).

### Complexidade: 🟡 média

Pouco código, mas exige configurar KB, data source, permissões IAM e entender o modelo de cobrança do índice.

### Trade-offs

- ✅ Menos código para manter; ingestão/retrieval gerenciados; citações prontas.
- ✅ Escala bem para bases grandes.
- ❌ Piso de custo do índice gerenciado pesa em volume pequeno.
- ❌ Menos controle fino sobre chunking/retrieval; algum lock-in na AWS.

---

## 6. Tabela comparativa

| Critério | A — SDK + FAQ no prompt 🟢 | B — LangChain + RAG 🟡 | C — Bedrock Agents + KB 🟡 |
|----------|---------------------------|------------------------|----------------------------|
| **Custo/mês (ilustrativo)** | ~US$ 35 | ~US$ 320 | ~US$ 685 |
| **Fator dominante do custo** | Tokens (barato) | Modelo médio | Piso do índice + modelo |
| **Complexidade** | 🟢 baixa | 🟡 média | 🟡 média |
| **Latência** | Muito baixa (modelo pequeno) | Média | Média |
| **Qualidade** | Boa p/ FAQ simples | Alta | Alta |
| **Escala da base** | Ruim (estoura contexto) | Boa | Ótima |
| **Operação** | Você mantém prompt | Você mantém índice | AWS mantém índice |
| **Quando escolher** | FAQ pequena e estável, alto volume, orçamento apertado | Base cresce, quer qualidade e controle | Quer gerenciado e base já grande |

---

## 7. Recomendação por perfil

- **MVP / validação:** **Opção A**. Custo mínimo, sobe em um dia, latência ótima. Enquanto a FAQ couber no prompt, é imbatível em custo-benefício.
- **Crescimento (base cresce, qualidade importa):** **Opção B** — mas **comece com o modelo barato** e só suba para o médio se a qualidade não bastar. O RAG te tira do problema de contexto sem obrigar modelo caro.
- **Escala (base grande, time pequeno, quer gerenciado):** **Opção C**. O piso do índice se dilui no volume e você para de operar pipeline de embeddings.

---

## 8. Armadilhas

- **Escolher modelo caro cedo demais.** Em FAQ, o modelo barato costuma bastar. O maior custo da Opção B/C vem do modelo, não da técnica.
- **Piso oculto do OpenSearch Serverless.** Em volume pequeno, o piso de OCUs pode custar mais que todos os tokens juntos. Meça antes.
- **FAQ inteira no prompt "para sempre".** Funciona até a base crescer; monitore o tamanho da entrada — cada token repetido em 100k mensagens vira dinheiro.
- **Não instruir a recusar.** Sem "se não souber, transfira", o modelo inventa (alucina) respostas de atendimento. Sempre restrinja ao contexto.
- **Pacote do LangChain no Lambda.** Dependências pesadas inflam cold start; use camadas (layers) e mantenha o mínimo.

---

## 9. Glossário

| Termo | Significado |
|-------|-------------|
| **Context stuffing** | Colocar todo o conhecimento (a FAQ) diretamente no prompt. |
| **RAG** | Retrieval-Augmented Generation: buscar trechos relevantes e injetá-los no prompt. |
| **top-k** | Os *k* trechos mais parecidos com a pergunta, recuperados do índice. |
| **Knowledge Base (Bedrock)** | Serviço gerenciado que faz ingestão, embeddings, indexação e retrieval por você. |
| **OCU** | OpenSearch Compute Unit — unidade de cobrança do OpenSearch Serverless (tem piso). |
| **Cold start** | Latência extra na primeira invocação de um Lambda "frio". |

---

## 10. Resumo em uma frase

> Para FAQ pequena e de alto volume, a técnica mais simples (FAQ no prompt com modelo barato) ganha em custo; RAG e Knowledge Base gerenciada só valem a pena quando a base cresce — e o custo é dominado pela escolha do modelo, não pela técnica.
