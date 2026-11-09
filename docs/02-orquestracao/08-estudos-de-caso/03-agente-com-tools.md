# Estudo de Caso 3 — Agente com Tools (Consulta Pedidos, Cria Tickets, Calcula)

> Um cenário end-to-end de **agente que usa ferramentas**: como o **número de iterações do loop** e o **roteamento de modelo** (barato para classificar, forte para agir) dominam o custo — para 200 mil requisições/mês e 3 tools.

📍 Volte ao índice dos estudos de caso: [README.md](README.md)

---

## Índice

1. [O problema e os requisitos](#1-o-problema-e-os-requisitos)
2. [Arquitetura geral](#2-arquitetura-geral)
3. [Opção A — Loop ReAct manual com SDK Bedrock 🟡](#3-opção-a--loop-react-manual-com-sdk-bedrock-)
4. [Opção B — LangChain AgentExecutor / Strands + roteamento de modelo 🟡](#4-opção-b--langchain-agentexecutor--strands--roteamento-de-modelo-)
5. [Opção C — Bedrock Agents gerenciado (Action Groups) 🟢 (oper., lock-in)](#5-opção-c--bedrock-agents-gerenciado-action-groups--oper-lock-in)
6. [O que domina o custo: iterações e roteamento](#6-o-que-domina-o-custo-iterações-e-roteamento)
7. [Tabela comparativa](#7-tabela-comparativa)
8. [Recomendação por perfil](#8-recomendação-por-perfil)
9. [Armadilhas](#9-armadilhas)
10. [Glossário](#10-glossário)
11. [Resumo em uma frase](#11-resumo-em-uma-frase)

---

## 1. O problema e os requisitos

Uma empresa quer um **assistente com ferramentas**: ele consulta o **status de um pedido**, **abre um ticket** de suporte e faz **cálculos** (ex.: prazo, valor com desconto). O modelo não responde sozinho — ele **decide qual tool chamar**, chama, lê o resultado e decide o próximo passo (loop de agente).

| Requisito | Alvo |
|-----------|------|
| **Volume** | ~200.000 requests/mês (~6.700/dia) |
| **Tools** | 3 (consultar pedido, criar ticket, calcular) |
| **Latência** | < 5 s (aceita algumas iterações) |
| **Qualidade** | Escolher a tool certa, montar argumentos corretos, não inventar dados |
| **Segurança** | Tools com efeito colateral (criar ticket) exigem validação de argumentos |

**Perfil das requisições (para as contas):**

- ~60% são simples (1 tool, **2 iterações** do loop: decidir → responder).
- ~40% são compostas (2 tools, **3–4 iterações**).
- Média de **~2,6 iterações/request**. Cada iteração é uma chamada ao LLM com o histórico acumulando (entrada cresce a cada passo).
- Estimativa média por request: **~2.500 tokens de entrada** (histórico + definições de tools, somando as iterações) e **~400 de saída**.

> ⚠️ **Todos os custos abaixo são ILUSTRATIVOS**, ordem de grandeza, para comparação. Confirme na [Calculadora de Preços da AWS](https://calculator.aws) e nos [preços do Bedrock](https://aws.amazon.com/bedrock/pricing/).

---

## 2. Arquitetura geral

O agente é um **loop**: o modelo decide uma ação (chamar tool ou responder), a ação é executada, o resultado volta ao modelo, e assim por diante até uma resposta final.

```
        Usuário
           │ (pedido)
           ▼
   ┌──────────────────┐
   │   ORQUESTRADOR    │◄───────────────┐
   │  (loop de agente) │                │ resultado da tool
   └───────┬───────────┘                │
           │ decisão do LLM             │
           ▼                            │
   [chamar tool?] ── sim ──►  ┌─────────┴───────────┐
           │                  │  TOOLS (Lambdas)    │
           │ não              │  • consultar pedido │
           ▼                  │  • criar ticket     │
     resposta final          │  • calcular         │
                             └─────────────────────┘
```

Cada volta no loop = **uma chamada ao LLM**. Mais iterações = mais tokens = mais custo e latência.

---

## 3. Opção A — Loop ReAct manual com SDK Bedrock 🟡

**Framework:** nenhum (boto3 + `converse` com tool use) · **Modelo:** médio (Claude 3.5 Sonnet) para todas as etapas · **Técnica:** loop ReAct escrito à mão · **Compute:** Lambda.

### Como funciona

Você implementa o loop: define as tools no formato `toolConfig` do Bedrock, chama `converse`, e se a resposta pedir uma tool (`stopReason == "tool_use"`), você executa a função, devolve o resultado como mensagem e chama de novo — até o modelo parar de pedir tools. Controle total do loop, do limite de iterações e do tratamento de erro.

### Exemplo de código (Lambda-ready)

```python
import json
import boto3

bedrock = boto3.client("bedrock-runtime")
MODEL_ID = "anthropic.claude-3-5-sonnet-20240620-v1:0"

TOOLS = {"toolSpec": [
    {"name": "consultar_pedido", "description": "Status de um pedido por ID",
     "inputSchema": {"json": {"type": "object", "properties": {"pedido_id": {"type": "string"}}}}},
    {"name": "criar_ticket", "description": "Abre ticket de suporte",
     "inputSchema": {"json": {"type": "object", "properties": {"assunto": {"type": "string"}}}}},
    {"name": "calcular", "description": "Avalia expressão numérica",
     "inputSchema": {"json": {"type": "object", "properties": {"expr": {"type": "string"}}}}},
]}

def exec_tool(nome, args):
    if nome == "consultar_pedido":
        return {"status": "em transporte"}          # chamaria API/DynamoDB
    if nome == "criar_ticket":
        return {"ticket_id": "TK-123"}
    if nome == "calcular":
        return {"resultado": eval_seguro(args["expr"])}

def handler(event, context):
    msgs = [{"role": "user", "content": [{"text": json.loads(event["body"])["mensagem"]}]}]
    for _ in range(5):  # LIMITE de iterações: protege contra loop infinito
        r = bedrock.converse(modelId=MODEL_ID, messages=msgs, toolConfig=TOOLS)
        out = r["output"]["message"]
        msgs.append(out)
        if r["stopReason"] != "tool_use":
            texto = next(b["text"] for b in out["content"] if "text" in b)
            return {"statusCode": 200, "body": json.dumps({"resposta": texto})}
        # executa cada tool pedida e devolve o resultado
        results = []
        for b in out["content"]:
            if "toolUse" in b:
                tu = b["toolUse"]
                res = exec_tool(tu["name"], tu["input"])
                results.append({"toolResult": {"toolUseId": tu["toolUseId"],
                                                "content": [{"json": res}]}})
        msgs.append({"role": "user", "content": results})
    return {"statusCode": 200, "body": json.dumps({"resposta": "Não consegui concluir."})}
```

### Custo estimado mensal (ILUSTRATIVO)

Modelo médio em todas as ~2,6 iterações. A entrada acumula histórico e definições de tools a cada volta.

| Item | Conta | Custo |
|------|-------|-------|
| Tokens entrada | 200k × 2.500 tok = 500M × US$ 3/1M | ~US$ 1.500 |
| Tokens saída | 200k × 400 tok = 80M × US$ 15/1M | ~US$ 1.200 |
| Lambda (orquestração + tools) | 200k × ~3 s × 512 MB | ~US$ 12 |
| API Gateway | 200k req | ~US$ 0,70 |
| **Total** | | **~US$ 2.715/mês** |

### Complexidade: 🟡 média

Loop, limite de iterações, parsing de tool use, tratamento de erro de tool — tudo por sua conta, mas é código direto e transparente.

### Trade-offs

- ✅ Controle total (limite de iterações, logs, retries, validação de argumentos).
- ✅ Sem dependências pesadas; fácil de auditar.
- ❌ Custo alto porque **todas** as iterações usam o modelo médio (não há roteamento).
- ❌ Você reimplementa o que frameworks já dão de graça.

---

## 4. Opção B — LangChain AgentExecutor / Strands + roteamento de modelo 🟡

**Framework:** LangChain AgentExecutor ou Strands Agents · **Modelo:** **roteamento** — modelo **barato** (Haiku/Nova Micro) para **classificar/decidir a tool**, modelo **forte** só para **redigir a resposta final** · **Técnica:** agente com roteamento de custo · **Compute:** Lambda.

### Como funciona

O framework cuida do loop e do binding das tools. A sacada de custo é **rotear o modelo**: a maioria das iterações (decidir qual tool, montar argumentos) é trabalho simples — um modelo barato resolve. Só a **redação final** (quando exige tom e síntese) usa o modelo forte. Isso corta o item que mais pesa: tokens de modelo caro nas iterações intermediárias.

### Exemplo de código (Lambda-ready)

```python
from langchain_aws import ChatBedrock
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.tools import tool
from langchain_core.prompts import ChatPromptTemplate

# Roteamento de modelo: barato para decidir, forte para a resposta final
llm_barato = ChatBedrock(model_id="amazon.nova-micro-v1:0")           # iterações/decisão
llm_forte  = ChatBedrock(model_id="anthropic.claude-3-5-sonnet-20240620-v1:0")  # redação

@tool
def consultar_pedido(pedido_id: str) -> str:
    "Retorna o status do pedido."
    return "em transporte"

@tool
def criar_ticket(assunto: str) -> str:
    "Abre um ticket de suporte e retorna o ID."
    return "TK-123"

@tool
def calcular(expr: str) -> str:
    "Avalia uma expressão numérica simples."
    return str(eval_seguro(expr))

tools = [consultar_pedido, criar_ticket, calcular]
prompt = ChatPromptTemplate.from_messages([
    ("system", "Use as ferramentas quando necessário. Não invente dados."),
    ("human", "{input}"), ("placeholder", "{agent_scratchpad}"),
])

# O agente usa o modelo barato no loop; a resposta final pode ser refinada pelo forte
agent = create_tool_calling_agent(llm_barato, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, max_iterations=5)

def handler(event, context):
    import json
    q = json.loads(event["body"])["mensagem"]
    r = executor.invoke({"input": q})
    # refino opcional da resposta final com o modelo forte
    final = llm_forte.invoke(f"Reescreva de forma clara e no tom da marca:\n{r['output']}")
    return {"statusCode": 200, "body": json.dumps({"resposta": final.content})}
```

### Custo estimado mensal (ILUSTRATIVO)

Iterações no barato (~US$ 0,25/1M in, ~US$ 1,25/1M out); só ~1 chamada final no forte por request. Divida os 2.500 tokens de entrada: ~2.000 no barato (iterações) + ~500 no forte (contexto da redação).

| Item | Conta | Custo |
|------|-------|-------|
| Entrada (barato) | 200k × 2.000 = 400M × US$ 0,25/1M | ~US$ 100 |
| Saída (barato) | 200k × 250 = 50M × US$ 1,25/1M | ~US$ 63 |
| Entrada (forte, final) | 200k × 500 = 100M × US$ 3/1M | ~US$ 300 |
| Saída (forte, final) | 200k × 300 = 60M × US$ 15/1M | ~US$ 900 |
| Lambda + API GW | 200k req | ~US$ 13 |
| **Total** | | **~US$ 1.376/mês** |

> 💡 **Metade do custo da Opção A**, mesmo produto. O ganho vem de tirar o modelo caro das iterações intermediárias. Se a redação final também puder ser feita pelo modelo barato, o total cai para a faixa de ~US$ 180.

### Complexidade: 🟡 média

O framework simplifica o loop, mas roteamento de modelo, empacotamento no Lambda e o comportamento do AgentExecutor exigem entendimento e testes.

### Trade-offs

- ✅ Loop e tools "de graça"; roteamento derruba o custo dramaticamente.
- ✅ Ecossistema maduro (LangChain) ou enxuto e AWS-first (Strands).
- ❌ Roteamento adiciona uma peça de decisão (o que é "simples" o bastante para o barato?).
- ❌ Modelo barato pode escolher a tool errada em casos ambíguos — precisa de avaliação.

---

## 5. Opção C — Bedrock Agents gerenciado (Action Groups) 🟢 (oper., lock-in)

**Framework:** Bedrock Agents (gerenciado) · **Modelo:** um modelo associado ao agente · **Técnica:** **Action Groups** (cada grupo de ações = um Lambda + schema OpenAPI) · **Compute:** loop gerenciado pela AWS; suas tools em Lambda.

### Como funciona

Você define o agente no Bedrock, descreve as ações via **OpenAPI schema** e liga cada Action Group a um **Lambda** (a tool). A AWS executa o **loop de raciocínio** (decidir ação → invocar seu Lambda → ler resultado → continuar). Você só chama `invoke_agent` e recebe a resposta. Mínimo de código de orquestração.

### Exemplo de código / config (Lambda-ready)

```python
import json
import boto3

agent_rt = boto3.client("bedrock-agent-runtime")
AGENT_ID = "XXXXXXXX"
ALIAS_ID = "YYYYYYYY"

def handler(event, context):
    q = json.loads(event["body"])["mensagem"]
    resp = agent_rt.invoke_agent(
        agentId=AGENT_ID, agentAliasId=ALIAS_ID,
        sessionId=event["requestContext"]["connectionId"],
        inputText=q,
    )
    # resposta chega em streaming de eventos
    texto = "".join(
        e["chunk"]["bytes"].decode()
        for e in resp["completion"] if "chunk" in e
    )
    return {"statusCode": 200, "body": json.dumps({"resposta": texto})}
```

O Lambda da tool recebe do agente o nome da ação e os parâmetros (definidos no OpenAPI) e retorna o resultado — a AWS cuida de encaixar isso no loop.

### Custo estimado mensal (ILUSTRATIVO)

O agente gerenciado usa **um** modelo em todas as iterações (sem roteamento nativo barato/forte). Tokens semelhantes à Opção A + Lambdas das tools.

| Item | Conta | Custo |
|------|-------|-------|
| Tokens entrada | 200k × 2.500 = 500M × US$ 3/1M | ~US$ 1.500 |
| Tokens saída | 200k × 400 = 80M × US$ 15/1M | ~US$ 1.200 |
| Lambdas (Action Groups) | 200k × ~2,6 invocações de tools × curtas | ~US$ 20 |
| API GW / streaming | 200k req | ~US$ 1 |
| **Total** | | **~US$ 2.720/mês** |

> ⚠️ Custo parecido com a Opção A: o gerenciado **não** faz roteamento barato/forte por você, então os tokens de modelo continuam dominando.

### Complexidade: 🟢 baixa (operacional) — mas com lock-in

Pouco código de orquestração; a AWS opera o loop. O custo é de **configuração** (agente, Action Groups, OpenAPI, IAM) e de **acoplamento**: sua lógica de agente vive no Bedrock (difícil portar).

### Trade-offs

- ✅ Menos código de loop; rastreamento (trace) e sessão gerenciados.
- ✅ Bom para times que querem foco no negócio, não no loop.
- ❌ Sem roteamento nativo barato/forte → custo de tokens alto.
- ❌ **Lock-in**: o desenho do agente fica preso ao Bedrock; menos controle fino do loop.

---

## 6. O que domina o custo: iterações e roteamento

Duas alavancas mandam na conta de um agente:

```
  CUSTO ≈ (nº de iterações) × (tokens por iteração) × (preço do modelo por token)
                 │                     │                        │
      corte com limite de       corte com histórico       corte com ROTEAMENTO
      iterações e boas          enxuto e tools bem        (barato p/ decidir,
      descrições de tool        descritas                 forte só p/ agir/redigir)
```

- **Cada iteração reenvia o histórico**: em 3–4 voltas, os tokens de entrada se multiplicam. Reduzir iterações (tools bem descritas, menos ambiguidade) corta custo direto.
- **Roteamento é a maior alavanca**: tirar o modelo caro das iterações de decisão (Opção B) cortou ~50% do custo neste exemplo, sem mudar o produto.
- **Mais tools ≠ linearmente mais caro**, mas infla as definições no prompt (entram em toda iteração). Mantenha descrições curtas e objetivas.

---

## 7. Tabela comparativa

| Critério | A — ReAct manual (SDK) 🟡 | B — LangChain/Strands + roteamento 🟡 | C — Bedrock Agents 🟢 (lock-in) |
|----------|---------------------------|----------------------------------------|----------------------------------|
| **Custo/mês (ilustrativo)** | ~US$ 2.715 | ~US$ 1.376 (ou ~US$ 180 sem forte no final) | ~US$ 2.720 |
| **Fator dominante** | Modelo médio em todas as iterações | Modelo forte só na redação | Modelo único em todas as iterações |
| **Complexidade (build)** | 🟡 média | 🟡 média | 🟢 baixa |
| **Complexidade (oper.)** | Você opera o loop | Você opera o loop (via framework) | AWS opera o loop |
| **Latência** | Média (iterações) | Média (barato é rápido) | Média |
| **Qualidade** | Alta (modelo médio) | Alta (forte na resposta) | Alta |
| **Controle do loop** | Máximo | Alto | Baixo |
| **Lock-in** | Nenhum | Baixo | Alto |
| **Quando escolher** | Quer controle total e auditabilidade | Quer cortar custo com roteamento | Quer gerenciado e aceita lock-in |

---

## 8. Recomendação por perfil

- **MVP / validação:** **Opção C** (Bedrock Agents) para chegar rápido sem escrever loop — ou **Opção A** se você já quer controle e evitar lock-in desde o início. Em baixo volume, a diferença de custo é pequena.
- **Crescimento (custo começa a doer):** **Opção B**. O roteamento de modelo é a alavanca de maior impacto: mantém a qualidade e corta o custo pela metade ou mais.
- **Escala (200k+/mês, custo crítico):** **Opção B** com roteamento agressivo (barato para tudo que der, forte só onde precisa) + limite de iterações e tools bem descritas. Meça a taxa de acerto do modelo barato antes de expandir.

---

## 9. Armadilhas

- **Sem limite de iterações.** Um agente pode entrar em loop e queimar tokens indefinidamente. Sempre defina `max_iterations` e um fallback.
- **Descrições de tool vagas.** Levam o modelo a chamar a tool errada ou a iterar demais tentando descobrir. Boas descrições = menos iterações = menos custo.
- **Usar modelo caro em todas as etapas.** É o erro mais caro (Opções A e C). Roteie: decidir é barato, agir/redigir pode ser caro.
- **Tools com efeito colateral sem validação.** "Criar ticket" com argumentos alucinados gera lixo/ações indevidas. Valide os argumentos antes de executar.
- **Histórico crescendo sem controle.** Em conversas longas, o contexto infla a entrada de cada iteração. Resuma ou trunque o histórico.
- **Lock-in silencioso (Opção C).** Migrar um agente Bedrock para outro stack depois é caro; escolha ciente do acoplamento.
- **`eval` inseguro na tool de cálculo.** Nunca use `eval` cru; use um avaliador seguro/sandbox para expressões.

---

## 10. Glossário

| Termo | Significado |
|-------|-------------|
| **Agente** | LLM que decide e executa ações (tools) em um loop até concluir a tarefa. |
| **Tool / ferramenta** | Função externa (API, banco, cálculo) que o agente pode invocar. |
| **ReAct** | Padrão "Reason + Act": o modelo alterna raciocínio e ações em iterações. |
| **Iteração** | Uma volta do loop = uma chamada ao LLM (decidir próxima ação ou responder). |
| **Tool use / function calling** | Mecanismo pelo qual o modelo pede a execução de uma tool com argumentos. |
| **Roteamento de modelo** | Usar modelos diferentes por etapa (barato p/ decidir, forte p/ agir). |
| **Action Group (Bedrock Agents)** | Conjunto de ações (schema OpenAPI) ligado a um Lambda que as executa. |
| **Lock-in** | Acoplamento a um provedor que dificulta migrar depois. |

---

## 11. Resumo em uma frase

> Num agente com tools, o custo é o produto de iterações × tokens × preço do modelo — e a maior economia vem de rotear modelos (barato para decidir, forte só para agir) e limitar iterações, enquanto o gerenciado (Bedrock Agents) troca esse controle por menos operação ao preço de lock-in.
