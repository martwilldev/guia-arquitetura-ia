# Serverless para IA — Compute, GPU e a Camada de Orquestração (AWS)

> Panorama do **serverless** numa arquitetura de IA: o **compute serverless "normal"** (sem GPU,
> ex: Lambda/Fargate), o **serverless GPU** (GPU sob demanda que escala a zero) e a **camada de
> gerenciamento/orquestração serverless** (Step Functions e cia.).
> Com comparação de custos, propostas de uso, trade-offs e análise por cenário.

---

## Índice

### Parte I — Panorama e serverless "normal" (sem GPU)
0. [As três peças do serverless para IA](#0-as-três-peças-do-serverless-para-ia)
0a. [Serverless normal (compute sem GPU)](#0a-serverless-normal-compute-sem-gpu)
0b. [A camada de gerenciamento/orquestração serverless](#0b-a-camada-de-gerenciamentoorquestração-serverless)

### Parte II — Serverless GPU
1. [O problema que o serverless GPU resolve](#1-o-problema-que-o-serverless-gpu-resolve)
2. [O que é serverless GPU](#2-o-que-é-serverless-gpu)
3. [Como funciona por dentro](#3-como-funciona-por-dentro)
4. [O calcanhar de aquiles: cold start](#4-o-calcanhar-de-aquiles-cold-start)
5. [Por que aparece em VÁRIAS camadas da arquitetura](#5-por-que-aparece-em-várias-camadas-da-arquitetura)
6. [Plataformas do mercado](#6-plataformas-do-mercado)
7. [Serverless GPU na AWS](#7-serverless-gpu-na-aws)
8. [Comparação: serverless GPU vs alternativas](#8-comparação-serverless-gpu-vs-alternativas)

### Parte III — Custos, propostas de uso e decisão
9. [Análise de custo (normal + GPU + orquestração)](#9-análise-de-custo-normal--gpu--orquestração)
9b. [Propostas de uso em uma arquitetura de IA](#9b-propostas-de-uso-em-uma-arquitetura-de-ia)
10. [Quando usar e quando NÃO usar](#10-quando-usar-e-quando-não-usar)
11. [Exemplo prático](#11-exemplo-prático)
12. [Árvore de decisão](#12-árvore-de-decisão)
13. [Boas práticas e armadilhas](#13-boas-práticas-e-armadilhas)
14. [Glossário](#14-glossário)

---

## 0. As três peças do serverless para IA

Quando falamos de "serverless" numa arquitetura de IA, há **três peças distintas** que costumam ser confundidas:

```
┌──────────────────────────────────────────────────────────────────┐
│                     SERVERLESS PARA IA                             │
│                                                                    │
│  1. COMPUTE SEM GPU        2. COMPUTE COM GPU     3. ORQUESTRAÇÃO   │
│     (serverless "normal")     (serverless GPU)       (gerência)    │
│                                                                    │
│   Lambda, Fargate           SageMaker Serverless   Step Functions  │
│   API Gateway               AWS Batch (GPU)        EventBridge     │
│                             Modal/RunPod/Replicate SQS, SAM        │
│                                                                    │
│   "roda a lógica,           "roda o modelo         "coordena quem  │
│    orquestra, chama API"     que exige placa"       roda o quê"    │
└──────────────────────────────────────────────────────────────────┘
```

| Peça | O que faz | Precisa de GPU? | Exemplos AWS |
|------|-----------|:---------------:|--------------|
| **1. Serverless normal** | Roda sua lógica: orquestrador, API, chamar Bedrock, ler DynamoDB | ❌ Não | Lambda, Fargate, API Gateway |
| **2. Serverless GPU** | Roda o modelo pesado (LLM aberto, embeddings, rerank) | ✅ Sim | SageMaker Serverless, AWS Batch GPU |
| **3. Orquestração serverless** | Coordena o fluxo entre 1 e 2 (etapas, retry, paralelismo, eventos) | ❌ Não | **Step Functions**, EventBridge, SQS |

> **A confusão comum:** achar que "serverless" = só Lambda. Na verdade, numa arquitetura de IA você combina os três: o **Lambda (normal)** orquestra a lógica, o **serverless GPU** roda o que exige placa, e o **Step Functions (orquestração)** costura tudo em um fluxo confiável.

---

## 0a. Serverless normal (compute sem GPU)

É o serverless "do dia a dia" — roda **código**, não modelos que exigem GPU. Na arquitetura de IA, é a **espinha dorsal**: recebe o request, orquestra, chama o LLM (via API como Bedrock), consulta bancos, aplica regras.

### As opções principais na AWS

| Serviço | Modelo | Escala a zero | Timeout | Melhor para |
|---------|--------|:-------------:|---------|-------------|
| **AWS Lambda** | Função por evento | ✅ | 15 min | Orquestrador, API de chat, webhooks, tools |
| **AWS Fargate** | Container sem gerenciar servidor | ⚠️ (tasks) | Ilimitado | Agentes longos, workers de fila, WebSocket |
| **API Gateway** | Entrada HTTP/WS gerenciada | ✅ | — | Auth, rate limit, roteamento de entrada |
| **App Runner** | Container web gerenciado | ⚠️ (opcional) | — | API web contínua simples |

### Por que o serverless normal domina a arquitetura de IA

```
90% de uma arquitetura de IA é LÓGICA (sem GPU):
  - receber e validar o request
  - autenticar o usuário
  - orquestrar o fluxo (loop de agente, roteamento)
  - chamar o LLM (Bedrock = API; a GPU é da AWS, não sua)
  - consultar RAG / DynamoDB
  - aplicar orçamento/quota
  - montar e devolver a resposta

Só a fração que roda MODELO PRÓPRIO precisa de GPU.
```

> **Insight-chave:** se você usa **Bedrock/OpenAI (API pronta)**, a GPU é do provedor. Sua arquitetura pode ser **100% serverless normal** (Lambda + Step Functions) e **zero GPU sua**. O serverless GPU só entra se você quiser **rodar um modelo próprio** (aberto/self-hosted).

### Custo do serverless normal

- **Lambda**: paga por invocação + GB-segundo. Em IA, é **irrisório** perto do custo dos tokens do LLM.
- **Fargate**: paga por vCPU-hora + GB-hora enquanto a task roda.
- Regra: o compute serverless normal quase nunca é o gargalo de custo — **os tokens do LLM dominam**.

---

## 0b. A camada de gerenciamento/orquestração serverless

> Esta é provavelmente a peça cujo nome você procurava. Na AWS, a **"camada de gerenciamento/coordenação de serverless"** é, na maioria dos casos, o **AWS Step Functions** — o serviço que orquestra funções serverless em fluxos (state machines). Abaixo, os candidatos e o que cada um faz, para você confirmar qual era.

### Candidatos (o que "camada de gerenciamento serverless" pode significar)

| Serviço AWS | O que é | Quando é "a camada" que você quis dizer |
|-------------|---------|------------------------------------------|
| **AWS Step Functions** ⭐ | Orquestrador de fluxos serverless (state machine): coordena Lambdas, tem estado, retry, paralelismo, escolhas | Se você pensa em **coordenar/gerenciar a execução** de várias funções em um fluxo |
| **AWS SAM** (Serverless Application Model) | Framework para **definir e deployar** apps serverless (IaC) | Se você pensa em **gerenciar o deploy/infra** do serverless |
| **Amazon EventBridge** | Barramento de eventos que **conecta** serviços serverless | Se você pensa em **rotear eventos** entre componentes |
| **Amazon SQS** | Fila que **desacopla** produtores e consumidores serverless | Se você pensa em **absorver picos** e processar async |

### Foco: AWS Step Functions (a orquestração serverless)

O Step Functions é o "maestro serverless": você desenha um **fluxo de estados** e ele coordena a execução, cuidando de estado, erros e paralelismo — **sem servidor** e **sem o limite de 15 min do Lambda** (roda até 1 ano).

```
                 ┌─────────────────────────────────────────┐
                 │           STEP FUNCTIONS                  │
                 │        (state machine serverless)         │
   Evento ──────►│                                           │
                 │  [Validar] ─► [Escolha] ─┬─► [Agente A]   │
                 │                          ├─► [Agente B]   │
                 │                          └─► [Paralelo]   │
                 │                               ├─ Lambda   │
                 │                               ├─ Lambda   │
                 │                               └─ GPU job  │
                 │  ◄── retry/erro por etapa, estado passado ─┤
                 └─────────────────────────────────────────┘
```

**Por que importa numa arquitetura de IA:**

| Recurso | Benefício para IA |
|---------|-------------------|
| **Sem timeout de 15 min** | Fluxos de agente longos que estourariam o Lambda |
| **Retry/erro por etapa** | Reintenta chamada ao LLM/tool que falhou (throttling do Bedrock) |
| **Paralelismo (Parallel/Map)** | Sub-agentes/fan-out rodando ao mesmo tempo |
| **Estado gerenciado** | Passa contexto entre etapas sem código extra |
| **Visual** | Enxerga o fluxo do orquestrador como diagrama |
| **Coordena GPU jobs** | Dispara e aguarda um job de GPU (Batch/SageMaker) como uma etapa |

> Ver também [`../02-orquestracao/03-orquestradores-ia-lambda.md`](../02-orquestracao/03-orquestradores-ia-lambda.md) (seção Step Functions) e a série [`../02-orquestracao/07-padroes-execucao/`](../02-orquestracao/07-padroes-execucao/) para como orquestrar agentes.

### Custo da camada de orquestração

- **Step Functions (Standard)**: cobra por **transição de estado** (cada passo do fluxo). Barato por transição, mas fluxos com muitos passos × muito volume somam.
- **Step Functions (Express)**: para alto volume/curta duração — cobra por execução + duração; mais barato em rajada.
- **EventBridge/SQS**: por evento/mensagem publicada — geralmente centavos por milhão.
- Regra: a orquestração é **barata** perto dos tokens do LLM, mas escolha **Express** para fluxos de altíssimo volume e curtos.

---

## 1. O problema que o serverless GPU resolve

Rodar IA "de verdade" (um modelo aberto, embeddings, reranking) exige **GPU**. Mas GPU é cara e tem um problema de modelo de consumo:

```
Uma GPU dedicada (ex: A100/L4) na nuvem custa por HORA, ligada ou não.
   │
   ├── Tráfego constante 24/7  → ok, ela fica sempre útil.
   └── Tráfego variável/baixo  → você paga a GPU parada a maior parte do tempo. 💸
```

O dilema clássico:

| Opção tradicional | Problema |
|-------------------|----------|
| **GPU dedicada (EC2 GPU)** | Paga 24/7 mesmo ociosa; desperdício em tráfego variável |
| **API pronta (Bedrock/OpenAI)** | Simples, mas você não controla o modelo; nem todo modelo está disponível |
| **CPU (Lambda comum)** | Barato e serverless, mas **não roda** modelos que exigem GPU |

**Serverless GPU** é a ponte: a conveniência do serverless (paga pelo uso, escala a zero) com o poder da GPU.

---

## 2. O que é serverless GPU

É um modelo de execução onde a plataforma **provisiona GPUs sob demanda** para rodar sua carga, e **desliga** quando não há tráfego.

```
Sem tráfego:   [ 0 GPUs ]                    → custo ~zero
Chega request: [ provisiona GPU ] → executa  → paga só o tempo de uso
Fim do tráfego:[ desliga após timeout ]      → volta a ~zero
```

Características centrais:

- **Pay-per-use** — cobra por segundo/minuto de GPU efetivamente usada.
- **Scale to zero** — sem request, sem custo (ou custo mínimo).
- **Autoscaling** — sobe mais GPUs sob rajada, desce quando acalma.
- **Sem gerenciar servidor** — você entrega o código/modelo, a plataforma cuida da infra.

> **Analogia:** é como Uber vs comprar um carro. A GPU dedicada é o carro na garagem (você paga mesmo parado). O serverless GPU é o Uber (paga só a corrida) — mas com um tempo de espera até o carro chegar (o **cold start**).

---

## 3. Como funciona por dentro

```
Request ──► Fila/roteador da plataforma
                 │
                 ├── Há uma GPU "quente" (warm) livre?
                 │       ├── Sim ──► executa imediato (baixa latência)
                 │       └── Não ──► COLD START:
                 │                    1. aloca uma GPU
                 │                    2. carrega o container/imagem
                 │                    3. carrega os PESOS do modelo na VRAM
                 │                    4. executa
                 ▼
             Resposta
                 │
   (após timeout de inatividade, a GPU é liberada → scale to zero)
```

O ponto crítico é o passo 3: **carregar os pesos do modelo na memória da GPU** pode levar de segundos a minutos, dependendo do tamanho do modelo.

---

## 4. O calcanhar de aquiles: cold start

O maior trade-off do serverless GPU é a **latência de partida a frio**.

```
Modelo pequeno (embeddings, ~100MB-1GB):   cold start ~ segundos
Modelo médio (7B-13B, vários GB):           cold start ~ dezenas de segundos
Modelo grande (70B+, dezenas de GB):        cold start ~ minutos
```

Por quê: quanto maior o modelo, mais tempo para baixar/carregar os pesos na VRAM.

### Mitigações comuns (oferecidas pelas plataformas)

| Técnica | O que faz | Custo |
|---------|-----------|-------|
| **Keep-warm / min instances** | Mantém 1+ GPU sempre quente | Perde parte do "scale to zero" (paga a warm) |
| **Snapshotting de memória** | Salva o estado da GPU já carregado e restaura rápido | Reduz cold start drasticamente |
| **Modelos menores** | Menos pesos = carrega mais rápido | Pode perder qualidade |
| **Cache de pesos no storage rápido** | Pesos ficam perto da GPU | Menor download |
| **Fila com timeout generoso** | Segura a GPU quente por mais tempo entre requests | Mais custo de idle |

> **Regra:** serverless GPU brilha em cargas **variáveis/esporádicas** onde o cold start ocasional é tolerável. Para latência baixíssima e constante, uma GPU **warm/dedicada** ainda ganha.

---

## 5. Por que aparece em VÁRIAS camadas da arquitetura

Você notou certo: "serverless GPU" é citado como opção em vários pontos. Isso porque **várias camadas de uma arquitetura de IA precisam de GPU**, mas nem todas têm tráfego constante — o cenário perfeito para serverless GPU.

```
┌─────────────────────────────────────────────────────────────┐
│                  ARQUITETURA DE IA                            │
│                                                               │
│  ┌────────────┐   Camada 1: SERVIR O LLM (inference engine)  │
│  │ Orquestrador│──► modelo aberto (Llama/Mistral) em GPU      │◄─ serverless GPU
│  └─────┬──────┘      (se não usar API pronta como Bedrock)    │
│        │                                                       │
│        ├──► Camada 2: EMBEDDINGS (RAG)                         │◄─ serverless GPU
│        │      gerar vetores de documentos/queries em GPU      │
│        │                                                       │
│        ├──► Camada 3: RERANKING (RAG)                          │◄─ serverless GPU
│        │      cross-encoder reordena resultados em GPU        │
│        │                                                       │
│        ├──► Camada 4: INGESTÃO/BATCH (RAG)                     │◄─ serverless GPU
│        │      vetorizar milhões de docs de vez em quando      │
│        │                                                       │
│        └──► Camada 5: MODELOS AUXILIARES                       │◄─ serverless GPU
│               transcrição (Whisper), imagem, OCR, classificador│
└─────────────────────────────────────────────────────────────┘
```

### Por camada

| Camada | Precisa de GPU para... | Por que serverless GPU encaixa |
|--------|------------------------|--------------------------------|
| **1. Servir o LLM** | Rodar um modelo aberto self-hosted (se você não usa API pronta) | Se o tráfego é variável, evita pagar GPU 24/7 |
| **2. Embeddings** | Gerar vetores com modelos open-source (bge, e5, gte) | Consultas em rajada; entre elas, escala a zero |
| **3. Reranking** | Cross-encoder reordenar top-K resultados | Só roda quando há busca; sob demanda |
| **4. Ingestão/batch** | Vetorizar grandes lotes de documentos ocasionalmente | Job esporádico e pesado — perfeito para sob demanda |
| **5. Auxiliares** | Whisper (áudio), visão, OCR, classificadores | Uso intermitente; não justifica GPU dedicada |

> **A razão de fundo:** todas essas camadas têm **tráfego intermitente ou em rajada**. Manter uma GPU dedicada para cada uma seria caro e ociosa a maior parte do tempo. Serverless GPU deixa cada camada "acender" a GPU só quando precisa.

### Quando a camada NÃO combina com serverless GPU

- **Servir o LLM com tráfego alto e constante** → GPU dedicada/reservada sai mais barata e sem cold start.
- **Latência crítica de tempo-real** (chat com resposta imediata sempre) → warm/dedicada.
- **Se você usa API pronta (Bedrock/OpenAI)** → a camada de LLM nem precisa de GPU sua; o serverless GPU fica só para embeddings/rerank/auxiliares que você queira self-hostar.

---

## 6. Plataformas do mercado

Plataformas especializadas em serverless GPU (fora dos hyperscalers):

| Plataforma | Destaque | Modelo de cobrança | DX (experiência de dev) |
|------------|----------|--------------------|-----------------------|
| **Modal** | Deploy via código Python puro; ótimo DX; scale-to-zero rápido | Por segundo de GPU/CPU | ⭐⭐⭐ Excelente |
| **RunPod** | GPUs baratas; "serverless" e "pods"; comunidade grande | Por segundo | ⭐⭐ Boa |
| **Replicate** | Roda modelos como API; catálogo enorme; paga por execução | Por execução/segundo | ⭐⭐⭐ Muito fácil |
| **Baseten** | Deploy de modelos com Truss; foco produção | Por minuto de GPU | ⭐⭐⭐ Boa |
| **Beam / Cerebrium** | Serverless GPU dev-first | Por segundo | ⭐⭐ Boa |
| **HuggingFace Inference Endpoints** | Deploy de modelos do Hub; scale-to-zero opcional | Por hora (com autoscale) | ⭐⭐⭐ Fácil |
| **Together AI / Fireworks / Groq** | APIs de modelos abertos (abstraem a GPU) | Por token | ⭐⭐⭐ (é API, não GPU crua) |

> Nota: **Together/Fireworks/Groq** tecnicamente entregam **API por token** (você nem vê a GPU). Servem quando você quer um modelo aberto sem gerenciar nada — é "serverless" no sentido de consumo, mas não é você quem roda o container na GPU.

---

## 7. Serverless GPU na AWS

Como o resto do estudo tem foco AWS, aqui estão as opções nativas:

| Serviço AWS | O que é | Scale to zero? | Nota |
|-------------|---------|:--------------:|------|
| **SageMaker Serverless Inference** | Endpoint de inferência que escala sob demanda | ✅ (escala a zero) | Bom para tráfego intermitente; **tinha limitações de GPU** — confira suporte a GPU/tamanho de modelo atual |
| **SageMaker Asynchronous Inference** | Fila para inferência pesada; escala a zero quando vazia | ✅ | Ótimo para **batch/ingestão** e payloads grandes |
| **SageMaker Real-time (com autoscaling)** | Endpoint sempre-on que escala instâncias | ⚠️ (mín. costuma ser ≥1) | Baixa latência, mas paga a instância mínima |
| **Amazon Bedrock** | API gerenciada de modelos (a AWS roda a GPU) | ✅ (você paga por token) | Se o modelo que você quer está no catálogo, é o caminho mais simples |
| **ECS/EKS + GPU + scale-to-zero (Karpenter/KEDA)** | Você monta seu próprio "serverless GPU" | ✅ (com config) | Mais controle, mais trabalho |
| **AWS Batch (GPU)** | Jobs em lote com GPU sob demanda | ✅ | Ideal para **ingestão/vetorização em massa** |
| **AWS Lambda** | ❌ **não tem GPU** | — | Use Lambda como orquestrador que *chama* uma das opções acima |

> **Padrão comum na AWS:** o **Lambda orquestra** (barato, serverless) e **delega a parte de GPU** para SageMaker Serverless/Async, Batch, ou uma plataforma externa (Modal/Replicate). O Lambda em si nunca roda o modelo pesado — ele não tem GPU.

> ⚠️ O suporte a GPU no SageMaker Serverless e os limites de tamanho de modelo **mudam com o tempo**. Confirme na documentação oficial e na [Calculadora AWS](https://calculator.aws) antes de decidir.

---

## 8. Comparação: serverless GPU vs alternativas

| Aspecto | API pronta (Bedrock) | Serverless GPU | GPU dedicada (EC2) |
|---------|----------------------|----------------|--------------------|
| Você gerencia a GPU? | ❌ Não | ⚠️ Parcial (entrega código/modelo) | ✅ Sim (total) |
| Escala a zero | ✅ (paga por token) | ✅ | ❌ (paga 24/7) |
| Cold start | ✅ Nenhum (é API) | ❌ Sim (segundos a min) | ✅ Nenhum (sempre on) |
| Custo em tráfego BAIXO/variável | 🟢 Ótimo | 🟢 Ótimo | 🔴 Ruim (ociosa) |
| Custo em tráfego ALTO/constante | 🟡 Pode ficar caro por token | 🟡 Médio | 🟢 Melhor (reservada) |
| Escolha de modelo | ⚠️ Só o catálogo | ✅ Qualquer modelo aberto | ✅ Qualquer |
| Controle/customização | 🔴 Baixo | 🟡 Médio | 🟢 Total |
| Esforço operacional | 🟢 Mínimo | 🟡 Médio | 🔴 Alto |

---

## 9. Análise de custo (normal + GPU + orquestração)

> ⚠️ Valores **ilustrativos** para raciocínio. Preços de GPU variam muito por tipo (L4, A10G, A100, H100), região e plataforma. Confirme nas páginas oficiais e na [Calculadora AWS](https://calculator.aws).

### 9.1 De onde vem o custo

```
Custo serverless GPU ≈ (tempo de GPU usado por request × preço/segundo da GPU)
                     + (eventual custo de keep-warm/min instances)
                     + storage dos pesos + rede
```

### 9.2 O ponto de virada (serverless vs dedicada)

```
       Custo mensal
          ▲
          │                              ╱ GPU dedicada (linha fixa: paga 24/7)
          │                         ╱────
          │                    ╱────
          │            ╱──────╱  ◄── ponto de virada
          │      ╱────╱      (acima daqui, dedicada é mais barata)
          │ ╱───╱   serverless GPU (cresce com o uso)
          └──────────────────────────────────► Uso (horas de GPU/mês)
```

- **Uso baixo/variável** → serverless GPU é muito mais barato (você não paga ociosidade).
- **Uso alto/constante** → GPU dedicada/reservada tende a sair mais barata por hora.

### 9.3 Exemplo ilustrativo

```
Suponha uma GPU a ~US$ 2/hora (dedicada) vs ~US$ 0,0006/segundo serverless (~US$ 2,16/h ativa).

Cenário A — embeddings esporádicos: 2 horas de GPU efetiva por mês
  Dedicada:    US$ 2 × 730h = US$ 1.460/mês (ligada o mês todo)  💸
  Serverless:  US$ 2,16 × 2h ≈ US$ 4,32/mês                       ✅
  → serverless ~340× mais barato aqui.

Cenário B — LLM servindo 24/7 em carga constante: ~730 horas/mês
  Dedicada (reservada): pode cair para ~US$ 1/h × 730 = US$ 730/mês  ✅
  Serverless:           US$ 2,16 × 730 = US$ 1.577/mês              💸
  → dedicada mais barata em uso constante.
```

> **Conclusão:** o serverless GPU **não é sempre mais barato** — ele é mais barato **em tráfego variável**. Meça as horas efetivas de GPU do seu caso.

### 9.4 Custo comparado das TRÊS peças serverless

Juntando compute normal + GPU + orquestração, veja a ordem de grandeza de cada uma numa arquitetura de IA típica:

| Peça | Modelo de cobrança | Peso no custo total | Observação |
|------|--------------------|--------------------|------------|
| **Serverless normal (Lambda/Fargate)** | Invocação + GB-segundo / vCPU-hora | 🟢 Baixo | Quase sempre irrisório perto dos tokens |
| **Orquestração (Step Functions)** | Por transição de estado (ou execução no Express) | 🟢 Baixo-médio | Cresce com nº de passos × volume; use Express em alto volume |
| **Serverless GPU** | Por segundo de GPU (+ keep-warm) | 🟡 Médio-alto | Só existe se você roda modelo próprio |
| **Tokens do LLM (Bedrock/API)** | Por token entrada/saída | 🔴 **Domina** | Normalmente 60–90% do custo total |

```
Distribuição típica de custo numa arquitetura de IA (ilustrativa):

  Tokens do LLM    ██████████████████████████████████  ~70%
  Serverless GPU   ████████████                        ~20% (se roda modelo próprio; senão 0)
  Orquestração     ███                                 ~5%
  Compute normal   ██                                  ~3%
  RAG/storage/rede ██                                  ~2%
```

> **Insight-chave:** otimizar Lambda/Step Functions economiza centavos; otimizar **tokens** (roteamento, cache, contexto) economiza a maior parte. Só o **serverless GPU** rivaliza com os tokens em custo — e apenas quando você roda modelo próprio em volume.

### 9.5 Trade-offs consolidados

| Decisão | Escolha A | Escolha B | Regra |
|---------|-----------|-----------|-------|
| Rodar modelo? | API pronta (Bedrock) | Serverless GPU (próprio) | API se está no catálogo e não precisa customizar |
| Compute da lógica | Lambda | Fargate | Lambda p/ curto; Fargate p/ longo (>15 min) ou warm |
| GPU: sob demanda ou fixa | Serverless GPU | GPU dedicada | Serverless se variável; dedicada se constante/alto |
| Orquestração | Step Functions Standard | Step Functions Express | Express p/ altíssimo volume e curto |
| Ingestão pesada | Lambda | AWS Batch / Fargate | Batch/Fargate p/ jobs longos com GPU |

---

## 9b. Propostas de uso em uma arquitetura de IA

Como combinar as três peças (normal + GPU + orquestração) em arranjos reais, do mais simples ao mais completo.

### Proposta 1 — 100% serverless, sem GPU sua (usa Bedrock)

O caso mais comum e barato de operar: você **não roda modelo próprio**, usa a API do Bedrock.

```
Usuário ─► API Gateway ─► Lambda (orquestrador) ─► Bedrock (API, GPU é da AWS)
                              │                        │
                              └─► DynamoDB (estado)    └─► sem GPU sua!
```

- **Peças:** serverless normal (Lambda) + orquestração leve. **Zero serverless GPU.**
- **Custo:** tokens do LLM dominam; compute irrisório.
- **Quando:** MVP, produto de chat, RAG com Bedrock Knowledge Bases. **Comece por aqui.**

### Proposta 2 — Serverless normal + serverless GPU para embeddings/rerank

Você usa Bedrock para o LLM, mas roda **embeddings/reranking próprios** (modelo open-source) em serverless GPU (economia ou modelo específico).

```
Usuário ─► API Gateway ─► Lambda (orquestrador) ─┬─► Bedrock (LLM)
                                                 ├─► SageMaker Serverless (embeddings GPU)
                                                 └─► endpoint GPU (rerank)
```

- **Peças:** normal + GPU (só nas camadas de vetor).
- **Custo:** tokens + fração de GPU sob demanda (escala a zero entre buscas).
- **Quando:** quer controlar embeddings/rerank, tem tráfego de busca variável.

### Proposta 3 — Fluxo complexo orquestrado por Step Functions

Fluxo de agente multi-etapa/longo, com paralelismo e um job de GPU no meio.

```
Evento ─► Step Functions (orquestração serverless)
             ├─ [Lambda] classifica/roteia
             ├─ [Parallel] ─┬─ [Lambda] busca web
             │              └─ [SageMaker Async] embeddings em lote (GPU)
             ├─ [Lambda] chama Bedrock (LLM)
             └─ [Lambda] consolida e responde
```

- **Peças:** as três — normal (Lambdas) + GPU (job) + orquestração (Step Functions).
- **Custo:** tokens + GPU do job + transições do Step Functions.
- **Quando:** fluxos longos que estouram 15 min, sub-agentes paralelos, ingestão como etapa.

### Proposta 4 — Ingestão de RAG (batch) desacoplada

Vetorizar muitos documentos ocasionalmente, sem manter GPU ligada.

```
Novo doc no S3 ─► EventBridge ─► Step Functions
                                    └─ [AWS Batch GPU] vetoriza em lote ─► vector store
                                                                             │
Consulta (tempo-real) ─► Lambda ─► só CONSULTA o índice já pronto ───────────┘
```

- **Peças:** GPU sob demanda (batch) para ingestão + normal (Lambda) para consulta + orquestração (EventBridge/Step Functions).
- **Custo:** GPU só durante o job de ingestão; consulta é compute barato.
- **Quando:** RAG com atualização periódica de base grande.

### Proposta 5 — Modelo próprio servido em serverless GPU (sem Bedrock)

Você quer um modelo aberto específico como LLM principal, com tráfego variável.

```
Usuário ─► API Gateway ─► Lambda (orquestrador) ─► Serverless GPU (LLM aberto)
                                                     (Modal/SageMaker/RunPod)
```

- **Peças:** normal (orquestra) + GPU (serve o LLM). Cuidado com **cold start** do modelo grande.
- **Custo:** GPU sob demanda domina; sem custo de token de API.
- **Quando:** modelo fora do catálogo Bedrock, tráfego variável, tolera cold start (ou usa keep-warm).

### Tabela-resumo das propostas

| Proposta | Normal | GPU | Orquestração | Custo dominante | Quando |
|----------|:------:|:---:|:------------:|-----------------|--------|
| 1 · Só Bedrock | ✅ | ❌ | leve | Tokens | MVP, maioria dos casos |
| 2 · + embeddings/rerank GPU | ✅ | parcial | leve | Tokens + GPU parcial | Controlar vetores |
| 3 · Step Functions complexo | ✅ | job | ✅ Step Functions | Tokens + GPU + transições | Fluxos longos/paralelos |
| 4 · Ingestão batch | ✅ | batch | ✅ EventBridge/SF | GPU do job | RAG periódico |
| 5 · LLM próprio em GPU | ✅ | ✅ | leve | GPU | Modelo fora do Bedrock |

---

## 10. Quando usar e quando NÃO usar

### ✅ Use serverless GPU quando

- Tráfego **variável, esporádico ou imprevisível**.
- **Ingestão/batch** ocasional (vetorizar documentos de vez em quando).
- **Protótipos e MVPs** (não quer pagar GPU 24/7 para validar).
- **Modelos auxiliares** de uso intermitente (Whisper, OCR, reranking).
- Você quer um **modelo aberto específico** que não está numa API pronta.
- Tolera um **cold start ocasional**.

### ❌ Evite quando

- Tráfego **alto e constante** (dedicada/reservada é mais barata).
- **Latência crítica sempre** (o cold start atrapalha) — a menos que use keep-warm (que come o scale-to-zero).
- O modelo que você quer **já está numa API pronta** (Bedrock) e você não precisa customizar — aí use a API.
- Modelos **gigantes** onde o cold start é proibitivo e você não pode manter warm.

---

## 11. Exemplo prático

Padrão típico na AWS: **Lambda orquestra, serverless GPU executa a parte pesada.**

```
Usuário ─► API Gateway ─► Lambda (orquestrador, SEM GPU)
                               │
                               │ 1. precisa de embedding?
                               ├──────────► SageMaker Serverless / Modal (GPU) ──► vetor
                               │
                               │ 2. precisa reordenar (rerank)?
                               ├──────────► endpoint GPU (cross-encoder) ──► top-K
                               │
                               │ 3. precisa do LLM?
                               └──────────► Bedrock (API) OU LLM aberto em serverless GPU
                               │
                               ▼
                          resposta
```

### Esboço do Lambda chamando um endpoint de GPU

```python
import json, boto3

# Cliente fora do handler (reuso em invocações quentes)
sm = boto3.client("sagemaker-runtime", region_name="us-east-1")
EMBED_ENDPOINT = "meu-endpoint-embeddings-serverless"  # SageMaker Serverless Inference

def gerar_embedding(texto: str) -> list:
    """Lambda (sem GPU) delega a geração de embedding ao endpoint GPU serverless."""
    resp = sm.invoke_endpoint(
        EndpointName=EMBED_ENDPOINT,
        ContentType="application/json",
        Body=json.dumps({"inputs": texto}),
    )
    return json.loads(resp["Body"].read())["embedding"]

def handler(event, context):
    texto = json.loads(event.get("body", "{}")).get("text", "")
    # A GPU "acende" sob demanda no endpoint serverless; entre requests, escala a zero.
    vetor = gerar_embedding(texto)
    return {"statusCode": 200, "body": json.dumps({"dim": len(vetor)})}
```

> O Lambda nunca toca a GPU — ele **chama** o endpoint serverless GPU. Isso combina o serverless barato (orquestração) com GPU sob demanda (a parte que exige placa).

---

## 12. Árvore de decisão

```
Sua camada precisa de GPU?
   ├── Não (só chama API pronta) ─────────► use Bedrock/OpenAI; sem GPU sua
   └── Sim ▼

O tráfego dessa camada é constante e alto?
   ├── Sim ───────────────────────────────► GPU DEDICADA/RESERVADA (EC2/SageMaker real-time)
   └── Não (variável/esporádico) ▼

É um job em lote (ingestão/vetorização em massa)?
   ├── Sim ───────────────────────────────► SERVERLESS GPU batch (SageMaker Async / AWS Batch)
   └── Não (requests interativos) ▼

A latência precisa ser baixíssima SEMPRE?
   ├── Sim ───────────────────────────────► serverless GPU com KEEP-WARM (ou dedicada)
   └── Não (tolera cold start ocasional) ──► SERVERLESS GPU puro (scale to zero) ✅
```

---

## 13. Boas práticas e armadilhas

### Boas práticas

- **Meça as horas efetivas de GPU** antes de escolher — é o que define serverless vs dedicada.
- **Lambda orquestra, GPU executa** — nunca tente rodar o modelo pesado no Lambda (não tem GPU).
- **Separe batch de tempo-real** — ingestão em fila/async; consultas em endpoint com keep-warm se preciso.
- **Modelos menores** para embeddings/rerank reduzem cold start e custo.
- **Keep-warm só no caminho crítico** — mantenha quente só o que precisa de latência baixa.
- **Monitore cold starts** e latência p95, não só a média.

### Armadilhas

| Armadilha | Consequência | Mitigação |
|-----------|--------------|-----------|
| Esperar GPU no Lambda | Lambda não tem GPU; não roda o modelo | Delegar a SageMaker/Modal/etc. |
| Ignorar cold start | Latência inaceitável em picos | Keep-warm ou snapshot |
| Serverless em tráfego constante | Fica mais caro que dedicada | Medir e migrar para reservada |
| Keep-warm exagerado | Perde o "scale to zero" (paga idle) | Warm mínimo, só no crítico |
| Modelo gigante em serverless | Cold start de minutos | Modelo menor ou dedicada warm |
| Não confirmar suporte a GPU do serviço | Escolher serviço que não roda seu modelo | Checar doc oficial atual |

---

## 14. Glossário

| Termo | Significado |
|-------|-------------|
| **Serverless GPU** | GPU sob demanda, paga por uso, que escala a zero quando ociosa |
| **Serverless normal** | Compute serverless sem GPU (Lambda/Fargate) — roda lógica, não modelos pesados |
| **Step Functions** | Orquestrador serverless de fluxos (state machine): coordena Lambdas, com estado/retry/paralelismo |
| **Standard vs Express (Step Functions)** | Standard cobra por transição (fluxos longos); Express por execução/duração (alto volume, curto) |
| **EventBridge** | Barramento de eventos que conecta/roteia serviços serverless |
| **SQS** | Fila que desacopla produtores/consumidores e absorve picos |
| **SAM** | Serverless Application Model — framework para definir/deployar apps serverless (IaC) |
| **Transição de estado** | Cada passo de uma state machine do Step Functions (unidade de cobrança Standard) |
| **Scale to zero** | Cair a zero instâncias (custo ~zero) sem tráfego |
| **Cold start** | Latência de partida a frio (alocar GPU + carregar pesos) |
| **Keep-warm / min instances** | Manter 1+ GPU sempre quente para evitar cold start |
| **Snapshotting** | Salvar/restaurar o estado da GPU já carregado, acelerando o start |
| **VRAM** | Memória da GPU onde os pesos do modelo são carregados |
| **Inference endpoint** | Endpoint que serve inferência de um modelo |
| **Batch/async inference** | Inferência em lote/assíncrona, tolerante a latência |
| **Cross-encoder (rerank)** | Modelo que reordena resultados de busca (usa GPU) |
| **Embedding** | Vetor numérico que representa texto (gerado em GPU por modelos abertos) |
| **Hyperscaler** | Grande provedor de nuvem (AWS/Azure/GCP) |
| **Ponto de virada** | Volume de uso onde dedicada passa a ser mais barata que serverless |

---

## Resumo em uma frase

> **Uma arquitetura de IA serverless combina três peças: o compute normal (Lambda/Fargate) que roda a lógica e orquestra, a camada de orquestração serverless (Step Functions e cia.) que costura o fluxo com estado/retry/paralelismo sem o limite de 15 min, e o serverless GPU (SageMaker Serverless/Async, AWS Batch, Modal/Replicate) que roda o que exige placa — LLM aberto, embeddings, reranking, ingestão — só quando há tráfego, escalando a zero; se você usa Bedrock, a GPU é do provedor e sua arquitetura pode ser 100% serverless normal + orquestração; e em todos os casos os tokens do LLM dominam o custo, com o serverless GPU sendo a única peça que rivaliza — e apenas quando você roda modelo próprio em volume, pois em tráfego alto e constante a GPU dedicada/reservada ainda vence o serverless GPU pelo cold start e pelo preço por hora.**

---

### Documentos relacionados

- [`01-hospedagem-e-orquestracao-llm.md`](01-hospedagem-e-orquestracao-llm.md) — panorama das 4 formas de consumo (serverless GPU é a "Opção C").
- [`02-hospedagem-inference-engine-guia.md`](02-hospedagem-inference-engine-guia.md) — aprofundamento de onde hospedar e custos.
- [`../03-rag-dados/01-bancos-vetoriais-e-documentos.md`](../03-rag-dados/01-bancos-vetoriais-e-documentos.md) — embeddings e reranking (camadas que usam GPU).
- [`../02-orquestracao/07-padroes-execucao/06-custo-latencia-throughput.md`](../02-orquestracao/07-padroes-execucao/06-custo-latencia-throughput.md) — impacto de cold start em latência/throughput.
