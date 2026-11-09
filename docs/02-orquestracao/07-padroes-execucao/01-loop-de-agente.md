# O Loop de Agente (ReAct) — a unidade básica de um orquestrador

> Primeiro documento da série **Padrões de Execução**. Foco em **um único conceito**: o
> **loop de agente** (padrão *ReAct* — *Reasoning + Acting*). É o ciclo
> **pensar → agir → observar → repetir** que transforma um LLM "que só responde texto"
> em um **agente** capaz de usar ferramentas para resolver uma tarefa.
>
> **Contexto:** AWS Bedrock + Lambda como pano de fundo.
> **Público:** quem está aprendendo a construir orquestradores.
>
> Multi-agente, sub-agentes e múltiplos orquestradores **não** são o assunto aqui — são
> outros padrões da série. Aqui olhamos só para **uma cabeça, um loop**.

---

## Índice

1. [O que é o loop de agente e por que existe](#1-o-que-é-o-loop-de-agente-e-por-que-existe)
2. [Anatomia de uma iteração](#2-anatomia-de-uma-iteração)
3. [Por que o contexto cresce (e o custo vira ~quadrático)](#3-por-que-o-contexto-cresce-e-o-custo-vira-quadrático)
4. [A trava de segurança: `MAX_ITERACOES`](#4-a-trava-de-segurança-max_iteracoes)
5. [Exemplo completo em Python (Lambda-ready) com Bedrock](#5-exemplo-completo-em-python-lambda-ready-com-bedrock)
6. [Ferramentas de mercado que implementam o loop por você](#6-ferramentas-de-mercado-que-implementam-o-loop-por-você)
7. [Boas práticas e armadilhas](#7-boas-práticas-e-armadilhas)
8. [Glossário e resumo em uma frase](#8-glossário-e-resumo-em-uma-frase)
9. [Próximo na série](#9-próximo-na-série)

---

## 1. O que é o loop de agente e por que existe

Um LLM "cru" só faz **uma** coisa: recebe texto e devolve texto. Ele não consulta um banco,
não chama uma API, não lê um arquivo. Se você perguntar "qual o saldo do cliente 123?", ele
**não sabe** — esse dado está fora dele.

O **loop de agente** resolve isso. Em vez de pedir a resposta final de uma vez, você deixa o
modelo **raciocinar** sobre o que precisa, **pedir uma ferramenta** (tool), receber o
**resultado** dessa ferramenta e então **decidir** se já pode responder ou se precisa de mais
um passo. Isso se repete num ciclo.

Esse padrão tem nome: **ReAct** — de *Reasoning* (raciocínio) + *Acting* (ação). A ideia é
intercalar "pensar" e "agir", usando o resultado de cada ação como nova informação para o
próximo pensamento.

### Por que existe

- LLM sozinho **não tem acesso** a dados/sistemas externos → precisa de **tools**.
- Uma tarefa raramente se resolve em **um** passo → precisa **repetir** (buscar, calcular, confirmar).
- O modelo precisa **reagir** ao que descobriu → o resultado de uma tool muda o próximo passo.

### O ciclo ReAct (diagrama)

```
                         ┌──────────────────────────────────────────┐
                         │               LOOP ReAct                  │
        pergunta ───────►│                                           │
                         │   ┌───────────────────────────────────┐   │
                         │   │ 1. PENSAR                         │   │
                         │   │    LLM raciocina sobre o objetivo │   │
                         │   └───────────────┬───────────────────┘   │
                         │                   ▼                       │
                         │   ┌───────────────────────────────────┐   │
                         │   │ 2. AGIR                           │   │
                         │   │    LLM escolhe uma tool + args    │   │
                         │   └───────────────┬───────────────────┘   │
                         │                   ▼                       │
                         │   ┌───────────────────────────────────┐   │
                         │   │ 3. EXECUTAR (fora do LLM)         │   │
                         │   │    o SEU código roda a tool       │   │
                         │   └───────────────┬───────────────────┘   │
                         │                   ▼                       │
                         │   ┌───────────────────────────────────┐   │
                         │   │ 4. OBSERVAR                       │   │
                         │   │    resultado volta ao LLM         │   │
                         │   └───────────────┬───────────────────┘   │
                         │                   ▼                       │
                         │        ┌──────────────────────┐           │
                         │        │ 5. DECIDIR:           │           │
                         │        │  já resolvi?          │           │
                         │        └───────┬──────────┬────┘           │
                         │      não (volta ao 1)      sim             │
                         │           ▲                │               │
                         │           └────────────────┘               │
                         └───────────────────────────┬───────────────┘
                                                      ▼
                                              resposta final
```

> **Quem faz o quê:** o **LLM** pensa e decide *qual* tool chamar (passos 1, 2, 5). O **seu
> orquestrador** (o código no Lambda) é quem **executa** a tool de verdade (passo 3) e devolve
> a observação (passo 4). O LLM **nunca** roda código sozinho — ele só pede.

---

## 2. Anatomia de uma iteração

Uma **iteração** = uma volta completa do loop = **uma chamada ao LLM** + (opcionalmente) a
execução de uma tool. Vamos abrir as quatro fases:

| Fase | Quem faz | O que acontece | Exemplo concreto |
|------|----------|----------------|------------------|
| **Raciocínio** | LLM | Analisa o objetivo e o histórico; decide o próximo passo | "Preciso do saldo; vou chamar `buscar_saldo`." |
| **Tool call** | LLM | Emite um pedido estruturado: nome da tool + argumentos (JSON) | `buscar_saldo(cliente_id="123")` |
| **Execução** | Seu código | Roda a função/API de verdade e captura o retorno | Consulta DynamoDB → `{"saldo": 42.50}` |
| **Observação** | Seu código → LLM | Devolve o resultado ao modelo como novo turno | "resultado da tool: saldo = 42.50" |

Depois da observação, o LLM **raciocina de novo** (nova iteração). Ou ele chama outra tool, ou
ele decide que já tem tudo e emite a **resposta final** (texto puro, sem tool call) — o que
**encerra** o loop.

### Como o loop "sabe" que terminou

O critério é simples: **o modelo parou de pedir tools**.

- Se a resposta do LLM contém um **pedido de tool** → executa e continua.
- Se a resposta do LLM é **só texto** (nenhuma tool) → é a resposta final → **para**.

No Bedrock (Claude via API Messages/Converse), isso aparece como o campo `stop_reason`:
`tool_use` significa "quero uma tool, continue"; `end_turn` significa "terminei".

```
Iteração 1:  LLM → stop_reason=tool_use  (pede buscar_saldo)   → executa, continua
Iteração 2:  LLM → stop_reason=tool_use  (pede converter_moeda)→ executa, continua
Iteração 3:  LLM → stop_reason=end_turn  (texto final)         → PARA
```

---

## 3. Por que o contexto cresce (e o custo vira ~quadrático)

Aqui mora o detalhe que pega quem está começando. **A cada iteração, você reenvia todo o
histórico** — pergunta original, cada raciocínio, cada tool call e cada resultado. O LLM não
tem memória entre chamadas; a "memória" é você reenviando tudo.

Resultado: o contexto **cresce a cada volta**, e como você **paga por token de entrada em
toda chamada**, o custo **não** é linear no número de iterações — é aproximadamente
**quadrático**.

### Exemplo numérico simples

Suponha que cada passo adiciona ~400 tokens (raciocínio + tool call + resultado):

```
Iteração 1: envia  500 tokens  (só a pergunta)
Iteração 2: envia  900 tokens  (pergunta + passo 1)
Iteração 3: envia 1.300 tokens (+ passo 2)
Iteração 4: envia 1.700 tokens (+ passo 3)
Iteração 5: envia 2.100 tokens (+ passo 4)
                 ─────────────
Total enviado: 6.500 tokens de ENTRADA em 5 iterações
```

Compare com uma chamada única "ingênua" de 500 tokens: **6.500 vs 500** — mais de **13×** só de
tokens de entrada, para 5 iterações. Dobre as iterações e o total não dobra: ele **mais que
dobra**. Esse é o comportamento quadrático (a soma de uma progressão: 1+2+3+...+n cresce com n²).

```
tokens
acumulados
  ▲
  │                                   ● (cresce cada vez mais rápido)
  │                            ●
  │                     ●
  │              ●
  │       ●
  │  ●
  └──────────────────────────────────────►  nº de iterações
```

> **Insight-chave:** iterações não são "de graça". Cada volta reenvia o histórico inteiro. Um
> agente que "pensa demais" em muitas voltas custa **muito** mais do que a intuição sugere.

**Consequências diretas:**

- **Custo** ↑ a cada iteração (contexto de entrada cresce).
- **Latência** ↑ (as chamadas são **em série** — uma espera a outra).
- **Throughput** ↓ (o Lambda fica ocupado mais tempo por request).

Isso leva naturalmente à próxima seção: você **tem** que limitar o número de voltas.

---

## 4. A trava de segurança: `MAX_ITERACOES`

Um agente pode entrar em **loop infinito**: chamar a mesma tool de novo e de novo, ou nunca
decidir que terminou. Sem trava, isso significa:

- **Custo descontrolado** (cada volta é uma chamada paga, com contexto crescente).
- **Timeout do Lambda** (que tem limite máximo de 15 minutos — e você não quer chegar perto).
- **Experiência ruim** (o usuário espera para sempre).

Por isso, **todo** loop de agente precisa de um limite duro de iterações — geralmente uma
constante `MAX_ITERACOES`. Quando o limite é atingido, você **para** e devolve uma resposta de
fallback (ou o melhor que tiver até ali), em vez de continuar indefinidamente.

```
para cada iteração de 1 até MAX_ITERACOES:
    resposta = chama_o_llm(historico)
    se resposta é texto final:
        retorna resposta          ◄── caminho feliz
    senão:
        executa a tool pedida
        adiciona resultado ao historico
# se saiu do for sem retornar:
retorna "não consegui concluir dentro do limite"   ◄── trava acionada
```

> **Regra de ouro:** `MAX_ITERACOES` **não é opcional**. É o cinto de segurança que impede
> custo infinito e travamento. Comece com um valor baixo (ex.: 5–8) e ajuste medindo a média
> real de iterações por request.

---

## 5. Exemplo completo em Python (Lambda-ready) com Bedrock

Loop ReAct **manual** (sem framework), usando o Bedrock Runtime com *tool use* do Claude.
Pontos de atenção marcados nos comentários:

- **Clientes criados FORA do handler** → reaproveitados entre invocações (menos cold start).
- **`MAX_ITERACOES`** → trava de segurança obrigatória.
- **Tratamento de tool** → validar nome e argumentos antes de executar.
- **Log por iteração** → observabilidade.

```python
# lambda_function.py
import json
import logging
import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# ---------------------------------------------------------------------------
# Clientes FORA do handler: criados uma vez por container e reaproveitados.
# Isso reduz latência e cold start (não recriar a cada invocação).
# ---------------------------------------------------------------------------
bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

MODEL_ID = "anthropic.claude-3-5-sonnet-20240620-v1:0"
MAX_ITERACOES = 6  # <<< trava de segurança OBRIGATÓRIA

# ---------------------------------------------------------------------------
# Definição das tools que o modelo PODE pedir (schema JSON).
# O modelo só sugere; quem executa é o nosso código.
# ---------------------------------------------------------------------------
TOOLS = [
    {
        "toolSpec": {
            "name": "buscar_saldo",
            "description": "Retorna o saldo atual de um cliente pelo ID.",
            "inputSchema": {
                "json": {
                    "type": "object",
                    "properties": {
                        "cliente_id": {"type": "string", "description": "ID do cliente"}
                    },
                    "required": ["cliente_id"],
                }
            },
        }
    },
    {
        "toolSpec": {
            "name": "converter_moeda",
            "description": "Converte um valor de BRL para outra moeda.",
            "inputSchema": {
                "json": {
                    "type": "object",
                    "properties": {
                        "valor": {"type": "number"},
                        "moeda_destino": {"type": "string", "description": "Ex.: USD"},
                    },
                    "required": ["valor", "moeda_destino"],
                }
            },
        }
    },
]


# ---------------------------------------------------------------------------
# Implementação REAL das tools (no mundo real: DynamoDB, APIs, etc.).
# Aqui simplificado para o exemplo.
# ---------------------------------------------------------------------------
def _tool_buscar_saldo(cliente_id: str) -> dict:
    saldos = {"123": 42.50, "456": 1000.00}
    if cliente_id not in saldos:
        return {"erro": f"cliente {cliente_id} não encontrado"}
    return {"cliente_id": cliente_id, "saldo_brl": saldos[cliente_id]}


def _tool_converter_moeda(valor: float, moeda_destino: str) -> dict:
    taxas = {"USD": 0.19, "EUR": 0.17}
    taxa = taxas.get(moeda_destino.upper())
    if taxa is None:
        return {"erro": f"moeda {moeda_destino} não suportada"}
    return {"valor_convertido": round(valor * taxa, 2), "moeda": moeda_destino.upper()}


def executar_tool(nome: str, args: dict) -> dict:
    """Despacha para a tool certa, validando nome e argumentos."""
    try:
        if nome == "buscar_saldo":
            return _tool_buscar_saldo(str(args["cliente_id"]))
        if nome == "converter_moeda":
            return _tool_converter_moeda(float(args["valor"]), str(args["moeda_destino"]))
        # Nome desconhecido: NUNCA execute algo que não reconhece.
        return {"erro": f"tool desconhecida: {nome}"}
    except (KeyError, TypeError, ValueError) as e:
        # Argumentos inválidos viram observação, não exceção que derruba o loop.
        return {"erro": f"argumentos inválidos para {nome}: {e}"}


# ---------------------------------------------------------------------------
# O LOOP ReAct em si.
# ---------------------------------------------------------------------------
def rodar_agente(pergunta_usuario: str) -> str:
    # O histórico ("messages") é a memória: nós reenviamos tudo a cada iteração.
    messages = [{"role": "user", "content": [{"text": pergunta_usuario}]}]

    system = [{"text": "Você é um assistente financeiro. Use as tools quando precisar de dados."}]

    for iteracao in range(1, MAX_ITERACOES + 1):
        logger.info("Iteração %d/%d | tokens aprox no histórico: %d",
                    iteracao, MAX_ITERACOES, len(json.dumps(messages)) // 4)

        resposta = bedrock.converse(
            modelId=MODEL_ID,
            messages=messages,
            system=system,
            toolConfig={"tools": TOOLS},
            inferenceConfig={"maxTokens": 1024, "temperature": 0.0},
        )

        stop_reason = resposta["stopReason"]
        saida = resposta["output"]["message"]
        # Guardamos a resposta do modelo no histórico (raciocínio + tool call).
        messages.append(saida)

        # ---- CAMINHO FELIZ: modelo respondeu só texto, sem pedir tool ----
        if stop_reason == "end_turn":
            textos = [b["text"] for b in saida["content"] if "text" in b]
            return "\n".join(textos).strip()

        # ---- O modelo pediu uma ou mais tools: executamos cada uma ----
        if stop_reason == "tool_use":
            resultados_tool = []
            for bloco in saida["content"]:
                if "toolUse" not in bloco:
                    continue
                uso = bloco["toolUse"]
                nome = uso["name"]
                args = uso.get("input", {})
                logger.info("Tool solicitada: %s | args=%s", nome, args)

                resultado = executar_tool(nome, args)  # validação embutida

                resultados_tool.append({
                    "toolResult": {
                        "toolUseId": uso["toolUseId"],
                        "content": [{"json": resultado}],
                    }
                })
            # A OBSERVAÇÃO volta como um turno "user" com os resultados.
            messages.append({"role": "user", "content": resultados_tool})
            continue  # próxima iteração

        # Qualquer outro stop_reason: encerra por segurança.
        logger.warning("stop_reason inesperado: %s", stop_reason)
        break

    # ---- TRAVA acionada: excedeu MAX_ITERACOES sem resposta final ----
    logger.warning("Limite de iterações atingido sem resposta final.")
    return "Não consegui concluir sua solicitação dentro do limite de passos. Tente reformular."


# ---------------------------------------------------------------------------
# Handler do Lambda (fino: só orquestra entrada/saída).
# ---------------------------------------------------------------------------
def lambda_handler(event, context):
    try:
        body = json.loads(event.get("body", "{}")) if isinstance(event.get("body"), str) else event
        pergunta = body.get("pergunta", "").strip()
        if not pergunta:
            return {"statusCode": 400, "body": json.dumps({"erro": "campo 'pergunta' é obrigatório"})}

        resposta = rodar_agente(pergunta)
        return {"statusCode": 200, "body": json.dumps({"resposta": resposta}, ensure_ascii=False)}
    except Exception as e:  # noqa: BLE001 — fronteira do Lambda: nunca vaze stacktrace cru
        logger.exception("Erro no agente")
        return {"statusCode": 500, "body": json.dumps({"erro": "erro interno"})}
```

**O que observar neste código:**

- O `for iteracao in range(1, MAX_ITERACOES + 1)` **é** o loop ReAct.
- `stop_reason == "end_turn"` é a **única** saída pelo caminho feliz.
- `stop_reason == "tool_use"` dispara **execução** e volta ao topo (observação vira novo turno).
- Se o `for` termina sem `return`, a **trava** devolve o fallback.
- `executar_tool` **valida** nome e argumentos — nunca executa algo desconhecido, e erro de
  argumento vira **observação** (o modelo pode corrigir na próxima volta) em vez de derrubar tudo.

---

## 6. Ferramentas de mercado que implementam o loop por você

Escrever o loop à mão (seção 5) é ótimo para **entender**. Em produção, muitas equipes usam
uma ferramenta pronta que já implementa o "pensar→agir→observar→repetir", o parsing de tool
calls e o controle de iterações.

| Ferramenta | O que é | Gerenciado ou biblioteca? | Roda em Lambda? | Self-hosted (seu ambiente) |
|-----------|---------|---------------------------|-----------------|----------------------------|
| **Bedrock Agents** | Serviço AWS que executa o loop por você (o agente vive na AWS) | **Gerenciado** (serviço) | Você chama via SDK do Lambda; o loop roda **na AWS**, não no seu processo | ❌ Não (SaaS AWS gerenciado) |
| **Strands Agents** | SDK leve da AWS para agentes; loop como biblioteca Python | **Biblioteca** | ✅ Sim (é código Python no seu runtime) | ✅ Sim (Lambda/ECS/EC2) |
| **LangChain `AgentExecutor`** | Executor de agente clássico; roda o loop ReAct sobre suas tools | **Biblioteca** | ✅ Sim (atenção ao tamanho do pacote/layers) | ✅ Sim (Lambda/ECS/EC2) |
| **LangGraph** | Modela o loop como **grafo de estados** (nós = passos, arestas = transições) | **Biblioteca** | ✅ Sim (código no seu runtime) | ✅ Sim (Lambda/ECS/EC2) |
| **LlamaIndex agents** | Agentes com foco em RAG/dados; loop de tool use embutido | **Biblioteca** | ✅ Sim | ✅ Sim (Lambda/ECS/EC2) |
| **Pydantic AI** | Framework de agentes com saída **tipada/validada** (Pydantic) | **Biblioteca** | ✅ Sim | ✅ Sim (Lambda/ECS/EC2) |
| **CrewAI** | Orquestra "papéis" de agentes; **por baixo** cada agente roda um loop de tool use | **Biblioteca** | ✅ Sim (mais pesado) | ✅ Sim (ECS/EC2 recomendado) |
| **OpenAI Agents SDK** | SDK de agentes da OpenAI; implementa o loop e o roteamento de tools | **Biblioteca** | ✅ Sim (código no seu runtime) | ✅ Sim (roda no seu ambiente; o **modelo** é chamado via API) |

> **Como ler a coluna "Self-hosted":** significa que o **loop roda no seu ambiente**
> (Lambda/ECS/EC2), sob seu controle. O **modelo** em si é quase sempre um serviço remoto
> (Bedrock, OpenAI, etc.) — "self-hosted" aqui se refere ao **executor do loop**, não a
> hospedar o LLM. O único caso **totalmente gerenciado** (loop na nuvem, fora do seu processo)
> é o **Bedrock Agents**.

> **Nuance do Lambda:** bibliotecas grandes (LangChain, CrewAI) podem estourar limites de
> tamanho do pacote e sofrer com cold start. Para essas, ECS/Fargate costuma ser mais
> confortável. SDKs leves (Strands, código manual) encaixam bem no Lambda.

Um panorama mais amplo de frameworks e o encaixe em Lambda está em
[`../04-frameworks-orquestracao.md`](../04-frameworks-orquestracao.md) e a
comparação aprofundada em
[`../05-comparacao-frameworks.md`](../05-comparacao-frameworks.md).

---

## 7. Boas práticas e armadilhas

### Boas práticas

| Prática | Por quê |
|--------|---------|
| **Limite de iterações (`MAX_ITERACOES`)** | Evita loop infinito e custo descontrolado. **Obrigatório.** |
| **Log por iteração** | Registrar nº da iteração, tool pedida, args e tamanho do histórico → observabilidade e depuração. |
| **Timeout curto por tool** | Uma tool travada não pode segurar o Lambda inteiro; defina timeout em cada chamada externa. |
| **Validar args da tool** | O LLM pode inventar argumentos; valide (tipos, obrigatórios) antes de executar. |
| **Erro de tool vira observação** | Devolver o erro ao modelo (em vez de crashar) permite que ele **corrija** na próxima volta. |
| **`temperature=0` para decisões** | Escolha de tool é decisão, não criatividade; determinismo ajuda a reduzir voltas erráticas. |
| **Medir média de iterações** | Se a média está alta, o prompt ou as tools estão mal desenhados. |

### Armadilhas

| Armadilha | Sintoma | Mitigação |
|-----------|---------|-----------|
| **Loop infinito** | Mesma tool chamada repetidamente; nunca chega em `end_turn` | `MAX_ITERACOES` + detectar repetição idêntica de tool call |
| **Contexto explodindo** | Custo/latência disparam; tokens de entrada enormes | Limitar iterações; resumir/truncar histórico; retornar resultados de tool enxutos |
| **Tool sem timeout** | Uma volta trava e o Lambda estoura os 15 min | Timeout por chamada externa; falha rápida vira observação |
| **Confiar cegamente nos args** | Execução com dados inválidos/perigosos | Validação estrita + allow-list de tools |
| **Muitas tools no mesmo agente** | Modelo escolhe a tool errada; mais voltas | Reduzir o conjunto de tools (ou dividir responsabilidades — assunto de outro doc da série) |
| **Handler "gordo"** | Cliente boto3 recriado a cada invocação; cold start alto | Criar clientes **fora** do handler (como no exemplo) |

---

## 8. Glossário e resumo em uma frase

| Termo | Significado |
|-------|-------------|
| **ReAct** | Padrão *Reasoning + Acting*: intercala raciocínio e ação (tool) em um ciclo. |
| **Loop de agente** | O ciclo pensar→agir→observar→repetir que roda até o modelo responder. |
| **Iteração** | Uma volta completa do loop = uma chamada ao LLM (+ eventual execução de tool). |
| **Tool (ferramenta)** | Função/API que o modelo pode **pedir** para acessar dados ou executar ações. |
| **Tool call** | O pedido estruturado do modelo (nome da tool + argumentos). |
| **Observação** | O resultado da tool, devolvido ao modelo como novo turno. |
| **`stop_reason`** | Sinal do modelo: `tool_use` = quero tool (continua); `end_turn` = terminei (para). |
| **`MAX_ITERACOES`** | Trava de segurança: número máximo de voltas antes de parar à força. |
| **Custo quadrático** | Como o contexto acumula e é reenviado, o custo cresce com ~n² nas iterações. |

> **Resumo em uma frase:** o **loop de agente (ReAct)** é o ciclo
> **pensar → agir → observar → repetir** que transforma um LLM em um agente que usa
> ferramentas — cada volta é uma chamada paga com contexto crescente, então ele **sempre**
> precisa de uma trava `MAX_ITERACOES`.

---

## 9. Próximo na série

➡️ **[`02-agente-unico.md`](02-agente-unico.md)** — como um **agente único** usa esse loop
com um conjunto de tools para resolver uma tarefa inteira, quando ele é a escolha certa e
quais são seus limites.
