# Múltiplos Orquestradores — Quando e Como Usar (vs Um Só)

> Guia focado em **uma decisão arquitetural**: usar **um único orquestrador** ou **vários**?
> Este documento mostra os **sinais** que indicam a hora de dividir, a **arquitetura hierárquica**
> (raiz/gateway → orquestradores de domínio), **como implementar na AWS** (Bedrock/Lambda) e o
> **custo real** de operar múltiplos. Regra de ouro no fim: **não divida cedo demais.**

---

## Índice

1. [Recap: o que é um orquestrador — e o que é "ter vários"](#1-recap-o-que-é-um-orquestrador--e-o-que-é-ter-vários)
2. [Sinais de que UM orquestrador basta](#2-sinais-de-que-um-orquestrador-basta)
3. [Sinais de que você PRECISA de MÚLTIPLOS](#3-sinais-de-que-você-precisa-de-múltiplos)
4. [Arquitetura hierárquica (raiz → domínio)](#4-arquitetura-hierárquica-raiz--domínio)
5. [Como implementar na AWS — opções e ferramentas](#5-como-implementar-na-aws--opções-e-ferramentas)
6. [Ferramentas/opções relacionadas (gerenciado vs self-hosted)](#6-ferramentasopções-relacionadas-gerenciado-vs-self-hosted)
7. [Exemplo: roteador raiz despachando para 2 orquestradores](#7-exemplo-roteador-raiz-despachando-para-2-orquestradores)
8. [O custo de ter múltiplos](#8-o-custo-de-ter-múltiplos)
9. [Aviso: não divida cedo demais](#9-aviso-não-divida-cedo-demais)
10. [Boas práticas](#10-boas-práticas)
11. [Armadilhas](#11-armadilhas)
12. [Glossário](#12-glossário)
13. [Resumo em uma frase](#13-resumo-em-uma-frase)

---

## 1. Recap: o que é um orquestrador — e o que é "ter vários"

Um **orquestrador** é a peça que **coordena tudo**: o loop de agente, as tools, o roteamento
de modelos, a memória/RAG e o estado da conversa. Ele é o "gerente" que recebe um pedido,
decide o que fazer e devolve a resposta.

> Se estes termos ainda estão nebulosos, volte aos documentos anteriores da série:
> **04-multi-agente.md** (vários agentes dentro de **um** orquestrador) e os fundamentos em
> `docs/02-orquestracao/01-orquestradores-estudo-didatico.md`.

**Cuidado com a confusão comum:** ter **vários agentes** não é o mesmo que ter **vários
orquestradores**.

```
┌─────────────────────────────────────────────────────────────┐
│  MULTI-AGENTE (1 orquestrador)                                │
│                                                               │
│      ┌──────────────── ORQUESTRADOR ÚNICO ───────────────┐   │
│      │   supervisor + agente A + agente B + agente C      │   │
│      │   (mesmo deploy, mesmo estado, mesmo dono)          │   │
│      └────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  MÚLTIPLOS ORQUESTRADORES                                     │
│                                                               │
│   ┌── ORQUESTRADOR 1 ──┐   ┌── ORQUESTRADOR 2 ──┐            │
│   │ atendimento         │   │ relatórios          │            │
│   │ (deploy próprio,    │   │ (deploy próprio,    │            │
│   │  time próprio,      │   │  time próprio,      │            │
│   │  estado próprio)    │   │  estado próprio)    │            │
│   └─────────────────────┘   └─────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

| Conceito | Fronteira | Compartilha deploy? | Compartilha dono/time? |
|----------|-----------|---------------------|------------------------|
| **Multi-agente** | Dentro de um orquestrador | Sim | Geralmente sim |
| **Múltiplos orquestradores** | Entre sistemas | Não (independentes) | Frequentemente não |

**Ter vários orquestradores** significa dividir o sistema em **unidades independentes** — cada
uma com seu deploy, seu estado, seus SLAs e (muitas vezes) seu time dono — e colocar algo na
frente para **rotear** o pedido ao orquestrador certo.

---

## 2. Sinais de que UM orquestrador basta

Comece **sempre** com um. Ele resolve a esmagadora maioria dos casos e é muito mais barato de
operar. Um único orquestrador é suficiente quando:

- ✅ **Um domínio só** — tudo gira em torno de um objetivo coeso (ex: só atendimento).
- ✅ **Um time dono** — as mesmas pessoas mantêm todo o fluxo; não há disputa de prioridades.
- ✅ **SLAs parecidos** — todos os fluxos toleram latências e disponibilidade semelhantes.
- ✅ **Escala parecida** — o volume de requests de cada fluxo é da mesma ordem de grandeza.
- ✅ **Mesmo ciclo de vida** — tudo evolui e faz deploy junto, sem travar uns aos outros.
- ✅ **Sem restrição de isolamento de dados** — não há requisito de separar dados por lei/contrato.

```
                 Usuário / App
                       │
                       ▼
           ┌───────────────────────┐
           │   ORQUESTRADOR ÚNICO   │
           │  (agentes + tools +    │
           │   roteamento + RAG)     │
           └───────────────────────┘
                       │
                    resposta
```

> **Se você marca quase tudo acima, pare aqui.** Não há benefício em dividir; só custo.

---

## 3. Sinais de que você PRECISA de MÚLTIPLOS

Divida quando surge **dor real** — não por antecipação. Os sinais mais fortes:

| Sinal | Por quê justifica múltiplos orquestradores |
|-------|--------------------------------------------|
| **Domínios distintos** | Prompts, tools, modelos e bases de dados diferentes. Juntar tudo num orquestrador gera prompt gigante, mais tokens por iteração e LLM confuso na escolha de tool. |
| **Times/donos diferentes (Conway)** | A arquitetura tende a espelhar a organização. Times separados querem **deploy e roadmap independentes** sem pedir permissão um ao outro. |
| **SLAs conflitantes (tempo-real vs batch)** | Um fluxo precisa responder em < 2 s; outro pode rodar por minutos. Misturá-los faz o batch "engolir" a capacidade do tempo-real. Separe para dimensionar cada um sob sua própria pressão. |
| **Escalas muito diferentes** | 10 mil req/min de atendimento vs 50 relatórios/dia. Escalar o conjunto pelo pico do maior desperdiça recursos; separado, cada um escala sozinho. |
| **Isolamento de falha (blast radius)** | Se o fluxo de relatórios estourar memória/tempo, não pode derrubar o atendimento. Orquestradores separados contêm a falha. |
| **Compliance / isolamento de dados** | Dados regulados (saúde, financeiro, PII) que exigem contas/VPCs/chaves separadas. É mais simples isolar num orquestrador dedicado do que segregar dentro de um só. |
| **Ciclos de vida diferentes** | Um fluxo muda toda semana; outro é estável há meses. Deploys independentes evitam que a mudança de um exija reteste do outro. |

```
     Um só orquestrador dói quando você tem...

     tempo-real  ─┐                       ┌─ batch pesado
     (SLA 2s)     ├── MESMO ORQUESTRADOR ──┤   (roda por minutos)
     alta escala ─┘   (tudo junto)         └─ baixa escala
                        │
                        ▼
              conflito: quem escala? quem faz deploy?
              quem cai junto quando um falha?
                        │
                        ▼
                 →  hora de DIVIDIR
```

**Heurística:** se **2 ou mais** sinais acima aparecem juntos e já causam atrito operacional,
provavelmente é hora de separar.

---

## 4. Arquitetura hierárquica (raiz → domínio)

O padrão mais comum e legível para múltiplos orquestradores é o **hierárquico**: um
**orquestrador raiz** (ou gateway) recebe todo pedido, **classifica/roteia** e delega ao
**orquestrador de domínio** correto. O raiz é fino — ele **não** faz o trabalho, só despacha.

```
                              ┌─────────────────────────┐
                              │     USUÁRIO / APLICAÇÃO   │
                              └────────────┬─────────────┘
                                           │
                              ┌────────────▼─────────────┐
                              │   ORQUESTRADOR RAIZ /     │
                              │   GATEWAY (roteador)      │
                              │  - classifica intenção    │
                              │  - aplica auth/quotas      │
                              │  - escolhe o domínio       │
                              │  - NÃO faz o trabalho      │
                              └───┬───────────┬───────┬───┘
                    ┌─────────────┘           │       └─────────────┐
                    ▼                         ▼                     ▼
        ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
        │  ORQUESTRADOR      │   │  ORQUESTRADOR      │   │  ORQUESTRADOR      │
        │  DE DOMÍNIO A      │   │  DE DOMÍNIO B      │   │  DE DOMÍNIO C      │
        │  "atendimento"     │   │  "relatórios"      │   │  "cobrança"        │
        │  (tempo-real)      │   │  (batch)           │   │  (tempo-real)      │
        │                    │   │                    │   │                    │
        │  agentes + tools + │   │  agentes + tools + │   │  agentes + tools + │
        │  RAG + estado A    │   │  RAG + estado B    │   │  RAG + estado C    │
        └─────────┬──────────┘   └─────────┬──────────┘   └─────────┬──────────┘
                  ▼                        ▼                        ▼
           Bedrock + tools           SQS + worker            Bedrock + tools
           (síncrono)                (assíncrono)            (síncrono)
```

**Propriedades do raiz bem feito:**

- **Fino e rápido:** decide a rota e sai do caminho (pouco ou nenhum LLM, ou só um classificador barato).
- **Ponto único de política:** autenticação, quotas por tenant, rate limit e observabilidade central.
- **Contrato estável:** cada orquestrador de domínio expõe um contrato (evento/payload) que o raiz conhece.
- **Sem lógica de negócio:** toda regra de domínio vive **dentro** do orquestrador de domínio.

> Alternativa ao hierárquico: **malha/eventos** (orquestradores se comunicam via EventBridge sem
> um raiz central). Mais desacoplado, porém mais difícil de rastrear. Para começar, prefira o
> hierárquico — é mais fácil de entender e depurar.

---

## 5. Como implementar na AWS — opções e ferramentas

Não existe "a" forma. Escolha os blocos conforme os sinais que você tem. Tabela de opções:

| Opção AWS | O que faz | Quando usar |
|-----------|-----------|-------------|
| **Lambdas separados por domínio + roteador na frente** | Cada domínio é um Lambda (ou grupo) independente; um Lambda roteador despacha. | Ponto de partida da divisão. Deploy independente por domínio, baixo custo, tudo síncrono e simples. |
| **Step Functions por domínio** | Cada domínio vira uma state machine própria (fluxos com passos, retries, paralelismo). | Fluxos com **múltiplas etapas**, retries e ramificações; quando você quer visualizar/auditar o fluxo do domínio. |
| **SQS por domínio** | Fila por domínio desacopla produtor (raiz) do consumidor (worker). | Separar **tempo-real de batch**: o raiz responde rápido e enfileira o trabalho pesado; o worker processa no seu ritmo. |
| **EventBridge (roteamento por eventos)** | Barramento com regras que roteiam eventos por conteúdo para o alvo certo. | Roteamento **orientado a eventos**, fan-out (um evento → vários orquestradores), integração desacoplada entre times. |
| **Contas / VPCs separadas** | Isolamento forte de rede, IAM e billing por domínio. | **Compliance/isolamento de dados**, blast radius crítico, billing por unidade de negócio. Maior custo operacional. |
| **API Gateway / ALB (entrada)** | Porta de entrada HTTP com auth, throttling e roteamento por rota/host. | Quando a entrada é HTTP/REST e você quer auth, rate limit e roteamento por path antes do raiz. |

**Combinações típicas:**

```
Tempo-real:   API Gateway ─► Lambda RAIZ ─► Lambda "atendimento" (síncrono, Bedrock)
Batch:        Lambda RAIZ ─► SQS "relatorios" ─► Worker (Lambda/Fargate) ─► Bedrock
Eventos:      EventBridge ─► regra por tipo ─► Step Functions do domínio
Compliance:   Conta/VPC dedicada por domínio regulado, com roteador cruzando via evento/API
```

**Regra prática de escolha:**

- Só precisa **dividir por domínio**? → **Lambdas separados + roteador**.
- Precisa **desacoplar tempo-real de batch**? → **SQS por domínio**.
- Fluxo do domínio tem **muitas etapas**? → **Step Functions**.
- Precisa de **isolamento regulatório forte**? → **contas/VPCs separadas**.

---

## 6. Ferramentas/opções relacionadas (gerenciado vs self-hosted)

Além dos blocos AWS "crus", há ferramentas que ajudam a estruturar múltiplos orquestradores:

| Ferramenta | Papel no contexto de múltiplos orquestradores | Gerenciado vs self-hosted |
|------------|-----------------------------------------------|---------------------------|
| **Bedrock Multi-Agent Collaboration** | Um **agente supervisor** gerenciado atua como **orquestrador-raiz**, delegando a agentes colaboradores. Bom quando você quer o raiz gerenciado pela AWS. | **Gerenciado** (AWS) |
| **LangGraph** | Cada **subgraph** pode funcionar como um orquestrador de domínio; um grafo raiz roteia entre subgraphs. Ótimo controle programático do fluxo. | **Self-hosted** (roda no seu Lambda/container) |
| **Gateways de LLM (LiteLLM, Portkey)** | Camada de **roteamento/observabilidade/quota** que pode ficar **entre** os orquestradores (ou na frente deles), centralizando políticas de modelo/custo. | **Self-hosted** (LiteLLM) ou **gerenciado** (Portkey SaaS) |

> **Nota:** essas ferramentas **não substituem** a decisão de dividir — elas **operacionalizam** a
> divisão. Você ainda precisa dos sinais da seção 3 para justificar múltiplos orquestradores.

```
        ┌────────────── ENTRADA (API GW / ALB) ──────────────┐
        │                                                      │
        │        (opcional) GATEWAY DE LLM: LiteLLM/Portkey     │
        │        - roteamento de modelo, quota, observabilidade │
        └───────────────────────┬──────────────────────────────┘
                                 ▼
                      ORQUESTRADOR-RAIZ
              (Bedrock supervisor gerenciado  OU
               LangGraph grafo-raiz self-hosted)
                    │            │            │
                    ▼            ▼            ▼
              domínio A     domínio B     domínio C
```

---

## 7. Exemplo: roteador raiz despachando para 2 orquestradores

Cenário: dois domínios com **SLAs conflitantes**.

- **`atendimento`** — tempo-real, precisa responder já → invoca um **Lambda síncrono** (Bedrock).
- **`relatorios`** — batch pesado, pode demorar → **enfileira em SQS**; um worker processa depois.

```
                         Usuário / App
                              │  (HTTP)
                              ▼
                     ┌──────────────────┐
                     │  Lambda RAIZ     │  ◄── classifica a intenção
                     │  (roteador)      │
                     └───────┬──────────┘
                 tempo-real  │        │  batch
             ┌───────────────┘        └───────────────┐
             ▼                                         ▼
   ┌───────────────────┐                    ┌───────────────────┐
   │ Lambda            │                    │  SQS "relatorios"  │
   │ "atendimento"     │                    └─────────┬──────────┘
   │ (síncrono, Bedrock)│                             ▼
   └─────────┬──────────┘                    ┌───────────────────┐
             ▼                                │ Worker            │
      resposta imediata                       │ "relatorios"      │
      ao usuário                              │ (Bedrock, async)   │
                                              └───────────────────┘
                                                       ▼
                                              resultado gravado
                                              (S3/DynamoDB) + notifica
```

### Código Python do roteador raiz (Lambda)

```python
"""
Lambda RAIZ (roteador). Não faz o trabalho: apenas classifica a intenção
e despacha para o orquestrador de domínio correto.

- 'atendimento' (tempo-real): invoca Lambda síncrono e devolve a resposta.
- 'relatorios' (batch): enfileira em SQS e devolve 202 (aceito).
"""
import json
import os
import boto3

lambda_client = boto3.client("lambda")
sqs = boto3.client("sqs")

LAMBDA_ATENDIMENTO = os.environ["LAMBDA_ATENDIMENTO_ARN"]
FILA_RELATORIOS_URL = os.environ["FILA_RELATORIOS_URL"]


def classificar_dominio(texto: str) -> str:
    """
    Classificador simples por palavra-chave (barato e determinístico).
    Em produção, troque por um classificador leve (modelo pequeno do Bedrock)
    se as intenções forem ambíguas — mas mantenha o raiz FINO e rápido.
    """
    t = (texto or "").lower()
    palavras_relatorio = ("relatório", "relatorio", "exportar", "consolidar", "fechamento")
    if any(p in t for p in palavras_relatorio):
        return "relatorios"
    return "atendimento"


def despachar_atendimento(payload: dict) -> dict:
    """Tempo-real: invoca o orquestrador de atendimento de forma SÍNCRONA."""
    resp = lambda_client.invoke(
        FunctionName=LAMBDA_ATENDIMENTO,
        InvocationType="RequestResponse",  # síncrono
        Payload=json.dumps(payload).encode("utf-8"),
    )
    corpo = json.loads(resp["Payload"].read())
    return {
        "statusCode": 200,
        "body": json.dumps({"dominio": "atendimento", "resposta": corpo}),
    }


def despachar_relatorios(payload: dict) -> dict:
    """Batch: enfileira em SQS e responde 202 (o worker processa depois)."""
    sqs.send_message(
        QueueUrl=FILA_RELATORIOS_URL,
        MessageBody=json.dumps(payload),
    )
    return {
        "statusCode": 202,  # aceito, processamento assíncrono
        "body": json.dumps({
            "dominio": "relatorios",
            "status": "enfileirado",
            "mensagem": "Seu relatório está sendo gerado; você será notificado.",
        }),
    }


def handler(event, context):
    # Entrada esperada (via API Gateway): { "body": "{\"texto\": \"...\", ...}" }
    corpo = json.loads(event.get("body") or "{}")
    texto = corpo.get("texto", "")
    tenant = corpo.get("tenant_id", "anon")

    # (aqui viveriam auth, quotas por tenant, rate limit — política centralizada)

    dominio = classificar_dominio(texto)
    payload = {"texto": texto, "tenant_id": tenant}

    if dominio == "relatorios":
        return despachar_relatorios(payload)
    return despachar_atendimento(payload)
```

**Pontos-chave do exemplo:**

- O raiz **não** chama o LLM para "resolver" — no máximo classifica (aqui, por palavra-chave).
- Cada domínio tem **seu próprio caminho de execução**: síncrono vs assíncrono.
- Falha/lentidão em `relatorios` **não** afeta `atendimento` (isolamento de falha).
- Escala independente: o worker de relatórios pode ter concorrência limitada; o atendimento não.

---

## 8. O custo de ter múltiplos

Dividir tem preço. Pese antes de fazer.

**Prós (por que vale):**

- ✅ **Isolamento de falha** — um domínio cair não derruba os outros.
- ✅ **Deploy independente** — cada time evolui no seu ritmo, sem coordenação.
- ✅ **Escala independente** — dimensiona cada domínio sob sua própria pressão.
- ✅ **Isolamento de dados/compliance** — mais fácil segregar dados regulados.
- ✅ **Clareza de dono** — cada orquestrador tem um responsável claro (Conway a seu favor).

**Contras (o que você paga):**

- ❌ **Mais infraestrutura** — mais Lambdas, filas, roles IAM, alarmes, pipelines para manter.
- ❌ **Saltos de rede e latência** — cada hop (raiz → domínio → tool) adiciona latência e pontos de falha.
- ❌ **Duplicação** — código de tools, RAG e utilitários repetidos entre orquestradores.
- ❌ **Observabilidade distribuída** — um pedido cruza vários sistemas; é preciso **tracing** (correlation id / X-Ray) para depurar ponta a ponta.
- ❌ **Consistência de estado** — coordenar estado entre domínios é mais difícil que num só.

```
    UM orquestrador          MÚLTIPLOS orquestradores
    ───────────────          ────────────────────────
    1 deploy                 N deploys (independentes ✅ mas mais para manter ❌)
    1 hop                    N hops (mais latência ❌)
    1 log/trace              tracing distribuído ❌ (precisa correlation id)
    fácil de depurar         precisa observabilidade ponta a ponta
    escala em bloco          escala por domínio ✅
    falha global ❌          falha isolada ✅
```

> **A análise numérica detalhada** de custo, latência e throughput (com contas e comparações) está no
> próximo documento: **06-custo-latencia-throughput.md**. Aqui, a mensagem é qualitativa: **múltiplos
> orquestradores compram isolamento e independência pagando em complexidade operacional.**

---

## 9. Aviso: não divida cedo demais

O erro mais caro é **dividir por antecipação**, sem dor real. Micro-orquestradores prematuros
trazem toda a complexidade da seção 8 **sem** os benefícios (porque você ainda não tem os
sinais da seção 3).

```
        Comece SIMPLES                     Divida sob DOR
        ──────────────                     ──────────────
   ┌──────────────────┐             ┌──────────────────────────┐
   │  1 orquestrador   │   ──────►   │  raiz + N orquestradores  │
   │  (multi-agente    │  quando     │  de domínio               │
   │   se preciso)     │  aparecer   │                           │
   └──────────────────┘  dor real    └──────────────────────────┘
```

**Sequência saudável de evolução:**

1. **Um orquestrador, um agente.**
2. Cresceu? → **multi-agente dentro do mesmo orquestrador** (ver `04-multi-agente.md`).
3. Apareceram **2+ sinais** da seção 3 causando atrito? → **divida em múltiplos orquestradores**.

> **Regra:** divida quando a **dor operacional** de manter tudo junto for maior que a **dor de
> operar sistemas separados**. Nem antes.

---

## 10. Boas práticas

- **Raiz fino:** o roteador raiz decide a rota e sai do caminho. Sem lógica de negócio nele.
- **Contratos explícitos:** cada orquestrador de domínio expõe um contrato de entrada/saída estável (schema de evento/payload). Versione-o.
- **Correlation id ponta a ponta:** propague um id único em todos os hops para tracing distribuído (X-Ray, logs correlacionados).
- **Isole o batch do tempo-real:** use SQS/EventBridge para que o pesado não engula a capacidade do rápido.
- **Política central, execução distribuída:** auth, quotas e rate limit no raiz/gateway; regra de negócio no domínio.
- **Dead-letter queues (DLQ):** filas por domínio com DLQ para não perder mensagens em falha.
- **Idempotência:** workers assíncronos devem tolerar reprocessar a mesma mensagem sem efeito duplicado.
- **Observabilidade por domínio + agregada:** métricas por orquestrador (latência, custo, iterações) e uma visão consolidada.
- **Comece com hierárquico:** só migre para malha/eventos puros quando o roteamento central virar gargalo.

---

## 11. Armadilhas

- ⚠️ **Dividir cedo demais** — complexidade sem benefício (ver seção 9).
- ⚠️ **Raiz gordo** — colocar lógica de negócio no roteador transforma o "fino" em um monólito escondido.
- ⚠️ **Roteamento com LLM caro no raiz** — usar um modelo grande só para classificar intenção adiciona latência e custo a **todo** request. Prefira classificador barato/determinístico.
- ⚠️ **Contratos implícitos** — domínios que mudam payload sem versionar quebram o raiz e os vizinhos.
- ⚠️ **Sem tracing** — sem correlation id, depurar um pedido que cruza 3 sistemas vira pesadelo.
- ⚠️ **Duplicação descontrolada** — copiar tools/RAG em cada orquestrador; extraia bibliotecas/camadas compartilhadas (Lambda Layers) quando fizer sentido.
- ⚠️ **Ignorar o custo dos hops** — cada salto de rede soma latência; medir é obrigatório (ver 06).
- ⚠️ **Estado espalhado sem dono** — estado compartilhado entre domínios sem fonte de verdade clara gera inconsistência.

---

## 12. Glossário

| Termo | Significado |
|-------|-------------|
| **Orquestrador** | Peça que coordena agentes, tools, roteamento, memória/RAG e estado. |
| **Orquestrador-raiz / gateway** | Orquestrador fino na entrada que classifica e roteia para orquestradores de domínio. |
| **Orquestrador de domínio** | Orquestrador dedicado a um domínio (atendimento, relatórios, cobrança...). |
| **Arquitetura hierárquica** | Raiz → domínios; um ponto central roteia para unidades independentes. |
| **Lei de Conway** | A arquitetura de um sistema tende a espelhar a estrutura da organização que o constrói. |
| **SLA** | Acordo de nível de serviço (ex: latência máxima, disponibilidade). |
| **Blast radius** | Alcance do estrago quando algo falha; isolamento reduz o blast radius. |
| **Tempo-real vs batch** | Síncrono e rápido vs assíncrono e pesado (processado depois). |
| **SQS** | Fila gerenciada da AWS; desacopla produtor de consumidor. |
| **EventBridge** | Barramento de eventos da AWS com roteamento por regras/conteúdo. |
| **Step Functions** | Orquestração de fluxos em etapas (state machine) na AWS. |
| **Correlation id** | Identificador único propagado entre serviços para rastrear um pedido ponta a ponta. |
| **Tracing distribuído** | Rastreamento de um request que cruza múltiplos serviços (ex: X-Ray). |
| **DLQ (dead-letter queue)** | Fila para mensagens que falharam repetidamente, para não perdê-las. |
| **Gateway de LLM** | Camada (LiteLLM/Portkey) para roteamento, quota e observabilidade de chamadas a modelos. |

---

## 13. Resumo em uma frase

> **Comece com um orquestrador; divida em múltiplos apenas quando domínios, times, SLAs, escalas
> ou compliance divergirem a ponto de doer — usando um raiz fino que roteia (Lambda + SQS/EventBridge/Step
> Functions) para orquestradores de domínio independentes, e pagando em complexidade o preço do isolamento.**

---

## Anterior / Próximo

- ⬅️ **Anterior:** [04-multi-agente.md](04-multi-agente.md) — vários agentes colaborando **dentro de um** orquestrador.
- ➡️ **Próximo:** [06-custo-latencia-throughput.md](06-custo-latencia-throughput.md) — a análise numérica de **custo, latência e throughput** dos padrões (incluindo o custo dos múltiplos orquestradores).
