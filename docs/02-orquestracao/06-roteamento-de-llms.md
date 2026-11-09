# Roteamento de LLMs (LLM Routing) — Guia Didático Completo

> O roteamento **vive dentro do orquestrador**: é a etapa que decide **qual modelo/caminho** usar para **cada requisição**, equilibrando custo, latência, qualidade e capacidade.

---

## Índice

1. [O que é roteamento de LLM (e por que existe)](#1-o-que-é-roteamento-de-llm-e-por-que-existe)
2. [Onde o roteador fica na arquitetura](#2-onde-o-roteador-fica-na-arquitetura)
3. [Tipos e estratégias de roteamento](#3-tipos-e-estratégias-de-roteamento)
   - [3.1 Regras estáticas](#31-roteamento-por-regras-estáticas)
   - [3.2 Classificação (LLM/classificador decide)](#32-roteamento-por-classificação)
   - [3.3 Complexidade da tarefa](#33-roteamento-por-complexidadedificuldade)
   - [3.4 Custo (cascata barato→caro)](#34-roteamento-por-custo)
   - [3.5 Semântico (embeddings)](#35-roteamento-semântico)
   - [3.6 Capacidade/feature](#36-roteamento-por-capacidadefeature)
   - [3.7 Latência/SLA](#37-roteamento-por-latênciasla)
   - [3.8 Fallback e retry](#38-fallback-e-retry)
   - [3.9 Load balancing](#39-load-balancing)
   - [3.10 Model cascading com verificação](#310-model-cascading-com-verificação)
   - [3.11 LLM-as-judge para re-rotear](#311-llm-as-judge-para-validar-e-re-rotear)
4. [Roteamento entre provedores e entre modelos](#4-roteamento-entre-provedores-e-entre-modelos)
5. [Ferramentas e bibliotecas do mercado](#5-ferramentas-e-bibliotecas-do-mercado)
6. [Exemplos de código Python (Lambda-ready)](#6-exemplos-de-código-python-lambda-ready)
7. [Métricas para avaliar um roteador](#7-métricas-para-avaliar-um-roteador)
8. [Análise de custo do roteamento](#8-análise-de-custo-do-roteamento)
9. [Boas práticas, armadilhas e observabilidade](#9-boas-práticas-armadilhas-e-observabilidade)
10. [Glossário](#10-glossário)
11. [Resumo em uma frase](#11-resumo-em-uma-frase)

---

## 1. O que é roteamento de LLM (e por que existe)

**Roteamento de LLM** é a decisão de **para onde enviar cada requisição**: qual modelo, qual provedor, qual caminho de execução. Em vez de mandar *tudo* para o modelo mais poderoso (e mais caro), o roteador olha para a requisição e escolhe o destino mais adequado.

Pense em um **triângulo de tensões** que o roteador administra:

```
                 QUALIDADE
                    /\
                   /  \
                  /    \
                 /      \
                /        \
          CUSTO ────────── LATÊNCIA
```

Você raramente maximiza os três ao mesmo tempo. O roteador é quem faz esse *trade-off* requisição a requisição.

### Por que ele existe

| Motivo | Explicação |
|--------|-----------|
| **Custo** | Um modelo "topo de linha" pode custar 10–50× mais por token que um modelo pequeno. A maioria das requisições reais é simples (saudações, FAQs, classificações). Mandar tudo para o modelo caro é desperdício. |
| **Latência** | Modelos pequenos respondem mais rápido. Para tarefas simples, o usuário nem precisa do modelo grande — e ganha resposta em menos tempo. |
| **Qualidade** | Tarefas difíceis (raciocínio complexo, código, matemática) precisam de modelos fortes. Roteá-las para o modelo certo evita respostas ruins. |
| **Especialização** | Alguns modelos são melhores em código, outros em resumo, outros em idiomas específicos. O roteador manda cada tarefa ao especialista. |
| **Fallback** | Se um provedor/modelo falhar (erro, throttling, indisponibilidade), o roteador redireciona para uma alternativa, mantendo o sistema no ar. |

### Roteamento de LLM ≠ roteamento de rede

São conceitos **diferentes** que compartilham só o nome:

| Aspecto | Roteamento de **rede** (routers, IP) | Roteamento de **LLM** |
|---------|--------------------------------------|-----------------------|
| **O que decide** | Por qual **caminho físico/lógico** um pacote trafega até um IP de destino | Qual **modelo/provedor** deve **processar** o conteúdo da requisição |
| **Critério** | Tabelas de rota, métricas de rede, endereço de destino | Conteúdo, complexidade, custo, capacidade, SLA |
| **Camada** | Rede (L3/L4) | Aplicação/semântica |
| **Estado do "pacote"** | Não interpreta o conteúdo | **Interpreta o conteúdo** (o prompt) para decidir |
| **Objetivo** | Entregar o pacote ao endereço certo | Entregar a requisição ao **cérebro certo** |

> Resumo: roteamento de rede leva **bytes** a um **endereço**. Roteamento de LLM leva um **prompt** ao **modelo mais adequado**.

---

## 2. Onde o roteador fica na arquitetura

O roteador é uma **etapa interna do orquestrador**, normalmente logo após a normalização do input e antes da chamada ao modelo:

```
Usuário
   │
   ▼
┌───────────────────────────────────────────────────────────┐
│                      ORQUESTRADOR                          │
│                                                            │
│   input ──► [pré-processamento] ──► ┌──────────────┐       │
│              (limpeza, guardrails)  │   ROTEADOR   │       │
│                                     │  decide QUAL │       │
│                                     │  modelo usar │       │
│                                     └──────┬───────┘       │
│                                            │               │
│                 ┌──────────────────────────┼───────────┐   │
│                 ▼               ▼           ▼           ▼   │
│           ┌──────────┐   ┌──────────┐ ┌──────────┐ ┌──────┐│
│           │ Modelo   │   │ Modelo   │ │ Modelo   │ │ RAG /││
│           │ BARATO   │   │ MÉDIO    │ │ FORTE    │ │ tool ││
│           │ (Haiku)  │   │ (Sonnet) │ │ (Opus)   │ │ chain││
│           └────┬─────┘   └────┬─────┘ └────┬─────┘ └──┬───┘│
│                └──────────────┴────────────┴──────────┘    │
│                                │                           │
│                    [pós-processamento / verificação]       │
└────────────────────────────────┬──────────────────────────┘
                                  ▼
                               Usuário
```

Fluxo mínimo (essência do roteamento):

```
input  ──►  ROTEADOR  ──►  modelo escolhido  ──►  resposta
```

Pontos importantes:

- O roteador **não gera a resposta final** — ele só **escolhe o caminho**. (Exceção: no roteamento por classificação, um LLM pequeno *decide*, mas quem responde é o modelo escolhido.)
- Em AWS Lambda, o roteador costuma ser algumas dezenas de linhas de código dentro do handler (regras) ou uma chamada extra rápida (classificador/embeddings).
- O roteador pode encadear com **fallback** e **verificação** (cascata) para garantir robustez.

---

## 3. Tipos e estratégias de roteamento

Cada estratégia tem um perfil de **precisão × custo × latência × complexidade**. Você pode (e frequentemente deve) **combiná-las**.

### 3.1 Roteamento por regras estáticas

**Como funciona:** `if/else` sobre sinais simples — palavras-chave, tamanho do input, tipo de tarefa declarado, presença de anexos/imagens, idioma detectado.

```
if "imagem" in input        -> modelo com visão
elif len(input) < 100       -> modelo barato
elif "código" in input      -> modelo forte
else                        -> modelo médio
```

| Prós | Contras |
|------|---------|
| Rápido (zero latência de rede extra) | Frágil: quebra com fraseados inesperados |
| Barato (nenhuma chamada de modelo) | Difícil de manter conforme regras crescem |
| Determinístico e fácil de depurar | Não entende **intenção**, só padrões literais |

**Quando usar:** MVPs, casos com poucos caminhos bem definidos, ou como **primeira camada barata** antes de estratégias mais caras.

### 3.2 Roteamento por classificação

**Como funciona:** um **modelo pequeno/barato** (ou um classificador treinado) lê a requisição e devolve um **rótulo de rota** (`simples`, `código`, `raciocínio`, etc.). O orquestrador então chama o modelo correspondente.

```
input ──► classificador (Haiku/modelo pequeno) ──► "código"
                                                      │
                                                      ▼
                                              modelo especialista em código
```

| Prós | Contras |
|------|---------|
| Entende intenção melhor que regras | Adiciona **uma chamada extra** (latência + custo) |
| Flexível: adicionar rota = adicionar rótulo | Depende da qualidade do classificador |
| Fácil de evoluir com exemplos | Pode errar e degradar a resposta final |

**Quando usar:** quando há várias rotas e a decisão exige **entender** o pedido, não só casar palavras.

### 3.3 Roteamento por complexidade/dificuldade

**Como funciona:** estima a **dificuldade** da tarefa e escala o modelo conforme ela. Sinais: comprimento, presença de múltiplas etapas, palavras de raciocínio ("explique passo a passo", "prove", "otimize"), estruturas de código, matemática.

| Prós | Contras |
|------|---------|
| Alinha diretamente custo à necessidade | "Complexidade" é difícil de medir com precisão |
| Ótimo para economia agregada | Sinais heurísticos podem enganar |

**Quando usar:** carga com **mistura** de perguntas triviais e difíceis (o caso mais comum em produção).

### 3.4 Roteamento por custo

**Como funciona:** tenta o modelo **mais barato primeiro**; se a resposta não passar em uma verificação (ver [3.10](#310-model-cascading-com-verificação)), **escala** para um modelo mais caro. É a base da **cascata**.

```
tenta BARATO ──► resposta OK? ──sim──► retorna
                    │
                    não
                    ▼
              tenta MÉDIO ──► OK? ──sim──► retorna
                    │
                    não
                    ▼
              tenta FORTE ──► retorna
```

| Prós | Contras |
|------|---------|
| Maximiza economia | Casos difíceis pagam **latência acumulada** (várias tentativas) |
| Degrada com elegância | Precisa de um bom sinal de "resposta OK" |

**Quando usar:** quando economia é prioridade e um pouco de latência extra nos casos difíceis é aceitável.

### 3.5 Roteamento semântico

**Como funciona:** gera o **embedding** da query e compara (similaridade de cosseno) com embeddings de **rotas de referência** (exemplos representativos de cada rota). A rota mais similar vence.

```
query ──► embedding ──► compara com centróides de rotas
                                │
        ┌───────────────┬───────┴────────┬──────────────┐
     "vendas"        "suporte"        "código"       "jurídico"
     sim=0.82         sim=0.31         sim=0.44        sim=0.12
        ▲
      escolhida
```

| Prós | Contras |
|------|---------|
| Barato e rápido (só embedding, sem geração) | Precisa curar bons exemplos por rota |
| Robusto a variações de fraseado | Rotas semanticamente próximas confundem |
| Determinístico e cacheável | Não capta "dificuldade", só "assunto" |

**Quando usar:** roteamento por **tópico/domínio** (qual especialista/prompt/base RAG usar), com muitas rotas.

### 3.6 Roteamento por capacidade/feature

**Como funciona:** roteia pela **feature exigida**, não pelo conteúdo em si:

- **Visão** → só modelos multimodais.
- **Function calling / tool use** → modelos que suportam.
- **Janela de contexto** → input gigante vai para modelo com contexto grande.
- **Idioma** → modelo com melhor desempenho naquele idioma.

| Prós | Contras |
|------|---------|
| Evita erros de "capacidade ausente" | Precisa de um catálogo de features por modelo |
| Regras claras e objetivas | Feature ≠ qualidade (o modelo pode ter a feature e ainda ser fraco) |

**Quando usar:** sempre como **filtro duro** antes de qualquer outra estratégia — não adianta rotear um pedido com imagem para um modelo sem visão.

### 3.7 Roteamento por latência/SLA

**Como funciona:** escolhe o modelo/rota que cabe no **orçamento de tempo** da requisição. Requests interativos (chat ao vivo) vão para modelos rápidos; jobs em batch podem usar modelos lentos e melhores.

| Prós | Contras |
|------|---------|
| Garante experiência responsiva | Pode sacrificar qualidade sob pressa |
| Casa bem com prioridades de negócio | Precisa medir latência real por modelo/região |

**Quando usar:** quando há SLAs distintos (ex.: chat < 2s; relatório noturno pode levar minutos).

### 3.8 Fallback e retry

**Como funciona:** se o destino primário falhar (erro 5xx, throttling, timeout), tenta **outro modelo/provedor** com **retry e backoff exponencial**.

```
provedor A ──falha──► espera 0.5s ──► retry A ──falha──► provedor B ──► ...
```

| Prós | Contras |
|------|---------|
| Aumenta disponibilidade | Fallback pode ter qualidade/custo diferentes |
| Absorve throttling e picos | Requer normalizar respostas entre provedores |

**Quando usar:** **sempre** em produção. É a rede de segurança do roteamento.

### 3.9 Load balancing

**Como funciona:** distribui carga entre **múltiplos provedores, regiões ou chaves de API** para respeitar limites de taxa (rate limits) e reduzir latência regional. Estratégias: round-robin, ponderado, "menos ocupado", por região mais próxima.

| Prós | Contras |
|------|---------|
| Escala além do limite de um único provedor | Complexidade de configuração |
| Reduz throttling e ponto único de falha | Diferenças sutis entre modelos "equivalentes" |

**Quando usar:** alto volume, múltiplas contas/regiões, ou quando um único endpoint não sustenta o tráfego.

### 3.10 Model cascading com verificação

**Como funciona:** é o roteamento por custo ([3.4](#34-roteamento-por-custo)) com uma **etapa de verificação** entre os degraus. A resposta barata só é aceita se passar num critério (confiança do modelo, heurística, ou um verificador). Caso contrário, escala.

```
BARATO ──► verificar (confiança/heurística/juiz)
              │aceita               │rejeita
              ▼                     ▼
           retorna              MÉDIO ──► verificar ──► ... ──► FORTE
```

| Prós | Contras |
|------|---------|
| Economia com "garantia de qualidade" | Latência somada nos casos difíceis |
| Ajustável via limiar de aceitação | O verificador também custa |

**Quando usar:** quando você consegue definir um **bom critério de aceitação** e quer economizar sem abrir mão de qualidade nos casos difíceis.

### 3.11 LLM-as-judge para validar e re-rotear

**Como funciona:** um LLM atua como **juiz**, avaliando a resposta gerada (nota, "passou/não passou", crítica). Se reprovar, o orquestrador **re-roteia** para um modelo mais forte ou refaz com instruções melhores.

```
modelo ──► resposta ──► JUIZ (LLM) ──► nota
                                        │≥ limiar ──► retorna
                                        │< limiar ──► re-rotear p/ modelo forte
```

| Prós | Contras |
|------|---------|
| Verificação semântica (não só heurística) | Juiz custa tokens e latência |
| Melhora qualidade percebida | Risco de **loops** se mal limitado |
| Serve também para avaliação offline | Juiz pode ter vieses próprios |

**Quando usar:** tarefas de alto valor onde a qualidade importa mais que o custo do juiz. **Sempre** limite o número de re-roteamentos (ex.: máx. 2) para evitar loops.

---

## 4. Roteamento entre provedores e entre modelos

Há dois níveis de roteamento, frequentemente combinados:

```
                        ROTEADOR
                           │
        ┌──────────────────┼───────────────────┐
        ▼                  ▼                    ▼
  ENTRE PROVEDORES   ENTRE MODELOS        HÍBRIDO
  (Bedrock, OpenAI,  (do mesmo provedor:  (escolhe provedor
   Anthropic, self-   Haiku/Sonnet/Opus)   E modelo)
   hosted)
```

### Entre modelos do mesmo provedor

O caso mais comum e mais simples. Ex.: no **Amazon Bedrock**, escolher entre `Claude Haiku` (barato/rápido), `Claude Sonnet` (equilíbrio) e `Claude Opus` (forte). Vantagens:

- **Mesmo SDK e mesmo formato** de request/response → troca de modelo é trocar um `modelId`.
- **Mesma região, mesmo IAM, mesma cobrança** → operação simples.
- Ótimo ponto de partida para roteamento por complexidade/custo.

### Entre provedores

Roteia entre **fornecedores diferentes**: Bedrock (multi-model: Anthropic, Meta, Mistral, Amazon Nova, etc.), OpenAI, Anthropic direto (API), modelos **self-hosted** (ex.: em SageMaker/EC2/EKS com vLLM).

| Vantagens | Desafios |
|-----------|----------|
| Redundância real (fallback entre fornecedores) | **Formatos de API diferentes** (precisa normalizar) |
| Aproveitar o "melhor de cada" | Credenciais, cobrança e limites separados |
| Negociar custo/quotas | Latência de rede para APIs externas (fora da AWS) |
| Independência de um único fornecedor | Governança/compliance por provedor |

> Dica: uma camada de **abstração** (ex.: LiteLLM) uniformiza a chamada, escondendo as diferenças de formato entre provedores. Isso facilita muito fallback e load balancing multi-provedor.

---

## 5. Ferramentas e bibliotecas do mercado

> Nota: recursos e disponibilidade de produtos mudam com frequência. Confirme na documentação oficial de cada ferramenta antes de decidir. A coluna "Roda no Lambda?" indica a viabilidade típica de empacotar/usar como **cliente** dentro de uma função Lambda.

| Ferramenta | O que é | Foco | Roda no Lambda? |
|------------|---------|------|-----------------|
| **RouteLLM** | Framework open-source (LMSYS) que treina roteadores para decidir entre modelo forte e fraco, otimizando custo × qualidade | Roteamento custo/qualidade baseado em dados | Sim (biblioteca Python; o roteador é leve) |
| **LiteLLM** | Proxy + Router que unifica 100+ provedores sob a API OpenAI; faz fallback, load balancing, retries, budgets | Abstração multi-provedor + roteamento | Sim como **SDK/router** dentro da Lambda; o **proxy** costuma rodar como serviço (ECS/EKS) |
| **Martian** | Serviço/roteador comercial que escolhe o melhor modelo por request otimizando custo × performance | Roteamento gerenciado | Sim como **cliente HTTP** (chamada externa) |
| **OpenRouter** | Gateway/API única que dá acesso a muitos modelos de vários provedores, com roteamento e fallback | Gateway multi-provedor | Sim como **cliente HTTP** (endpoint externo) |
| **NotDiamond** | Roteador "meta" que escolhe o melhor LLM por prompt via modelos de decisão | Roteamento por qualidade/custo | Sim como **cliente** (SDK/HTTP) |
| **Portkey** | AI Gateway com roteamento, fallback, cache semântico, observabilidade e guardrails | Gateway + observabilidade | Sim como **cliente/SDK**; o gateway é serviço gerenciado/hospedável |
| **Amazon Bedrock — Intelligent Prompt Routing** | Recurso do Bedrock que roteia automaticamente entre modelos de uma família para otimizar custo/qualidade | Roteamento gerenciado nativo AWS | Sim (é só chamar o Bedrock; nada extra para empacotar) |
| **LangChain routing** (`RunnableBranch`, `RunnableLambda`) | Primitivas do LangChain para ramificar o fluxo (condicional ou por função) | Roteamento programático em chains | Sim (importe só o núcleo; use container se pesar) |
| **semantic-router** | Biblioteca leve para roteamento **por embeddings** (rotas definidas por exemplos) | Roteamento semântico | Sim (leve; ótimo para Lambda) |

**Leitura rápida por objetivo:**

- Quero **abstrair provedores** e ter fallback/load balancing → **LiteLLM** (ou **OpenRouter/Portkey** como gateway).
- Quero **roteamento gerenciado** dentro da AWS → **Bedrock Intelligent Prompt Routing**.
- Quero **roteamento semântico simples** → **semantic-router**.
- Quero **otimizar custo × qualidade com dados** → **RouteLLM** / **NotDiamond** / **Martian**.
- Já uso **LangChain** → `RunnableBranch`/`RunnableLambda`.

---

## 6. Exemplos de código Python (Lambda-ready)

> Convenção Lambda: **clientes inicializados fora do handler** (reaproveitados entre invocações no mesmo *warm container*), imports mínimos, sem estado em memória entre invocações. Os `modelId`, preços e nomes são **ilustrativos** — ajuste aos seus.

### (a) Roteador por regras simples

```python
import json
import boto3

# Cliente inicializado FORA do handler (reuso em invocações quentes)
bedrock = boto3.client("bedrock-runtime")

MODELO_BARATO = "anthropic.claude-3-haiku-20240307-v1:0"
MODELO_MEDIO  = "anthropic.claude-3-5-sonnet-20240620-v1:0"

def escolher_modelo(texto: str) -> str:
    t = texto.lower()
    # Filtro por feature/keyword primeiro
    if any(k in t for k in ["código", "code", "função", "bug", "stacktrace"]):
        return MODELO_MEDIO
    # Heurística de complexidade por tamanho
    if len(texto) < 120:
        return MODELO_BARATO
    if any(k in t for k in ["explique passo a passo", "prove", "otimize", "analise"]):
        return MODELO_MEDIO
    return MODELO_BARATO

def invocar(model_id: str, prompt: str) -> str:
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [{"role": "user", "content": prompt}],
    }
    resp = bedrock.invoke_model(modelId=model_id, body=json.dumps(body))
    payload = json.loads(resp["body"].read())
    return payload["content"][0]["text"]

def handler(event, context):
    prompt = event.get("prompt", "")
    model_id = escolher_modelo(prompt)
    resposta = invocar(model_id, prompt)
    return {"modelo": model_id, "resposta": resposta}
```

### (b) Roteador por classificação (modelo pequeno decide)

```python
import json
import boto3

bedrock = boto3.client("bedrock-runtime")

MODELO_CLASSIFICADOR = "anthropic.claude-3-haiku-20240307-v1:0"  # barato/rápido
ROTAS = {
    "simples":    "anthropic.claude-3-haiku-20240307-v1:0",
    "codigo":     "anthropic.claude-3-5-sonnet-20240620-v1:0",
    "raciocinio": "anthropic.claude-3-5-sonnet-20240620-v1:0",
}

PROMPT_CLASSIFICADOR = (
    "Classifique a mensagem do usuário em UMA palavra dentre: "
    "simples, codigo, raciocinio. Responda SOMENTE a palavra.\n\n"
    "Mensagem: {msg}"
)

def _invocar(model_id: str, prompt: str, max_tokens: int = 1024) -> str:
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": max_tokens,
        "messages": [{"role": "user", "content": prompt}],
    }
    resp = bedrock.invoke_model(modelId=model_id, body=json.dumps(body))
    return json.loads(resp["body"].read())["content"][0]["text"].strip()

def classificar(msg: str) -> str:
    rotulo = _invocar(
        MODELO_CLASSIFICADOR,
        PROMPT_CLASSIFICADOR.format(msg=msg),
        max_tokens=5,
    ).lower()
    # Normaliza para uma rota conhecida; default seguro = "simples"
    return rotulo if rotulo in ROTAS else "simples"

def handler(event, context):
    prompt = event.get("prompt", "")
    rota = classificar(prompt)
    model_id = ROTAS[rota]
    resposta = _invocar(model_id, prompt)
    return {"rota": rota, "modelo": model_id, "resposta": resposta}
```

### (c) Roteador semântico com embeddings

```python
import json
import math
import boto3

bedrock = boto3.client("bedrock-runtime")

MODELO_EMBEDDING = "amazon.titan-embed-text-v2:0"

# Rotas definidas por exemplos representativos (curados uma vez)
ROTAS_EXEMPLOS = {
    "vendas":   ["quero comprar", "qual o preço", "tem desconto", "fechar pedido"],
    "suporte":  ["não consigo logar", "está com erro", "como resolvo", "abrir chamado"],
    "codigo":   ["revise esta função", "corrija o bug", "escreva um script python"],
}

MODELO_POR_ROTA = {
    "vendas":  "anthropic.claude-3-haiku-20240307-v1:0",
    "suporte": "anthropic.claude-3-haiku-20240307-v1:0",
    "codigo":  "anthropic.claude-3-5-sonnet-20240620-v1:0",
}

def _embed(texto: str) -> list[float]:
    resp = bedrock.invoke_model(
        modelId=MODELO_EMBEDDING,
        body=json.dumps({"inputText": texto}),
    )
    return json.loads(resp["body"].read())["embedding"]

def _cosseno(a: list[float], b: list[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    na = math.sqrt(sum(x * x for x in a))
    nb = math.sqrt(sum(y * y for y in b))
    return dot / (na * nb + 1e-9)

# Centróides das rotas: calculados no cold start (fora do handler)
CENTROIDES = {}
for rota, exemplos in ROTAS_EXEMPLOS.items():
    vetores = [_embed(e) for e in exemplos]
    dim = len(vetores[0])
    CENTROIDES[rota] = [sum(v[i] for v in vetores) / len(vetores) for i in range(dim)]

def rotear_semantico(query: str) -> str:
    q = _embed(query)
    melhor, melhor_sim = None, -1.0
    for rota, centroide in CENTROIDES.items():
        sim = _cosseno(q, centroide)
        if sim > melhor_sim:
            melhor, melhor_sim = rota, sim
    return melhor

def _invocar(model_id: str, prompt: str) -> str:
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [{"role": "user", "content": prompt}],
    }
    resp = bedrock.invoke_model(modelId=model_id, body=json.dumps(body))
    return json.loads(resp["body"].read())["content"][0]["text"]

def handler(event, context):
    prompt = event.get("prompt", "")
    rota = rotear_semantico(prompt)
    model_id = MODELO_POR_ROTA[rota]
    return {"rota": rota, "modelo": model_id, "resposta": _invocar(model_id, prompt)}
```

> Otimização: em produção, pré-calcule os centróides **em build/deploy** e carregue de um arquivo/parametro (evita gastar embeddings a cada cold start).

### (d) Cascata custo-crescente com verificação

```python
import json
import boto3

bedrock = boto3.client("bedrock-runtime")

# Degraus do mais barato ao mais forte
CASCATA = [
    "anthropic.claude-3-haiku-20240307-v1:0",
    "anthropic.claude-3-5-sonnet-20240620-v1:0",
]

def _invocar(model_id: str, prompt: str, max_tokens: int = 1024) -> str:
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": max_tokens,
        "messages": [{"role": "user", "content": prompt}],
    }
    resp = bedrock.invoke_model(modelId=model_id, body=json.dumps(body))
    return json.loads(resp["body"].read())["content"][0]["text"]

def resposta_aceitavel(pergunta: str, resposta: str) -> bool:
    """Verificação simples. Em produção, pode ser um LLM-as-judge."""
    if not resposta or len(resposta.strip()) < 10:
        return False
    recusas = ["não sei", "não tenho certeza", "não consigo responder"]
    return not any(r in resposta.lower() for r in recusas)

def handler(event, context):
    prompt = event.get("prompt", "")
    tentativas = []
    for model_id in CASCATA:
        resposta = _invocar(model_id, prompt)
        tentativas.append(model_id)
        if resposta_aceitavel(prompt, resposta):
            return {"modelo": model_id, "tentativas": tentativas, "resposta": resposta}
    # Se nenhum passou, retorna a última (do modelo mais forte)
    return {"modelo": CASCATA[-1], "tentativas": tentativas, "resposta": resposta}
```

### (e) Fallback entre modelos com retry/backoff

```python
import json
import time
import random
import boto3
from botocore.config import Config

# Config de retry do próprio boto3 (modo adaptativo ajuda com throttling)
_cfg = Config(retries={"max_attempts": 2, "mode": "adaptive"})
bedrock = boto3.client("bedrock-runtime", config=_cfg)

# Ordem de preferência: primário -> fallbacks
CADEIA_FALLBACK = [
    "anthropic.claude-3-5-sonnet-20240620-v1:0",  # primário
    "anthropic.claude-3-haiku-20240307-v1:0",     # fallback 1
    "meta.llama3-1-8b-instruct-v1:0",             # fallback 2 (outro fornecedor no Bedrock)
]

def _invocar_anthropic(model_id: str, prompt: str) -> str:
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [{"role": "user", "content": prompt}],
    }
    resp = bedrock.invoke_model(modelId=model_id, body=json.dumps(body))
    return json.loads(resp["body"].read())["content"][0]["text"]

def _invocar_llama(model_id: str, prompt: str) -> str:
    body = {"prompt": prompt, "max_gen_len": 1024, "temperature": 0.2}
    resp = bedrock.invoke_model(modelId=model_id, body=json.dumps(body))
    return json.loads(resp["body"].read())["generation"]

def _invocar(model_id: str, prompt: str) -> str:
    # Normaliza formatos diferentes por família de modelo
    if model_id.startswith("anthropic."):
        return _invocar_anthropic(model_id, prompt)
    if model_id.startswith("meta."):
        return _invocar_llama(model_id, prompt)
    raise ValueError(f"Formato não suportado: {model_id}")

def invocar_com_fallback(prompt: str, tentativas_por_modelo: int = 2) -> dict:
    ultimo_erro = None
    for model_id in CADEIA_FALLBACK:
        for tentativa in range(tentativas_por_modelo):
            try:
                return {"modelo": model_id, "resposta": _invocar(model_id, prompt)}
            except Exception as e:  # em produção, capture exceções específicas do boto3
                ultimo_erro = e
                # Backoff exponencial com jitter
                espera = (2 ** tentativa) * 0.5 + random.uniform(0, 0.3)
                time.sleep(espera)
    raise RuntimeError(f"Todos os modelos falharam. Último erro: {ultimo_erro}")

def handler(event, context):
    prompt = event.get("prompt", "")
    return invocar_com_fallback(prompt)
```

---

## 7. Métricas para avaliar um roteador

Um roteador só é bom se você **medir**. Principais métricas:

| Métrica | O que mede | Como interpretar |
|---------|-----------|------------------|
| **Taxa de acerto de rota** | % de requisições enviadas à rota "correta" (segundo um rótulo de referência) | Alta = roteador confiável; baixa = roteando errado |
| **Custo médio por request** | Custo total ÷ nº de requests (inclui o custo do roteador) | Comparar com o baseline "tudo no modelo forte" |
| **Latência p50 / p95** | Mediana e cauda de latência ponta a ponta | p95 alto indica que casos difíceis/cascata estão pesando |
| **Taxa de fallback** | % de requests que caíram em fallback | Alta = instabilidade de provedor ou primário mal escolhido |
| **Qualidade** | Nota da resposta (LLM-as-judge, avaliação humana, ou métricas de tarefa) | Garante que a economia não degradou o resultado |
| **Taxa de re-roteamento** | % que precisou escalar/re-rotear | Alta = classificador/limiar mal calibrados |
| **Overhead do roteador** | Latência e custo *só* da decisão de roteamento | Deve ser pequeno frente ao ganho |

### A/B testing

Divida o tráfego real em dois grupos:

```
tráfego ──┬── 50% ──► ROTEADOR NOVO      ─┐
          └── 50% ──► baseline/atual      ─┴─► comparar custo, latência, qualidade
```

- Compare **as mesmas métricas** entre os grupos.
- Rode tempo suficiente para significância estatística.
- Segmente por tipo de tarefa (o roteador pode ganhar em umas e perder em outras).

### Shadow testing (teste sombra)

Roda o roteador **em paralelo, sem afetar o usuário**:

```
request ──► roteador ATUAL ──► resposta ao usuário (produção)
        └─► roteador NOVO  ──► só registra (log/análise), NÃO responde
```

- Zero risco para o usuário (o novo roteador não decide de verdade).
- Coleta dados reais para comparar decisões e estimar custo/qualidade antes de promover o novo roteador.

---

## 8. Análise de custo do roteamento

### Como o roteamento economiza

A ideia central: **a maioria das requisições reais é simples** e não precisa do modelo mais caro. Mandar as simples para um modelo barato reduz o custo agregado.

> ⚠️ **Valores ilustrativos.** Os números abaixo servem só para mostrar o **raciocínio**. Preços de modelos, tokens e regiões mudam — **confirme sempre na [calculadora de preços da AWS](https://calculator.aws/) e na página de preços do Amazon Bedrock**.

Cenário ilustrativo — 100.000 requests/mês:

| Estratégia | Distribuição | Custo relativo (ilustrativo) |
|------------|--------------|------------------------------|
| **Tudo no modelo forte** | 100% forte | **1,00× (baseline)** |
| **Com roteamento** | 70% barato + 30% forte | **~0,40×** |

Se o modelo forte custa, digamos, 10× o barato por token, e 70% do tráfego vai para o barato:

```
Custo baseline (tudo forte)      = 100% × 10  = 1000 (unidades ilustrativas)
Custo roteado (70% barato/30% forte) = 70% × 1 + 30% × 10 = 0,7 + 3,0 = 3,7 unidades
                                     ≈ 63% de economia (ilustrativo)
```

### O custo do próprio roteador

O roteamento **não é grátis**. Some ao custo:

| Tipo de roteador | Custo adicional | Latência adicional |
|------------------|-----------------|--------------------|
| **Regras estáticas** | ~zero | desprezível |
| **Semântico (embedding)** | 1 embedding por request (barato) | baixa (dezenas de ms) |
| **Classificação (LLM pequeno)** | 1 chamada de LLM pequeno por request | média (uma ida ao modelo) |
| **Cascata / LLM-as-judge** | 1+ chamadas extras nos casos difíceis | pode somar bastante |

Regra de ouro: **o custo do roteador precisa ser muito menor que a economia que ele gera.** Um classificador que custa quase tanto quanto o modelo caro anula o benefício.

### Trade-off latência × economia

```
        Economia
           ▲
           │        ● cascata agressiva (economiza muito, mas p95 sobe)
           │     ●
           │   ●  roteamento por classificação
           │ ●    roteamento semântico
           │●     regras estáticas (rápido, economia moderada)
           └────────────────────────────► Latência
```

- Cascata e LLM-as-judge **economizam mais**, mas **pioram a cauda de latência (p95)** nos casos difíceis.
- Regras/semântico têm **overhead mínimo**, mas decisão menos "inteligente".
- Escolha conforme seu SLA: chat interativo tolera menos latência que jobs batch.

---

## 9. Boas práticas, armadilhas e observabilidade

### Boas práticas

- **Comece simples.** Regras estáticas + fallback já resolvem muito. Adicione sofisticação só quando os dados justificarem.
- **Feature como filtro duro primeiro.** Nunca roteie um pedido com imagem para um modelo sem visão, nem um input gigante para um modelo de contexto pequeno.
- **Default seguro.** Se o roteador ficar em dúvida, prefira uma rota que **não degrade** a experiência (ou o modelo forte, em tarefas críticas).
- **Sempre tenha fallback.** Throttling e indisponibilidade acontecem. Tenha um plano B (e um C).
- **Limite re-roteamentos.** Defina um teto (ex.: máx. 2) para evitar loops.
- **Normalize respostas entre provedores.** Formatos diferem; padronize a saída do orquestrador.
- **Cacheie decisões repetidas.** Queries idênticas/semelhantes podem reusar a rota (e até a resposta).
- **Clientes fora do handler no Lambda.** Reaproveite conexões em invocações quentes.

### Armadilhas comuns

| Armadilha | Sintoma | Mitigação |
|-----------|---------|-----------|
| **Overhead do roteador** | Roteador custa/demora quase tanto quanto a economia | Use roteador mais leve; meça o overhead |
| **Loops de re-roteamento** | Latência explode; custo dispara | Limite tentativas; corte com timeout |
| **Roteamento errado** | Qualidade cai sem motivo aparente | Meça taxa de acerto; ajuste com dados reais |
| **Rotas semânticas coladas** | Confusão entre rotas parecidas | Cure exemplos melhores; aumente limiar |
| **Fallback silencioso ruim** | Usuário recebe respostas de qualidade inferior sem sinal | Logue o modelo usado; alerte em fallback |
| **Otimizar só custo** | Economiza mas usuários reclamam | Monitore qualidade junto com custo |

### Observabilidade

Registre **por requisição** (ex.: em logs estruturados / CloudWatch):

```json
{
  "request_id": "abc-123",
  "rota_escolhida": "codigo",
  "modelo_usado": "claude-3-5-sonnet",
  "estrategia": "classificacao",
  "fallback": false,
  "re_roteamentos": 0,
  "latencia_ms": 842,
  "tokens_in": 210,
  "tokens_out": 640,
  "custo_estimado": 0.0041,
  "verificacao": "aceita"
}
```

Com isso você monta dashboards de **custo por rota**, **latência p50/p95**, **taxa de fallback** e **distribuição de rotas** — a base para ajustar o roteador com evidências.

---

## 10. Glossário

| Termo | Definição |
|-------|-----------|
| **Roteamento de LLM** | Decisão de qual modelo/provedor/caminho processa cada requisição |
| **Roteador (router)** | Componente do orquestrador que faz essa decisão |
| **Cascata (cascading)** | Tentar modelos do mais barato ao mais caro, escalando se necessário |
| **Fallback** | Rota alternativa acionada quando a primária falha |
| **Load balancing** | Distribuição de carga entre provedores/regiões/chaves |
| **Roteamento semântico** | Escolha de rota por similaridade de embeddings |
| **Classificador** | Modelo pequeno que rotula a requisição para decidir a rota |
| **LLM-as-judge** | LLM que avalia respostas para aceitar/rejeitar/re-rotear |
| **Embedding** | Vetor numérico que representa o significado de um texto |
| **Similaridade de cosseno** | Medida de proximidade entre dois vetores (embeddings) |
| **Throttling** | Limitação de taxa imposta pelo provedor |
| **Backoff exponencial** | Espera crescente entre tentativas de retry |
| **Jitter** | Aleatoriedade adicionada ao backoff para evitar sincronização |
| **p50 / p95** | Percentis de latência (mediana / cauda) |
| **Shadow testing** | Rodar o novo roteador em paralelo sem afetar o usuário |
| **A/B testing** | Comparar duas estratégias dividindo o tráfego real |
| **Intelligent Prompt Routing** | Recurso nativo do Amazon Bedrock que roteia entre modelos automaticamente |
| **Cold start** | Primeira invocação "fria" de uma Lambda (inicialização) |
| **Warm container** | Contêiner Lambda reaproveitado entre invocações |

---

## 11. Resumo em uma frase

> **Roteamento de LLM é a etapa do orquestrador que, para cada requisição, escolhe o modelo/caminho certo — equilibrando custo, latência, qualidade e capacidade — em vez de mandar tudo para o modelo mais caro.**
