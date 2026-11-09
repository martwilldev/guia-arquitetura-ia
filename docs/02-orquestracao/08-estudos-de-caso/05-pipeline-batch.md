# Estudo de Caso 5 — Pipeline Batch de Processamento de Documentos

> **Cenário:** todo fim de dia chegam **milhares de documentos** (contratos, notas fiscais, laudos) que precisam ser **extraídos**, **resumidos** e **classificados** antes da manhã seguinte. Não é chat em tempo real — é **processamento em lote**, onde **custo por documento** e **throughput** importam mais que latência.

> 🔙 Volte ao índice dos estudos de caso: [README.md](./README.md)

---

## Índice

1. [Problema, produto e requisitos](#1-problema-produto-e-requisitos)
2. [Arquitetura de referência](#2-arquitetura-de-referência)
3. [As três alavancas de custo em batch](#3-as-três-alavancas-de-custo-em-batch)
4. [Opção A — Lambda fan-out + SQS + Bedrock on-demand (modelo barato) 🟡](#4-opção-a--lambda-fan-out--sqs--bedrock-on-demand-modelo-barato-)
5. [Opção B — Step Functions Map + Bedrock Batch (modelo médio, com desconto) 🟡](#5-opção-b--step-functions-map--bedrock-batch-modelo-médio-com-desconto-)
6. [Opção C — Fargate/AWS Batch + modelo aberto em serverless GPU 🔴](#6-opção-c--fargateaws-batch--modelo-aberto-em-serverless-gpu-)
7. [Tabela comparativa](#7-tabela-comparativa)
8. [Recomendação final por perfil](#8-recomendação-final-por-perfil)
9. [Armadilhas](#9-armadilhas)
10. [Glossário](#10-glossário)
11. [Resumo em uma frase](#11-resumo-em-uma-frase)

---

## 1. Problema, produto e requisitos

Uma seguradora recebe documentos ao longo do dia e quer que **cada documento** passe por três etapas: (1) **extração** de campos-chave, (2) **resumo** de 3 linhas, (3) **classificação** por tipo/risco. O resultado alimenta o sistema interno e um índice de busca.

| Requisito | Alvo |
|-----------|------|
| **Volume** | ~500.000 documentos/mês (~16.700/dia, com picos noturnos) |
| **Tamanho médio** | ~3.000 tokens de entrada por documento |
| **Saída** | ~500 tokens (campos + resumo + rótulo) |
| **Janela** | Processar o lote da noite em **até 6 h** (não precisa ser instantâneo) |
| **Qualidade** | Extração correta > estilo; classificação com acurácia mensurável |
| **Restrições** | Custo por documento baixo; reprocessamento idempotente; observabilidade do lote |

> 💡 **Insight central:** batch libera duas economias que o tempo real não permite — **APIs de batch com desconto** (Bedrock Batch) e **GPU própria amortizada em volume** (modelo aberto). O trade-off é sempre **complexidade operacional**.

---

## 2. Arquitetura de referência

```
   Documentos (S3: raw/)
        │  (evento ou agendamento noturno)
        ▼
   ┌──────────────────────┐
   │  DISPARADOR           │  EventBridge (cron) ou S3 event
   └──────────┬───────────┘
              ▼
   ┌──────────────────────┐        ┌───────────────────────┐
   │  DISTRIBUIÇÃO         │ ─────► │  WORKERS (processam)   │
   │  (fan-out / Map)      │        │  extrair→resumir→class.│
   └──────────────────────┘        └───────────┬───────────┘
                                                ▼
                                    ┌───────────────────────┐
                                    │  LLM (Bedrock/aberto)  │
                                    └───────────┬───────────┘
                                                ▼
                                    ┌───────────────────────┐
                                    │  Resultado: S3 done/   │
                                    │  + DynamoDB (status)   │
                                    │  + índice de busca     │
                                    └───────────────────────┘
```

As opções mudam **como os workers e o LLM são implementados**: enxame de Lambdas + API on-demand (A), job de batch com desconto (B), ou frota GPU com modelo aberto (C).

Todos os valores abaixo são **ILUSTRATIVOS**. Confirme na [Calculadora AWS](https://calculator.aws) e nos [preços do Bedrock](https://aws.amazon.com/bedrock/pricing/).

---

## 3. As três alavancas de custo em batch

```
   1) PREÇO DO TOKEN     → modelo barato vs médio vs aberto (self-host)
   2) DESCONTO DE BATCH  → Bedrock Batch costuma dar ~50% off vs on-demand
   3) AMORTIZAÇÃO DE GPU → GPU própria só compensa em VOLUME ALTO e constante
```

Referência de volume deste caso: **500k docs/mês × ~3,5k tokens ≈ 1,75 bilhão de tokens/mês** de entrada (+ ~0,25 B de saída). Nessa faixa, escolher a alavanca certa muda a conta em **ordem de grandeza**.

---

## 4. Opção A — Lambda fan-out + SQS + Bedrock on-demand (modelo barato) 🟡

### Como funciona

Um disparador lê a fila de documentos e faz **fan-out**: publica uma mensagem por documento em **SQS**; um **Lambda** consome em paralelo (concorrência controlada) e chama o **Bedrock on-demand** com um **modelo barato** (Nova Lite / Claude Haiku). SQS dá retry, DLQ e desacoplamento. Simples e elástico.

```
S3 raw/ ──► Lambda "lister" ──► SQS (1 msg/doc) ──► Lambda worker (N em paralelo)
                                                     └► Bedrock on-demand (Haiku/Nova Lite)
                                                     └► S3 done/ + DynamoDB status
```

### Exemplo de código (Lambda-ready)

```python
# worker.py — cliente FORA do handler
import json, boto3

s3 = boto3.client("s3")
bedrock = boto3.client("bedrock-runtime")
ddb = boto3.client("dynamodb")
MODEL = "us.amazon.nova-lite-v1:0"

PROMPT = (
    "Extraia os campos-chave, gere um resumo de 3 linhas e classifique o tipo/risco. "
    "Responda em JSON com as chaves: campos, resumo, tipo, risco.\n\nDOC:\n{texto}"
)

def handler(event, _ctx):
    for record in event["Records"]:          # SQS entrega em lotes
        msg = json.loads(record["body"])
        doc = s3.get_object(Bucket=msg["bucket"], Key=msg["key"])["Body"].read().decode()
        resp = bedrock.converse(
            modelId=MODEL,
            messages=[{"role": "user", "content": [{"text": PROMPT.format(texto=doc[:12000])}]}],
            inferenceConfig={"maxTokens": 600, "temperature": 0},
        )
        out = resp["output"]["message"]["content"][0]["text"]
        s3.put_object(Bucket=msg["bucket"], Key=msg["key"].replace("raw/", "done/") + ".json", Body=out)
        ddb.put_item(TableName="doc_status",
                     Item={"key": {"S": msg["key"]}, "status": {"S": "done"}})
    return {"ok": True}
```

### Custo estimado mensal (ILUSTRATIVO)

Premissa: 500k docs × 3,5k in + 0,5k out. Modelo barato (Nova Lite ~US$0,06 in / US$0,24 out por M — verifique).

| Item | Conta | Custo/mês |
|------|-------|-----------|
| Tokens de entrada | 500k × 3,5k = 1,75 B @ ~US$0,06/M | ~US$ 105 |
| Tokens de saída | 500k × 0,5k = 250 M @ ~US$0,24/M | ~US$ 60 |
| Lambda | 500k invocações × ~3 s × 512MB | ~US$ 40 |
| SQS | 500k msgs (1M req grátis inicial) | ~US$ 1 |
| S3 + DynamoDB | armazenamento + status | ~US$ 25 |
| **Total** | | **~US$ 231/mês** |

### Complexidade: 🟡 média

### Trade-offs

- ✅ **Elástico e simples**; SQS resolve retry/DLQ/backpressure.
- ✅ Sem infra sempre-ligada; paga pelo que usa.
- ✅ Ótimo custo com modelo barato para tarefas objetivas.
- ⚠️ On-demand não tem desconto de batch (paga preço cheio por token).
- ⚠️ Modelo barato pode não dar conta de docs muito complexos (avalie acurácia).
- ⚠️ Cuidado com **throttling** do Bedrock e limites de concorrência do Lambda.

---

## 5. Opção B — Step Functions Map + Bedrock Batch (modelo médio, com desconto) 🟡

### Como funciona

Você monta um arquivo de entrada (JSONL, um registro por documento) e submete um **job de inferência em lote** do **Bedrock Batch**, que processa tudo de forma assíncrona **com desconto** (tipicamente ~50% vs on-demand) usando um **modelo médio** (ex.: Claude Haiku/Sonnet ou Nova Pro). O **Step Functions (Distributed Map)** orquestra: prepara o JSONL, dispara o job, aguarda, coleta a saída do S3 e grava resultados.

```
Step Functions
  ├─ Prepara JSONL (S3 input)
  ├─ StartModelInvocationJob (Bedrock Batch, ~50% off)  ── assíncrono ──►
  ├─ Aguarda conclusão (polling)
  └─ Distributed Map: lê saída S3 → grava DynamoDB/índice
```

### Exemplo de config (Bedrock Batch — esboço)

```python
# submit_batch.py — roda 1x por lote (não por documento)
import boto3
bedrock = boto3.client("bedrock")

# input.jsonl no S3: {"recordId":"doc-123","modelInput":{...converse payload...}}
job = bedrock.create_model_invocation_job(
    jobName="docs-noturno-2026-09-25",
    modelId="anthropic.claude-3-haiku-20240307-v1:0",
    roleArn="arn:aws:iam::123456789012:role/BedrockBatchRole",
    inputDataConfig={"s3InputDataConfig": {
        "s3Uri": "s3://meu-bucket/batch/input/2026-09-25.jsonl"}},
    outputDataConfig={"s3OutputDataConfig": {
        "s3Uri": "s3://meu-bucket/batch/output/2026-09-25/"}},
)
print(job["jobArn"])  # Step Functions faz o polling do status até COMPLETED
```

```json
// trecho da State Machine (Step Functions) — orquestração
{
  "StartAt": "PrepararJSONL",
  "States": {
    "PrepararJSONL": {"Type": "Task", "Resource": "arn:...:prepara", "Next": "SubmeterBatch"},
    "SubmeterBatch": {"Type": "Task", "Resource": "arn:...:submit_batch", "Next": "Aguardar"},
    "Aguardar": {"Type": "Wait", "Seconds": 300, "Next": "ChecarStatus"},
    "ChecarStatus": {"Type": "Task", "Resource": "arn:...:status", "End": true}
  }
}
```

### Custo estimado mensal (ILUSTRATIVO)

Premissa: mesmo volume, modelo **médio** (Haiku ~US$0,25 in / US$1,25 out por M — verifique), com **~50% de desconto de batch**.

| Item | Conta | Custo/mês |
|------|-------|-----------|
| Tokens entrada (batch, ~50% off) | 1,75 B @ ~US$0,25/M × 0,5 | ~US$ 219 |
| Tokens saída (batch, ~50% off) | 250 M @ ~US$1,25/M × 0,5 | ~US$ 156 |
| Step Functions | transições (Distributed Map) | ~US$ 30 |
| Lambda (prep/coleta) | baixo | ~US$ 15 |
| S3 (input/output JSONL) | | ~US$ 20 |
| **Total** | | **~US$ 440/mês** |

> Modelo médio custa mais que o barato mesmo com desconto — mas entrega **mais qualidade**. Se o modelo barato (Opção A) já for suficiente, ele vence no custo; se não, o desconto de batch torna o modelo médio viável.

### Complexidade: 🟡 média

### Trade-offs

- ✅ **Desconto de batch** (~50%) reduz o custo por token do modelo médio.
- ✅ Assíncrono por natureza — perfeito para janela noturna.
- ✅ Sem gerenciar concorrência/throttling manualmente.
- ⚠️ **Não é tempo real** (jobs levam minutos a horas).
- ⚠️ Mais partes móveis (JSONL, job, polling, coleta).
- ⚠️ Reprocessar 1 documento isolado é menos direto que na Opção A.

---

## 6. Opção C — Fargate/AWS Batch + modelo aberto em serverless GPU 🔴

### Como funciona

Para **volume muito alto e constante**, hospedar um **modelo aberto** (Llama, Qwen, Mistral) num **inference engine** (vLLM/TGI) sobre **GPU** pode ficar **mais barato por token** que qualquer API — desde que a GPU fique **bem utilizada**. **AWS Batch/Fargate** orquestra os workers; a inferência roda em **serverless GPU** (SageMaker Serverless, ou plataformas como Modal/RunPod) que escala e, idealmente, **escala a zero** fora do lote.

```
AWS Batch (fila de jobs)
   └─► worker (container) ──► endpoint GPU (vLLM/TGI, modelo aberto)
                                   ▲  escala com o lote, some depois
   S3 raw/ → job array → S3 done/ + índice
```

### Exemplo de config (esboço)

```python
# worker_gpu.py — chama o endpoint self-hosted (compatível OpenAI)
import os, requests

ENDPOINT = os.environ["LLM_ENDPOINT"]          # vLLM/TGI (OpenAI-compatible)
SESSION = requests.Session()                    # reuso de conexão

def processa(texto: str) -> str:
    r = SESSION.post(f"{ENDPOINT}/v1/chat/completions", json={
        "model": "meta-llama/Llama-3.1-8B-Instruct",
        "messages": [{"role": "user",
                      "content": f"Extraia campos, resuma em 3 linhas e classifique.\n\n{texto[:12000]}"}],
        "max_tokens": 600, "temperature": 0,
    }, timeout=120)
    return r.json()["choices"][0]["message"]["content"]
```

```yaml
# infra (esboço) — endpoint GPU serverless que escala a zero
gpu_endpoint:
  engine: vLLM
  model: meta-llama/Llama-3.1-8B-Instruct
  gpu: 1x A10G           # ~US$1,0-1,5/h ILUSTRATIVO — verifique
  min_replicas: 0        # escala a zero fora do lote
  max_replicas: 8        # sobe durante a janela noturna
```

### Custo estimado mensal (ILUSTRATIVO)

Premissa: modelo 8B em vLLM num A10G processa ~alguns milhares de docs/hora por réplica. Para 500k docs, estime ~**300–400 GPU-horas/mês** concentradas nas janelas. GPU serverless ~US$1,2/h (ILUSTRATIVO). **Sem custo de token de API** (modelo é seu).

| Item | Conta | Custo/mês |
|------|-------|-----------|
| GPU (serverless, escala a zero) | ~350 GPU-h × ~US$1,2/h | ~US$ 420 |
| AWS Batch/Fargate (orquestração) | workers CPU durante o lote | ~US$ 60 |
| S3 + DynamoDB + índice | | ~US$ 30 |
| Egress/rede | | ~US$ 10 |
| Engenharia/manutenção (não-AWS) | tempo de time (não faturado aqui) | — |
| **Total AWS** | | **~US$ 520/mês** |

> ⚠️ Nesta faixa (500k/mês) a Opção A (modelo barato via API) ainda tende a **ganhar em custo puro**. A Opção C só domina quando o volume é **muito maior** (milhões/dia) e a GPU fica **quase 100% ocupada** — aí o custo por token do modelo aberto despenca. Abaixo desse ponto, você paga GPU ociosa + engenharia.

### Complexidade: 🔴 alta

### Trade-offs

- ✅ **Menor custo por token em volume muito alto e constante** (GPU amortizada).
- ✅ Controle total do modelo (fine-tune, privacidade, sem vendor lock de token).
- ✅ Sem limites de throttling de API de terceiros.
- ❌ **Complexidade operacional alta** (engine GPU, escala, cold start, upgrades).
- ❌ GPU ociosa = dinheiro jogado fora; exige utilização alta para compensar.
- ❌ Precisa de time com skill de MLOps/infra GPU.

---

## 7. Tabela comparativa

| Critério | A — Lambda+SQS+on-demand 🟡 | B — Step Functions + Bedrock Batch 🟡 | C — AWS Batch + GPU (modelo aberto) 🔴 |
|----------|-----------------------------|----------------------------------------|-----------------------------------------|
| **Custo/mês (ILUSTRATIVO, 500k docs)** | ~US$ 231 | ~US$ 440 | ~US$ 520 (AWS) + engenharia |
| **Custo por token** | 🟢 baixo (modelo barato) | 🟡 médio c/ 50% off | 🟢 muito baixo **só em volume enorme** |
| **Complexidade** | 🟡 média | 🟡 média | 🔴 alta |
| **Latência do lote** | 🟢 rápido/elástico | 🟡 minutos–horas (assíncrono) | 🟡 depende da frota GPU |
| **Throughput** | 🟡 limitado por concorrência/throttle | 🟢 alto (job gerenciado) | 🟢 altíssimo (escala GPU) |
| **Qualidade** | 🟡 boa (modelo barato) | 🟢 melhor (modelo médio) | 🟢 configurável (modelo/fine-tune) |
| **Quando escolher** | Volume médio, tarefa objetiva, quer simplicidade | Quer qualidade média + desconto, janela flexível | Volume gigante e constante, time de MLOps |

---

## 8. Recomendação final por perfil

- **MVP / validação:** **Opção A**. Lambda + SQS + modelo barato entrega o pipeline em dias, com custo mínimo. Meça a acurácia antes de complicar.
- **Crescimento:** **Opção B**. Quando a qualidade do modelo barato não bastar e o volume subir, o **Bedrock Batch** dá o modelo médio com ~50% de desconto sem você operar GPU. Melhor relação qualidade/custo/esforço para lotes noturnos.
- **Escala / volume gigante:** **Opção C**. Só quando o volume for **muito alto e constante** (a ponto de manter a GPU quase sempre ocupada) e você tiver time de MLOps. Aí o modelo aberto self-hosted vence em custo por token.

> Ponto de virada: rode a conta **por documento** em cada opção com o **seu** volume real. GPU própria só ganha depois de um limiar alto de utilização.

---

## 9. Armadilhas

- **GPU ociosa:** a maior armadilha da Opção C. Se a GPU não fica ocupada, você paga caro por token efetivo. Escala-a-zero e alta utilização são obrigatórios.
- **Confundir batch com tempo real:** Bedrock Batch é assíncrono; não use para requisições que precisam de resposta imediata.
- **Throttling silencioso (Opção A):** sem controle de concorrência, o Bedrock throttla e o lote atrasa. Ajuste `reservedConcurrency` e trate `ThrottlingException` com backoff.
- **Idempotência:** documentos reprocessados devem sobrescrever, não duplicar. Use a chave do doc como identificador e status em DynamoDB.
- **Chunking de docs grandes:** truncar em `[:12000]` perde informação. Para docs longos, quebre e agregue.
- **Medir só custo AWS:** a Opção C tem um custo **oculto de engenharia** (MLOps) que não aparece na fatura da AWS, mas é real.
- **Modelo barato demais:** economizar no modelo e errar a extração custa mais lá na frente (retrabalho, decisão errada). Valide acurácia com um conjunto rotulado.

---

## 10. Glossário

| Termo | Significado |
|-------|-------------|
| **Fan-out** | Espalhar N unidades de trabalho (1 msg por documento) para processar em paralelo. |
| **SQS** | Fila gerenciada da AWS; dá retry, DLQ e desacoplamento. |
| **Bedrock Batch** | Inferência assíncrona em lote no Bedrock, com desconto vs on-demand. |
| **Step Functions (Distributed Map)** | Orquestrador de workflow que itera sobre milhares de itens em paralelo. |
| **AWS Batch** | Serviço para rodar jobs em lote em contêineres (CPU/GPU). |
| **Serverless GPU** | GPU sob demanda que escala (idealmente a zero) — ex.: SageMaker Serverless, Modal, RunPod. |
| **vLLM/TGI** | Inference engines para servir modelos abertos com alto throughput. |
| **Idempotência** | Reprocessar o mesmo item produz o mesmo resultado, sem duplicar. |

---

## 11. Resumo em uma frase

> **Para lotes de documentos, comece com Lambda+SQS e um modelo barato; suba para Bedrock Batch (modelo médio com ~50% off) quando precisar de mais qualidade; e só migre para GPU própria com modelo aberto quando o volume for grande e constante o bastante para manter a GPU quase sempre ocupada.**
