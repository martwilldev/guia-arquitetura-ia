# Estudo de Caso 2 — Chat com Meus Documentos (RAG) sobre Base Grande

> Um cenário end-to-end de **RAG em escala**: como a escolha do **vector store** (gerenciado vs. pgvector vs. self-hosted) e da **técnica de recuperação** (retrieval simples vs. híbrido + reranking) muda custo, complexidade e qualidade — para 5 milhões de vetores e 50 mil consultas/mês.

📍 Volte ao índice dos estudos de caso: [README.md](README.md)

---

## Índice

1. [O problema e os requisitos](#1-o-problema-e-os-requisitos)
2. [Arquitetura geral](#2-arquitetura-geral)
3. [Opção A — Bedrock Knowledge Bases + OpenSearch Serverless 🟢 (oper.)](#3-opção-a--bedrock-knowledge-bases--opensearch-serverless--oper)
4. [Opção B — LlamaIndex + pgvector (Aurora) + Titan 🟡](#4-opção-b--llamaindex--pgvector-aurora--titan-)
5. [Opção C — Haystack + Qdrant self-hosted + rerank em serverless GPU 🔴](#5-opção-c--haystack--qdrant-self-hosted--rerank-em-serverless-gpu-)
6. [Tabela comparativa](#6-tabela-comparativa)
7. [Recomendação por perfil](#7-recomendação-por-perfil)
8. [Armadilhas](#8-armadilhas)
9. [Glossário](#9-glossário)
10. [Resumo em uma frase](#10-resumo-em-uma-frase)

---

## 1. O problema e os requisitos

Uma empresa quer um **"chat com meus documentos"**: manuais, contratos, wikis internas, tickets — uma base **grande e que cresce**. O usuário faz uma pergunta e recebe uma resposta fundamentada nos documentos, com citações.

| Requisito | Alvo |
|-----------|------|
| **Volume de dados** | ~5.000.000 de vetores (chunks) — base grande |
| **Volume de consultas** | ~50.000 queries/mês (~1.700/dia) |
| **Latência** | < 4 s por resposta (retrieval + geração) |
| **Qualidade** | Alta precisão de recuperação; respostas fundamentadas com fontes |
| **Ingestão** | Contínua (novos docs entram todo dia) |

**Tamanho típico da requisição (para as contas):**

- Retrieval traz ~6 chunks × ~200 tokens = ~1.200 tokens de contexto + ~300 de instrução/pergunta ≈ **~1.500 tokens de entrada**.
- Saída: ~350 tokens.

> ⚠️ **Todos os custos abaixo são ILUSTRATIVOS**, em ordem de grandeza, para raciocínio comparativo. Confirme na [Calculadora de Preços da AWS](https://calculator.aws), nos [preços do Bedrock](https://aws.amazon.com/bedrock/pricing/) e nas páginas de OpenSearch, Aurora, ECS/Fargate e da plataforma de GPU escolhida.

---

## 2. Arquitetura geral

RAG tem sempre **dois caminhos**: **ingestão** (offline, indexar documentos) e **consulta** (online, responder). O que varia é onde ficam os vetores e quão sofisticado é o retrieval.

```
   INGESTÃO (offline)                         CONSULTA (online)
  ┌───────────┐                              Usuário
  │ Documentos│                                 │ (pergunta)
  │  (S3)     │                                 ▼
  └─────┬─────┘                          ┌─────────────┐
        ▼                                │ Lambda /    │
  chunk + embeddings ──►  VECTOR STORE ◄─┤ orquestração│
                          (varia A/B/C)  └──────┬──────┘
                                                ▼
                                      [retrieval → (rerank?) →
                                       contexto → LLM Bedrock]
                                                ▼
                                        resposta + fontes
```

---

## 3. Opção A — Bedrock Knowledge Bases + OpenSearch Serverless 🟢 (oper.)

**Framework:** Bedrock Knowledge Bases (gerenciado) · **Modelo:** médio (Claude 3.5 Sonnet / Nova Lite) · **Técnica:** retrieval gerenciado (semântico, opção de híbrido) · **Compute:** gerenciado (OpenSearch Serverless + Bedrock).

### Como funciona

Você aponta a Knowledge Base para o S3; a AWS faz chunking, embeddings (Titan) e indexa em **OpenSearch Serverless**. Ingestão contínua = sincronizar o data source. Na consulta, um `retrieve_and_generate` faz tudo: busca, monta contexto e gera com citações. **Mínimo de operação**, você não mantém servidor de banco.

### Exemplo de código (Lambda-ready)

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
                "retrievalConfiguration": {
                    "vectorSearchConfiguration": {"numberOfResults": 6}
                },
            },
        },
    )
    out = resp["output"]["text"]
    fontes = [c["retrievedReferences"] for c in resp.get("citations", [])]
    return {"statusCode": 200, "body": json.dumps({"resposta": out, "fontes": fontes})}
```

### Custo estimado mensal (ILUSTRATIVO)

Com 5M de vetores, o índice do OpenSearch Serverless precisa de mais OCUs (memória para o índice), e há custo de storage. Some tokens do modelo.

| Item | Conta | Custo |
|------|-------|-------|
| Tokens entrada | 50k × 1.500 tok = 75M × US$ 3/1M | ~US$ 225 |
| Tokens saída | 50k × 350 tok = 17,5M × US$ 15/1M | ~US$ 263 |
| Embeddings ingestão | 5M chunks × ~200 tok = 1B tok × ~US$ 0,02/1M (one-time/amortizado) | ~US$ 20 |
| OpenSearch Serverless | ~4–6 OCU p/ 5M vetores × ~US$ 0,24/OCU-h × 730h | ~US$ 700–1.050 |
| Storage vetorial gerenciado | incluído/associado ao índice | ~US$ 30 |
| Lambda + API GW | 50k req | ~US$ 2 |
| **Total** | | **~US$ 1.250–1.600/mês** |

### Complexidade: 🟢 baixa (operacional)

Pouco código e nada de servidor. A complexidade é de **configuração e custo**, não de engenharia.

### Trade-offs

- ✅ Menor esforço de operação; ingestão e retrieval gerenciados; citações prontas.
- ✅ Escala sem você gerenciar cluster.
- ❌ OpenSearch Serverless tem custo relevante em escala (OCUs para 5M vetores).
- ❌ Menos controle sobre chunking, reranking e retrieval híbrido fino; lock-in AWS.

---

## 4. Opção B — LlamaIndex + pgvector (Aurora) + Titan 🟡

**Framework:** LlamaIndex · **Modelo:** médio + **embeddings Titan** · **Técnica:** retrieval semântico (opção de híbrido com full-text do Postgres) · **Compute:** **Lambda** na consulta, **Fargate** na ingestão · **Vector store:** **pgvector** em Aurora PostgreSQL.

### Como funciona

Você mesmo indexa: um job em **Fargate** lê do S3, faz chunking + embeddings (Titan) e grava no **pgvector** (Aurora). Na consulta, o **Lambda** gera o embedding da pergunta, faz `ORDER BY embedding <=> :q LIMIT k` no Postgres, monta o contexto e chama o Bedrock. Você controla tudo, reaproveitando um banco relacional que talvez já exista.

### Exemplo de código (Lambda-ready, consulta)

```python
import json
import os
import boto3
import psycopg2

# Fora do handler: cliente Bedrock + conexão reutilizável
bedrock = boto3.client("bedrock-runtime")
conn = psycopg2.connect(os.environ["DB_DSN"])  # Aurora + pgvector

EMB_MODEL = "amazon.titan-embed-text-v2:0"
LLM_MODEL = "anthropic.claude-3-5-sonnet-20240620-v1:0"

def embed(texto: str):
    r = bedrock.invoke_model(modelId=EMB_MODEL,
                             body=json.dumps({"inputText": texto}))
    return json.loads(r["body"].read())["embedding"]

def handler(event, context):
    q = json.loads(event["body"])["mensagem"]
    vq = embed(q)
    with conn.cursor() as cur:
        cur.execute(
            "SELECT texto FROM chunks ORDER BY embedding <=> %s::vector LIMIT 6",
            (vq,),
        )
        ctx = "\n---\n".join(row[0] for row in cur.fetchall())

    prompt = f"Responda com base no CONTEXTO e cite trechos.\n\nCONTEXTO:\n{ctx}\n\nPERGUNTA: {q}"
    resp = bedrock.converse(
        modelId=LLM_MODEL,
        messages=[{"role": "user", "content": [{"text": prompt}]}],
        inferenceConfig={"maxTokens": 400, "temperature": 0.2},
    )
    return {"statusCode": 200,
            "body": json.dumps({"resposta": resp["output"]["message"]["content"][0]["text"]})}
```

### Custo estimado mensal (ILUSTRATIVO)

Aurora tem custo de instância + storage. 5M vetores em pgvector ocupam alguns GB (dimensão do embedding × 5M). Ingestão em Fargate é intermitente.

| Item | Conta | Custo |
|------|-------|-------|
| Tokens entrada | 50k × 1.500 = 75M × US$ 3/1M | ~US$ 225 |
| Tokens saída | 50k × 350 = 17,5M × US$ 15/1M | ~US$ 263 |
| Embeddings query | 50k × ~20 tok = 1M × ~US$ 0,02/1M | ~US$ 0,02 |
| Embeddings ingestão | amortizado (novos docs/mês) | ~US$ 5 |
| Aurora PostgreSQL | 1 instância (ex.: db.r6g.large) contínua + storage ~US$ 350 + US$ 30 | ~US$ 380 |
| Fargate (ingestão) | jobs intermitentes, ~algumas horas/dia | ~US$ 40 |
| Lambda + API GW | 50k req | ~US$ 2 |
| **Total** | | **~US$ 915/mês** |

> 💡 Mais barato que a Opção A **nesse volume** porque uma instância Aurora fixa custa menos que 4–6 OCUs de OpenSearch. A conta se inverte se o volume de vetores/consultas crescer muito (Aurora precisaria de instância maior).

### Complexidade: 🟡 média

Você opera o banco (schema, índice HNSW/IVFFlat no pgvector, vacuum), o pipeline de ingestão (Fargate) e a conexão do Lambda ao Aurora (VPC, pool de conexões).

### Trade-offs

- ✅ Controle total; reaproveita Postgres; híbrido (vetor + full-text) fácil no mesmo banco.
- ✅ Custo previsível e competitivo neste volume.
- ❌ Você mantém banco, índice e ingestão; Lambda em VPC adiciona latência/complexidade de conexão.
- ❌ Escalar pgvector para dezenas de milhões de vetores exige tuning (índice, memória).

---

## 5. Opção C — Haystack + Qdrant self-hosted + rerank em serverless GPU 🔴

**Framework:** Haystack · **Modelo:** forte (Claude 3.5 Sonnet / topo) · **Técnica:** **busca híbrida** (denso + esparso) **+ reranking cross-encoder** · **Compute:** **Qdrant self-hosted em ECS**, **reranker em serverless GPU**, orquestração em Lambda/ECS · **Vector store:** **Qdrant**.

### Como funciona

Pipeline de recuperação de ponta: o Qdrant (rodando em ECS) faz busca **híbrida** (vetor denso + BM25) e traz ~30 candidatos; um **cross-encoder** reordena esses candidatos por relevância real (rerank) em uma **GPU serverless** (escala a zero entre picos); os melhores 6 vão para um **modelo forte** gerar a resposta. Máxima qualidade, máxima complexidade.

### Exemplo de config/código (Haystack, pipeline de consulta)

```python
# Pipeline montado fora do handler (componentes reutilizados entre invocações)
from haystack import Pipeline
from haystack_integrations.document_stores.qdrant import QdrantDocumentStore
from haystack_integrations.components.retrievers.qdrant import QdrantHybridRetriever
from haystack.components.rankers import TransformersSimilarityRanker  # cross-encoder

store = QdrantDocumentStore(url="http://qdrant.internal:6333", index="docs")

retriever = QdrantHybridRetriever(document_store=store, top_k=30)   # denso + esparso
ranker = TransformersSimilarityRanker(                              # roda no endpoint GPU
    model="BAAI/bge-reranker-large", top_k=6,
    # endpoint apontando para a função serverless-GPU
)

rag = Pipeline()
rag.add_component("retriever", retriever)
rag.add_component("ranker", ranker)
rag.connect("retriever.documents", "ranker.documents")

def handler(event, context):
    import json
    q = json.loads(event["body"])["mensagem"]
    res = rag.run({"retriever": {"query": q}, "ranker": {"query": q}})
    ctx = "\n---\n".join(d.content for d in res["ranker"]["documents"])
    # ... chamar modelo forte no Bedrock com ctx (converse) e retornar
    return {"statusCode": 200, "body": json.dumps({"contexto_top": ctx[:2000]})}
```

### Custo estimado mensal (ILUSTRATIVO)

Modelo **forte** encarece tokens; Qdrant em ECS é uma tarefa contínua; a GPU serverless cobra por segundo de uso (rerank de 50k queries).

| Item | Conta | Custo |
|------|-------|-------|
| Tokens entrada | 50k × 1.500 = 75M × US$ 3/1M (forte) | ~US$ 225 |
| Tokens saída | 50k × 350 = 17,5M × US$ 15/1M (forte) | ~US$ 263 |
| Qdrant (ECS Fargate) | tarefa contínua (ex.: 2 vCPU / 8 GB) + storage p/ 5M vetores | ~US$ 180 |
| Rerank em serverless GPU | 50k queries × ~0,3 s de GPU × preço/s (ex.: ~US$ 0,0006/s) + cold starts | ~US$ 60 |
| Embeddings ingestão | amortizado | ~US$ 5 |
| Lambda/ECS orquestração + API GW | 50k req | ~US$ 5 |
| **Total** | | **~US$ 740/mês** |

> 💡 Curiosamente pode ficar **mais barato que a Opção A** em infraestrutura (Qdrant + GPU on-demand < OCUs fixas), **mas** a complexidade operacional é muito maior. O rerank melhora a qualidade sem inflar tokens do LLM (você manda menos chunks, porém melhores).

### Complexidade: 🔴 alta

Você opera Qdrant (deploy, backup, escala), o endpoint de GPU serverless (empacotar o modelo, gerenciar cold start), a busca híbrida e o pipeline Haystack. Muitas peças móveis.

### Trade-offs

- ✅ Melhor qualidade de recuperação (híbrido + rerank) e controle total do stack.
- ✅ Sem lock-in de vector store; portável para outra nuvem.
- ❌ Alta complexidade operacional (vários serviços, GPU, cold start do reranker).
- ❌ Precisa de time com maturidade em infra/ML; mais superfície para falhar.

---

## 6. Tabela comparativa

| Critério | A — Bedrock KB + OpenSearch 🟢 | B — LlamaIndex + pgvector 🟡 | C — Haystack + Qdrant + rerank 🔴 |
|----------|-------------------------------|------------------------------|-----------------------------------|
| **Custo/mês (ilustrativo)** | ~US$ 1.250–1.600 | ~US$ 915 | ~US$ 740 |
| **Fator dominante** | OCUs do OpenSearch | Instância Aurora | Modelo forte + Qdrant |
| **Complexidade** | 🟢 baixa (oper.) | 🟡 média | 🔴 alta |
| **Latência** | Média | Média (VPC/Aurora) | Média-alta (rerank + híbrido) |
| **Qualidade de retrieval** | Boa | Boa (híbrido possível) | Melhor (híbrido + rerank) |
| **Controle / portabilidade** | Baixo (lock-in) | Alto | Máximo (sem lock-in) |
| **Operação** | AWS gerencia | Você gerencia banco | Você gerencia tudo |
| **Quando escolher** | Time pequeno, quer gerenciado | Já usa Postgres, quer controle e custo previsível | Qualidade é crítica e há time de infra/ML |

---

## 7. Recomendação por perfil

- **MVP / validação:** **Opção A**. Sobe rápido, sem operar banco. Aceite o custo do OpenSearch enquanto valida o produto; a velocidade de chegar ao mercado compensa.
- **Crescimento (custo previsível, quer controle):** **Opção B**. pgvector em Aurora dá controle, híbrido no mesmo banco e custo previsível — ótimo se o time já domina Postgres.
- **Escala / qualidade crítica:** **Opção C**. Quando a **qualidade de recuperação** é diferencial (jurídico, saúde, suporte técnico denso) e há time para operar Qdrant + rerank. O reranking é o maior ganho de qualidade por dólar aqui.

---

## 8. Armadilhas

- **Ignorar o piso do OpenSearch Serverless.** Em 5M de vetores, as OCUs necessárias podem dominar a conta — modele antes de escolher a Opção A.
- **Lambda em VPC sem cuidado (Opção B).** Conexões ao Aurora esgotam rápido; use pool/RDS Proxy e mantenha a conexão fora do handler.
- **Índice pgvector mal configurado.** Sem HNSW/IVF adequado, a busca em 5M vetores fica lenta. Ajuste `lists`/`ef_search`.
- **Rerank sem medir ganho (Opção C).** Reranking custa GPU e latência; só vale se melhorar métricas de recuperação de forma mensurável.
- **Chunking ruim em qualquer opção.** Chunks grandes demais desperdiçam tokens; pequenos demais perdem contexto. É a alavanca de qualidade mais subestimada.
- **Cold start do reranker serverless GPU.** O primeiro request após ociosidade pode estourar o SLA; considere warmup ou concorrência mínima.

---

## 9. Glossário

| Termo | Significado |
|-------|-------------|
| **RAG** | Retrieval-Augmented Generation: recuperar trechos e gerar resposta fundamentada. |
| **Vector store** | Banco especializado em busca por similaridade de embeddings. |
| **pgvector** | Extensão do PostgreSQL para armazenar/buscar vetores. |
| **Busca híbrida** | Combina busca densa (embeddings) com esparsa (BM25/palavra-chave). |
| **Reranking (cross-encoder)** | Reordenar candidatos por relevância real usando um modelo que lê par (pergunta, trecho). |
| **OCU** | OpenSearch Compute Unit — unidade de cobrança do OpenSearch Serverless. |
| **Serverless GPU** | GPU sob demanda que escala a zero, cobrada por tempo de uso. |
| **HNSW / IVF** | Índices de vizinhos aproximados (ANN) para busca vetorial rápida. |

---

## 10. Resumo em uma frase

> Em RAG de base grande, a escolha do vector store define o custo fixo (OpenSearch gerenciado vs. Aurora/pgvector vs. Qdrant self-hosted) e a técnica de recuperação (híbrido + rerank) define a qualidade — o gerenciado ganha em esforço, o self-hosted ganha em controle e qualidade, ao preço de operação.
