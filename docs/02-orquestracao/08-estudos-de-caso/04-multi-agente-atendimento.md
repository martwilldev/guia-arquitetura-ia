# Estudo de Caso 4 — Atendimento Complexo Multi-Agente

> **Cenário:** central de atendimento que precisa **triar** o pedido, acionar **especialistas** (cobrança, técnico, comercial) e **consolidar** uma resposta única. O foco deste caso é decidir **quando vale a pena** pagar o custo (3–10×) de uma arquitetura multi-agente **versus** um único agente bem projetado.

> 🔙 Volte ao índice dos estudos de caso: [README.md](./README.md)

---

## Índice

1. [Problema, produto e requisitos](#1-problema-produto-e-requisitos)
2. [Arquitetura de referência](#2-arquitetura-de-referência)
3. [O custo escondido da coordenação multi-agente](#3-o-custo-escondido-da-coordenação-multi-agente)
4. [Opção A — Agente único + muitas tools + roteamento de modelo 🟡](#4-opção-a--agente-único--muitas-tools--roteamento-de-modelo-)
5. [Opção B — Supervisor + especialistas (CrewAI/LangGraph em Fargate) 🔴](#5-opção-b--supervisor--especialistas-crewailanggraph-em-fargate-)
6. [Opção C — Bedrock multi-agent collaboration (gerenciado) 🟡](#6-opção-c--bedrock-multi-agent-collaboration-gerenciado-)
7. [Tabela comparativa](#7-tabela-comparativa)
8. [Recomendação final por perfil](#8-recomendação-final-por-perfil)
9. [Armadilhas](#9-armadilhas)
10. [Glossário](#10-glossário)
11. [Resumo em uma frase](#11-resumo-em-uma-frase)

---

## 1. Problema, produto e requisitos

Uma empresa de telecom quer um assistente de atendimento que resolva pedidos que **cruzam domínios**: um cliente pergunta "por que minha fatura subiu e minha internet está lenta?" — isso mistura **cobrança** (analisar fatura) e **suporte técnico** (diagnóstico de rede). O sistema deve triar, buscar dados em cada domínio e devolver **uma** resposta coerente.

| Requisito | Alvo |
|-----------|------|
| **Volume** | ~50.000 conversas/mês (~1.700/dia) |
| **Turnos por conversa** | 3–6 mensagens (média 4) |
| **Latência** | < 6 s por turno (tolerável até 10 s em casos complexos) |
| **Qualidade** | Resposta correta e "single voice" (sem contradição entre domínios); handoff para humano quando incerto |
| **Domínios** | Cobrança, Técnico, Comercial/Retenção |
| **Restrições** | Auditoria (quem decidiu o quê), custo previsível, LGPD |

> ⚠️ **Insight central:** nem todo problema "que parece multi-agente" precisa de multi-agente. Um único agente com boas *tools* resolve a maioria. Multi-agente só se paga quando os domínios têm **raciocínio próprio profundo** e/ou **prompts/contextos grandes demais para caber num agente só**.

---

## 2. Arquitetura de referência

```
                          ┌───────────────────────┐
        Cliente  ───────► │  API Gateway / WS      │
                          └───────────┬────────────┘
                                      │
                          ┌───────────▼────────────┐
                          │   ORQUESTRADOR (Lambda) │
                          │   - carrega histórico   │
                          │   - TRIAGEM             │
                          └───────────┬────────────┘
                       ┌──────────────┼──────────────┐
                       ▼              ▼              ▼
                ┌────────────┐ ┌────────────┐ ┌────────────┐
                │ COBRANÇA   │ │  TÉCNICO   │ │ COMERCIAL  │
                │ (especial.)│ │ (especial.)│ │ (especial.)│
                └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
                      │              │              │
                      └──────────────┼──────────────┘
                                     ▼
                          ┌───────────────────────┐
                          │  CONSOLIDAÇÃO (síntese)│
                          │  "single voice"        │
                          └───────────┬────────────┘
                                      ▼
                          ┌───────────────────────┐
                          │ Memória: DynamoDB      │
                          │ Dados: APIs internas   │
                          │ Bedrock (LLM)          │
                          └───────────────────────┘
```

A diferença entre as opções está em **como** as caixas TRIAGEM → ESPECIALISTAS → CONSOLIDAÇÃO são implementadas: tudo num prompt (A), como agentes separados que conversam (B), ou orquestrado pela AWS (C).

---

## 3. O custo escondido da coordenação multi-agente

Quando você quebra o problema em N agentes, cada "passada" no LLM **re-lê o contexto** (histórico + resultado dos outros agentes). O custo de tokens **não soma**, ele **multiplica**:

```
Agente único:   1 chamada  → lê o contexto 1x
Multi-agente:   supervisor + 3 especialistas + consolidação = 5 chamadas
                cada uma relendo (parte do) contexto  →  ~3x a 10x mais tokens
```

Exemplo intuitivo (mesma conversa):

| Padrão | Chamadas ao LLM | Tokens lidos (aprox.) | Fator |
|--------|-----------------|-----------------------|-------|
| Agente único | 1–2 | 1× | 1× |
| Supervisor + 2 especialistas + síntese | 4–5 | 3–4× | ~3–4× |
| Supervisor + 3 especialistas + debate/verificação | 6–10 | 6–10× | ~6–10× |

> 💡 **Regra de bolso:** só adote multi-agente quando o ganho de **qualidade/manutenibilidade** compensar um custo por conversa **3–10× maior**. Caso contrário, agente único + roteamento de modelo é mais barato e mais rápido.

Todos os valores abaixo são **ILUSTRATIVOS** para raciocínio comparativo. Confirme na [Calculadora AWS](https://calculator.aws) e nos [preços do Amazon Bedrock](https://aws.amazon.com/bedrock/pricing/).

---

## 4. Opção A — Agente único + muitas tools + roteamento de modelo 🟡

### Como funciona

Um **único** agente (loop ReAct) recebe a conversa. Ele tem **tools** para cada domínio (`consultar_fatura`, `diagnosticar_rede`, `checar_planos`) e um **roteador de modelo**: usa um modelo barato (ex.: Claude Haiku / Nova Lite) para triagem e turnos simples, e escala para um modelo forte (Claude Sonnet) só quando a tarefa é complexa. Não há "vários agentes" — há **um cérebro com muitas ferramentas**.

```
Triagem (modelo barato) → decide tools → chama APIs → responde
                        └── se complexo → escala p/ modelo forte
```

### Exemplo de código (Lambda-ready)

```python
# handler.py — cliente FORA do handler (reuso entre invocações)
import json
import boto3

bedrock = boto3.client("bedrock-runtime")

MODEL_CHEAP = "us.amazon.nova-lite-v1:0"
MODEL_STRONG = "us.anthropic.claude-3-5-sonnet-20241022-v2:0"

TOOLS = [
    {"name": "consultar_fatura", "description": "Detalhes da fatura do cliente",
     "input_schema": {"type": "object", "properties": {"conta_id": {"type": "string"}}}},
    {"name": "diagnosticar_rede", "description": "Diagnóstico técnico da conexão",
     "input_schema": {"type": "object", "properties": {"conta_id": {"type": "string"}}}},
    {"name": "checar_planos", "description": "Planos e ofertas disponíveis",
     "input_schema": {"type": "object", "properties": {"perfil": {"type": "string"}}}},
]

def _roteia_modelo(texto: str) -> str:
    # Roteamento simples por heurística; poderia ser um classificador
    gatilhos = ("por que", "não funciona", "reclamação", "cancelar", "erro")
    return MODEL_STRONG if any(g in texto.lower() for g in gatilhos) else MODEL_CHEAP

def _executa_tool(nome, args):
    # Aqui entram as APIs internas reais (billing, NOC, CRM)
    return {"consultar_fatura": {"valor": 189.9, "aumento": "+30 (roaming)"},
            "diagnosticar_rede": {"status": "degradado", "causa": "CGNAT"},
            "checar_planos": {"oferta": "Fibra 500MB por R$120"}}.get(nome, {})

def handler(event, _ctx):
    msgs = json.loads(event["body"])["messages"]
    model = _roteia_modelo(msgs[-1]["content"][0]["text"])
    for _ in range(5):  # loop de agente com teto
        resp = bedrock.converse(
            modelId=model,
            messages=msgs,
            toolConfig={"tools": [{"toolSpec": t} for t in TOOLS]},
        )
        out = resp["output"]["message"]
        msgs.append(out)
        tool_uses = [c for c in out["content"] if "toolUse" in c]
        if not tool_uses:
            break
        results = []
        for c in tool_uses:
            tu = c["toolUse"]
            results.append({"toolResult": {
                "toolUseId": tu["toolUseId"],
                "content": [{"json": _executa_tool(tu["name"], tu["input"])}],
            }})
        msgs.append({"role": "user", "content": results})
    return {"statusCode": 200, "body": json.dumps(out["content"][-1].get("text", ""))}
```

### Custo estimado mensal (ILUSTRATIVO)

Premissa: 50k conversas × 4 turnos = 200k turnos. ~70% simples (modelo barato), ~30% escalam (modelo forte). Média por turno com tools: ~2k tokens in + 600 out.

| Item | Conta | Custo/mês |
|------|-------|-----------|
| Tokens modelo barato (Nova Lite) | 140k turnos × 2.6k tok ≈ 364M tok @ ~US$0,06–0,24/M | ~US$ 35 |
| Tokens modelo forte (Sonnet) | 60k turnos × 2.6k tok ≈ 156M tok @ ~US$3 in / US$15 out | ~US$ 550 |
| Compute (Lambda) | 200k invocações × ~4 s × 512MB | ~US$ 15 |
| DynamoDB (memória) | on-demand, baixo | ~US$ 10 |
| **Total** | | **~US$ 610/mês** |

> Verifique preços atualizados no [Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/). Nova Lite/Haiku são baratíssimos; Sonnet domina a conta.

### Complexidade: 🟡 média

### Trade-offs

- ✅ **Mais barato** (sem multiplicação de contexto entre agentes).
- ✅ Latência menor (1–2 chamadas ao LLM por turno).
- ✅ Um só lugar para depurar/observar.
- ⚠️ Prompt cresce e fica difícil de manter quando os domínios são muito distintos.
- ⚠️ Sem isolamento: uma tool ruim pode "poluir" o raciocínio.

---

## 5. Opção B — Supervisor + especialistas (CrewAI/LangGraph em Fargate) 🔴

### Como funciona

Um **supervisor** faz triagem e delega a **agentes especialistas** independentes (cada um com seu prompt, suas tools, possivelmente seu modelo). Um passo de **consolidação** funde as respostas em "single voice". Como o grafo pode ser longo e stateful, roda em **Fargate** (container sempre pronto) em vez de Lambda (evita timeout de 15 min e cold starts em fluxos longos).

```
        LangGraph (grafo de estado) em Fargate
        supervisor ──► cobranca ──┐
                   ──► tecnico  ──┤──► consolidacao ──► resposta
                   ──► comercial ─┘
```

### Exemplo de config (LangGraph — esboço)

```python
# graph.py — construído uma vez no boot do container (fora do request)
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

class Estado(TypedDict):
    historico: List[dict]
    dominios: List[str]         # decididos pelo supervisor
    achados: dict               # resultado de cada especialista
    resposta: str

def supervisor(s: Estado) -> Estado:
    # LLM decide QUAIS especialistas acionar
    s["dominios"] = decidir_dominios(s["historico"])   # ex.: ["cobranca","tecnico"]
    return s

def especialista_cobranca(s): s["achados"]["cobranca"] = analisar_fatura(s); return s
def especialista_tecnico(s):  s["achados"]["tecnico"]  = diagnosticar(s);   return s
def consolidar(s):            s["resposta"] = sintetizar(s["achados"]);      return s

g = StateGraph(Estado)
g.add_node("supervisor", supervisor)
g.add_node("cobranca", especialista_cobranca)
g.add_node("tecnico", especialista_tecnico)
g.add_node("consolidar", consolidar)
g.set_entry_point("supervisor")
# roteamento condicional: só ativa os nós dos domínios escolhidos
g.add_conditional_edges("supervisor", lambda s: s["dominios"])
g.add_edge("cobranca", "consolidar")
g.add_edge("tecnico", "consolidar")
g.add_edge("consolidar", END)
APP = g.compile()   # reusado entre requests
```

### Custo estimado mensal (ILUSTRATIVO)

Premissa: 50k conversas. Nem toda conversa aciona todos; média ~2,5 especialistas + supervisor + consolidação ≈ **4,5 chamadas** ao LLM por conversa complexa. Assuma metade das conversas simples (roteadas direto). Modelo predominante: Sonnet.

| Item | Conta | Custo/mês |
|------|-------|-----------|
| Tokens (multiplicação ~3,5×) | ~50k conv × ~4,5 chamadas × ~2,5k tok ≈ 560M tok @ mix Sonnet | ~US$ 2.000 |
| Compute (Fargate) | 2 tasks × 1 vCPU/2GB × 730h (sempre-ligado p/ latência) | ~US$ 70 |
| Load balancer (ALB) | 1 ALB | ~US$ 20 |
| DynamoDB + logs | | ~US$ 25 |
| **Total** | | **~US$ 2.115/mês** |

> A conta de tokens (~3–4×) é o que domina. Fargate acrescenta custo fixo de infra que Lambda não tem.

### Complexidade: 🔴 alta

### Trade-offs

- ✅ **Melhor qualidade** em domínios profundos e distintos (cada especialista tem prompt/contexto dedicado).
- ✅ Isolamento e manutenção por domínio (times separados).
- ✅ Auditabilidade do grafo (quem decidiu o quê).
- ❌ **3–4× o custo de tokens** + infra Fargate fixa.
- ❌ Latência maior (várias passadas sequenciais).
- ❌ Mais superfície de falha e observabilidade complexa.

---

## 6. Opção C — Bedrock multi-agent collaboration (gerenciado) 🟡

### Como funciona

Você usa o **Amazon Bedrock Agents** com **multi-agent collaboration**: cria um **agente supervisor** e **agentes colaboradores** (especialistas) declarativamente no console/IaC. A AWS gerencia a orquestração, o roteamento entre agentes, memória de sessão e o "trace". Você não escreve o loop nem gerencia container — só define agentes, instruções e action groups (tools via Lambda).

```
   Bedrock Agent (supervisor, gerenciado)
        ├─► colaborador: Cobrança  (action group → Lambda)
        ├─► colaborador: Técnico   (action group → Lambda)
        └─► colaborador: Comercial (action group → Lambda)
   (orquestração, memória e trace = responsabilidade da AWS)
```

### Exemplo de config (IaC — esboço)

```python
# bootstrap_agents.py — provisionamento (roda no deploy, não por request)
import boto3
agent = boto3.client("bedrock-agent")

# 1) cria colaboradores (um por domínio) — cada um aponta p/ Lambda de tools
#    aqui só o esqueleto; parâmetros completos no doc da API
sup = agent.create_agent(
    agentName="atendimento-supervisor",
    foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
    instruction="Você tria o pedido e delega aos colaboradores; consolide em UMA resposta.",
    agentCollaboration="SUPERVISOR",  # habilita multi-agente gerenciado
)
# associar colaboradores (cobranca/tecnico/comercial) via associate_agent_collaborator(...)
# invocação em runtime: bedrock-agent-runtime.invoke_agent(...)
```

```python
# runtime_handler.py — invocação em produção
import boto3
rt = boto3.client("bedrock-agent-runtime")

def handler(event, _ctx):
    r = rt.invoke_agent(
        agentId=event["agentId"], agentAliasId=event["aliasId"],
        sessionId=event["sessionId"], inputText=event["texto"],
    )
    chunks = [e["chunk"]["bytes"].decode() for e in r["completion"] if "chunk" in e]
    return {"statusCode": 200, "body": "".join(chunks)}
```

### Custo estimado mensal (ILUSTRATIVO)

O modelo de cobrança é **por tokens** dos modelos usados (supervisor + colaboradores) — a AWS **não cobra taxa de orquestração à parte**, mas a coordenação **consome mais tokens** (semelhante à Opção B). Compute de tools = Lambda.

| Item | Conta | Custo/mês |
|------|-------|-----------|
| Tokens (multiplicação ~3×, gerenciada) | ~50k conv × ~4 chamadas × ~2,5k tok ≈ 500M tok @ mix Sonnet | ~US$ 1.750 |
| Compute (Lambda das tools) | ~200k invocações | ~US$ 15 |
| DynamoDB/logs | memória gerenciada + app | ~US$ 15 |
| **Total** | | **~US$ 1.780/mês** |

> Confirme se há custo de memória/sessão e recursos associados no [Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/). O grosso continua sendo tokens.

### Complexidade: 🟡 (baixa a média para operar; a AWS carrega a orquestração)

### Trade-offs

- ✅ **Sem infra para gerenciar** (nem loop, nem Fargate).
- ✅ Trace/observabilidade nativos; escala automática.
- ✅ Multi-agente "de verdade" sem escrever o orquestrador.
- ⚠️ Custo de tokens ainda ~3× (é multi-agente).
- ⚠️ Menos controle fino sobre o grafo do que LangGraph.
- ⚠️ Acoplamento ao Bedrock (portabilidade menor).

---

## 7. Tabela comparativa

| Critério | A — Agente único 🟡 | B — Supervisor+especialistas (Fargate) 🔴 | C — Bedrock multi-agente 🟡 |
|----------|---------------------|-------------------------------------------|-----------------------------|
| **Custo/mês (ilustrativo)** | ~US$ 610 | ~US$ 2.115 | ~US$ 1.780 |
| **Complexidade** | 🟡 média | 🔴 alta | 🟡 média (operar) |
| **Latência/turno** | 🟢 baixa (1–2 chamadas) | 🔴 alta (4–10 chamadas) | 🟡 média-alta |
| **Qualidade em domínios profundos** | 🟡 boa (limitada por prompt único) | 🟢 melhor (isolamento total) | 🟢 boa (isolamento gerenciado) |
| **Manutenção por time** | 🔴 tudo num lugar | 🟢 por domínio | 🟢 por agente |
| **Portabilidade** | 🟢 alta | 🟢 alta (código próprio) | 🔴 acoplado à AWS |
| **Quando escolher** | Domínios sobrepostos, custo/latência mandam | Domínios muito distintos, times separados, qualidade crítica | Quer multi-agente sem operar infra |

---

## 8. Recomendação final por perfil

- **MVP / validação:** **Opção A**. Comece com **um** agente e tools por domínio + roteamento de modelo. Você provavelmente descobre que 80% dos casos não precisavam de multi-agente. Mais barato, mais rápido, mais fácil de depurar.
- **Crescimento:** **Opção C** (Bedrock multi-agent). Quando a qualidade em domínios distintos começar a doer e você não quiser operar Fargate, migre para multi-agente gerenciado — paga o custo de tokens, mas não o de infra/engenharia.
- **Escala / requisitos fortes:** **Opção B** (LangGraph/CrewAI em Fargate) quando você tem **times separados por domínio**, precisa de **controle fino** do grafo, verificação/debate entre agentes, e o volume justifica engenharia dedicada.

> Caminho evolutivo saudável: **A → C → B**, subindo de nível só quando a dor de qualidade superar o custo extra.

---

## 9. Armadilhas

- **Multi-agente "porque é legal":** o maior erro. Começa complexo, gasta 3–10× em tokens, e o resultado não é melhor que um agente único bem feito.
- **Esquecer o teto do loop:** agentes que se chamam sem limite geram loops caros. Sempre `max_iterations` e circuit breaker.
- **Contexto duplicado:** reenviar o histórico inteiro para cada especialista explode a conta. Passe **resumos**, não a conversa crua.
- **Latência sequencial:** 5 chamadas em série = 5× a latência. Paralelize especialistas independentes quando possível.
- **"Single voice" quebrada:** sem um passo de consolidação forte, os especialistas se contradizem. A síntese é obrigatória.
- **Lambda em fluxo longo:** grafos multi-agente podem estourar 15 min / cold start. Se for longo e stateful, use Fargate/Step Functions.
- **Não medir por conversa:** meça **custo por conversa resolvida**, não por token, para comparar A/B/C honestamente.

---

## 10. Glossário

| Termo | Significado |
|-------|-------------|
| **Supervisor** | Agente que tria e delega tarefas a especialistas, depois consolida. |
| **Especialista** | Agente focado em um domínio (prompt/tools próprias). |
| **Single voice** | Resposta final única e coerente, sem contradição entre agentes. |
| **Multiplicação de contexto** | Custo extra por reenviar contexto a cada agente/passada. |
| **LangGraph** | Framework para orquestrar agentes como grafo de estado. |
| **CrewAI** | Framework de agentes colaborativos com papéis/tarefas. |
| **Multi-agent collaboration** | Recurso gerenciado do Bedrock Agents para supervisor + colaboradores. |
| **Fargate** | Container serverless (sem gerenciar servidores), bom p/ fluxos longos/stateful. |

---

## 11. Resumo em uma frase

> **Comece com um agente único + tools + roteamento de modelo; só adote multi-agente (Bedrock gerenciado ou LangGraph/CrewAI em Fargate) quando domínios profundos e distintos justificarem pagar 3–10× mais em tokens por conversa.**
