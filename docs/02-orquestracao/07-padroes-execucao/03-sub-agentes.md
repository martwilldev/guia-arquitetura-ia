# Sub-tarefas e Sub-agentes — Decomposição Top-Down

> Como quebrar **uma tarefa grande** em **partes menores**, e resolver cada parte com um
> **agente focado** (sub-agente). Foco em **decomposição top-down efêmera** — não em
> multi-agente colaborativo (isso é o [próximo documento](04-multi-agente.md)) nem em
> múltiplos orquestradores. Contexto **AWS Bedrock + Lambda**.

---

## Índice

1. [O que é decompor: sub-tarefa e sub-agente](#1-o-que-é-decompor-sub-tarefa-e-sub-agente)
2. [Sub-agente ≠ multi-agente](#2-sub-agente--multi-agente)
3. [Por que decompor: foco, paralelismo, reuso](#3-por-que-decompor-foco-paralelismo-reuso)
4. [O trade-off central: série vs paralelo](#4-o-trade-off-central-série-vs-paralelo)
5. [Exemplo Python — série (sub-tarefas dependentes)](#5-exemplo-python--série-sub-tarefas-dependentes)
6. [Exemplo Python — paralelo (fan-out / fan-in, Lambda-ready)](#6-exemplo-python--paralelo-fan-out--fan-in-lambda-ready)
7. [Ferramentas e opções de mercado](#7-ferramentas-e-opções-de-mercado)
8. [Padrão de consolidação (juntar resultados)](#8-padrão-de-consolidação-juntar-resultados)
9. [Boas práticas e armadilhas](#9-boas-práticas-e-armadilhas)
10. [Glossário](#10-glossário)
11. [Resumo em uma frase](#11-resumo-em-uma-frase)
12. [Anterior / Próximo](#12-anterior--próximo)

---

## 1. O que é decompor: sub-tarefa e sub-agente

**Decompor** é pegar uma tarefa grande e difícil e quebrá-la em **pedaços menores**, cada um
mais simples de resolver. É a mesma ideia de "dividir para conquistar" que você já usa ao
programar: uma função grande vira várias funções pequenas.

| Conceito | O que é | Analogia |
|----------|---------|----------|
| **Tarefa** | O objetivo completo do usuário | "Escreva um relatório sobre o produto X" |
| **Sub-tarefa** | Um pedaço menor e bem definido da tarefa | "Pesquise os concorrentes de X" |
| **Sub-agente** | Um **agente focado**, criado para resolver **uma** sub-tarefa, com prompt e tools enxutos, e que **some** depois de entregar o resultado | Um estagiário contratado só para aquela tarefa |

Um **sub-agente** é, tecnicamente, o mesmo que um agente (LLM + loop + tools + objetivo — veja
[02-agente-unico.md](02-agente-unico.md)), mas com três características:

1. **Escopo estreito** — resolve **uma** sub-tarefa, não a tarefa toda.
2. **Prompt e tools enxutos** — só recebe o que precisa para aquela sub-tarefa.
3. **Efêmero** — nasce para a sub-tarefa e é descartado quando termina (não é um "colega"
   permanente).

```
                    ┌──────────────────────────┐
   Tarefa grande ──►│      ORQUESTRADOR         │
                    │  (decompõe a tarefa)      │
                    └──────────┬────────────────┘
                               │
             ┌─────────────────┼─────────────────┐
             ▼                 ▼                 ▼
       ┌───────────┐     ┌───────────┐     ┌───────────┐
       │ SUB-AGENTE│     │ SUB-AGENTE│     │ SUB-AGENTE│  ◄── efêmeros,
       │ sub-tarefa│     │ sub-tarefa│     │ sub-tarefa│      focados
       │     A     │     │     B     │     │     C     │
       └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
             └─────────────────┼─────────────────┘
                               ▼
                    ┌──────────────────────────┐
                    │  CONSOLIDAÇÃO (junta A+B+C)│
                    └──────────┬────────────────┘
                               ▼
                        Resposta final
```

> O orquestrador continua sendo o **maestro**: ele decide como quebrar, dispara os sub-agentes,
> coleta os resultados e junta tudo. O sub-agente é uma **ferramenta de foco**, não um
> participante que "conversa" com os outros.

---

## 2. Sub-agente ≠ multi-agente

Essa é a confusão mais comum. Os dois usam "vários agentes", mas a **relação** entre eles é
diferente.

| Aspecto | **Sub-agente** (este documento) | **Multi-agente** ([04](04-multi-agente.md)) |
|---------|--------------------------------|---------------------------------------------|
| Origem | Decomposição **top-down** de uma tarefa | **Especialistas** que coexistem por design |
| Relação | Hierárquica (orquestrador → sub-agente) | Colaborativa (agentes conversam entre si) |
| Duração | **Efêmero** (some após a sub-tarefa) | Geralmente **persistente**/reutilizável |
| Quem decide o "quê" | O orquestrador decide e distribui | Os agentes negociam/roteiam entre si |
| Analogia | Estagiários contratados para tarefas pontuais | Um time de especialistas fixos (jurídico, dados, redação) |

```
  SUB-AGENTE (top-down, efêmero)          MULTI-AGENTE (colaborativo, persistente)

        Orquestrador                          Agente A ◄────► Agente B
         │  │  │                                  ▲              │
         ▼  ▼  ▼                                  │              ▼
        SA  SA  SA   (descartados)             Agente C ◄──── (conversa contínua)
```

> **Regra prática:** se você está **quebrando uma tarefa** para resolver mais rápido/melhor,
> é **sub-agente**. Se você tem **papéis especialistas que colaboram** de forma contínua, é
> **multi-agente** — veja [04-multi-agente.md](04-multi-agente.md).

---

## 3. Por que decompor: foco, paralelismo, reuso

Três motivos principais para quebrar uma tarefa em sub-tarefas + sub-agentes:

### 3.1 Foco (prompt e tools enxutos)

Um agente único que faz tudo precisa de um **prompt gigante** (todas as instruções) e de
**todas as tools** (todos os schemas). Isso:

- gasta mais tokens por iteração (mais caro);
- confunde o LLM na hora de escolher a tool certa (mais erros).

Um sub-agente recebe **só o que importa** para a sua sub-tarefa:

```
Agente único:   prompt = 3.000 tokens + 12 tools  → escolhe errado às vezes
Sub-agente A:   prompt =   400 tokens +  2 tools  → foco total, menos erro
```

### 3.2 Paralelismo

Sub-tarefas **independentes** podem rodar **ao mesmo tempo**, reduzindo a latência total
(veja a [seção 4](#4-o-trade-off-central-série-vs-paralelo)).

### 3.3 Reuso

Um sub-agente bem definido (ex.: "resumidor de documento", "extrator de entidades") pode ser
**reaproveitado** em vários fluxos, como uma função utilitária.

---

## 4. O trade-off central: série vs paralelo

Ao decompor, a primeira decisão é: as sub-tarefas são **dependentes** (uma precisa do
resultado da outra) ou **independentes** (não dependem entre si)?

### Série — sub-tarefas dependentes

O resultado de A alimenta B, que alimenta C. **Não dá para paralelizar** — a ordem importa.

```
   A ──► B ──► C ──► resultado
   │     │     │
   └─────┴─────┴──► LATÊNCIA = tempo(A) + tempo(B) + tempo(C)   (soma tudo)
```

### Paralelo — sub-tarefas independentes

A, B e C não dependem entre si; disparam juntas e você espera todas terminarem (fan-out /
fan-in).

```
        ┌──► A ──┐
   ─────┼──► B ──┼──► consolidar ──► resultado
        └──► C ──┘
              │
              └──► LATÊNCIA = max(tempo(A), tempo(B), tempo(C))   (só o mais lento)
```

### O ponto que você precisa gravar

| Dimensão | Série (dependente) | Paralelo (independente) |
|----------|--------------------|-------------------------|
| **Latência** | **Soma** dos tempos (mais lento) | **Máximo** dos tempos (mais rápido) |
| **Custo** | Paga todos os tokens | **Paga todos os tokens (igual!)** |
| Quando usar | B precisa de A | A, B, C não dependem entre si |

> ⚠️ **Insight-chave:** paralelizar reduz **latência**, mas **NÃO reduz custo**. Você faz o
> mesmo número de chamadas ao LLM e paga o mesmo total de tokens — só recebe as respostas mais
> rápido, porque elas acontecem simultaneamente em vez de em fila.
>
> (A física completa de custo/latência/throughput está no documento
> [06 sobre desempenho](06-custo-latencia-throughput.md).)

---

## 5. Exemplo Python — série (sub-tarefas dependentes)

Orquestrador que quebra uma tarefa em **3 sub-tarefas dependentes** e chama sub-agentes em
sequência. Cada sub-agente é uma chamada focada ao Bedrock (Claude), com prompt/tools próprios.

Cenário: **"Escreva um post de blog sobre o tema X"** →
`(A) pesquisar pontos-chave` → `(B) montar outline com base nos pontos` → `(C) redigir o post`.

```python
import json
import boto3

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
MODEL_ID = "anthropic.claude-3-5-sonnet-20240620-v1:0"


def sub_agente(system: str, tarefa: str, entrada: str = "") -> str:
    """
    Um sub-agente = uma chamada focada ao LLM, com prompt enxuto para UMA sub-tarefa.
    (Aqui simplificado como single-shot; poderia ser um loop ReAct com tools próprias.)
    """
    prompt = f"{tarefa}\n\n---\nEntrada:\n{entrada}" if entrada else tarefa
    resp = bedrock.invoke_model(
        modelId=MODEL_ID,
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 1024,
            "system": system,                       # prompt ENXUTO, só o papel da sub-tarefa
            "messages": [{"role": "user", "content": prompt}],
        }),
    )
    return json.loads(resp["body"].read())["content"][0]["text"]


def orquestrar_serie(tema: str) -> str:
    # A → B → C : cada etapa depende da anterior (SÉRIE, latência soma)
    pontos = sub_agente(
        system="Você é um pesquisador. Liste pontos-chave em bullets, sem enrolação.",
        tarefa=f"Liste 5 pontos-chave sobre: {tema}",
    )

    outline = sub_agente(
        system="Você é um editor. Produza um outline claro em tópicos.",
        tarefa="Crie um outline de post de blog a partir destes pontos:",
        entrada=pontos,                              # depende de A
    )

    post = sub_agente(
        system="Você é um redator. Escreva em PT-BR, tom didático.",
        tarefa="Escreva o post completo seguindo este outline:",
        entrada=outline,                             # depende de B
    )
    return post


if __name__ == "__main__":
    print(orquestrar_serie("bancos de dados vetoriais"))
```

> Latência total ≈ `t(A) + t(B) + t(C)`. Não há como acelerar sem quebrar a dependência.

---

## 6. Exemplo Python — paralelo (fan-out / fan-in, Lambda-ready)

Quando as sub-tarefas são **independentes**, dispare todas ao mesmo tempo e consolide.
Chamadas ao Bedrock são **I/O-bound** (esperam a rede), então tanto `asyncio` quanto
`concurrent.futures` funcionam bem dentro de um único Lambda.

Cenário: **"Faça um comparativo de 3 produtos"** → 3 sub-agentes (um por produto) rodam em
paralelo, depois consolidamos.

### 6.1 Com `concurrent.futures` (simples e Lambda-friendly)

```python
import json
import boto3
from concurrent.futures import ThreadPoolExecutor

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
MODEL_ID = "anthropic.claude-3-5-sonnet-20240620-v1:0"


def sub_agente(produto: str) -> dict:
    resp = bedrock.invoke_model(
        modelId=MODEL_ID,
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 512,
            "system": "Você analisa UM produto. Retorne prós e contras em bullets.",
            "messages": [{"role": "user", "content": f"Analise o produto: {produto}"}],
        }),
    )
    texto = json.loads(resp["body"].read())["content"][0]["text"]
    return {"produto": produto, "analise": texto}


def orquestrar_paralelo(produtos: list[str]) -> list[dict]:
    # FAN-OUT: dispara todos os sub-agentes ao mesmo tempo
    with ThreadPoolExecutor(max_workers=len(produtos)) as pool:
        resultados = list(pool.map(sub_agente, produtos))
    # FAN-IN: resultados chegam; latência ≈ o mais lento, não a soma
    return resultados


def lambda_handler(event, context):
    produtos = event.get("produtos", ["A", "B", "C"])
    return {"resultados": orquestrar_paralelo(produtos)}
```

### 6.2 Com `asyncio` (alta concorrência de I/O)

```python
import asyncio
import json
import boto3

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
MODEL_ID = "anthropic.claude-3-5-sonnet-20240620-v1:0"


def _chamar(produto: str) -> dict:
    resp = bedrock.invoke_model(
        modelId=MODEL_ID,
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 512,
            "system": "Você analisa UM produto. Prós e contras em bullets.",
            "messages": [{"role": "user", "content": f"Analise: {produto}"}],
        }),
    )
    return {"produto": produto,
            "analise": json.loads(resp["body"].read())["content"][0]["text"]}


async def orquestrar_async(produtos: list[str]) -> list[dict]:
    loop = asyncio.get_running_loop()
    # to_thread evita bloquear o event loop com o cliente boto3 (síncrono)
    tarefas = [loop.run_in_executor(None, _chamar, p) for p in produtos]
    return await asyncio.gather(*tarefas)   # fan-in


def lambda_handler(event, context):
    produtos = event.get("produtos", ["A", "B", "C"])
    return {"resultados": asyncio.run(orquestrar_async(produtos))}
```

### 6.3 Paralelismo "de verdade" em produção (além de um único Lambda)

O paralelismo dentro de um Lambda é **bom até certo ponto** (limitado por memória/CPU e pelo
timeout de 15 min). Para paralelismo **robusto e em escala**, prefira orquestração nativa da AWS:

| Opção AWS | Como paraleliza | Quando usar |
|-----------|-----------------|-------------|
| **Step Functions — `Parallel` state** | Ramos declarados em paralelo, cada um chama um Lambda | Poucas ramificações fixas e conhecidas (ex.: 3-5 sub-agentes) |
| **Step Functions — `Map` state** | Aplica o mesmo Lambda a uma lista (paralelismo dinâmico) | Número de sub-tarefas varia (ex.: N documentos) |
| **Lambda fan-out via SQS** | Publica N mensagens; N invocações de Lambda consomem em paralelo | Alto volume, assíncrono, com retries/DLQ |
| **Lambda fan-out via EventBridge** | Um evento dispara vários alvos (regras) | Roteamento por tipo de evento para diferentes sub-agentes |

```
   Step Functions (Parallel state)          Lambda fan-out (SQS)

        ┌─► Lambda(A) ─┐                    Orquestrador
   ─────┼─► Lambda(B) ─┼─► Join                 │  publica N msgs
        └─► Lambda(C) ─┘                        ▼
                                            [  Fila SQS  ]
                                          ┌────┼────┬────┐
                                          ▼    ▼    ▼    ▼
                                        L(A) L(B) L(C) L(D)  (paralelo real)
```

> O `Join`/consolidação no SQS costuma exigir um passo extra (ex.: gravar cada resultado no
> DynamoDB/S3 e um Lambda final que agrega quando todos chegarem). Com Step Functions, o
> `Parallel`/`Map` já entrega os resultados agrupados para o próximo estado.

---

## 7. Ferramentas e opções de mercado

Você raramente precisa escrever o fan-out/fan-in do zero. Opções comuns:

| Ferramenta | O que é (para sub-tarefas) | Self-hosted vs SaaS/Gerenciado | Encaixe no Lambda |
|------------|----------------------------|-------------------------------|-------------------|
| **LangGraph** | Grafo de execução; permite **sub-grafos** e **branches** paralelos como nós | Biblioteca (self-hosted) | ✅ roda como lib dentro do Lambda |
| **CrewAI** | Define **tasks** (sub-tarefas) atribuídas a agentes; suporta execução sequencial | Biblioteca (self-hosted) | ✅ empacotável no Lambda (atenção ao tamanho) |
| **AWS Step Functions** | `Parallel`/`Map` orquestram sub-tarefas como estados | **Gerenciado (AWS)** | ✅✅ nativo: cada estado chama um Lambda |
| **Lambda + SQS fan-out** | Padrão de infra puro para distribuir sub-tarefas | **Gerenciado (AWS)** | ✅✅ nativo, escala horizontal |
| **LlamaIndex — Sub-Question Query Engine** | Quebra uma pergunta complexa em **sub-perguntas** e responde cada uma (ótimo exemplo de sub-tarefas) | Biblioteca (self-hosted) | ✅ roda como lib; ideal para RAG |
| **OpenAI Agents SDK** | **Handoffs**: um agente delega parte da tarefa a outro | Biblioteca (self-hosted) | ✅ lib, mas usa modelos OpenAI (não Bedrock) |

> **Destaque didático:** o *Sub-Question Query Engine* do LlamaIndex é a materialização mais
> direta da ideia "quebre a tarefa em sub-tarefas": ele transforma "Compare receita e lucro
> de A e B em 2023" em sub-perguntas ("receita de A?", "lucro de A?", "receita de B?"...),
> responde cada uma (potencialmente em paralelo) e consolida.

---

## 8. Padrão de consolidação (juntar resultados)

Depois do fan-in, você tem N resultados soltos. **Consolidar** é transformá-los na resposta
final. Há duas estratégias:

1. **Consolidação por código** — junta/formata sem LLM (barato, determinístico). Bom quando os
   resultados já vêm estruturados (ex.: JSON).
2. **Consolidação por LLM (reduce)** — um sub-agente "sintetizador" lê todos os resultados e
   escreve a resposta final coesa (mais caro, mas melhor para texto corrido).

```
   [resultado A]─┐
   [resultado B]─┼──► CONSOLIDADOR ──► resposta final
   [resultado C]─┘     (código OU LLM sintetizador)
```

Exemplo de consolidação por LLM (padrão map-reduce simples):

```python
def consolidar_com_llm(pergunta: str, resultados: list[dict]) -> str:
    contexto = "\n\n".join(
        f"### {r['produto']}\n{r['analise']}" for r in resultados
    )
    return sub_agente(
        system="Você sintetiza análises parciais em uma resposta final coesa em PT-BR.",
        tarefa=f"Pergunta original: {pergunta}\n\nConsolide as análises abaixo:",
        entrada=contexto,
    )

# Uso:
resultados = orquestrar_paralelo(["Produto A", "Produto B", "Produto C"])
resposta = consolidar_com_llm("Qual produto é melhor e por quê?", resultados)
```

> ⚠️ A consolidação por LLM adiciona **mais uma chamada** (custo + latência). Se os resultados
> já servem como estão, prefira consolidar por código.

---

## 9. Boas práticas e armadilhas

### ✅ Boas práticas

- **Só paralelize o que é independente.** Se B depende de A, paralelizar quebra a lógica —
  mantenha em série.
- **Prompt e tools enxutos por sub-agente.** É o principal ganho de foco (menos erro, menos
  tokens por iteração).
- **Limite o número de sub-agentes.** Cada um custa tokens; decompor demais explode o custo
  (paralelo **não** reduz custo — [seção 4](#4-o-trade-off-central-série-vs-paralelo)).
- **Defina o formato de saída de cada sub-agente** (ex.: JSON) para facilitar a consolidação
  por código.
- **Em produção, use Step Functions (`Parallel`/`Map`)** para paralelismo confiável, com
  retries e visibilidade — em vez de só threads dentro de um Lambda.
- **Trate falhas parciais.** Se um sub-agente falha, decida: abortar tudo, ignorar, ou usar
  um fallback.

### ⚠️ Armadilhas

- **Paralelizar dependentes** → resultados incoerentes (B rodou sem o resultado de A).
- **Achar que paralelo é mais barato** → não é; você paga todos os tokens igual.
- **Decompor demais** → overhead de coordenação + consolidação supera o ganho.
- **Ignorar limites do Lambda** (timeout de 15 min, memória/CPU) ao fazer muito fan-out
  interno — migre para Step Functions/SQS.
- **Confundir sub-agente com multi-agente** → sub-agente é decomposição efêmera; para
  especialistas colaborando, veja [04](04-multi-agente.md).

---

## 10. Glossário

| Termo | Significado |
|-------|-------------|
| **Sub-tarefa** | Pedaço menor e bem definido de uma tarefa maior |
| **Sub-agente** | Agente focado e efêmero criado para resolver uma sub-tarefa |
| **Decomposição top-down** | Quebrar uma tarefa grande em partes menores, de cima para baixo |
| **Fan-out** | Disparar várias sub-tarefas ao mesmo tempo |
| **Fan-in** | Coletar/aguardar os resultados de todas as sub-tarefas |
| **Consolidação (reduce)** | Juntar os resultados parciais na resposta final |
| **Série** | Sub-tarefas dependentes, executadas em sequência (latência soma) |
| **Paralelo** | Sub-tarefas independentes, executadas juntas (latência = a mais lenta) |
| **Map/Parallel (Step Functions)** | Estados AWS que executam ramos/itens em paralelo |

---

## 11. Resumo em uma frase

> **Sub-agentes** são agentes focados e efêmeros criados ao **decompor** uma tarefa em
> sub-tarefas; execute as **dependentes em série** e as **independentes em paralelo**
> (paralelo reduz latência, **não** custo), e depois **consolide** os resultados.

---

## 12. Anterior / Próximo

- ⬅️ **Anterior:** [02-agente-unico.md](02-agente-unico.md) — um único agente resolve a tarefa inteira.
- ➡️ **Próximo:** [04-multi-agente.md](04-multi-agente.md) — vários **especialistas colaborando** (não decomposição efêmera).
