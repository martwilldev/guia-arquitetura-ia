# Multi-Agente — Vários Agentes Especializados Colaborando

> Guia didático do padrão **multi-agente**: quando você tem **vários agentes persistentes**, cada um
> com um **papel** definido, que **colaboram** (delegam, conversam ou passam trabalho adiante) para
> resolver uma tarefa. Contexto prático: **AWS Bedrock + Lambda**.
>
> Este documento foca **apenas em multi-agente**. A questão de **quando você precisa de mais de um
> orquestrador** (vários "gerentes" coordenando domínios diferentes) fica para o
> [próximo arquivo (05)](./05-multiplos-orquestradores.md).

---

## Índice

1. [O que é multi-agente (e como difere de sub-agentes)](#1-o-que-é-multi-agente-e-como-difere-de-sub-agentes)
2. [Os três padrões de multi-agente](#2-os-três-padrões-de-multi-agente)
   - [2.1 Supervisor (orquestrador–trabalhadores)](#21-supervisor-orquestradortrabalhadores)
   - [2.2 Rede (agentes conversam entre si)](#22-rede-agentes-conversam-entre-si)
   - [2.3 Pipeline (cada agente uma etapa)](#23-pipeline-cada-agente-uma-etapa)
3. [Custo e latência do multi-agente](#3-custo-e-latência-do-multi-agente)
4. [Exemplo Python: supervisor Lambda-ready (Bedrock)](#4-exemplo-python-supervisor-lambda-ready-bedrock)
5. [Ferramentas e opções de mercado](#5-ferramentas-e-opções-de-mercado)
6. [Onde rodar multi-agente na AWS](#6-onde-rodar-multi-agente-na-aws)
7. [Boas práticas e armadilhas](#7-boas-práticas-e-armadilhas)
8. [Glossário](#8-glossário)
9. [Resumo em uma frase](#9-resumo-em-uma-frase)
10. [Anterior / Próximo](#10-anterior--próximo)

---

## 1. O que é multi-agente (e como difere de sub-agentes)

**Multi-agente** é quando você tem **vários agentes** — cada um um LLM dentro do seu próprio loop
(pensar → agir → observar) — com **papéis especializados**, colaborando para atingir um objetivo.

Pense em uma **equipe**: um pesquisador, um redator, um revisor. Cada um tem uma função, mantém seu
próprio contexto e "conversa" com os outros para produzir o resultado final.

### Diferença crucial: sub-agente vs multi-agente

Já vimos **sub-agentes** no [documento anterior (03)](./03-sub-agentes.md). A confusão é comum, então
vale separar bem:

| Aspecto | Sub-agente (doc 03) | Multi-agente (este doc) |
|---------|---------------------|--------------------------|
| **Natureza** | Efêmero: criado para **uma** sub-tarefa e descartado | Persistente: tem **papel fixo** e vive durante toda a colaboração |
| **Papel** | Genérico ("resolva este pedaço") | Especializado ("você é o revisor jurídico") |
| **Comunicação** | Só devolve o resultado ao "pai" | Pode **conversar** com outros agentes, trocar mensagens em rodadas |
| **Quem decide o fluxo** | O agente pai decide tudo | Pode ser um supervisor, ou os próprios agentes negociando |
| **Analogia** | Estagiário contratado para um recado | Membro fixo de uma equipe multidisciplinar |

```
   SUB-AGENTES (efêmeros)                 MULTI-AGENTE (persistentes)
   ─────────────────────                  ────────────────────────────
        AGENTE PAI                          ┌──────────┐   ┌──────────┐
            │                               │ Pesquisa │◄─►│  Redação │
     ┌──────┼──────┐                        └────┬─────┘   └────┬─────┘
     ▼      ▼      ▼                              │              │
   sub    sub    sub   ◄─ some após                └──────┬───────┘
  (some) (some) (some)    entregar                        ▼
                                                    ┌──────────┐
   fluxo top-down, um nível                         │  Revisão │  papéis fixos,
                                                    └──────────┘  podem conversar
```

> **Regra prática:** se os "trabalhadores" têm **identidade e papel estável** e/ou precisam
> **trocar mensagens entre si**, você está em multi-agente. Se são só "braços" descartáveis de
> um agente, são sub-agentes.

**Quando multi-agente faz sentido:** tarefas que se beneficiam de **especialização** (cada papel com
seu prompt/tools/modelo), de **revisão cruzada** (um agente critica o outro), ou de **etapas distintas
e reutilizáveis** (pesquisar → escrever → revisar → publicar).

---

## 2. Os três padrões de multi-agente

Existem três topologias fundamentais. A maioria dos sistemas reais é uma combinação delas.

| Padrão | Como flui | Quem controla | Melhor para |
|--------|-----------|----------------|-------------|
| **Supervisor** | Estrela: um centro delega e consolida | O supervisor | Tarefas com "chefe" que decide e junta |
| **Rede** | Malha: todos podem falar com todos | Emergente (negociação) | Brainstorm, debate, revisão cruzada |
| **Pipeline** | Linha: A → B → C em sequência | O desenho fixo do fluxo | Etapas bem definidas e ordenadas |

### 2.1 Supervisor (orquestrador–trabalhadores)

Um **agente supervisor** recebe a tarefa, decide **qual especialista** aciona, coleta os resultados e
**consolida** a resposta final. É o padrão mais comum e mais fácil de controlar.

```
                       ┌─────────────────────┐
        tarefa ───────►│     SUPERVISOR      │───────► resposta final
                       │ (decide + consolida)│
                       └──────────┬──────────┘
             ┌────────────────────┼────────────────────┐
             ▼                    ▼                    ▼
     ┌──────────────┐    ┌──────────────┐     ┌──────────────┐
     │  Agente A    │    │  Agente B    │     │  Agente C    │
     │ (pesquisa)   │    │  (cálculo)   │     │  (redação)   │
     └──────────────┘    └──────────────┘     └──────────────┘
             │                    │                    │
             └────────────────────┴────────────────────┘
                        resultados voltam ao supervisor
```

**Quando usar:**
- Há uma decisão clara de "quem faz o quê" e alguém precisa **juntar** as partes.
- Você quer **controle** e **previsibilidade** (o supervisor limita rodadas, valida saídas).
- A tarefa é decomponível em especialidades independentes.

**Cuidado:** o supervisor "pensa" a cada passo (uma chamada de LLM por decisão), então ele é o maior
gerador de custo. Use um **modelo barato** no supervisor sempre que possível.

### 2.2 Rede (agentes conversam entre si)

Não há um chefe fixo. Os agentes **trocam mensagens** em uma espécie de **group chat**, cada um
contribuindo quando seu papel é relevante, até chegarem a um consenso ou a um critério de parada.

```
        ┌──────────┐          ┌──────────┐
        │ Agente A │◄────────►│ Agente B │
        │ (crítico)│          │ (autor)  │
        └────┬─────┘          └─────┬────┘
             │      group chat      │
             │   (mensagens vão e   │
             ▼    vêm em rodadas)   ▼
        ┌──────────┐          ┌──────────┐
        │ Agente C │◄────────►│ Agente D │
        │(pesquisa)│          │ (revisor)│
        └──────────┘          └──────────┘
             └───────────► consenso / parada ► resposta
```

**Quando usar:**
- Problemas **abertos** que se beneficiam de debate, crítica mútua ou múltiplas perspectivas.
- Geração + crítica iterativa (autor propõe, crítico aponta falhas, autor refina).

**Cuidado:** é o padrão **mais caro e imprevisível**. A conversa pode **explodir em rodadas** (cada
rodada = várias chamadas de LLM). **Sempre** defina limite de rodadas e critério de parada explícito.

### 2.3 Pipeline (cada agente uma etapa)

Fluxo **sequencial fixo**: a saída de um agente é a entrada do próximo. Cada agente é uma **estação**
de uma linha de montagem.

```
   entrada
     │
     ▼
  ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
  │ Agente 1   │──►│ Agente 2   │──►│ Agente 3   │──►│ Agente 4   │──► saída
  │ (extrair)  │   │(transformar)│  │ (redigir)  │   │ (revisar)  │
  └────────────┘   └────────────┘   └────────────┘   └────────────┘
     etapa 1          etapa 2          etapa 3          etapa 4
```

**Quando usar:**
- As etapas são **conhecidas e ordenadas** (ETL de texto, doc → resumo → tradução → formatação).
- Você quer o fluxo **mais barato e previsível** dentre os multi-agente (sem "supervisor pensando"
  a cada passo, sem conversa em rodadas).

**Cuidado:** é rígido. Se a tarefa exige decisões dinâmicas de roteamento, o pipeline puro não basta —
combine com um supervisor no início.

---

## 3. Custo e latência do multi-agente

> Números aqui são **ordens de grandeza ilustrativas** para raciocínio comparativo, **não** medições.
> A análise aprofundada (com throughput e contas) está no [documento 06](./06-custo-latencia-throughput.md).

Multi-agente **coordena**, e coordenação **custa**. As duas fontes principais:

1. **O supervisor pensa a cada passo.** Cada decisão de "quem chamo agora?" é uma chamada de LLM
   adicional, que **não existia** no agente único. Se ele decide 5 vezes, são 5 chamadas só de
   coordenação — antes de qualquer trabalho útil.
2. **A conversa pode explodir em rodadas.** No padrão rede, N agentes por R rodadas ≈ **N × R**
   chamadas de LLM. Sem limite, isso cresce rápido e pode virar **loop infinito** de "concordo/discordo".

### Noção de grandeza (ilustrativo)

| Métrica | Agente único | Multi-agente (típico) | Por quê |
|---------|--------------|------------------------|---------|
| **Custo (tokens)** | 1× (base) | **3×–10×** | Coordenação + contexto repassado + várias chamadas |
| **Latência** | 1× (base) | **2×–5×** | Passos sequenciais somam; rede em rodadas soma mais ainda |

```
   Custo relativo (ilustrativo)
   Agente único   ▇▇▇                      1×
   Pipeline       ▇▇▇▇▇▇                   ~2–3×  (etapas fixas, sem "pensar" extra)
   Supervisor     ▇▇▇▇▇▇▇▇▇                ~3–6×  (supervisor decide a cada passo)
   Rede (chat)    ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇          ~5–10× (N agentes × R rodadas)
```

**Leitura rápida:** pipeline é o mais econômico dos três; rede é o mais caro. O supervisor fica no
meio, e seu custo depende muito de quantas rodadas ele executa e do modelo que usa para decidir.

---

## 4. Exemplo Python: supervisor Lambda-ready (Bedrock)

Um **supervisor mínimo** que: (1) classifica a intenção da pergunta, (2) delega para 1–2 **agentes
especialistas** (funções), e (3) **consolida** a resposta. Sem framework — só `boto3` + Bedrock —
para caber bem em uma **AWS Lambda** curta.

```python
import json
import boto3

bedrock = boto3.client("bedrock-runtime")

# Modelos: barato para coordenar, capaz para o trabalho pesado
MODEL_SUPERVISOR = "anthropic.claude-3-haiku-20240307-v1:0"   # barato → decisões
MODEL_ESPECIALISTA = "anthropic.claude-3-5-sonnet-20240620-v1:0"  # capaz → conteúdo


def _chamar_bedrock(model_id: str, system: str, user: str, max_tokens: int = 800) -> str:
    """Chamada única ao Bedrock (Messages API da Anthropic)."""
    resp = bedrock.invoke_model(
        modelId=model_id,
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": max_tokens,
            "system": system,
            "messages": [{"role": "user", "content": user}],
        }),
    )
    payload = json.loads(resp["body"].read())
    return payload["content"][0]["text"].strip()


# ---------- AGENTES ESPECIALISTAS (papéis fixos) ----------

def agente_pesquisa(pergunta: str) -> str:
    system = "Você é um pesquisador. Levante fatos e contexto relevantes, de forma objetiva."
    return _chamar_bedrock(MODEL_ESPECIALISTA, system, pergunta)


def agente_calculo(pergunta: str) -> str:
    system = ("Você é um analista quantitativo. Resolva cálculos e estimativas numéricas, "
              "mostrando o raciocínio de forma sucinta.")
    return _chamar_bedrock(MODEL_ESPECIALISTA, system, pergunta)


def agente_redacao(pergunta: str) -> str:
    system = "Você é um redator. Produza um texto claro e bem estruturado."
    return _chamar_bedrock(MODEL_ESPECIALISTA, system, pergunta)


ESPECIALISTAS = {
    "pesquisa": agente_pesquisa,
    "calculo": agente_calculo,
    "redacao": agente_redacao,
}


# ---------- SUPERVISOR ----------

def supervisor_classifica(pergunta: str) -> list[str]:
    """Supervisor (modelo barato) decide QUAIS especialistas acionar."""
    system = (
        "Você é um supervisor que despacha tarefas. "
        "Dado o pedido do usuário, responda APENAS com um JSON no formato "
        '{"agentes": ["pesquisa"|"calculo"|"redacao", ...]}. '
        "Escolha de 1 a 2 agentes, os mais relevantes."
    )
    bruto = _chamar_bedrock(MODEL_SUPERVISOR, system, pergunta, max_tokens=100)
    try:
        agentes = json.loads(bruto).get("agentes", [])
    except json.JSONDecodeError:
        agentes = ["redacao"]  # fallback seguro
    # valida contra os especialistas conhecidos e limita a 2 (controla custo)
    return [a for a in agentes if a in ESPECIALISTAS][:2] or ["redacao"]


def supervisor_consolida(pergunta: str, contribuicoes: dict[str, str]) -> str:
    """Supervisor junta as contribuições em uma resposta final coesa."""
    partes = "\n\n".join(f"[{papel}]\n{texto}" for papel, texto in contribuicoes.items())
    system = ("Você é um supervisor. Consolide as contribuições dos especialistas em "
              "UMA resposta final, coesa e sem repetições.")
    user = f"Pergunta original:\n{pergunta}\n\nContribuições dos especialistas:\n{partes}"
    return _chamar_bedrock(MODEL_SUPERVISOR, system, user, max_tokens=800)


def resolver(pergunta: str) -> dict:
    """Fluxo supervisor: classifica → delega → consolida."""
    escolhidos = supervisor_classifica(pergunta)          # 1 chamada (barata)
    contribuicoes = {p: ESPECIALISTAS[p](pergunta) for p in escolhidos}  # N chamadas
    final = supervisor_consolida(pergunta, contribuicoes)  # 1 chamada (barata)
    return {"agentes_usados": escolhidos, "resposta": final}


# ---------- HANDLER LAMBDA ----------

def lambda_handler(event, context):
    pergunta = (event or {}).get("pergunta", "")
    if not pergunta:
        return {"statusCode": 400, "body": json.dumps({"erro": "campo 'pergunta' ausente"})}
    resultado = resolver(pergunta)
    return {"statusCode": 200, "body": json.dumps(resultado, ensure_ascii=False)}
```

**Contagem de chamadas neste exemplo:** `1 (classificar) + N (especialistas, N≤2) + 1 (consolidar)`
= 3 a 4 chamadas de LLM por request. Compare com **1** chamada do agente único — é daí que sai o
multiplicador de custo da [seção 3](#3-custo-e-latência-do-multi-agente).

> **Por que cabe em Lambda:** o fluxo é **curto e limitado** (sem loop de conversa, no máximo ~4
> chamadas). Se você adicionar rodadas de debate (padrão rede) ou muitos passos, o tempo pode
> ultrapassar os **15 min** da Lambda — veja a [seção 6](#6-onde-rodar-multi-agente-na-aws).

---

## 5. Ferramentas e opções de mercado

Você pode montar multi-agente "na mão" (como na seção 4) ou usar **frameworks** e **serviços
gerenciados**. Panorama:

### 5.1 Frameworks open-source / self-hosted

| Ferramenta | O que é | Padrão que favorece | Roda em Lambda? |
|-----------|---------|---------------------|-----------------|
| **LangGraph** | Orquestração de agentes como **grafo com estado**; nós = agentes, arestas = transições | Supervisor, Rede e Pipeline (o grafo modela qualquer um) | Sim para grafos curtos; loops longos → Fargate/Step Functions |
| **CrewAI** | Monta **"crews"** de agentes com **papéis, metas e tarefas** | Supervisor e Pipeline (processos sequenciais/hierárquicos) | Sim se a crew for curta; crews grandes → Fargate |
| **AutoGen / AG2** (Microsoft) | Framework de **conversação** entre agentes (group chat) | **Rede** (group chat) e Supervisor | Group chat longo tende a estourar 15 min → Fargate/Step Functions |
| **OpenAI Swarm / Agents SDK** | Leve, baseado em **handoffs** (um agente passa a vez a outro) | Supervisor e Pipeline (handoffs encadeados) | Sim para handoffs curtos |
| **Semantic Kernel Agent Framework** (Microsoft) | Agentes + orquestração dentro do SK, com plugins/skills | Supervisor e Pipeline | Sim para fluxos curtos |
| **LlamaIndex AgentWorkflow** | Workflows de agentes orientados a eventos; forte em RAG | Supervisor e Pipeline | Sim para workflows curtos |
| **Strands** (AWS, open-source) | SDK de agentes da AWS, integra bem com Bedrock; suporta agentes-como-tools (multi-agente) | Supervisor (agentes como ferramentas) | Sim, pensado para AWS/Lambda em fluxos curtos |

**Self-hosted por natureza:** todos acima. Você empacota a lib no seu deploy (Lambda layer/container,
ECS/Fargate, EC2). O modelo continua vindo do **Bedrock** (ou de outro provedor) — o framework só
**coordena** os agentes.

### 5.2 Gerenciados / na nuvem

| Serviço | O que é | Padrão que favorece | Self-hosted vs gerenciado |
|---------|---------|---------------------|---------------------------|
| **Amazon Bedrock — Multi-Agent Collaboration** | Um **agente supervisor gerenciado** coordena **agentes colaboradores** (subagentes), todos definidos no Bedrock Agents | **Supervisor** (nativo) | **Gerenciado** (AWS hospeda o loop/estado; você define agentes e instruções) |
| **Azure AI Agent Service** | Serviço gerenciado da Microsoft para criar e conectar agentes (integra com o ecossistema Azure/SK) | Supervisor e Rede | **Gerenciado** (SaaS Azure) |

**Diferença-chave:** com o **Bedrock Multi-Agent Collaboration** você **não gerencia o loop** — a AWS
executa a coordenação supervisor↔colaboradores por você (bom para não estourar limites de tempo de
uma Lambda que você mesmo escreveria). Com os frameworks self-hosted, **você controla tudo** (e
também é responsável por limites, retries e onde rodar).

> **Contexto do estudo:** para AWS, os caminhos naturais são **Strands** (self-hosted, SDK) e
> **Bedrock Multi-Agent Collaboration** (gerenciado). Os demais frameworks rodam por cima do Bedrock
> como provedor de modelo.

---

## 6. Onde rodar multi-agente na AWS

A escolha depende de **quanto tempo** o fluxo leva e de **quanta coordenação** ele exige.

| Onde | Ideal para | Limite/Nota |
|------|-----------|-------------|
| **AWS Lambda** | Fluxos **curtos**: supervisor com poucas chamadas, pipeline enxuto | Teto de **15 min** por execução; ótimo para o exemplo da seção 4 |
| **ECS / Fargate** | Fluxos **longos**: group chat com muitas rodadas, crews grandes, agentes que ficam "vivos" | Sem teto de 15 min; container roda o tempo necessário |
| **AWS Step Functions** | **Coordenação** de etapas: orquestrar pipeline/supervisor com estado, retries e paralelismo entre agentes | Cada etapa pode ser uma Lambda; a máquina de estados gerencia o fluxo longo sem estourar tempo |
| **Bedrock Multi-Agent Collaboration** | Deixar a **AWS coordenar** o supervisor↔colaboradores | Gerenciado; você não se preocupa com o loop nem com o tempo |

```
   Fluxo curto (≤ ~min)         Fluxo longo / muitas rodadas
   ────────────────────         ────────────────────────────
        Lambda                     ECS/Fargate  ou  Step Functions
   (barato, simples)              (sem teto)      (coordena etapas)

   Quer que a AWS coordene?  ──►  Bedrock Multi-Agent Collaboration (gerenciado)
```

**Regra de bolso:** comece na **Lambda**. Se a coordenação/conversa começar a se aproximar dos 15 min,
migre a orquestração para **Step Functions** (etapas em Lambdas) ou rode o agente em **Fargate**.

---

## 7. Boas práticas e armadilhas

### Boas práticas

- **Supervisor com modelo barato.** A coordenação é decisão simples (quem chamo? já terminei?). Use um
  modelo econômico (ex.: Haiku) no supervisor e reserve o modelo caro para o trabalho de conteúdo.
- **Limite rodadas e passos.** Defina um **máximo de rodadas** (rede) e de **passos** (supervisor).
  Sem isso, você corre risco de conversa infinita e custo descontrolado.
- **Papéis nítidos e prompts curtos.** Cada agente com um papel único e bem descrito; papéis
  sobrepostos geram retrabalho e discussão desnecessária.
- **Critério de parada explícito.** "Pare quando o revisor aprovar" ou "após no máximo 3 rodadas".
- **Não repasse contexto inteiro a todos.** Envie a cada agente só o que ele precisa (reduz tokens).
- **Comece simples.** Prefira **pipeline** ou **supervisor** antes de partir para **rede** (chat), que
  é o mais caro e difícil de controlar.

### Armadilhas

- **Multi-agente para tarefa simples.** Se um agente único resolve, multi-agente só adiciona custo,
  latência e pontos de falha. Não use "porque é legal".
- **Conversa infinita (rede).** Agentes que ficam "concordando/discordando" sem convergir. Sempre com
  limite de rodadas + critério de parada.
- **Supervisor caro.** Usar um modelo topo de linha só para despachar tarefas — desperdício.
- **Explosão de contexto.** Repassar todo o histórico para todos os agentes multiplica tokens
  silenciosamente.
- **Estourar o tempo da Lambda.** Loops longos passam de 15 min; migre para Fargate/Step Functions.

---

## 8. Glossário

| Termo | Significado |
|-------|-------------|
| **Multi-agente** | Vários agentes com papéis, colaborando para uma tarefa |
| **Agente** | LLM + loop (pensar→agir→observar) + tools + objetivo |
| **Papel (role)** | Especialidade fixa de um agente (pesquisador, revisor, etc.) |
| **Supervisor** | Agente que delega para outros e consolida o resultado |
| **Rede / group chat** | Agentes conversando entre si em rodadas, sem chefe fixo |
| **Pipeline** | Sequência fixa de agentes; a saída de um é a entrada do próximo |
| **Handoff** | Um agente "passa a vez" a outro (padrão de alguns frameworks) |
| **Rodada** | Um ciclo de troca de mensagens entre agentes (padrão rede) |
| **Critério de parada** | Condição que encerra a colaboração (aprovação, nº de rodadas) |
| **Colaborador** | No Bedrock, um agente subordinado ao supervisor gerenciado |

---

## 9. Resumo em uma frase

> **Multi-agente é uma equipe de agentes especializados que colaboram** — via **supervisor**
> (um chefe delega e junta), **rede** (todos conversam) ou **pipeline** (linha de montagem) —
> entregando mais qualidade em troca de **mais custo (~3–10×) e latência (~2–5×)**, o que exige
> **modelo barato no supervisor, limite de rodadas** e a infraestrutura AWS certa (Lambda para curto,
> Fargate/Step Functions para longo, ou Bedrock Multi-Agent Collaboration gerenciado).

---

## 10. Anterior / Próximo

| ◄ Anterior | Próximo ► |
|------------|-----------|
| [03 — Sub-agentes](./03-sub-agentes.md) | [05 — Múltiplos Orquestradores](./05-multiplos-orquestradores.md) |
