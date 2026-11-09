# Frameworks de Orquestração de IA do Mercado — Guia Didático (e uso no AWS Lambda)

> Comparativo prático dos principais frameworks para orquestrar LLMs e agentes, com foco em **como cada um roda (ou não) no AWS Lambda**.

---

## Índice

1. [Por que usar um framework?](#1-por-que-usar-um-framework)
2. [Visão geral do mercado](#2-visão-geral-do-mercado)
3. [Como avaliar um framework para Lambda](#3-como-avaliar-um-framework-para-lambda)
4. [LangChain](#4-langchain)
5. [LangGraph](#5-langgraph)
6. [LlamaIndex](#6-llamaindex)
7. [CrewAI](#7-crewai)
8. [AutoGen (Microsoft)](#8-autogen-microsoft)
9. [Semantic Kernel (Microsoft)](#9-semantic-kernel-microsoft)
10. [Haystack (deepset)](#10-haystack-deepset)
11. [Amazon Bedrock Agents](#11-amazon-bedrock-agents)
12. [Strands Agents (AWS)](#12-strands-agents-aws)
13. [Pydantic AI](#13-pydantic-ai)
14. [Frameworks JavaScript/TypeScript](#14-frameworks-javascripttypescript)
15. [Tabela comparativa](#15-tabela-comparativa)
16. [Padrões de empacotamento no Lambda](#16-padrões-de-empacotamento-no-lambda)
17. [Como decidir](#17-como-decidir)
18. [Glossário](#18-glossário)

---

## 1. Por que usar um framework?

Você **pode** orquestrar IA só com o SDK do Bedrock/OpenAI (como no guia anterior). Mas frameworks entregam de graça:

- **Abstração de tools/agentes** — loop ReAct, tool calling, retries prontos.
- **Memória e estado** — histórico de conversa, RAG, checkpoints.
- **Conectores** — dezenas de integrações (bancos vetoriais, APIs, loaders).
- **Multi-agente** — coordenação entre especialistas.
- **Observabilidade** — tracing integrado (LangSmith, etc.).

O custo: **dependências pesadas**, curva de aprendizado e, às vezes, "mágica" difícil de depurar.

> Regra prática: para fluxos simples, SDK puro. Para agentes complexos, multi-agente ou RAG avançado, um framework economiza semanas.

---

## 2. Visão geral do mercado

```
                    FRAMEWORKS DE ORQUESTRAÇÃO DE IA

  Propósito geral        Multi-agente         RAG-first        AWS-nativo
  ┌──────────────┐      ┌────────────┐      ┌───────────┐    ┌──────────────┐
  │ LangChain    │      │ CrewAI     │      │ LlamaIndex│    │ Bedrock      │
  │ LangGraph    │      │ AutoGen    │      │ Haystack  │    │  Agents      │
  │ Semantic     │      │ (LangGraph)│      │           │    │ Strands      │
  │  Kernel      │      │            │      │           │    │  Agents      │
  │ Pydantic AI  │      │            │      │           │    │              │
  └──────────────┘      └────────────┘      └───────────┘    └──────────────┘
       │                     │                    │                 │
       └─────────────────────┴────────────────────┴─────────────────┘
                                    │
                          Todos podem rodar no Lambda
                       (com maior ou menor esforço)
```

---

## 3. Como avaliar um framework para Lambda

O Lambda tem restrições que impactam a escolha:

| Restrição do Lambda | Impacto no framework |
|---------------------|----------------------|
| **Pacote de deploy** (250 MB descompactado / 50 MB zip; 10 GB via container) | Frameworks pesados exigem **container image** |
| **Stateless** | Memória/checkpoints precisam de store externo (DynamoDB, S3) |
| **Timeout 15 min** | Loops de agente longos precisam de Step Functions |
| **Cold start** | Import pesado (ex: LangChain completo) aumenta latência |
| **Sem estado entre invocações** | Não confie em cache em memória do framework |

### Estratégia de tamanho

- **Framework leve** → zip normal ou Lambda Layer.
- **Framework pesado** (LangChain + integrações, LlamaIndex) → **container image** (até 10 GB).
- Importe **só o necessário** (ex: `from langchain_aws import ChatBedrock`, não o pacote inteiro).

---

## 4. LangChain

O framework mais popular e maduro para aplicações com LLM.

### O que é

Biblioteca de "blocos de montar": modelos, prompts, chains, tools, memória, retrievers. Rica em integrações.

### Pontos fortes

- Ecossistema gigante (centenas de integrações).
- Abstrações prontas para RAG, agentes, chains.
- Integração nativa com Bedrock via `langchain-aws`.
- Observabilidade com LangSmith.

### Pontos fracos

- **Pesado** — muitas dependências transitivas.
- Abstrações podem esconder o que acontece (difícil depurar).
- API mudou bastante entre versões.

### Uso no Lambda

✅ **Funciona bem**, mas:
- Use **container image** se instalar muitas integrações.
- Instale só o núcleo + o provedor: `langchain-core`, `langchain-aws`.
- Inicialize o modelo fora do handler.

### Exemplo (Bedrock + LangChain no Lambda)

```python
import json
from langchain_aws import ChatBedrock
from langchain_core.messages import HumanMessage, SystemMessage

# Fora do handler (reuso em invocações quentes)
llm = ChatBedrock(
    model_id="anthropic.claude-3-5-sonnet-20241022-v2:0",
    region_name="us-east-1",
    model_kwargs={"max_tokens": 1024},
)

def handler(event, context):
    body = json.loads(event.get("body", "{}"))
    mensagens = [
        SystemMessage(content="Você é um assistente conciso."),
        HumanMessage(content=body.get("message", "")),
    ]
    resposta = llm.invoke(mensagens)
    return {
        "statusCode": 200,
        "body": json.dumps({"reply": resposta.content}),
    }
```

### Empacotamento recomendado

```dockerfile
# Dockerfile para Lambda container image
FROM public.ecr.aws/lambda/python:3.12

COPY requirements.txt .
RUN pip install -r requirements.txt --target "${LAMBDA_TASK_ROOT}"

COPY app.py ${LAMBDA_TASK_ROOT}
CMD ["app.handler"]
```

```txt
# requirements.txt (mínimo)
langchain-core
langchain-aws
boto3
```

---

## 5. LangGraph

Extensão do LangChain para **agentes com estado** modelados como **grafos**.

### O que é

Você define nós (etapas) e arestas (transições), com estado compartilhado. Ideal para agentes que fazem loops, ramificam e precisam de checkpoints.

### Pontos fortes

- Controle explícito do fluxo (nada de "mágica").
- **Checkpoints** — persiste o estado do grafo (bom para retomar).
- Suporta ciclos, condicionais, humano-no-loop.
- Base para multi-agente robusto.

### Pontos fracos

- Curva de aprendizado do modelo de grafo.
- Ainda carrega dependências do ecossistema LangChain.

### Uso no Lambda

✅ **Bom encaixe**, especialmente porque os **checkpointers** podem usar stores externos.
- Use um checkpointer compatível com Lambda (ex: persistir em DynamoDB/Postgres) — **não** o `MemorySaver` em memória, que se perde entre invocações.
- Para fluxos longos, combine com Step Functions (um nó grande por Lambda).

### Exemplo conceitual

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class Estado(TypedDict):
    pergunta: str
    resposta: str

def no_pensar(estado: Estado) -> Estado:
    # chama o LLM, decide...
    estado["resposta"] = "..."
    return estado

def decidir(estado: Estado) -> str:
    return "fim" if estado["resposta"] else "pensar"

grafo = StateGraph(Estado)
grafo.add_node("pensar", no_pensar)
grafo.set_entry_point("pensar")
grafo.add_conditional_edges("pensar", decidir, {"fim": END, "pensar": "pensar"})
app = grafo.compile()  # em produção: compile(checkpointer=<store externo>)
```

> No Lambda: cada invocação recria o grafo (barato), mas o **estado** deve vir de um checkpointer persistente.

---

## 6. LlamaIndex

Framework **RAG-first**: focado em indexar dados e responder com base neles.

### O que é

Especialista em ingestão de documentos, indexação (vetorial, árvore, grafo) e consultas fundamentadas.

### Pontos fortes

- Melhor experiência para **RAG** e "chat com seus dados".
- Muitos loaders (PDF, Notion, SQL, S3, etc.).
- Também suporta agentes e tools.

### Pontos fracos

- Pesado com muitos conectores.
- Menos focado em agentes complexos que LangGraph/AutoGen.

### Uso no Lambda

✅ Funciona, com atenção:
- A **indexação** (ingestão pesada) é melhor fazer **offline** ou em Lambda separado/batch, não a cada requisição.
- Guarde o índice em store persistente (S3, OpenSearch, Pinecone) e apenas **consulte** no Lambda de resposta.
- Use container image pela quantidade de dependências.

### Padrão recomendado

```
Ingestão (offline/batch) ──► índice vetorial (OpenSearch/S3)
                                        │
Consulta (Lambda) ──► carrega índice ──► query ──► resposta RAG
```

---

## 7. CrewAI

Framework focado em **equipes de agentes** ("crews") com papéis definidos.

### O que é

Você define agentes com papel/objetivo/ferramentas e tarefas; a "crew" coordena a colaboração (sequencial ou hierárquica).

### Pontos fortes

- Abstração intuitiva (papéis, tarefas, crew).
- Rápido para prototipar multi-agente.
- Boa para automações de negócio (pesquisa, redação, análise).

### Pontos fracos

- Menos controle fino que LangGraph.
- Dependências consideráveis.

### Uso no Lambda

⚠️ **Possível, com cuidado**:
- Fluxos multi-agente podem ser **longos** → risco de timeout de 15 min. Prefira tarefas curtas ou divida em Step Functions.
- Use container image.
- Persistir estado entre etapas em store externo.

### Exemplo conceitual

```python
from crewai import Agent, Task, Crew

pesquisador = Agent(role="Pesquisador", goal="Coletar dados", backstory="...")
redator = Agent(role="Redator", goal="Escrever resumo", backstory="...")

t1 = Task(description="Pesquisar tema X", agent=pesquisador)
t2 = Task(description="Resumir achados", agent=redator)

crew = Crew(agents=[pesquisador, redator], tasks=[t1, t2])
resultado = crew.kickoff()
```

---

## 8. AutoGen (Microsoft)

Framework de **conversação entre múltiplos agentes**.

### O que é

Agentes conversam entre si (e com humanos) para resolver tarefas. Forte em code execution e colaboração dinâmica.

### Pontos fortes

- Multi-agente conversacional poderoso.
- Padrões de "group chat" entre agentes.
- Bom para tarefas de código e raciocínio colaborativo.

### Pontos fracos

- Pode gerar conversas longas (caras e demoradas).
- Alguns recursos assumem execução de código local (arriscado/complexo no Lambda).

### Uso no Lambda

⚠️ **Parcial**:
- Conversas multi-agente podem estourar 15 min → Step Functions ou async.
- Execução de código dos agentes exige sandbox — **não** rode código arbitrário no Lambda de produção sem isolamento.
- Container image.

---

## 9. Semantic Kernel (Microsoft)

SDK para integrar LLMs em apps, com **plugins** e planejamento.

### O que é

Orientado a engenharia de software: "kernels", plugins (funções nativas + prompts), planners. Forte em .NET, também Python/Java.

### Pontos fortes

- Ótimo para integrar IA em software existente.
- Multi-linguagem (C#, Python, Java).
- Modelo de plugins limpo.

### Pontos fracos

- Menos "baterias inclusas" para RAG que LlamaIndex.
- Comunidade menor que LangChain em Python.

### Uso no Lambda

✅ Funciona (Python ou .NET no Lambda). Relativamente leve. Bom para quem já vive no ecossistema Microsoft/.NET.

---

## 10. Haystack (deepset)

Framework de **pipelines** para busca e RAG em produção.

### O que é

Constrói pipelines componíveis (retriever → reader → generator). Pensado para produção e escala.

### Pontos fortes

- Pipelines explícitos e testáveis.
- Forte em busca/RAG de qualidade.
- Bom para times de engenharia.

### Pontos fracos

- Menos foco em agentes autônomos.
- Curva para casos além de RAG.

### Uso no Lambda

✅ Funciona; separe ingestão (offline) da consulta (Lambda), como no LlamaIndex.

---

## 11. Amazon Bedrock Agents

Orquestração de agentes **gerenciada pela AWS** — sem servidor para gerenciar.

### O que é

Você configura um agente no Bedrock: instruções, grupos de ação (que apontam para **Lambdas** ou APIs via OpenAPI schema), e Knowledge Bases (RAG gerenciado). A AWS roda o loop de orquestração.

### Pontos fortes

- **Totalmente gerenciado** — a AWS faz o loop ReAct por você.
- Integração nativa: as **tools são Lambdas**.
- RAG gerenciado via Knowledge Bases.
- Guardrails, memória e tracing integrados.
- Sem se preocupar com timeout do orquestrador (a AWS gerencia).

### Pontos fracos

- Menos flexível que código próprio.
- Preso ao ecossistema AWS/Bedrock.

### Relação com Lambda

🎯 **Modelo invertido e ideal**: aqui o Lambda **não é o orquestrador** — ele é a **ferramenta**. O Bedrock Agents orquestra e chama seus Lambdas para executar ações.

```
Bedrock Agent (orquestra o loop)
      │
      ├──► Action Group ──► Lambda: consultar_pedido()
      ├──► Action Group ──► Lambda: criar_ticket()
      └──► Knowledge Base ──► RAG sobre docs no S3
```

### Exemplo de Lambda como ação do agente

```python
def handler(event, context):
    """Lambda chamado pelo Bedrock Agent como uma ação."""
    # O Bedrock envia qual função/API foi invocada e os parâmetros
    action_group = event["actionGroup"]
    function = event.get("function", "")
    params = {p["name"]: p["value"] for p in event.get("parameters", [])}

    if function == "consultar_pedido":
        resultado = f"Pedido {params.get('id')}: enviado."
    else:
        resultado = "Ação desconhecida."

    # Formato de resposta esperado pelo Bedrock Agent
    return {
        "response": {
            "actionGroup": action_group,
            "function": function,
            "functionResponse": {
                "responseBody": {"TEXT": {"body": resultado}}
            },
        }
    }
```

---

## 12. Strands Agents (AWS)

SDK open-source da AWS para construir agentes com poucas linhas, "model-driven".

### O que é

Framework leve onde você declara o modelo, as tools (funções Python decoradas) e o loop de agente cuida do resto. Feito para rodar bem em ambientes AWS.

### Pontos fortes

- **Leve** — ótimo para Lambda.
- Simples: decore funções como tools.
- Integra bem com Bedrock e infra AWS.
- Open-source e flexível.

### Pontos fracos

- Ecossistema mais novo/menor que LangChain.

### Uso no Lambda

✅ **Excelente encaixe** — projetado para ambientes serverless/AWS. Pacote enxuto reduz cold start.

### Exemplo conceitual

```python
from strands import Agent, tool

@tool
def consultar_clima(cidade: str) -> str:
    """Retorna o clima de uma cidade."""
    return f"{cidade}: 25°C, ensolarado."

agente = Agent(tools=[consultar_clima])

def handler(event, context):
    body = event.get("body", "{}")
    resposta = agente("Qual o clima em São Paulo?")
    return {"statusCode": 200, "body": str(resposta)}
```

---

## 13. Pydantic AI

Framework de agentes com **tipagem forte** (do time do Pydantic).

### O que é

Agentes type-safe: saídas validadas por modelos Pydantic, tools tipadas, DX limpa. Foco em confiabilidade e produção.

### Pontos fortes

- **Saídas estruturadas e validadas** (menos erro de parsing).
- Leve e "pythônico".
- Ótimo quando você precisa de JSON confiável do LLM.

### Pontos fracos

- Mais novo; menos integrações prontas.

### Uso no Lambda

✅ **Ótimo** — leve, tipado, fácil de empacotar. Bom para APIs que exigem respostas estruturadas.

### Exemplo conceitual

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class Resposta(BaseModel):
    resumo: str
    confianca: float

agente = Agent("bedrock:anthropic.claude-3-5-sonnet", result_type=Resposta)

def handler(event, context):
    r = agente.run_sync("Resuma o texto X")
    return {"statusCode": 200, "body": r.data.model_dump_json()}
```

---

## 14. Frameworks JavaScript/TypeScript

Se seu Lambda é Node.js:

| Framework | Descrição | Lambda |
|-----------|-----------|--------|
| **LangChain.js** | Porta do LangChain para JS/TS | ✅ Bom; pacotes menores que Python |
| **LlamaIndex.TS** | RAG em TypeScript | ✅ Funciona |
| **Vercel AI SDK** | SDK enxuto para streaming e tool calling | ✅ Excelente para Lambda (leve, streaming) |
| **Mastra** | Framework TS de agentes/workflows | ✅ Funciona |

O ecossistema Node costuma gerar **pacotes menores** que Python — vantagem para cold start.

### Exemplo (Vercel AI SDK + Bedrock)

```typescript
import { bedrock } from "@ai-sdk/amazon-bedrock";
import { generateText } from "ai";

export const handler = async (event: any) => {
  const { message } = JSON.parse(event.body ?? "{}");
  const { text } = await generateText({
    model: bedrock("anthropic.claude-3-5-sonnet-20241022-v2:0"),
    prompt: message,
  });
  return { statusCode: 200, body: JSON.stringify({ reply: text }) };
};
```

---

## 15. Tabela comparativa

| Framework | Foco | Peso | Multi-agente | RAG | Encaixe no Lambda |
|-----------|------|------|--------------|-----|-------------------|
| **LangChain** | Propósito geral | Pesado | Médio | Bom | ✅ (container) |
| **LangGraph** | Agentes com estado (grafo) | Médio-pesado | Forte | Bom | ✅ (checkpointer externo) |
| **LlamaIndex** | RAG | Pesado | Médio | Excelente | ✅ (separar ingestão) |
| **CrewAI** | Equipes de agentes | Médio | Forte | Médio | ⚠️ (timeout/Step Functions) |
| **AutoGen** | Conversa multi-agente | Médio | Forte | Médio | ⚠️ (timeout/sandbox) |
| **Semantic Kernel** | Integração em apps | Leve-médio | Médio | Médio | ✅ |
| **Haystack** | Pipelines RAG | Médio | Fraco | Excelente | ✅ (separar ingestão) |
| **Bedrock Agents** | Gerenciado AWS | N/A (managed) | Forte | Excelente (KB) | 🎯 Lambda vira a tool |
| **Strands Agents** | Agentes leves AWS | Leve | Médio | Bom | ✅ Excelente |
| **Pydantic AI** | Agentes tipados | Leve | Médio | Médio | ✅ Excelente |
| **Vercel AI SDK** | JS/TS enxuto | Leve | Médio | Médio | ✅ Excelente |

Legenda: ✅ bom encaixe · ⚠️ possível com cuidado · 🎯 modelo ideal (Lambda como ferramenta)

---

## 16. Padrões de empacotamento no Lambda

### 16.1 Zip (frameworks leves)

Strands, Pydantic AI, Semantic Kernel, Vercel AI SDK.

```bash
pip install -r requirements.txt -t package/
cp app.py package/
cd package && zip -r ../deploy.zip . && cd ..
```

### 16.2 Lambda Layer (dependências compartilhadas)

Coloque o framework numa Layer e mantenha o código da função pequeno. Bom quando várias funções compartilham as mesmas libs.

### 16.3 Container Image (frameworks pesados)

LangChain completo, LlamaIndex, CrewAI, AutoGen. Até 10 GB.

```dockerfile
FROM public.ecr.aws/lambda/python:3.12
COPY requirements.txt .
RUN pip install -r requirements.txt --target "${LAMBDA_TASK_ROOT}"
COPY app.py ${LAMBDA_TASK_ROOT}
CMD ["app.handler"]
```

```bash
docker build -t meu-agente .
# push para ECR e aponte a Lambda para a imagem
```

### 16.4 Dicas para reduzir cold start

- Importe só o submódulo necessário.
- Inicialize modelos/clients fora do handler.
- Use Provisioned Concurrency se latência for crítica.
- Prefira frameworks leves quando possível.

---

## 17. Como decidir

```
Precisa de RAG "chat com seus dados"?
   ├── Sim, gerenciado ──────────► Bedrock Knowledge Bases (+ Agents)
   └── Sim, controle no código ──► LlamaIndex ou Haystack

Precisa de agente autônomo com tools?
   ├── Quer gerenciado (AWS faz o loop) ─► Bedrock Agents (Lambda = tool)
   ├── Quer leve/serverless ─────────────► Strands Agents ou Pydantic AI
   └── Quer controle fino do fluxo ──────► LangGraph

Precisa de múltiplos agentes colaborando?
   ├── Prototipagem rápida ──────────────► CrewAI
   ├── Conversa entre agentes ───────────► AutoGen
   └── Robusto com estado/checkpoints ───► LangGraph

Já usa .NET / ecossistema Microsoft?
   └────────────────────────────────────► Semantic Kernel

Lambda em Node.js?
   └────────────────────────────────────► Vercel AI SDK ou LangChain.js

Fluxo simples (1-2 chamadas ao LLM)?
   └────────────────────────► SDK puro do Bedrock (sem framework)
```

### Recomendações rápidas por cenário

- **API de chat serverless enxuta:** Strands Agents, Pydantic AI ou Vercel AI SDK.
- **Agentes gerenciados sem se preocupar com orquestração:** Bedrock Agents (seus Lambdas viram as ações).
- **RAG de produção:** Bedrock Knowledge Bases, LlamaIndex ou Haystack.
- **Fluxo de agente complexo com ramificações e retomada:** LangGraph + checkpointer externo (+ Step Functions se longo).
- **Multi-agente rápido de montar:** CrewAI (atento ao timeout).

---

## 18. Glossário

| Termo | Significado |
|-------|-------------|
| **Framework** | Biblioteca que abstrai a orquestração de LLM/agentes |
| **Chain** | Sequência de passos (LangChain) |
| **Grafo de estado** | Fluxo com nós/arestas e estado (LangGraph) |
| **Crew** | Equipe de agentes com papéis (CrewAI) |
| **Plugin** | Função/prompt reutilizável (Semantic Kernel) |
| **Pipeline** | Sequência componível de etapas (Haystack) |
| **Action Group** | Conjunto de ações (Lambdas/APIs) de um Bedrock Agent |
| **Knowledge Base** | RAG gerenciado do Bedrock sobre dados no S3 |
| **Checkpointer** | Persistência do estado de um grafo/agente |
| **Container Image** | Imagem Docker usada como pacote do Lambda (até 10 GB) |
| **Lambda Layer** | Pacote de dependências compartilhado entre funções |
| **Cold Start** | Latência da primeira invocação do Lambda |
| **Provisioned Concurrency** | Instâncias Lambda pré-aquecidas |
| **Tool / Ferramenta** | Função/API que o agente pode chamar |

---

## Resumo em uma frase

> **No Lambda, prefira frameworks leves (Strands, Pydantic AI, Vercel AI SDK) para orquestração direta; use LangGraph com checkpointer externo para agentes complexos; e considere Bedrock Agents quando quiser que a AWS orquestre e seus Lambdas sejam apenas as ferramentas — empacotando frameworks pesados como container image.**

---

### Relação com o guia anterior

Este documento complementa `03-orquestradores-ia-lambda.md`:
- Lá você aprendeu a **construir do zero** com SDK puro.
- Aqui você vê **quando um framework vale a pena** e qual escolher para o Lambda.
