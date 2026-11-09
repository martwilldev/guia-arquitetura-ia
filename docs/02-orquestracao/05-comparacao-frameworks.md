# LangChain vs LlamaIndex vs Haystack vs Bedrock Agents — Comparação Profunda

> Análise detalhada dos 4 frameworks: arquitetura, implementação em **Lambda**, alternativas de compute (**EC2 / ECS / Fargate / App Runner**) e **custos** reais.

---

## Índice

1. [Panorama e filosofia de cada um](#1-panorama-e-filosofia-de-cada-um)
2. [LangChain em profundidade](#2-langchain-em-profundidade)
3. [LlamaIndex em profundidade](#3-llamaindex-em-profundidade)
4. [Haystack em profundidade](#4-haystack-em-profundidade)
5. [Bedrock Agents em profundidade](#5-bedrock-agents-em-profundidade)
6. [Comparação completa (tabelões)](#6-comparação-completa-tabelões)
7. [Onde rodar: Lambda vs EC2 vs ECS/Fargate vs App Runner](#7-onde-rodar-lambda-vs-ec2-vs-ecsfargate-vs-app-runner)
8. [Implementação em cada compute](#8-implementação-em-cada-compute)
9. [Análise de custos](#9-análise-de-custos)
10. [Matriz de decisão final](#10-matriz-de-decisão-final)

---

## 1. Panorama e filosofia de cada um

| Framework | Filosofia central | Melhor para |
|-----------|-------------------|-------------|
| **LangChain** | "Blocos de montar" para qualquer app com LLM | Agentes, chains, protótipos versáteis |
| **LlamaIndex** | Tudo gira em torno de **indexar e consultar dados** | RAG, "chat com seus documentos" |
| **Haystack** | **Pipelines** explícitos e produtizáveis | Busca/RAG de produção, times de engenharia |
| **Bedrock Agents** | **Serviço gerenciado**: a AWS orquestra o loop | Agentes sem gerenciar infra de orquestração |

### A grande divisão

```
                     VOCÊ CONTROLA O CÓDIGO           AWS CONTROLA A ORQUESTRAÇÃO
                     ┌───────────────────────┐        ┌──────────────────────┐
  Propósito geral →  │      LangChain        │        │                      │
  RAG-first       →  │      LlamaIndex       │        │   Bedrock Agents     │
  Pipelines       →  │      Haystack         │        │  (Lambda = tool)     │
                     └───────────────────────┘        └──────────────────────┘
                       roda em Lambda/EC2/ECS            roda no Bedrock;
                       (você empacota e escala)          você só provê tools
```

---

## 2. LangChain em profundidade

### 2.1 Arquitetura de componentes

```
┌─────────────────────────────────────────────────┐
│                   LangChain                       │
│                                                   │
│  Models ──► Prompts ──► Chains ──► Output Parsers │
│     │          │          │                       │
│     │          │          ├──► Agents (ReAct)     │
│     │          │          │       │               │
│     │          │          │       └──► Tools      │
│     │          │          │                       │
│  Memory ◄──────┴──────────┤                       │
│                           │                       │
│  Retrievers ◄─── Vector Stores ◄─── Embeddings    │
└─────────────────────────────────────────────────┘
```

**Componentes-chave:**
- **Models** — wrappers para LLMs (`ChatBedrock`, `ChatOpenAI`...).
- **Prompts** — templates parametrizáveis.
- **Chains** — encadeamento de passos (LCEL — LangChain Expression Language).
- **Agents** — loop ReAct com tools.
- **Memory** — histórico de conversa.
- **Retrievers / Vector Stores** — base para RAG.

### 2.2 Peso e dependências

O `langchain` "completo" puxa **muitas** dependências transitivas. A partir da v0.1+ foi modularizado:

- `langchain-core` — abstrações base (leve).
- `langchain-aws` — integração Bedrock.
- `langchain-community` — integrações da comunidade (pesado, opcional).
- `langgraph` — agentes com estado.

> **Regra para Lambda:** instale só `langchain-core` + `langchain-aws`. Evite `langchain-community` inteiro.

### 2.3 Tamanho aproximado do pacote

| Instalação | Tamanho descompactado (aprox.) | Cabe em zip (250 MB)? |
|------------|-------------------------------|------------------------|
| `langchain-core` + `langchain-aws` | ~50–90 MB | ✅ Sim |
| `+ langchain` + retrievers básicos | ~150–200 MB | ⚠️ No limite |
| `+ langchain-community` + integrações | 300 MB+ | ❌ Use container |

> Valores aproximados — variam por versão e dependências. Sempre meça o seu pacote real.

### 2.4 Implementação Lambda (agente com tool + memória)

```python
import json
import boto3
from langchain_aws import ChatBedrock
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.tools import tool
from langchain.agents import AgentExecutor, create_tool_calling_agent

# --- Inicialização fora do handler (reuso em invocações quentes) ---
llm = ChatBedrock(
    model_id="anthropic.claude-3-5-sonnet-20241022-v2:0",
    region_name="us-east-1",
    model_kwargs={"max_tokens": 1024},
)

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("langchain-sessions")

@tool
def consultar_pedido(pedido_id: str) -> str:
    """Consulta o status de um pedido pelo ID."""
    return f"Pedido {pedido_id}: em transporte, chega amanhã."

tools = [consultar_pedido]

prompt = ChatPromptTemplate.from_messages([
    ("system", "Você é um assistente de atendimento conciso."),
    MessagesPlaceholder("chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),
])

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, max_iterations=5)


def carregar_historico(session_id):
    item = table.get_item(Key={"session_id": session_id}).get("Item")
    return item.get("history", []) if item else []

def salvar_historico(session_id, history):
    table.put_item(Item={"session_id": session_id, "history": history})


def handler(event, context):
    body = json.loads(event.get("body", "{}"))
    session_id = body.get("session_id", "default")
    pergunta = body.get("message", "")

    historico = carregar_historico(session_id)  # estado externo (Lambda é stateless)
    resultado = executor.invoke({"input": pergunta, "chat_history": historico})

    historico.append({"role": "user", "content": pergunta})
    historico.append({"role": "assistant", "content": resultado["output"]})
    salvar_historico(session_id, historico)

    return {"statusCode": 200, "body": json.dumps({"reply": resultado["output"]})}
```

### 2.5 Quando LangChain brilha / atrapalha

✅ **Brilha:** protótipos rápidos, agentes versáteis, muitas integrações, ecossistema (LangSmith para tracing).
❌ **Atrapalha:** quando você quer controle fino e previsibilidade — as abstrações escondem chamadas; pacote pesado prejudica cold start.

---

## 3. LlamaIndex em profundidade

### 3.1 Arquitetura (pipeline de RAG)

```
      INGESTÃO (offline/batch)                CONSULTA (online/Lambda)
┌──────────────────────────────┐      ┌────────────────────────────────┐
│ Documentos (PDF, S3, SQL...)  │      │ Pergunta do usuário            │
│         │                      │      │        │                        │
│         ▼                      │      │        ▼                        │
│   Loaders/Readers              │      │  Query Engine                   │
│         │                      │      │        │                        │
│         ▼                      │      │        ▼                        │
│   Node Parser (chunking)       │      │  Retriever ──► busca vetorial   │
│         │                      │      │        │                        │
│         ▼                      │      │        ▼                        │
│   Embeddings                   │      │  Nós relevantes ──► LLM         │
│         │                      │      │        │                        │
│         ▼                      │      │        ▼                        │
│   Vector Store (índice)  ──────┼──────┼──►  Resposta fundamentada       │
└──────────────────────────────┘      └────────────────────────────────┘
```

### 3.2 O princípio: separar ingestão da consulta

Este é o ponto **mais importante** para usar LlamaIndex no Lambda:

- **Ingestão** (ler docs, chunk, gerar embeddings, indexar) é **pesada e demorada**. NÃO faça a cada requisição.
- **Consulta** (buscar + responder) é leve e rápida. Essa sim vai no Lambda de resposta.

```
❌ ERRADO: Lambda que a cada request lê PDFs, gera embeddings e indexa (lento, caro, timeout).
✅ CERTO:  Lambda/batch de ingestão popula um vector store persistente.
           Lambda de consulta só faz query nesse índice já pronto.
```

### 3.3 Onde guardar o índice (vector store persistente)

| Store | Nota |
|-------|------|
| **Amazon OpenSearch Serverless** | Gerenciado, escala, k-NN vetorial |
| **Aurora PostgreSQL + pgvector** | SQL familiar + vetores |
| **Pinecone / Weaviate / Qdrant** | Especializados, gerenciados |
| **S3 (persistência simples)** | Barato para índices pequenos |

### 3.4 Implementação Lambda (só consulta)

```python
import json
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.vector_stores.opensearch import OpensearchVectorStore, OpensearchVectorClient
from llama_index.llms.bedrock import Bedrock
from llama_index.embeddings.bedrock import BedrockEmbedding

# --- Fora do handler: conecta ao índice JÁ EXISTENTE ---
llm = Bedrock(model="anthropic.claude-3-5-sonnet-20241022-v2:0", region_name="us-east-1")
embed = BedrockEmbedding(model_name="amazon.titan-embed-text-v2:0", region_name="us-east-1")

client = OpensearchVectorClient(
    endpoint="https://<opensearch-endpoint>",
    index="documentos",
    dim=1024,
    embedding_field="embedding",
    text_field="content",
)
vector_store = OpensearchVectorStore(client)
storage = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex.from_vector_store(vector_store, storage_context=storage)
query_engine = index.as_query_engine(llm=llm, embed_model=embed, similarity_top_k=4)


def handler(event, context):
    body = json.loads(event.get("body", "{}"))
    resposta = query_engine.query(body.get("message", ""))
    return {"statusCode": 200, "body": json.dumps({"reply": str(resposta)})}
```

### 3.5 Lambda de ingestão (separado, acionado por evento S3 ou schedule)

```python
from llama_index.core import SimpleDirectoryReader, VectorStoreIndex, StorageContext

def handler(event, context):
    """Acionado quando um novo doc chega no S3 (ou por agendamento)."""
    # Baixa docs do S3 para /tmp (Lambda tem até 10 GB de /tmp configurável)
    documentos = SimpleDirectoryReader("/tmp/docs").load_data()
    storage = StorageContext.from_defaults(vector_store=vector_store)  # mesmo store da consulta
    VectorStoreIndex.from_documents(documentos, storage_context=storage)
    return {"statusCode": 200, "body": "Indexação concluída"}
```

> Se a ingestão for **muito** pesada (milhares de docs), use ECS/Fargate batch ou AWS Batch em vez de Lambda (timeout de 15 min).

---

## 4. Haystack em profundidade

### 4.1 Filosofia: pipelines componíveis e explícitos

Haystack (2.x) modela tudo como um **pipeline** de componentes conectados. Nada de "mágica" — você vê o grafo de dados.

```
       PIPELINE DE INDEXAÇÃO                  PIPELINE DE CONSULTA (RAG)
┌────────────────────────────┐      ┌─────────────────────────────────────┐
│ Converter (PDF→texto)      │      │ Text Embedder (pergunta→vetor)        │
│      │                      │      │      │                               │
│      ▼                      │      │      ▼                               │
│ Splitter (chunking)        │      │ Retriever (busca no store)            │
│      │                      │      │      │                               │
│      ▼                      │      │      ▼                               │
│ Doc Embedder               │      │ Prompt Builder (monta contexto)       │
│      │                      │      │      │                               │
│      ▼                      │      │      ▼                               │
│ Document Store  ───────────┼──────┼──► Generator (LLM) ──► Resposta       │
└────────────────────────────┘      └─────────────────────────────────────┘
```

### 4.2 Por que times de engenharia gostam

- **Explícito e testável** — cada componente é isolado e mockável.
- **Produção-first** — pensado para deploy e escala, não só notebooks.
- **Componentes customizados** — fácil escrever o seu próprio.
- Menos "surpresas" que abstrações de alto nível.

### 4.3 Implementação Lambda (pipeline de consulta)

```python
import json
from haystack import Pipeline
from haystack.components.builders import PromptBuilder
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack_integrations.document_stores.opensearch import OpenSearchDocumentStore
from haystack_integrations.components.retrievers.opensearch import OpenSearchEmbeddingRetriever
from haystack_integrations.components.generators.amazon_bedrock import AmazonBedrockGenerator

# --- Fora do handler: monta o pipeline uma vez ---
store = OpenSearchDocumentStore(hosts="https://<opensearch-endpoint>", index="documentos")

template = """
Contexto:
{% for doc in documents %}{{ doc.content }}
{% endfor %}
Pergunta: {{ question }}
Resposta:
"""

pipe = Pipeline()
pipe.add_component("embedder", SentenceTransformersTextEmbedder())
pipe.add_component("retriever", OpenSearchEmbeddingRetriever(document_store=store, top_k=4))
pipe.add_component("prompt", PromptBuilder(template=template))
pipe.add_component("llm", AmazonBedrockGenerator(model="anthropic.claude-3-5-sonnet-20241022-v2:0"))

pipe.connect("embedder.embedding", "retriever.query_embedding")
pipe.connect("retriever.documents", "prompt.documents")
pipe.connect("prompt.prompt", "llm.prompt")


def handler(event, context):
    body = json.loads(event.get("body", "{}"))
    pergunta = body.get("message", "")
    resultado = pipe.run({"embedder": {"text": pergunta}, "prompt": {"question": pergunta}})
    resposta = resultado["llm"]["replies"][0]
    return {"statusCode": 200, "body": json.dumps({"reply": resposta})}
```

> ⚠️ **Atenção ao cold start:** `SentenceTransformersTextEmbedder` carrega um modelo local (centenas de MB). Para Lambda, prefira **embeddings via API** (ex: `AmazonBedrockTextEmbedder` com Titan) para não carregar modelo na função. Se usar modelo local, vá de container image + memória alta.

---

## 5. Bedrock Agents em profundidade

### 5.1 O modelo invertido

Nos 3 anteriores, **seu código é o orquestrador**. No Bedrock Agents, **a AWS é o orquestrador** — você só fornece:

1. **Instruções** (o "system prompt" do agente).
2. **Action Groups** — ações que o agente pode executar → apontam para **Lambdas** (ou APIs).
3. **Knowledge Bases** — RAG gerenciado sobre dados no S3.
4. **Guardrails** (opcional) — filtros de segurança.

```
┌──────────────────────────────────────────────────────┐
│                  BEDROCK AGENT (AWS)                    │
│                                                         │
│   Recebe pergunta ──► LLM raciocina (loop ReAct)        │
│         │                    │                          │
│         │        ┌───────────┼───────────┐             │
│         ▼        ▼           ▼           ▼             │
│   Knowledge   Action      Action     resposta          │
│    Base       Group 1     Group 2      final            │
│   (RAG S3)      │           │                           │
└─────────────────┼───────────┼───────────────────────────┘
                  ▼           ▼
              Lambda A    Lambda B     ◄── VOCÊ implementa só estas
           (sua tool)   (sua tool)
```

### 5.2 Vantagem central para Lambda

Você **não gerencia**: loop de agente, retries, parsing de tool calls, memória de sessão, timeout de orquestração. A AWS faz tudo. Seus Lambdas são funções pequenas e focadas — fáceis de empacotar (sem framework pesado!).

### 5.3 Lambda como Action Group

```python
def handler(event, context):
    """Chamado pelo Bedrock Agent quando decide executar uma ação."""
    action_group = event["actionGroup"]
    function = event.get("function", "")
    params = {p["name"]: p["value"] for p in event.get("parameters", [])}

    if function == "consultar_pedido":
        resultado = f"Pedido {params.get('pedido_id')}: entregue em 20/09."
    elif function == "cancelar_pedido":
        resultado = f"Pedido {params.get('pedido_id')} cancelado com sucesso."
    else:
        resultado = "Função não reconhecida."

    return {
        "response": {
            "actionGroup": action_group,
            "function": function,
            "functionResponse": {
                "responseBody": {"TEXT": {"body": resultado}}
            },
        },
        "messageVersion": "1.0",
    }
```

### 5.4 Definição do Action Group (schema de função)

Você declara as funções disponíveis (via console, CDK, ou API). Exemplo simplificado do que o agente enxerga:

```json
{
  "functions": [
    {
      "name": "consultar_pedido",
      "description": "Consulta o status de um pedido",
      "parameters": {
        "pedido_id": {"type": "string", "description": "ID do pedido", "required": true}
      }
    },
    {
      "name": "cancelar_pedido",
      "description": "Cancela um pedido",
      "parameters": {
        "pedido_id": {"type": "string", "description": "ID do pedido", "required": true}
      }
    }
  ]
}
```

### 5.5 Invocando o agente

```python
import boto3
agent_runtime = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

resp = agent_runtime.invoke_agent(
    agentId="AGENT123",
    agentAliasId="ALIAS123",
    sessionId="sessao-do-usuario",   # a AWS gerencia a memória por sessão!
    inputText="Qual o status do pedido 4567?",
)
# resposta vem em stream de eventos
for evento in resp["completion"]:
    if "chunk" in evento:
        print(evento["chunk"]["bytes"].decode())
```

### 5.6 Prós e contras

✅ **Prós:** zero gestão de orquestração, memória de sessão gerenciada, RAG integrado (KB), guardrails, tracing, tools = Lambdas leves.
❌ **Contras:** menos controle fino do loop, preso ao Bedrock, custo por invocação do agente + tokens, personalização limitada vs código próprio.

---

## 6. Comparação completa (tabelões)

### 6.1 Capacidades

| Critério | LangChain | LlamaIndex | Haystack | Bedrock Agents |
|----------|-----------|------------|----------|----------------|
| Foco principal | Propósito geral | RAG | Pipelines RAG | Agentes gerenciados |
| Agentes/tools | ✅ Forte | ✅ Médio | ⚠️ Básico | ✅ Gerenciado |
| RAG | ✅ Bom | ✅ Excelente | ✅ Excelente | ✅ Excelente (KB) |
| Multi-agente | ✅ (LangGraph) | ⚠️ | ❌ | ⚠️ (via collaboration) |
| Controle do fluxo | Médio (abstrações) | Médio | ✅ Alto (explícito) | ❌ Baixo (gerenciado) |
| Curva de aprendizado | Média | Média | Média-alta | Baixa (config) |
| Observabilidade | LangSmith | Callbacks | Tracing próprio | CloudWatch/X-Ray |
| Lock-in AWS | Não | Não | Não | ✅ Total |

### 6.2 Peso e empacotamento no Lambda

| Framework | Peso do pacote | Zip (250 MB)? | Recomendado |
|-----------|----------------|---------------|-------------|
| LangChain (core+aws) | Médio (~50–90 MB) | ✅ | Zip ou Layer |
| LangChain (full+community) | Pesado (300 MB+) | ❌ | Container |
| LlamaIndex | Pesado | ⚠️/❌ | Container |
| Haystack (embeddings via API) | Médio | ⚠️ | Layer/Container |
| Haystack (embeddings locais) | Muito pesado | ❌ | Container + memória alta |
| Bedrock Agents (a tool) | Mínimo (só boto3) | ✅ | Zip |

### 6.3 Encaixe no Lambda

| Framework | Encaixe | Observação-chave |
|-----------|---------|------------------|
| LangChain | ✅ Bom | Import só o necessário; container se pesado |
| LlamaIndex | ✅ Com disciplina | Separe ingestão (batch) da consulta (Lambda) |
| Haystack | ✅ Com atenção | Evite embedder local no Lambda; use API |
| Bedrock Agents | 🎯 Ideal | Lambda vira tool leve; AWS orquestra |

---

## 7. Onde rodar: Lambda vs EC2 vs ECS/Fargate vs App Runner

Nem sempre Lambda é a melhor casa. Comparação por característica:

| Aspecto | Lambda | EC2 | ECS/Fargate | App Runner |
|---------|--------|-----|-------------|------------|
| Modelo | Serverless, por evento | Servidor sempre ligado | Contêiner gerenciado | Contêiner web gerenciado |
| Escala | Automática (0→N) | Manual/Auto Scaling | Automática (tasks) | Automática |
| Timeout | 15 min (duro) | Ilimitado | Ilimitado | Requests longos ok |
| Estado em memória | ❌ Stateless | ✅ Persiste | ⚠️ Por task | ⚠️ Por instância |
| Cold start | Sim | Não (sempre on) | Baixo (task warm) | Baixo |
| Pacote/deps | 10 GB (container) | Ilimitado | Ilimitado (imagem) | Imagem |
| GPU | ❌ | ✅ | ✅ (alguns tipos) | ❌ |
| Custo ocioso | Zero | Paga sempre | Paga task rodando | Paga (com scale-to-zero opcional) |
| Ideal para | APIs event-driven, tools | Cargas pesadas/GPU, ingestão longa | Serviços longos, workers, agentes demorados | APIs web contínuas simples |

### 7.1 Regras de bolso

- **Lambda** → chat/API event-driven, tools do Bedrock Agents, consultas RAG rápidas, ingestão pequena.
- **ECS/Fargate** → agentes multi-etapa longos (CrewAI/AutoGen), ingestão pesada contínua, workers de fila, WebSocket persistente, quando o pacote/latência do framework não cabe bem no Lambda.
- **EC2** → precisa de GPU (modelos locais/embeddings pesados), controle total, cargas 24/7 previsíveis e altas (mais barato que serverless no volume).
- **App Runner** → API web contínua e simples baseada em container, sem querer gerenciar cluster ECS.

### 7.2 Árvore de decisão de compute

```
A tarefa dura > 15 min ou é sempre-ligada?
   ├── Sim, precisa GPU ──────────────► EC2 (GPU) ou SageMaker
   ├── Sim, sem GPU, longa/worker ────► ECS/Fargate
   └── Não (curta, por evento) ───────► Lambda

Precisa carregar modelo local (embeddings/rerank)?
   ├── Sim, grande ───────────────────► ECS/Fargate ou EC2
   └── Não (usa API Bedrock) ─────────► Lambda

Ingestão de RAG?
   ├── Poucos docs, eventual ─────────► Lambda (evento S3)
   └── Muitos docs, contínuo ─────────► ECS/Fargate batch ou AWS Batch

Só quer expor uma API web de container?
   └──────────────────────────────────► App Runner
```

---

## 8. Implementação em cada compute

### 8.1 ECS/Fargate — agente LangChain de longa duração

Quando o agente faz muitas iterações/tools e pode passar de 15 min, ou o pacote é grande demais para conforto no Lambda.

```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8080"]
```

```python
# server.py — API FastAPI com LangChain rodando em Fargate
from fastapi import FastAPI
from pydantic import BaseModel
from langchain_aws import ChatBedrock

app = FastAPI()
llm = ChatBedrock(model_id="anthropic.claude-3-5-sonnet-20241022-v2:0", region_name="us-east-1")

class Req(BaseModel):
    message: str

@app.post("/chat")
def chat(req: Req):
    # sem timeout de 15 min; estado pode viver em memória enquanto a task roda
    resposta = llm.invoke(req.message)
    return {"reply": resposta.content}
```

O serviço roda como uma **task** no cluster ECS (Fargate = sem gerenciar EC2). Escala por número de tasks. Bom para workers de SQS também.

### 8.2 EC2 — ingestão pesada de RAG com embeddings locais (GPU)

Quando você indexa milhões de documentos e roda embeddings/rerankers localmente (precisa de GPU e não tem timeout).

```python
# script rodando numa EC2 g5.xlarge (GPU), ou como job agendado
from llama_index.core import SimpleDirectoryReader, VectorStoreIndex, StorageContext
# embeddings locais em GPU (rápido para grande volume)
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

embed = HuggingFaceEmbedding(model_name="BAAI/bge-large-en-v1.5", device="cuda")
docs = SimpleDirectoryReader("/data/documentos").load_data()
storage = StorageContext.from_defaults(vector_store=meu_vector_store)
VectorStoreIndex.from_documents(docs, storage_context=storage, embed_model=embed)
# Depois, o Lambda de consulta só CONSULTA esse índice já pronto.
```

> Padrão comum: **EC2/Fargate faz a ingestão pesada** → **Lambda faz a consulta leve**. O melhor dos dois mundos.

### 8.3 Bedrock Agents — tool em Lambda (já visto na seção 5.3)

Aqui não há "onde rodar o orquestrador": a AWS roda. Você só provê Lambdas leves. É a arquitetura de menor esforço operacional.

---

## 9. Análise de custos

> ⚠️ **Importante:** os valores abaixo são **ordens de grandeza ilustrativas** para raciocínio, baseadas em preços públicos típicos da região us-east-1. Preços mudam e variam por região. **Sempre confirme na [Calculadora de Preços da AWS](https://calculator.aws) e nas páginas oficiais de preços** antes de decidir.

### 9.1 De onde vem o custo (em qualquer opção)

```
CUSTO TOTAL = Custo de COMPUTE  +  Custo do LLM (tokens)  +  Custo de STORE/RAG  +  Rede/outros
                    │                     │                        │
              Lambda/EC2/ECS        Bedrock (por token)      DynamoDB/OpenSearch/S3
```

Na maioria dos casos de IA, **o custo dos tokens do LLM domina** — o compute costuma ser uma fração pequena. Otimizar prompt/contexto/modelo geralmente economiza mais que trocar o compute.

### 9.2 Custo de compute — comparação de modelo

| Opção | Modelo de cobrança | Barato quando... | Caro quando... |
|-------|--------------------|--------------------|----------------|
| **Lambda** | Por invocação + GB-segundo de execução | Tráfego variável/baixo, picos | Tráfego alto e constante 24/7 |
| **Fargate** | Por vCPU-hora + GB-hora enquanto a task roda | Cargas médias contínuas | Muitas tasks ociosas |
| **EC2** | Por hora da instância (ou Savings Plan) | Carga alta previsível 24/7, GPU | Ocioso (paga mesmo parado) |
| **App Runner** | Por recurso provisionado/ativo | API web contínua simples | Muito ocioso sem scale-to-zero |

### 9.3 Exemplo ilustrativo — API de chat

Cenário: **API de chat**, cada request ≈ 3s de compute, 512 MB de memória.

**Caso A — 100.000 requests/mês (tráfego baixo/variável)**

| Opção | Raciocínio | Ordem de grandeza |
|-------|-----------|-------------------|
| **Lambda** | 100k × 3s × 0.5GB = 150.000 GB-s + 100k invocações | **~US$ 2–3 / mês** de compute |
| **Fargate** | 1 task 0.25 vCPU/0.5GB ligada o mês todo | **~US$ 9–12 / mês** (paga ocioso) |
| **EC2 t4g.small** | instância ligada o mês todo | **~US$ 12–15 / mês** (paga ocioso) |

➡️ Com tráfego baixo/variável, **Lambda ganha fácil** (paga só o que usa).

**Caso B — 50.000.000 requests/mês (tráfego alto constante)**

| Opção | Raciocínio | Ordem de grandeza |
|-------|-----------|-------------------|
| **Lambda** | 50M × 3s × 0.5GB = 75M GB-s + 50M invocações | **centenas a ~US$ 1.000+ / mês** de compute |
| **Fargate/EC2** | poucas instâncias saturadas 24/7 (com Savings Plan) | frequentemente **mais barato** no volume |

➡️ Com tráfego alto e constante, **EC2/Fargate reservado tende a sair mais barato** que Lambda por unidade.

> O "ponto de virada" existe: baixo/variável → Lambda; alto/constante → EC2/Fargate. Meça o seu caso.

### 9.4 Custo do LLM (Bedrock) — o que geralmente domina

Cobrado por **tokens de entrada e saída**. Exemplo ilustrativo com um modelo classe Sonnet:

```
Suponha ~US$ 3 / milhão de tokens de entrada e ~US$ 15 / milhão de saída (ilustrativo).
Um chat com 2.000 tokens de entrada + 500 de saída por request:
  entrada: 2.000 / 1.000.000 × US$3   = US$ 0,006
  saída:     500 / 1.000.000 × US$15  = US$ 0,0075
  total por request ≈ US$ 0,0135

100.000 requests ≈ US$ 1.350  ◄── MUITO maior que o compute (~US$2-3 no Lambda!)
```

➡️ **Conclusão:** brigar por centavos de compute enquanto ignora tokens é otimizar o lugar errado. Reduza contexto, use modelos menores para tarefas simples (classificação/roteamento), faça cache.

### 9.5 Custo de RAG / vector store

| Store | Modelo | Nota de custo |
|-------|--------|---------------|
| **OpenSearch Serverless** | Por OCU (unidades de compute) + storage | Tem um piso mensal relevante mesmo ocioso |
| **Aurora pgvector** | Por instância/ACU + storage | Serverless v2 escala; piso menor |
| **S3 (índice simples)** | Por GB armazenado | Muito barato para índices pequenos |
| **Bedrock Knowledge Bases** | Usa o store por baixo + embeddings | Some o custo do store escolhido |
| **Embeddings (Titan)** | Por token embedado | Custo de ingestão proporcional ao volume |

> OpenSearch Serverless costuma ter **custo mínimo mensal** não trivial — para volumes pequenos, pgvector ou S3 podem sair bem mais barato.

### 9.6 Custo do Bedrock Agents

```
Custo Bedrock Agents ≈ tokens do LLM usados no loop (entrada+saída, que costumam ser MAIORES
                        por causa das instruções + tool schemas + resultados de tool)
                      + invocações dos Lambdas (tools) — geralmente irrisório
                      + Knowledge Base (store + embeddings) se usar RAG
```

Você troca **esforço operacional** (não gerencia orquestração) por **um pouco mais de tokens** (o loop gerenciado tende a injetar mais contexto). Para muitos times, vale a troca.

### 9.7 Resumo de custos

- **Compute é quase sempre a menor parte** — não superotimize.
- **Tokens do LLM dominam** — otimize prompt, contexto e escolha de modelo.
- **RAG store pode ter piso fixo** — cuidado com OpenSearch Serverless em volumes pequenos.
- **Lambda vence em tráfego baixo/variável; EC2/Fargate em alto/constante.**
- **Bedrock Agents:** paga a conveniência com tokens extras do loop.

---

## 10. Matriz de decisão final

### 10.1 Qual framework?

```
Quero que a AWS orquestre e só escrever tools?
   └──► BEDROCK AGENTS (tools = Lambdas leves)

Meu caso é essencialmente "chat com meus documentos" (RAG)?
   ├── Quero DX focada em dados/loaders ──► LLAMAINDEX
   └── Quero pipelines explícitos/produção ─► HAYSTACK

Quero versatilidade, agentes, muitas integrações, protótipo rápido?
   └──► LANGCHAIN (+ LangGraph se precisar de estado/grafo)
```

### 10.2 Qual compute?

```
Tarefa curta, por evento, tráfego variável, sem GPU, sem modelo local?
   └──► LAMBDA

Longa (>15min), worker de fila, ou pacote grande, sem GPU?
   └──► ECS/FARGATE

Precisa de GPU (embeddings/modelos locais) ou carga alta 24/7?
   └──► EC2 (+ Savings Plan) ou SageMaker

Só uma API web de container, simples, sem cluster?
   └──► APP RUNNER

Ingestão pesada de RAG?
   └──► EC2/Fargate/Batch faz a ingestão → LAMBDA faz a consulta
```

### 10.3 Combinações recomendadas (framework × compute)

| Cenário | Framework | Compute |
|---------|-----------|---------|
| API de chat serverless simples | Bedrock Agents ou LangChain(core) | Lambda |
| "Chat com documentos" gerenciado | Bedrock KB + Agents | Lambda (tools) |
| RAG com controle, consulta rápida | LlamaIndex/Haystack | Lambda (só consulta) |
| Ingestão massiva de RAG | LlamaIndex/Haystack | Fargate/EC2/Batch |
| Agente longo multi-etapa | LangChain/LangGraph | Fargate |
| Embeddings locais em GPU | LlamaIndex/Haystack | EC2 GPU |
| API web contínua de container | qualquer | App Runner |

---

## Resumo em uma frase

> **Use Bedrock Agents quando quiser que a AWS orquestre (Lambda vira tool leve); LlamaIndex/Haystack para RAG (separando ingestão pesada em Fargate/EC2 da consulta leve no Lambda); e LangChain para versatilidade — rodando no Lambda quando a carga é curta e variável, e migrando para ECS/Fargate ou EC2 quando há longa duração, GPU ou tráfego alto e constante — lembrando que o custo dos tokens do LLM quase sempre domina o do compute.**

---

### Documentos relacionados

- `03-orquestradores-ia-lambda.md` — construir orquestradores do zero com SDK puro.
- `04-frameworks-orquestracao.md` — panorama de todos os frameworks do mercado.
- Este doc — aprofundamento dos 4 principais + compute + custos.
