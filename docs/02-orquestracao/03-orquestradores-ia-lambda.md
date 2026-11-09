# Orquestradores de IA com AWS Lambda — Guia Didático Completo

> Um guia progressivo para entender, projetar e construir orquestradores de IA usando AWS Lambda como base de execução.

---

## Índice

1. [O que é um Orquestrador de IA?](#1-o-que-é-um-orquestrador-de-ia)
2. [Por que usar Lambda?](#2-por-que-usar-lambda)
3. [Conceitos Fundamentais](#3-conceitos-fundamentais)
4. [Padrões de Orquestração](#4-padrões-de-orquestração)
5. [Arquiteturas de Referência](#5-arquiteturas-de-referência)
6. [Mão na Massa: Orquestrador Simples](#6-mão-na-massa-orquestrador-simples)
7. [Orquestração com Step Functions](#7-orquestração-com-step-functions)
8. [Padrão Agente (Tool Calling / ReAct)](#8-padrão-agente-tool-calling--react)
9. [Multi-Agente e Roteamento](#9-multi-agente-e-roteamento)
10. [Memória, Estado e Contexto](#10-memória-estado-e-contexto)
11. [Streaming de Respostas](#11-streaming-de-respostas)
12. [Observabilidade e Custos](#12-observabilidade-e-custos)
13. [Segurança e Boas Práticas](#13-segurança-e-boas-práticas)
14. [Limitações do Lambda para IA](#14-limitações-do-lambda-para-ia)
15. [Checklist de Produção](#15-checklist-de-produção)
16. [Glossário](#16-glossário)

---

## 1. O que é um Orquestrador de IA?

Um **orquestrador de IA** é o componente que coordena o fluxo entre:

- **Modelos de linguagem (LLMs)** — ex: Claude via Amazon Bedrock, OpenAI, etc.
- **Ferramentas (tools)** — APIs, bancos de dados, funções, buscas.
- **Memória/estado** — histórico de conversa, contexto acumulado.
- **Regras de negócio** — validações, decisões, roteamento.

Pense nele como um **maestro**: o LLM é um músico talentoso, mas sozinho não sabe quando tocar, com quais instrumentos combinar, ou quando parar. O orquestrador rege tudo isso.

```
Usuário
   │
   ▼
┌─────────────────────┐
│   ORQUESTRADOR      │◄──── decide o que fazer
│                     │
│  1. Recebe input    │
│  2. Consulta LLM    │──────► LLM (Bedrock/Claude)
│  3. Chama tools     │──────► APIs, DB, busca
│  4. Monta contexto  │──────► Memória (DynamoDB)
│  5. Retorna resposta│
└─────────────────────┘
   │
   ▼
Usuário
```

### Analogia simples

| Papel real | Componente de IA |
|------------|------------------|
| Recepcionista que entende o pedido | Orquestrador |
| Especialista que responde | LLM |
| Arquivo/consulta de dados | Tools + Memória |
| Gerente que decide quem faz o quê | Roteador/Agente |

---

## 2. Por que usar Lambda?

AWS Lambda é **serverless**: você roda código sem gerenciar servidores.

### Vantagens para orquestração de IA

- **Escala automática** — de 0 a milhares de execuções simultâneas.
- **Paga pelo uso** — cobra por milissegundo de execução, não por servidor ocioso.
- **Integração nativa** — API Gateway, Step Functions, EventBridge, SQS, Bedrock.
- **Event-driven** — reage a eventos (mensagem chegou, arquivo subiu, etc.).

### Quando Lambda é ideal

✅ Fluxos de IA de curta/média duração (até 15 min)
✅ Cargas variáveis ou imprevisíveis
✅ Integração com outros serviços AWS
✅ APIs de chat, webhooks, processamento assíncrono

### Quando Lambda NÃO é ideal

❌ Tarefas > 15 min (limite duro do Lambda)
❌ Modelos rodando localmente na função (falta GPU/memória)
❌ Conexões persistentes longas (WebSocket puro — use API Gateway WS)
❌ Estado em memória entre requisições (Lambda é stateless)

---

## 3. Conceitos Fundamentais

### 3.1 Stateless

Cada invocação do Lambda começa "do zero". Não guarde estado em variáveis globais esperando persistência. Use serviços externos:

- **DynamoDB** — estado de conversa, sessões.
- **S3** — arquivos grandes, documentos.
- **ElastiCache/Redis** — cache rápido de contexto.

### 3.2 Cold Start

A primeira invocação (ou após inatividade) tem latência extra ("cold start"). Para IA:

- Mantenha o pacote leve.
- Use **Provisioned Concurrency** se latência for crítica.
- Inicialize clientes (ex: Bedrock) **fora** do handler para reaproveitar em invocações quentes.

```python
import boto3

# ✅ Fora do handler — reutilizado em invocações quentes
bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

def handler(event, context):
    # ❌ NÃO crie o client aqui dentro toda vez
    ...
```

### 3.3 Timeout

Lambda tem timeout máximo de **15 minutos**. LLMs podem demorar. Para fluxos longos:

- Divida em etapas (Step Functions).
- Use processamento assíncrono (SQS + callback).

### 3.4 Idempotência

Configure retries com cuidado. Se o Lambda for reinvocado, evite chamar o LLM/tool duas vezes com efeitos colaterais. Use chaves de idempotência.

---

## 4. Padrões de Orquestração

### 4.1 Pipeline (Sequencial)

Etapas em ordem fixa. Ex: extrair → resumir → traduzir.

```
Input ──► Etapa 1 ──► Etapa 2 ──► Etapa 3 ──► Output
```

### 4.2 Roteamento (Router)

Um "classificador" decide qual caminho seguir.

```
              ┌──► Agente Suporte
Input ──► Router ──► Agente Vendas
              └──► Agente Técnico
```

### 4.3 Agente (ReAct / Tool Calling)

O LLM decide dinamicamente quais ferramentas usar em loop até resolver.

```
┌─────────────────────────────┐
│  LLM pensa → escolhe tool   │
│       ▲             │        │
│       │             ▼        │
│  resultado ◄── executa tool  │
└─────────────────────────────┘
     (repete até ter resposta)
```

### 4.4 Multi-Agente (Orquestrador-Trabalhadores)

Um agente "gerente" delega para agentes especialistas.

```
        ┌──► Agente Pesquisa ──┐
Gerente ┼──► Agente Código  ───┼──► Consolida
        └──► Agente Revisão ───┘
```

### 4.5 Fan-out / Fan-in (Paralelo)

Divide o trabalho, executa em paralelo, junta os resultados.

```
        ┌──► Lambda A ──┐
Input ──┼──► Lambda B ──┼──► Agrega ──► Output
        └──► Lambda C ──┘
```

---

## 5. Arquiteturas de Referência

### 5.1 API de Chat Simples

```
Cliente ──► API Gateway ──► Lambda (orquestrador) ──► Bedrock
                                  │
                                  └──► DynamoDB (histórico)
```

### 5.2 Orquestração Complexa com Step Functions

```
API Gateway ──► Lambda (entrada)
                     │
                     ▼
              Step Functions (state machine)
              ├── Lambda: classifica
              ├── Choice: roteia
              ├── Parallel: chama tools
              └── Lambda: consolida resposta
                     │
                     ▼
              DynamoDB / resposta
```

### 5.3 Processamento Assíncrono

```
Cliente ──► API Gateway ──► Lambda (enfileira) ──► SQS
                                                    │
                                                    ▼
                                          Lambda (worker/orquestrador)
                                                    │
                                                    ▼
                                          Bedrock + Tools + resultado
                                                    │
                                                    ▼
                                          Notifica (WebSocket/SNS/webhook)
```

---

## 6. Mão na Massa: Orquestrador Simples

Um orquestrador de chat básico em Python, invocando o Claude via Amazon Bedrock, com memória em DynamoDB.

### 6.1 Handler Lambda (Python)

```python
import json
import boto3
import os
from datetime import datetime

# Clientes inicializados fora do handler (reuso em invocações quentes)
bedrock = boto3.client("bedrock-runtime", region_name=os.environ.get("AWS_REGION", "us-east-1"))
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(os.environ["SESSIONS_TABLE"])

MODEL_ID = "anthropic.claude-3-5-sonnet-20241022-v2:0"


def carregar_historico(session_id: str) -> list:
    """Busca o histórico da conversa no DynamoDB."""
    resp = table.get_item(Key={"session_id": session_id})
    item = resp.get("Item")
    return item.get("messages", []) if item else []


def salvar_historico(session_id: str, messages: list) -> None:
    """Persiste o histórico atualizado."""
    table.put_item(Item={
        "session_id": session_id,
        "messages": messages,
        "updated_at": datetime.utcnow().isoformat(),
    })


def chamar_llm(messages: list) -> str:
    """Invoca o Claude via Bedrock com o histórico da conversa."""
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": messages,
    }
    resp = bedrock.invoke_model(modelId=MODEL_ID, body=json.dumps(body))
    payload = json.loads(resp["body"].read())
    return payload["content"][0]["text"]


def handler(event, context):
    """Orquestrador principal: recebe mensagem, mantém contexto, responde."""
    body = json.loads(event.get("body", "{}"))
    session_id = body.get("session_id", "default")
    user_message = body.get("message", "")

    # 1. Carrega o contexto anterior
    messages = carregar_historico(session_id)

    # 2. Adiciona a mensagem do usuário
    messages.append({"role": "user", "content": user_message})

    # 3. Consulta o LLM
    resposta = chamar_llm(messages)

    # 4. Adiciona a resposta ao histórico e persiste
    messages.append({"role": "assistant", "content": resposta})
    salvar_historico(session_id, messages)

    # 5. Retorna
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"reply": resposta, "session_id": session_id}),
    }
```

### 6.2 Infraestrutura (AWS SAM)

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Orquestrador de IA simples com Lambda + Bedrock + DynamoDB

Resources:
  SessionsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: ia-sessions
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: session_id
          AttributeType: S
      KeySchema:
        - AttributeName: session_id
          KeyType: HASH

  OrquestradorFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: python3.12
      Handler: app.handler
      CodeUri: ./src
      Timeout: 60
      MemorySize: 512
      Environment:
        Variables:
          SESSIONS_TABLE: !Ref SessionsTable
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref SessionsTable
        - Statement:
            - Effect: Allow
              Action:
                - bedrock:InvokeModel
              Resource: "*"
      Events:
        Api:
          Type: Api
          Properties:
            Path: /chat
            Method: post
```

### 6.3 Deploy

```bash
sam build
sam deploy --guided
```

### 6.4 Testar

```bash
curl -X POST https://<api-id>.execute-api.us-east-1.amazonaws.com/Prod/chat \
  -H "Content-Type: application/json" \
  -d '{"session_id": "abc123", "message": "Explique o que é serverless"}'
```

---

## 7. Orquestração com Step Functions

Para fluxos com múltiplas etapas, ramificações e paralelismo, o **AWS Step Functions** é o orquestrador ideal. Cada etapa pode ser um Lambda.

### 7.1 Por que Step Functions?

- **Visual** — você vê o fluxo como um diagrama.
- **Estado gerenciado** — passa dados entre etapas automaticamente.
- **Retry/erro nativo** — configura tentativas e tratamento de falhas por etapa.
- **Sem timeout de 15 min** — a máquina de estados pode rodar até 1 ano.
- **Paralelismo e escolhas** — ramificações declarativas.

### 7.2 Exemplo: fluxo classifica → roteia → responde

```json
{
  "Comment": "Orquestrador de IA com roteamento",
  "StartAt": "Classificar",
  "States": {
    "Classificar": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:classificar-intencao",
      "Next": "Rotear"
    },
    "Rotear": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.intencao",
          "StringEquals": "suporte",
          "Next": "AgenteSuporte"
        },
        {
          "Variable": "$.intencao",
          "StringEquals": "vendas",
          "Next": "AgenteVendas"
        }
      ],
      "Default": "AgenteGeral"
    },
    "AgenteSuporte": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:agente-suporte",
      "End": true
    },
    "AgenteVendas": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:agente-vendas",
      "End": true
    },
    "AgenteGeral": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:agente-geral",
      "End": true
    }
  }
}
```

### 7.3 Etapas em paralelo (fan-out)

```json
{
  "ConsultarFontes": {
    "Type": "Parallel",
    "Branches": [
      {
        "StartAt": "BuscaWeb",
        "States": {
          "BuscaWeb": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:...:function:busca-web",
            "End": true
          }
        }
      },
      {
        "StartAt": "BuscaBanco",
        "States": {
          "BuscaBanco": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:...:function:busca-banco",
            "End": true
          }
        }
      }
    ],
    "Next": "Consolidar"
  }
}
```

---

## 8. Padrão Agente (Tool Calling / ReAct)

Um **agente** deixa o próprio LLM decidir quais ferramentas usar, em loop, até resolver a tarefa. Isso é o coração dos orquestradores modernos.

### 8.1 Como funciona o loop ReAct

```
1. LLM recebe a pergunta + lista de ferramentas disponíveis
2. LLM responde: "preciso usar a tool X com esses argumentos"
3. Orquestrador executa a tool X
4. Orquestrador devolve o resultado ao LLM
5. LLM decide: usar outra tool OU dar a resposta final
6. Repete 2–5 até resposta final (com limite de iterações!)
```

### 8.2 Implementação (Python + Bedrock Tool Use)

```python
import json
import boto3
import os

bedrock = boto3.client("bedrock-runtime", region_name=os.environ.get("AWS_REGION", "us-east-1"))
MODEL_ID = "anthropic.claude-3-5-sonnet-20241022-v2:0"
MAX_ITERACOES = 5  # trava de segurança contra loops infinitos

# Definição das ferramentas disponíveis para o LLM
TOOLS = [
    {
        "name": "consultar_clima",
        "description": "Retorna o clima atual de uma cidade",
        "input_schema": {
            "type": "object",
            "properties": {
                "cidade": {"type": "string", "description": "Nome da cidade"}
            },
            "required": ["cidade"],
        },
    },
    {
        "name": "calcular",
        "description": "Executa um cálculo matemático simples",
        "input_schema": {
            "type": "object",
            "properties": {
                "expressao": {"type": "string", "description": "Ex: 2 + 2"}
            },
            "required": ["expressao"],
        },
    },
]


def executar_tool(nome: str, args: dict) -> str:
    """Implementação real de cada ferramenta."""
    if nome == "consultar_clima":
        # Aqui você chamaria uma API real
        return f"O clima em {args['cidade']} é 25°C, ensolarado."
    if nome == "calcular":
        try:
            # Em produção, use um parser seguro, NÃO eval direto
            resultado = eval(args["expressao"], {"__builtins__": {}}, {})
            return str(resultado)
        except Exception as e:
            return f"Erro no cálculo: {e}"
    return "Ferramenta desconhecida."


def agente(pergunta: str) -> str:
    """Loop de agente com tool calling."""
    messages = [{"role": "user", "content": pergunta}]

    for iteracao in range(MAX_ITERACOES):
        body = {
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 1024,
            "tools": TOOLS,
            "messages": messages,
        }
        resp = bedrock.invoke_model(modelId=MODEL_ID, body=json.dumps(body))
        payload = json.loads(resp["body"].read())

        # O LLM quer usar uma ferramenta?
        if payload.get("stop_reason") == "tool_use":
            messages.append({"role": "assistant", "content": payload["content"]})

            # Executa cada tool solicitada
            tool_results = []
            for bloco in payload["content"]:
                if bloco["type"] == "tool_use":
                    resultado = executar_tool(bloco["name"], bloco["input"])
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": bloco["id"],
                        "content": resultado,
                    })

            # Devolve os resultados ao LLM e continua o loop
            messages.append({"role": "user", "content": tool_results})
        else:
            # Resposta final — extrai o texto
            for bloco in payload["content"]:
                if bloco["type"] == "text":
                    return bloco["text"]
            return "Sem resposta textual."

    return "Limite de iterações atingido sem resposta final."


def handler(event, context):
    body = json.loads(event.get("body", "{}"))
    resposta = agente(body.get("message", ""))
    return {
        "statusCode": 200,
        "body": json.dumps({"reply": resposta}),
    }
```

> ⚠️ **Segurança:** o exemplo usa `eval` apenas para ilustrar. Em produção, use um parser matemático seguro (ex: `simpleeval`) e valide todas as entradas das tools.

### 8.3 Regras de ouro do agente

- **Sempre** tenha um limite de iterações (`MAX_ITERACOES`).
- Valide os argumentos que o LLM passa para as tools.
- Registre (log) cada decisão do agente para depurar.
- Trate erros de tool graciosamente e devolva ao LLM.

---

## 9. Multi-Agente e Roteamento

Quando um único agente fica complexo demais, divida em **especialistas**.

### 9.1 Padrão Orquestrador-Trabalhadores

```python
def orquestrador(pergunta: str) -> str:
    # 1. Um LLM "gerente" decide qual especialista chamar
    especialista = classificar(pergunta)  # ex: "codigo", "pesquisa", "revisao"

    # 2. Delega para o agente certo
    if especialista == "codigo":
        return agente_codigo(pergunta)
    elif especialista == "pesquisa":
        return agente_pesquisa(pergunta)
    else:
        return agente_geral(pergunta)
```

### 9.2 Como implementar cada agente

Cada agente pode ser:

- **Uma função Lambda separada** — isolamento, escala e deploy independentes.
- **Um estado no Step Functions** — orquestração visual.
- **Uma função Python dentro do mesmo Lambda** — simples, mas acopla tudo.

### 9.3 Trade-offs

| Abordagem | Prós | Contras |
|-----------|------|---------|
| Lambdas separados | Isolamento, escala independente | Mais latência (invocações), mais infra |
| Step Functions | Visual, robusto, retry nativo | Curva de aprendizado, custo por transição |
| Tudo num Lambda | Simples, rápido de começar | Acopla, difícil de escalar por parte |

---

## 10. Memória, Estado e Contexto

LLMs não têm memória entre chamadas. O orquestrador precisa gerenciá-la.

### 10.1 Tipos de memória

- **Curto prazo** — histórico da conversa atual (últimas N mensagens).
- **Longo prazo** — fatos sobre o usuário, preferências (persistidos).
- **Semântica (RAG)** — busca em documentos por similaridade.

### 10.2 Onde guardar

| Serviço | Uso |
|---------|-----|
| DynamoDB | Histórico de sessão, estado estruturado |
| S3 | Documentos, transcrições longas |
| ElastiCache/Redis | Cache de contexto quente, baixa latência |
| OpenSearch / Aurora pgvector / Bedrock KB | Busca vetorial (RAG) |

### 10.3 Gerenciando janela de contexto

LLMs têm limite de tokens. Estratégias:

- **Truncar** — mantenha só as últimas N mensagens.
- **Resumir** — periodicamente peça ao LLM para resumir o histórico antigo.
- **RAG** — busque só os trechos relevantes em vez de mandar tudo.

```python
def preparar_contexto(historico: list, max_mensagens: int = 20) -> list:
    """Mantém só as últimas mensagens para caber na janela de contexto."""
    if len(historico) <= max_mensagens:
        return historico
    # Mantém as mais recentes (poderia resumir as antigas antes)
    return historico[-max_mensagens:]
```

### 10.4 RAG (Retrieval-Augmented Generation) resumido

```
Pergunta ──► Gera embedding ──► Busca vetorial ──► Trechos relevantes
                                                          │
                                                          ▼
                          LLM recebe: pergunta + trechos ──► Resposta fundamentada
```

Na AWS, use **Amazon Bedrock Knowledge Bases** para RAG gerenciado (indexa documentos no S3 e resolve a busca vetorial pra você).

---

## 11. Streaming de Respostas

Para chat, é melhor enviar a resposta token a token (como digitação) em vez de esperar tudo.

### 11.1 Opções no Lambda

- **Lambda Response Streaming** — para Function URLs, streaming nativo.
- **API Gateway WebSocket** — envia chunks conforme o LLM gera.
- **Bedrock `invoke_model_with_response_stream`** — o modelo emite tokens em stream.

### 11.2 Exemplo de consumo do stream do Bedrock

```python
def stream_llm(messages: list):
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": messages,
    }
    resp = bedrock.invoke_model_with_response_stream(
        modelId=MODEL_ID, body=json.dumps(body)
    )
    for evento in resp["body"]:
        chunk = json.loads(evento["chunk"]["bytes"])
        if chunk["type"] == "content_block_delta":
            yield chunk["delta"]["text"]  # emite cada pedaço
```

---

## 12. Observabilidade e Custos

### 12.1 O que monitorar

- **Latência** — tempo total e por etapa (LLM, tools, DB).
- **Tokens consumidos** — entrada e saída (impacta custo direto).
- **Taxa de erro** — falhas de LLM, tools, timeouts.
- **Iterações do agente** — loops longos indicam problema.

### 12.2 Ferramentas AWS

- **CloudWatch Logs** — logs estruturados de cada invocação.
- **CloudWatch Metrics** — métricas customizadas (tokens, latência).
- **X-Ray** — tracing distribuído entre Lambdas/Step Functions.
- **Bedrock Model Invocation Logging** — registra prompts e respostas.

### 12.3 Log estruturado (recomendado)

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def log_evento(tipo: str, **dados):
    logger.info(json.dumps({"evento": tipo, **dados}))

# Uso
log_evento("llm_call", modelo=MODEL_ID, tokens_entrada=350, latencia_ms=1200)
log_evento("tool_call", tool="consultar_clima", cidade="São Paulo")
```

### 12.4 Controle de custos

- **Cache** respostas repetidas (mesma pergunta → mesma resposta).
- **Escolha o modelo certo** — modelos menores para tarefas simples (roteamento, classificação).
- **Limite `max_tokens`** — evite respostas maiores que o necessário.
- **Trunque o contexto** — menos tokens de entrada = menos custo.
- **Defina orçamentos** no AWS Budgets com alertas.

---

## 13. Segurança e Boas Práticas

### 13.1 IAM — menor privilégio

Dê ao Lambda **apenas** as permissões necessárias:

```json
{
  "Effect": "Allow",
  "Action": ["bedrock:InvokeModel"],
  "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-*"
}
```

### 13.2 Segredos

- **Nunca** hardcode chaves/tokens no código.
- Use **AWS Secrets Manager** ou **Parameter Store** para credenciais de tools externas.

### 13.3 Validação de entrada

- Valide e sanitize todo input do usuário antes de mandar ao LLM.
- Cuidado com **prompt injection**: trate o conteúdo do usuário como dado não confiável, nunca como instrução ao sistema.

### 13.4 Prompt Injection

```
❌ Perigoso: concatenar input do usuário direto no system prompt
✅ Seguro: usar mensagens de papel (role: user) separadas do system prompt,
           validar tool arguments, e limitar o que as tools podem fazer.
```

### 13.5 Guardrails

Use **Amazon Bedrock Guardrails** para filtrar conteúdo tóxico, PII e tópicos proibidos automaticamente.

### 13.6 Timeouts e limites

- Configure timeout do Lambda com folga sobre a latência esperada do LLM.
- Sempre limite iterações de agente.
- Use retry com backoff exponencial para throttling do Bedrock.

---

## 14. Limitações do Lambda para IA

| Limitação | Impacto | Contorno |
|-----------|---------|----------|
| Timeout 15 min | Fluxos longos falham | Step Functions, processamento assíncrono |
| Sem GPU | Não roda modelos localmente | Use Bedrock/SageMaker endpoints |
| Stateless | Perde estado entre chamadas | DynamoDB/Redis para estado |
| Cold start | Latência inicial | Provisioned Concurrency, init fora do handler |
| Payload 6 MB (sync) | Respostas grandes cortam | S3 para payloads grandes, streaming |
| Memória até 10 GB | Contexto muito grande | RAG, truncamento |

---

## 15. Checklist de Produção

- [ ] Clientes (Bedrock, DynamoDB) inicializados **fora** do handler
- [ ] Estado persistido em DynamoDB/Redis (nada em variável global)
- [ ] Limite de iterações no agente
- [ ] Timeout do Lambda ajustado à latência do LLM
- [ ] Retry com backoff para throttling do Bedrock
- [ ] Truncamento/resumo de contexto para controlar tokens
- [ ] Logs estruturados (tokens, latência, decisões)
- [ ] Tracing com X-Ray habilitado
- [ ] IAM com menor privilégio
- [ ] Segredos no Secrets Manager
- [ ] Validação de input + guardrails contra prompt injection
- [ ] Alertas de custo no AWS Budgets
- [ ] Tratamento gracioso de erros de tool e LLM
- [ ] Idempotência para invocações repetidas
- [ ] Cache de respostas repetidas (quando aplicável)

---

## 16. Glossário

| Termo | Significado |
|-------|-------------|
| **LLM** | Large Language Model — modelo de linguagem (ex: Claude) |
| **Bedrock** | Serviço AWS gerenciado para acessar LLMs |
| **Orquestrador** | Componente que coordena LLM + tools + estado |
| **Agente** | Orquestrador onde o LLM decide as ações dinamicamente |
| **Tool Calling** | LLM solicita a execução de funções/APIs |
| **ReAct** | Reasoning + Acting — loop de pensar e agir |
| **RAG** | Retrieval-Augmented Generation — resposta com busca em documentos |
| **Cold Start** | Latência da primeira invocação do Lambda |
| **Step Functions** | Serviço AWS de orquestração de fluxos (state machine) |
| **Fan-out/Fan-in** | Paralelizar e depois consolidar |
| **Embedding** | Representação vetorial de texto para busca semântica |
| **Guardrails** | Filtros de segurança sobre entrada/saída do LLM |
| **Prompt Injection** | Ataque onde input malicioso manipula o LLM |
| **Provisioned Concurrency** | Instâncias Lambda pré-aquecidas (sem cold start) |

---

## Resumo em uma frase

> **Um orquestrador de IA no Lambda coordena o LLM (Bedrock), ferramentas e memória (DynamoDB) de forma serverless, escalável e paga-por-uso — usando Step Functions para fluxos complexos e o padrão de agente para deixar o próprio LLM decidir as ações.**

---

### Próximos passos sugeridos

1. Comece pelo [orquestrador simples](#6-mão-na-massa-orquestrador-simples).
2. Evolua para o [padrão agente](#8-padrão-agente-tool-calling--react).
3. Quando o fluxo crescer, adote [Step Functions](#7-orquestração-com-step-functions).
4. Adicione [memória/RAG](#10-memória-estado-e-contexto) e [observabilidade](#12-observabilidade-e-custos).
5. Rode o [checklist de produção](#15-checklist-de-produção) antes de ir ao ar.
