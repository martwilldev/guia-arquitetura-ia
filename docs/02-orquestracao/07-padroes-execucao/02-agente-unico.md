# Agente Único (Single-Agent)

> Documento **focado em um só conceito**: o **agente único**. Um LLM, um loop, um conjunto de
> tools e um objetivo. Aqui você entende o que ele é, quando usar (e quando **não** usar), o
> impacto prático em custo/latência/throughput em nível alto, e vê exemplos completos prontos
> para **AWS Bedrock + Lambda** — com SDK puro e com framework.
>
> Multi-agente, sub-agentes e múltiplos orquestradores **não** são aprofundados aqui: são temas
> dos próximos documentos da série (só os citamos remetendo aos arquivos certos).

---

## Índice

1. [O que é um agente único](#1-o-que-é-um-agente-único)
2. [Quando usar (e quando NÃO usar)](#2-quando-usar-e-quando-não-usar)
3. [Impacto prático: custo, latência e throughput (nível alto)](#3-impacto-prático-custo-latência-e-throughput-nível-alto)
4. [Exemplo completo: agente único em Lambda com Bedrock (SDK puro)](#4-exemplo-completo-agente-único-em-lambda-com-bedrock-sdk-puro)
5. [Exemplo equivalente com framework (Strands / LangChain)](#5-exemplo-equivalente-com-framework-strands--langchain)
6. [Ferramentas e opções de mercado](#6-ferramentas-e-opções-de-mercado)
7. [Como escalar um agente único antes de partir para multi-agente](#7-como-escalar-um-agente-único-antes-de-partir-para-multi-agente)
8. [Boas práticas e armadilhas](#8-boas-práticas-e-armadilhas)
9. [Glossário curto](#9-glossário-curto)
10. [Resumo em uma frase](#10-resumo-em-uma-frase)
11. [Anterior / Próximo na série](#11-anterior--próximo-na-série)

---

## 1. O que é um agente único

Um **agente único** (*single-agent*) é a forma mais simples de um agente autônomo: **um** LLM,
rodando dentro de **um** loop, com acesso a **um conjunto de tools**, trabalhando para atingir
**um objetivo**. Não há hierarquia, não há outro agente coordenando — é um "funcionário"
resolvendo a tarefa do início ao fim, sozinho.

```
  ┌──────────┐        ┌──────────────────────────────────────────┐        ┌───────────┐
  │          │  msg   │                AGENTE ÚNICO                │  resp  │           │
  │ USUÁRIO  ├───────►│                                            ├───────►│  RESPOSTA │
  │          │        │   ┌─────────── LOOP ────────────┐          │        │           │
  └──────────┘        │   │  1. LLM pensa               │          │        └───────────┘
                      │   │  2. escolhe tool + args     │          │
                      │   │  3. executa a tool          │◄──┐      │
                      │   │  4. observa resultado       │   │      │
                      │   │  5. continua OU responde    │───┘      │
                      │   └─────────────────────────────┘          │
                      │            │        │        │             │
                      │            ▼        ▼        ▼             │
                      │        [tool A] [tool B] [tool C]          │
                      └────────────────────────────────────────────┘
```

As quatro peças, sempre:

| Peça | O que é | Exemplo |
|------|---------|---------|
| **1 LLM** | O modelo que raciocina e decide | Claude 3.5 Sonnet via Bedrock |
| **1 loop** | O ciclo pensar → agir → observar → repetir (padrão ReAct) | `while` com limite de iterações |
| **Conjunto de tools** | Funções que o agente pode chamar | `consultar_pedido`, `consultar_estoque` |
| **1 objetivo** | A meta que define quando o loop termina | "atender a dúvida do cliente sobre o pedido" |

> O **loop** em si é o assunto do documento anterior da série
> ([`01-loop-de-agente.md`](./01-loop-de-agente.md)). Aqui assumimos que você já entende o loop
> e focamos em como ele forma um **agente único** completo.

**Diferença para os outros padrões (só para situar):** no agente único há **um** loop e **um**
LLM tomando todas as decisões. Quando você quebra a tarefa e delega pedaços a outros agentes, já
entrou em **sub-agentes** ([`03-sub-agentes.md`](./03-sub-agentes.md)) ou **multi-agente** — que
**não** são o foco aqui.

---

## 2. Quando usar (e quando NÃO usar)

O agente único é o **default**: comece por ele. Só suba a complexidade quando ele não der conta.

### ✅ Quando USAR

- **Tarefa coesa**: um objetivo claro que cabe num raciocínio contínuo (ex.: "responder dúvidas
  de pós-venda", "gerar um relatório a partir de uma consulta").
- **Um único domínio**: o agente lida com um assunto (suporte, finanças, RH) e não precisa trocar
  de "personalidade" no meio.
- **Poucas tools** (regra prática: ~3 a 8): o LLM consegue escolher entre elas sem se perder.
- **Prompt de sistema enxuto**: as instruções cabem em poucas centenas de tokens.
- **Latência importa**: menos coordenação = menos idas ao LLM = resposta mais rápida.
- **Você está começando**: prototipar com agente único é mais barato e mais fácil de depurar.

### ❌ Quando NÃO usar

- **Muitas tools** (dezenas): o LLM começa a confundir qual chamar, erra argumentos e "alucina"
  ferramentas. Sintoma clássico de que passou da hora de **especializar** (ver §7 e depois a série
  de sub-agentes/multi-agente).
- **Prompt gigante**: se o *system prompt* virou um manual de 3 mil tokens com regras de vários
  domínios, o modelo perde foco e o custo por chamada dispara.
- **Domínios muito distintos** no mesmo agente (jurídico + financeiro + suporte técnico): cada um
  quer instruções e tools diferentes; misturar degrada a qualidade.
- **Fluxos com etapas paralelas independentes** que se beneficiariam de executar ao mesmo tempo
  (isso é caso de decomposição — fora do escopo deste doc).

```
  Comece SEMPRE aqui ──►  ┌─────────────────┐
                          │  AGENTE ÚNICO   │
                          └───────┬─────────┘
                                  │  "está confuso, prompt inchado,
                                  │   tools demais, domínios misturados"?
                                  ▼
                    ┌──────────────────────────────┐
                    │ Primeiro: OTIMIZE (§7)         │
                    │ (limitar tools, rotear modelo, │
                    │  cache, encolher prompt)       │
                    └───────────────┬────────────────┘
                                    │ ainda não resolve?
                                    ▼
                    ┌──────────────────────────────┐
                    │ Só então: sub-agentes /        │
                    │ multi-agente (outros docs)     │
                    └────────────────────────────────┘
```

---

## 3. Impacto prático: custo, latência e throughput (nível alto)

> Aqui é só o essencial. A **análise profunda** (fórmulas, comparação entre padrões, números)
> está em [`06-custo-latencia-throughput.md`](./06-custo-latencia-throughput.md).

O agente único é o padrão **mais econômico e mais rápido** entre os padrões agênticos, porque tem
o **menor overhead de coordenação**. Ainda assim, cada iteração do loop é **uma chamada ao LLM** —
e o histórico cresce a cada passo.

| Dimensão | Comportamento no agente único | Regra prática |
|----------|-------------------------------|---------------|
| **Custo** | ∝ (nº de iterações) × (tokens de entrada + saída por iteração). O histórico reenviado a cada passo é o principal vilão. | Limite iterações; use modelo mais barato quando possível (roteamento). |
| **Latência** | Soma **sequencial** das iterações. 1 tool call ≈ 1 round-trip ao LLM + execução da tool. | Menos iterações = menos latência. Streaming melhora a *latência percebida*. |
| **Throughput** | Alto e simples de escalar horizontalmente: cada request é independente (ótimo para Lambda). | Concorrência do Lambda + limites de TPM/RPM do Bedrock são o teto real. |

Mini-visão da conta de custo (ilustrativa, não use como preço real):

```
custo_request ≈ Σ (por iteração)  [ tokens_entrada × preço_in  +  tokens_saída × preço_out ]

  iteração 1: prompt + histórico(0)          ─┐
  iteração 2: prompt + histórico(cresce)      ├─ histórico REENVIADO a cada passo
  iteração 3: prompt + histórico(cresce mais) ─┘   ↑ é aqui que o custo escala
```

**Leitura rápida:** poucos passos e prompt enxuto = barato e rápido. Muitos passos e histórico
grande = caro e lento. Por isso o **limite de iterações** e o **corte de histórico** são
obrigatórios (ver exemplo na §4 e boas práticas na §8).

---

## 4. Exemplo completo: agente único em Lambda com Bedrock (SDK puro)

Agente de **pós-venda** com **duas tools** (`consultar_pedido` e `consultar_estoque`), memória de
conversa em **DynamoDB**, clientes instanciados **fora do handler** (reuso entre invocações), e
**limite de iterações** para não rodar o loop infinitamente.

Pontos de atenção marcados no código:
- Clientes (`bedrock`, `dynamodb`) **fora do handler** → reaproveitados em invocações quentes.
- `MAX_ITERACOES` → trava de segurança do loop.
- Memória por `session_id` no DynamoDB → conversa multi-turno.
- Uso do **Bedrock `converse` com tool use** (a API unificada de tool calling).

```python
# agente_unico_lambda.py
import json
import os
import boto3
from decimal import Decimal

# ── Clientes FORA do handler (reuso em invocações quentes) ───────────────────
bedrock = boto3.client("bedrock-runtime", region_name=os.environ.get("AWS_REGION", "us-east-1"))
dynamodb = boto3.resource("dynamodb")
tabela_memoria = dynamodb.Table(os.environ.get("TABELA_MEMORIA", "conversas-agente"))

# ── Configuração ─────────────────────────────────────────────────────────────
MODEL_ID = os.environ.get("MODEL_ID", "anthropic.claude-3-5-sonnet-20240620-v1:0")
MAX_ITERACOES = int(os.environ.get("MAX_ITERACOES", "6"))
SYSTEM_PROMPT = (
    "Você é um assistente de pós-venda de uma loja. "
    "Responda em português do Brasil, de forma objetiva. "
    "Use as ferramentas para consultar pedidos e estoque quando necessário. "
    "Se não souber, diga que não sabe."
)

# ── Definição das tools (esquema que o Bedrock envia ao modelo) ──────────────
TOOL_CONFIG = {
    "tools": [
        {
            "toolSpec": {
                "name": "consultar_pedido",
                "description": "Retorna status e itens de um pedido pelo seu ID.",
                "inputSchema": {"json": {
                    "type": "object",
                    "properties": {"pedido_id": {"type": "string", "description": "ID do pedido"}},
                    "required": ["pedido_id"],
                }},
            }
        },
        {
            "toolSpec": {
                "name": "consultar_estoque",
                "description": "Retorna a quantidade em estoque de um produto pelo seu SKU.",
                "inputSchema": {"json": {
                    "type": "object",
                    "properties": {"sku": {"type": "string", "description": "SKU do produto"}},
                    "required": ["sku"],
                }},
            }
        },
    ]
}


# ── Implementação real das tools (aqui, dados fake; troque por sua API/DB) ────
def consultar_pedido(pedido_id: str) -> dict:
    pedidos = {
        "1001": {"status": "enviado", "itens": ["SKU-ABC", "SKU-XYZ"], "previsao": "2 dias"},
        "1002": {"status": "processando", "itens": ["SKU-ABC"], "previsao": "5 dias"},
    }
    return pedidos.get(pedido_id, {"erro": "pedido não encontrado"})


def consultar_estoque(sku: str) -> dict:
    estoque = {"SKU-ABC": 42, "SKU-XYZ": 0}
    return {"sku": sku, "quantidade": estoque.get(sku, 0)}


EXECUTORES = {
    "consultar_pedido": lambda args: consultar_pedido(args["pedido_id"]),
    "consultar_estoque": lambda args: consultar_estoque(args["sku"]),
}


# ── Memória de conversa no DynamoDB ──────────────────────────────────────────
def carregar_historico(session_id: str) -> list:
    resp = tabela_memoria.get_item(Key={"session_id": session_id})
    return resp.get("Item", {}).get("mensagens", [])


def salvar_historico(session_id: str, mensagens: list) -> None:
    tabela_memoria.put_item(Item={"session_id": session_id, "mensagens": mensagens})


# ── O agente único: um LLM, um loop, um conjunto de tools, um objetivo ───────
def executar_agente(session_id: str, mensagem_usuario: str) -> str:
    mensagens = carregar_historico(session_id)
    mensagens.append({"role": "user", "content": [{"text": mensagem_usuario}]})

    for _ in range(MAX_ITERACOES):
        resposta = bedrock.converse(
            modelId=MODEL_ID,
            system=[{"text": SYSTEM_PROMPT}],
            messages=mensagens,
            toolConfig=TOOL_CONFIG,
            inferenceConfig={"maxTokens": 1024, "temperature": 0.2},
        )

        saida = resposta["output"]["message"]
        mensagens.append(saida)  # guarda o turno do assistente no histórico

        stop = resposta.get("stopReason")

        # O modelo quer chamar uma ou mais tools:
        if stop == "tool_use":
            resultados_tools = []
            for bloco in saida["content"]:
                if "toolUse" not in bloco:
                    continue
                tool_use = bloco["toolUse"]
                nome = tool_use["name"]
                args = tool_use["input"]
                try:
                    resultado = EXECUTORES[nome](args)
                except Exception as e:  # tool falhou → devolve erro ao modelo
                    resultado = {"erro": str(e)}
                resultados_tools.append({
                    "toolResult": {
                        "toolUseId": tool_use["toolUseId"],
                        "content": [{"json": resultado}],
                    }
                })
            # Devolve o resultado das tools como mensagem do usuário e continua o loop
            mensagens.append({"role": "user", "content": resultados_tools})
            continue

        # O modelo respondeu (não pediu mais tools) → fim do loop
        texto = "".join(b.get("text", "") for b in saida["content"])
        salvar_historico(session_id, mensagens)
        return texto

    # Estourou o limite de iterações (trava de segurança)
    salvar_historico(session_id, mensagens)
    return "Não consegui concluir a solicitação dentro do limite de passos. Pode reformular?"


# ── Handler Lambda ───────────────────────────────────────────────────────────
def handler(event, context):
    body = json.loads(event.get("body", "{}")) if isinstance(event.get("body"), str) else event
    session_id = body.get("session_id", "anon")
    mensagem = body.get("mensagem", "")

    resposta = executar_agente(session_id, mensagem)

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"resposta": resposta}, ensure_ascii=False),
    }
```

**IAM mínimo** (política do papel da Lambda), como referência:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel"],
      "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20240620-v1:0"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/conversas-agente"
    }
  ]
}
```

> **Nota sobre timeout do Lambda:** o loop é sequencial, então some as latências de cada iteração.
> Com `MAX_ITERACOES = 6` e um modelo médio, configure o timeout do Lambda com folga (ex.: 30–60s)
> e considere `maxTokens` baixo para respostas rápidas.

---

## 5. Exemplo equivalente com framework (Strands / LangChain)

O loop da §4 (pensar → tool → observar → repetir, com parsing manual do `stopReason`) é exatamente
o que os frameworks empacotam. Veja como o **mesmo agente** fica bem mais curto.

### 5a. Strands Agents (AWS) — orientado a Bedrock

O Strands Agents implementa o loop agêntico para você; basta declarar as tools com o decorator
`@tool` e criar o `Agent`.

```python
# agente_strands.py
import boto3
from strands import Agent, tool
from strands.models import BedrockModel

# Cliente/modelo FORA do handler (reuso em invocações quentes)
modelo = BedrockModel(model_id="anthropic.claude-3-5-sonnet-20240620-v1:0", region="us-east-1")

@tool
def consultar_pedido(pedido_id: str) -> dict:
    """Retorna status e itens de um pedido pelo seu ID."""
    pedidos = {"1001": {"status": "enviado", "previsao": "2 dias"}}
    return pedidos.get(pedido_id, {"erro": "pedido não encontrado"})

@tool
def consultar_estoque(sku: str) -> dict:
    """Retorna a quantidade em estoque de um produto pelo seu SKU."""
    estoque = {"SKU-ABC": 42, "SKU-XYZ": 0}
    return {"sku": sku, "quantidade": estoque.get(sku, 0)}

# O agente único: modelo + tools + prompt. O loop é interno ao framework.
agente = Agent(
    model=modelo,
    tools=[consultar_pedido, consultar_estoque],
    system_prompt="Você é um assistente de pós-venda. Responda em pt-BR, objetivo.",
)

def handler(event, context):
    mensagem = event.get("mensagem", "")
    resultado = agente(mensagem)          # roda o loop completo internamente
    return {"resposta": str(resultado)}
```

### 5b. LangChain — `create_tool_calling_agent`

Mesma ideia com o ecossistema LangChain, usando o Bedrock como modelo.

```python
# agente_langchain.py
from langchain_aws import ChatBedrockConverse
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.tools import tool

llm = ChatBedrockConverse(
    model="anthropic.claude-3-5-sonnet-20240620-v1:0",
    region_name="us-east-1",
    temperature=0.2,
)

@tool
def consultar_pedido(pedido_id: str) -> dict:
    """Retorna status e itens de um pedido pelo seu ID."""
    return {"1001": {"status": "enviado"}}.get(pedido_id, {"erro": "não encontrado"})

@tool
def consultar_estoque(sku: str) -> dict:
    """Retorna a quantidade em estoque de um produto pelo seu SKU."""
    return {"sku": sku, "quantidade": {"SKU-ABC": 42}.get(sku, 0)}

prompt = ChatPromptTemplate.from_messages([
    ("system", "Você é um assistente de pós-venda. Responda em pt-BR, objetivo."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),  # onde o loop de tools acontece
])

agent = create_tool_calling_agent(llm, [consultar_pedido, consultar_estoque], prompt)
executor = AgentExecutor(agent=agent, tools=[consultar_pedido, consultar_estoque],
                         max_iterations=6)  # limite de iterações

def handler(event, context):
    resultado = executor.invoke({"input": event.get("mensagem", "")})
    return {"resposta": resultado["output"]}
```

**O que o framework economiza:** o parsing do `stopReason`, o roteamento das chamadas de tool, a
montagem do histórico e o controle do loop. **O que você troca:** uma dependência a mais (pesa no
tamanho do pacote/*cold start* do Lambda) e um pouco menos de controle fino. Comparação aprofundada
de frameworks está em
[`../05-comparacao-frameworks.md`](../05-comparacao-frameworks.md).

---

## 6. Ferramentas e opções de mercado

Todas viabilizam **agente único**. A diferença está em ser um serviço **gerenciado** (a AWS/o
fornecedor roda o loop e o estado por você) vs. uma **biblioteca** (você roda o loop no seu código,
ex.: dentro do Lambda).

| Opção | O que é | Gerenciado vs Biblioteca | Self-hosted vs SaaS | Encaixe no Lambda |
|-------|---------|--------------------------|---------------------|-------------------|
| **Bedrock Agents** | Serviço gerenciado da AWS que executa o loop, tools (via Lambda) e memória | **Gerenciado** (AWS roda o agente) | **SaaS** (serviço AWS); suas tools rodam no seu Lambda | O agente é o serviço; o Lambda vira *action group* (tool). Você não roda o loop. |
| **Strands Agents** | Biblioteca open source da AWS, focada em Bedrock, com loop pronto | **Biblioteca** | **Self-hosted** (seu código) | Ótimo: leve, feito para rodar no seu runtime, inclusive Lambda |
| **LangChain** | Framework amplo (agents, chains, tools, memória) | **Biblioteca** | **Self-hosted** (seu código) | Bom, mas atenção ao tamanho do pacote / cold start |
| **LlamaIndex** | Framework com forte foco em RAG + agentes | **Biblioteca** | **Self-hosted** (seu código) | Bom para agente único com RAG; cuidar do peso do pacote |
| **Pydantic AI** | Framework leve, agentes com validação de tipos (Pydantic) | **Biblioteca** | **Self-hosted** (seu código) | Muito bom: leve, tipado, amigável a funções serverless |
| **OpenAI Agents SDK** | SDK de agentes da OpenAI (loop, tools, handoffs) | **Biblioteca** | **Self-hosted** (seu código), modelos via API | Roda no Lambda; modelos são SaaS externo (não Bedrock por padrão) |
| **Semantic Kernel** | SDK da Microsoft (.NET/Python/Java) para agentes e plugins | **Biblioteca** | **Self-hosted** (seu código) | Roda no Lambda; ecossistema mais orientado a Azure/OpenAI |

> **Regra rápida de escolha:**
> - Quer o mínimo de código de infra e aceitar o "jeito AWS" → **Bedrock Agents** (gerenciado).
> - Quer controle e leveza no Lambda com Bedrock → **Strands** ou **Pydantic AI**.
> - Já usa um ecossistema amplo / RAG pesado → **LangChain** ou **LlamaIndex**.

Panorama completo de frameworks e o encaixe no Lambda:
[`../04-frameworks-orquestracao.md`](../04-frameworks-orquestracao.md).

---

## 7. Como escalar um agente único antes de partir para multi-agente

Antes de introduzir sub-agentes ou multi-agente (mais complexidade, mais custo de coordenação),
**esgote as otimizações do agente único**. Quase sempre resolvem.

```
  AGENTE ÚNICO "no limite"
        │
        ├─ 1) LIMITAR / CURAR TOOLS  ──► exponha só as tools relevantes ao contexto
        │
        ├─ 2) ROTEAR O MODELO ───────► pergunta simples → modelo barato;
        │                              complexa → modelo forte
        │                              (ver 06-roteamento-de-llms.md)
        │
        ├─ 3) CACHE ─────────────────► prompt caching (Bedrock) + cache de
        │                              respostas/tools frequentes
        │
        ├─ 4) ENCOLHER O PROMPT ─────► instruções enxutas, sem regras de outros domínios
        │
        └─ 5) CORTAR HISTÓRICO ──────► janela deslizante / resumo do histórico
                                       (reduz tokens reenviados por iteração)
```

1. **Limitar e curar as tools.** Menos tools = decisões mais fáceis para o LLM. Se um agente
   precisa de 30 tools, provavelmente ele está fazendo trabalho de vários domínios (sinal de que
   um dia pode virar sub-agentes — mas ainda não é hoje).
2. **Roteamento de modelo.** Nem todo request precisa do modelo mais caro. Direcione por
   complexidade/custo. Detalhes e padrões (custo, semântico, cascata, fallback) em
   [`../06-roteamento-de-llms.md`](../06-roteamento-de-llms.md).
3. **Cache.** Use *prompt caching* do Bedrock para o *system prompt* e definições de tools
   estáveis; e cache de resultados de tools deterministas (ex.: estoque consultado há segundos).
4. **Encolher o prompt.** Remova regras que não se aplicam ao domínio do agente. Prompt menor =
   mais barato e mais focado.
5. **Cortar histórico.** Janela deslizante (últimos N turnos) ou resumo do histórico antigo, para
   não reenviar a conversa inteira a cada iteração.

Só quando **todas** essas alavancas foram usadas e o agente **ainda** está sobrecarregado
(tools/domínios demais) é hora de considerar **sub-agentes**
([`03-sub-agentes.md`](./03-sub-agentes.md)) ou **multi-agente** — assuntos dos próximos documentos.

---

## 8. Boas práticas e armadilhas

### ✅ Boas práticas

- **Sempre um limite de iterações** (`MAX_ITERACOES`). Loop sem trava é loop caro e potencialmente
  infinito.
- **Clientes fora do handler** (Bedrock, DynamoDB): reuso entre invocações quentes reduz latência.
- **Tools idempotentes e com timeout próprio.** Uma tool lenta trava todo o loop.
- **Descrições de tool claras e curtas.** O LLM escolhe a tool pela descrição; texto ambíguo gera
  chamada errada.
- **Trate erro de tool devolvendo o erro ao modelo** (como no exemplo da §4), para ele tentar se
  recuperar em vez de quebrar a Lambda.
- **Observabilidade:** logue cada iteração (tool chamada, args, tokens). Facilita achar loops caros.
- **Streaming** (`converse_stream`) para melhorar a latência **percebida** em respostas longas.

### ⚠️ Armadilhas

- **Excesso de tools.** A principal armadilha do agente único: o LLM confunde/erra a tool. Sintoma
  de que é hora de otimizar (§7) — e não necessariamente de partir para multi-agente.
- **Prompt inchado.** System prompt de vários domínios degrada a qualidade e aumenta o custo por
  chamada.
- **Histórico ilimitado.** Cresce a cada turno; sem corte, cada iteração fica mais cara.
- **Sem limite de iterações.** Custo e latência descontrolados; risco de timeout do Lambda.
- **Confiar cegamente na tool.** Valide os argumentos que o modelo passou antes de executar
  ações sensíveis (ex.: escrever no banco).

---

## 9. Glossário curto

| Termo | Significado |
|-------|-------------|
| **Agente único** | Um LLM + um loop + um conjunto de tools + um objetivo, sem outros agentes. |
| **Loop (ReAct)** | Ciclo pensar → agir (tool) → observar → repetir, até responder. |
| **Tool (ferramenta)** | Função que o agente pode chamar (API, consulta a banco, cálculo). |
| **Tool use** | Mecanismo pelo qual o modelo pede para chamar uma tool (no Bedrock, via `converse`). |
| **`stopReason`** | Campo da resposta do Bedrock que indica se o modelo quer tool (`tool_use`) ou terminou. |
| **Iteração** | Uma volta do loop = uma chamada ao LLM. |
| **Memória de conversa** | Histórico de mensagens persistido (aqui, em DynamoDB) para multi-turno. |
| **Roteamento de modelo** | Escolher o modelo por request (custo/complexidade). Ver `06-roteamento-de-llms.md`. |
| **Gerenciado** | O provedor roda o loop/estado por você (ex.: Bedrock Agents). |
| **Biblioteca** | Você roda o loop no seu próprio código (ex.: Strands, LangChain). |

---

## 10. Resumo em uma frase

> **Agente único = um LLM em um loop, com poucas tools e um objetivo claro; é o ponto de partida
> ideal, e você deve otimizá-lo ao máximo (menos tools, roteamento, cache, prompt enxuto) antes de
> pensar em sub-agentes ou multi-agente.**

---

## 11. Anterior / Próximo na série

| ◄ Anterior | Próximo ► |
|------------|-----------|
| [`01-loop-de-agente.md`](./01-loop-de-agente.md) — a unidade básica (o loop) | [`03-sub-agentes.md`](./03-sub-agentes.md) — quando o agente único delega |

**Documentos relacionados:**
- Custo/latência/throughput em detalhe: [`06-custo-latencia-throughput.md`](./06-custo-latencia-throughput.md)
- Roteamento de modelos: [`../06-roteamento-de-llms.md`](../06-roteamento-de-llms.md)
- Frameworks e encaixe no Lambda: [`../04-frameworks-orquestracao.md`](../04-frameworks-orquestracao.md)
- Comparação de frameworks: [`../05-comparacao-frameworks.md`](../05-comparacao-frameworks.md)
