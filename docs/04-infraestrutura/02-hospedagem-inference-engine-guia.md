# Hospedagem de LLM (Inference Engine) — Guia Completo

> Aprofundamento da seção 2 do guia principal
> (`01-hospedagem-e-orquestracao-llm.md`). Cobre como servir/consumir um LLM: as 4
> formas de consumo (A–D), **onde hospedar de fato** (AWS, Azure, GCP e plataformas
> especializadas), comparação de custos com contas reais e casos de uso por custo.
> Para cada opção: como funciona por dentro, quando usar, prós/contras e exemplos de código.

---

## O que é o "Inference Engine"?

No diagrama, o **Inference Engine** é o bloco ⚙️ logo abaixo do 🧠 **Modelo (LLM)**.

- O **Modelo (LLM)** é apenas um monte de pesos (arquivos numéricos). Sozinho, não faz nada.
- O **Inference Engine** é o software que:
  1. Carrega esses pesos na memória (RAM/VRAM da GPU).
  2. Recebe seu prompt (texto → tokens).
  3. Executa a "matemática" da rede neural para prever o próximo token.
  4. Repete até formar a resposta e devolve via API/HTTP.

```
Prompt ──► [Tokenizer] ──► [Inference Engine roda o modelo] ──► [Detokenizer] ──► Resposta
                                    ▲
                              Pesos do LLM
                          (na VRAM da GPU / RAM)
```

A grande decisão é: **quem opera esse engine?**
Você (self-hosted) ou um terceiro (API/serviço gerenciado)? Entre esses extremos
existem 4 grandes modelos, do mais simples ao mais complexo.

```
Mais simples / menos controle                         Mais complexo / mais controle
├───────────────┬──────────────────┬──────────────────┬──────────────────────────┤
  A. API           B. Modelo aberto     C. Serverless GPU /    D. Self-hosted
  proprietária     hospedado (3º)       Inference Endpoints    (sua GPU)
```

---

## Opção A — API Proprietária

### Como funciona
Você manda um POST para o endpoint do dono do modelo (OpenAI, Anthropic, Google).
Toda a infraestrutura — GPUs, escala, disponibilidade, atualizações — é deles.
Você só envia o prompt e recebe a resposta. Paga por token consumido.

```
Seu app ──HTTPS──► api.openai.com ──► (infra da OpenAy) ──► resposta
```

### Exemplo de código (OpenAI)
```python
from openai import OpenAI
client = OpenAI(api_key="sk-...")

resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Explique RAG em 1 frase."}],
)
print(resp.choices[0].message.content)
```

### Quem oferece
| Provedor | Modelos principais | Diferencial |
|---|---|---|
| OpenAI | GPT-4o, GPT-4o mini, o1 | Ecossistema maduro, tooling forte |
| Anthropic | Claude 3.5 Sonnet/Haiku, Opus | Excelente em código e raciocínio |
| Google | Gemini 1.5 Pro/Flash | Contexto gigante (1M+ tokens) |
| xAI | Grok | Integração com X |
| Cohere | Command R+ | Foco em RAG empresarial |

### Prós
- ✅ **Zero infraestrutura** — nada para instalar ou manter.
- ✅ **Melhor qualidade** disponível no mercado.
- ✅ **Escala automática** — aguenta picos sem você fazer nada.
- ✅ **Rápido para começar** — funciona em minutos.
- ✅ Recursos extras prontos: visão, áudio, function calling, structured output.

### Contras
- ❌ **Custo por token** escala rápido em volume alto.
- ❌ **Dados saem** do seu ambiente (verifique retenção/treino).
- ❌ **Lock-in** de API e comportamento do modelo.
- ❌ **Rate limits** e possível indisponibilidade fora do seu controle.
- ❌ Não dá para customizar o engine (batching, quantização, etc.).

### Custo típico
Por token. Ex: GPT-4o ~$2,50 input / $10 output por 1M tokens.
GPT-4o mini ~$0,15 / $0,60. (Confirme sempre no pricing oficial.)

### Quando escolher
Protótipos, MVPs, produtos que precisam da máxima qualidade, times sem
especialista em infra, volumes baixos a médios.

---

## Opção B — Modelo Aberto Hospedado por Terceiros

### Como funciona
Modelos *open-weight* (Llama, Mistral, Qwen, DeepSeek, Gemma) são servidos por
plataformas que cuidam da infra. Você paga por token, como na Opção A, mas usa um
modelo aberto — o que reduz lock-in (o mesmo modelo existe em vários provedores).

```
Seu app ──HTTPS──► api.together.ai (ou Groq, Fireworks...) ──► Llama 3.1 ──► resposta
```

### Exemplo de código (compatível com o SDK da OpenAI)
```python
from openai import OpenAI
client = OpenAI(
    api_key="...",
    base_url="https://api.together.xyz/v1",  # só muda a base_url!
)
resp = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3.1-70B-Instruct-Turbo",
    messages=[{"role": "user", "content": "Olá!"}],
)
```
> 💡 Muitos provedores expõem uma API **compatível com a da OpenAI**. Trocar de
> provedor costuma ser só mudar `base_url` + `model`.

### Quem oferece
| Provedor | Diferencial |
|---|---|
| **Together AI** | Catálogo amplo de modelos abertos, bom preço |
| **Fireworks AI** | Rápido, foco em produção |
| **Groq** | Latência ultrabaixa (hardware LPU dedicado) |
| **Replicate** | Fácil, paga por execução, muitos modelos |
| **AWS Bedrock** | Modelos abertos + proprietários dentro da AWS |
| **Azure AI / Vertex AI** | Idem nos ecossistemas MS/Google |
| **OpenRouter** | Gateway: uma API para dezenas de modelos/provedores |

### Prós
- ✅ **Mais barato** que os modelos proprietários de topo.
- ✅ **Menos lock-in** — o modelo é aberto e existe em vários lugares.
- ✅ **Ainda sem infra** para gerenciar.
- ✅ Groq entrega latência que a Opção A dificilmente alcança.
- ✅ Alguns permitem **fine-tuning** hospedado.

### Contras
- ❌ Qualidade geralmente **abaixo** do GPU-4o/Claude nos casos mais difíceis.
- ❌ Dados ainda saem do seu ambiente.
- ❌ Disponibilidade/latência dependem do provedor.

### Custo típico
Por token, mais barato. Ex: Llama 3.1 8B ~$0,10/$0,10; 70B ~$0,60/$0,80;
405B ~$3/$3 por 1M tokens.

### Quando escolher
Quando quer bom custo-benefício, evitar lock-in, ou precisa de latência
mínima (Groq), sem assumir operação de GPU.

---

## Opção C — Serverless GPU / Inference Endpoints

### Como funciona
Você **sobe o seu próprio modelo** (aberto ou fine-tunado por você) e a plataforma
provisiona GPUs sob demanda, com autoscaling e (geralmente) "scale to zero".
Você paga por **tempo de GPU** (segundo/hora), não por token.

```
Requisição ──► plataforma acorda uma GPU ──► carrega seu modelo ──► responde
                        │
                 (se ociosa, desliga → você para de pagar)
```

### Quem oferece
| Plataforma | Diferencial |
|---|---|
| **Modal** | Deploy via código Python, ótimo DX |
| **RunPod** | GPUs baratas, serverless e dedicadas |
| **Replicate** | Publicar modelo como API facilmente |
| **HuggingFace Inference Endpoints** | Integrado ao Hub da HF |
| **AWS SageMaker** | Endpoints gerenciados na AWS |
| **Baseten / Beam** | Foco em deploy de ML serverless |

### Exemplo conceitual (Modal)
```python
import modal
app = modal.App("meu-llm")

@app.function(gpu="A100")
def gerar(prompt: str):
    # aqui você carrega o modelo (vLLM/transformers) e roda a inferência
    ...
```

### Prós
- ✅ **Controle do modelo** (inclusive fine-tunado) sem gerenciar cluster.
- ✅ **Scale to zero** — não paga quando ninguém usa.
- ✅ Bom meio-termo entre "API pronta" e "self-hosted puro".

### Contras
- ❌ **Cold start** — a primeira requisição após ociosidade demora (carregar modelo na GPU).
- ❌ Você paga **GPU ligada**, não por token → pode sair caro se ficar ociosa acesa.
- ❌ Exige configurar o engine (vLLM, TGI) e a imagem de deploy.

### Custo típico
Por hora de GPU. Ex: A100 80GB ~$1,50–$3,50/h; H100 ~$2,50–$8/h.
"Scale to zero" ajuda em cargas intermitentes.

### Quando escolher
Modelo próprio/fine-tunado, cargas intermitentes (picos), quando você quer
controle do modelo mas não quer operar servidor 24/7.

---

## Opção D — Self-Hosted (máximo controle)

### Como funciona
Você mesmo roda o inference engine na sua GPU — on-premise ou em uma VM com GPU
que você aluga na cloud. Você controla tudo: modelo, engine, batching, quantização,
disponibilidade, atualizações.

```
Seu servidor/GPU:  [vLLM/Ollama/TGI] ──serve──► http://localhost:8000/v1/...
                          ▲
                    Pesos do modelo (no seu disco/VRAM)
```

### Engines disponíveis (comparados)

| Engine | Melhor para | Hardware | Facilidade | Throughput |
|---|---|---|---|---|
| **Ollama** | Local, protótipo, DX simples | CPU/GPU modesta | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **LM Studio** | Desktop com GUI | CPU/GPU consumer | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **llama.cpp** | Edge, CPU, hardware limitado | CPU/GPU pequena | ⭐⭐⭐ | ⭐⭐ |
| **vLLM** | **Produção séria, alta vazão** | GPU (A100/H100) | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **TGI** (HuggingFace) | Produção, ecossistema HF | GPU | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **SGLang** | Produção avançada, features novas | GPU | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **TensorRT-LLM** | Máxima performance NVIDIA | GPU NVIDIA | ⭐ | ⭐⭐⭐⭐⭐ |

### Exemplos de código

**Ollama (o mais simples — ótimo para começar):**
```bash
# instala, baixa e roda um modelo com um comando
ollama run llama3.1
# ou serve via API compatível com OpenAI em http://localhost:11434/v1
```
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
resp = client.chat.completions.create(
    model="llama3.1",
    messages=[{"role": "user", "content": "Oi!"}],
)
```

**vLLM (produção, alta vazão):**
```bash
pip install vllm
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Meta-Llama-3.1-8B-Instruct
# expõe API compatível com OpenAI em http://localhost:8000/v1
```

### Conceito-chave: Quantização
Reduz a precisão dos pesos (ex: 16 bits → 4 bits) para o modelo caber em GPUs menores.

| Formato | Uso |
|---|---|
| FP16 / BF16 | Precisão total, precisa de mais VRAM |
| **GGUF (Q4/Q5/Q8)** | llama.cpp/Ollama, roda em hardware modesto |
| **AWQ / GPTQ** | Quantização para GPU (vLLM/TGI) |

> Regra prática de VRAM (aproximada): um modelo de N bilhões de parâmetros precisa de
> ~2N GB em FP16, ~N GB em 8 bits, ~N/2 GB em 4 bits.
> Ex: Llama 70B ≈ 140 GB (FP16) → ~40 GB (Q4). Por isso quantização é essencial.

### Prós
- ✅ **Custo previsível** (paga a GPU, não tokens).
- ✅ **Privacidade total** — dados nunca saem da sua rede.
- ✅ **Sem lock-in**, sem rate limit, controle fino (batching, cache, quantização).
- ✅ Melhor custo em **volume alto e constante**.

### Contras
- ❌ **Você opera tudo**: GPU, uptime, escala, patches, monitoramento.
- ❌ GPU boa é **cara** (comprar ou alugar 24/7).
- ❌ Requer **conhecimento** de ML ops e infra.
- ❌ Qualidade limitada aos modelos abertos disponíveis.

### Custo típico
- GPU cloud A100 80GB: ~$1,50–$3,50/h → ~$1.000–$2.500/mês se 24/7.
- GPU consumer RTX 4090 (comprar): ~$1.800 uma vez, roda modelos até ~30B quantizados.
- Ollama em Mac M-series: só o custo de energia (modelos 7B–13B rodam bem).

### Quando escolher
Dados sensíveis, exigência regulatória, volume alto e constante, necessidade de
customização profunda, ou uso local de desenvolvedor.

---

## Onde efetivamente rodar cada opção — infraestrutura de mercado

As opções A–D dizem *o modelo de consumo*. Esta seção mostra **onde** você
efetivamente hospeda, com serviços realmente usados no mercado. Foquei nos que são
viáveis e comuns para LLM — serviços inadequados (ex: rodar LLM grande em Lambda,
que tem limite de memória/tempo e sem GPU adequada) foram omitidos de propósito.

### AWS (o que o mercado usa para LLM)

| Serviço | Para qual opção | Quando usar | Observação |
|---|---|---|---|
| **Amazon Bedrock** | A / B (gerenciado) | Consumir Claude, Llama, Titan, Mistral por API sem infra | Padrão AWS para LLM gerenciado; tem RAG (Knowledge Bases) e Agents |
| **Amazon SageMaker (Endpoints)** | C / D | Servir modelo aberto/fine-tunado em endpoint gerenciado com GPU | Real-time ou serverless inference; bom p/ modelo próprio |
| **EC2 com GPU** (p4d/p5 = A100/H100, g5/g6 = A10G/L4) | D | Self-hosted com vLLM/TGI, controle total | Você opera tudo; g5/g6 são o custo-benefício comum |
| **Amazon ECS / EKS + GPU** | D | Self-hosted containerizado, escala com orquestração | EKS (Kubernetes) é o padrão para produção em escala |

> 💡 Fluxo típico AWS: começa em **Bedrock** (zero infra) → migra para **SageMaker**
> ou **EKS + EC2 GPU** quando precisa de modelo próprio, privacidade ou custo em escala.
> **Não** se usa Lambda para servir o LLM em si (sem GPU/limites); Lambda serve só
> para a orquestração leve em volta.

### Microsoft Azure

| Serviço | Para qual opção | Quando usar |
|---|---|---|
| **Azure OpenAI Service** | A | Consumir GPT-4o/GPT-4 com governança e rede da Azure (muito usado por empresas) |
| **Azure AI Foundry (ex-AI Studio)** | A / B | Catálogo de modelos (abertos + proprietários) + orquestração gerenciada |
| **Azure ML (Managed Online Endpoints)** | C / D | Servir modelo próprio/aberto em endpoint gerenciado com GPU |
| **Azure VM série N (NC/ND/NV)** | D | Self-hosted em VM com GPU NVIDIA |
| **AKS + GPU** | D | Self-hosted containerizado em Kubernetes |

> 💡 Azure OpenAI é o caminho preferido de quem já é "casa Microsoft" e precisa de
> compliance/rede privada com os modelos da OpenAI.

### Google Cloud (GCP)

| Serviço | Para qual opção | Quando usar |
|---|---|---|
| **Vertex AI (Model Garden)** | A / B | Consumir Gemini + modelos abertos por API gerenciada |
| **Vertex AI Endpoints** | C / D | Servir modelo próprio/aberto com GPU/TPU gerenciados |
| **GKE + GPU/TPU** | D | Self-hosted containerizado; GCP oferece TPU além de GPU |
| **Compute Engine com GPU** | D | Self-hosted em VM |

> 💡 Diferencial do GCP: acesso a **TPU** (além de GPU), interessante para cargas
> muito grandes; e Gemini nativo no Vertex.

### Plataformas especializadas em GPU/LLM (fora das 3 grandes)

Muito usadas por serem mais simples e frequentemente mais baratas que as big clouds:

| Plataforma | Modelo de uso | Diferencial |
|---|---|---|
| **RunPod** | Serverless GPU (C) e GPU dedicada (D) | GPUs baratas, comunidade grande |
| **Modal** | Serverless GPU (C) | Deploy por código Python, DX excelente, scale-to-zero |
| **Together AI** | API aberta hospedada (B) | Catálogo grande de modelos abertos, bom preço |
| **Fireworks AI** | API aberta hospedada (B) | Foco em produção e velocidade |
| **Groq** | API aberta hospedada (B) | Latência ultrabaixa (hardware LPU) |
| **Replicate** | Serverless (C) / API (B) | Publicar/consumir modelo como API fácil |
| **HuggingFace Inference Endpoints** | Serverless/dedicado (C/D) | Integrado ao Hub, 1 clique para servir |
| **Baseten** | Serverless GPU (C) | Deploy de ML em produção |
| **Lambda Labs / CoreWeave** | GPU dedicada (D) | Clouds focadas em GPU para IA, boa disponibilidade de H100 |

### Como escolher a infraestrutura

| Sua situação | Caminho recomendado |
|---|---|
| Já uso AWS e quero rápido | Bedrock → depois SageMaker/EKS se precisar |
| Já uso Azure / casa Microsoft | Azure OpenAI → Azure ML se precisar de modelo próprio |
| Já uso GCP / quero Gemini ou TPU | Vertex AI → GKE se precisar de self-host |
| Não tenho cloud e quero simples/barato | RunPod, Modal ou Together (especializadas) |
| Preciso de muitas H100 disponíveis | CoreWeave / Lambda Labs / RunPod |
| Latência mínima (tempo real) | Groq (hosted) ou GPU dedicada própria |

> 💡 **Regra geral:** fique na cloud que você já usa (menos atrito de rede, IAM,
> billing e compliance). Só vá para uma plataforma especializada se ela resolver algo
> específico — preço de GPU, disponibilidade de H100, latência (Groq) ou simplicidade (Modal).

---

## Tabela-resumo das 4 opções

| Critério | A. API proprietária | B. Aberto hospedado | C. Serverless GPU | D. Self-hosted |
|---|---|---|---|---|
| Modelo do dono do LLM? | Sim | Não (aberto) | Seu/aberto | Seu/aberto |
| Quem opera a infra | Provedor | Provedor | Plataforma | **Você** |
| Cobrança | Por token | Por token | Por hora GPU | Por hora GPU / hardware |
| Facilidade de começar | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| Privacidade dos dados | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Qualidade máxima | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Custo em volume alto | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Controle/customização | ⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Esforço operacional | Nenhum | Baixo | Médio | Alto |
| Cold start | Não | Não | **Sim** | Não (se sempre ligado) |

---

## Fluxo de decisão

```
Os dados podem sair da sua rede?
│
├── NÃO ─────────────────────────► D. Self-hosted (vLLM/TGI na sua GPU)
│
└── SIM
     │
     Preciso da máxima qualidade possível?
     │
     ├── SIM ────────────────────► A. API proprietária (GPT-4o/Claude)
     │
     └── NÃO
          │
          O volume é alto e constante?
          │
          ├── SIM ───────────────► C. Serverless/GPU dedicada, ou D. Self-hosted
          │                          (simule custo de GPU vs. tokens)
          │
          └── NÃO ───────────────► B. Modelo aberto hospedado
                                     (Together/Groq/OpenRouter)
```

---

## Comparação de custos (com contas reais)

> ⚠️ Valores aproximados (meados de 2025) só para ordem de grandeza. Confirme no
> pricing oficial. O objetivo aqui é **aprender a fazer a conta**, não decorar preços.

### Como estimar o custo por token

A conta base é sempre a mesma:

```
custo = (tokens_input / 1.000.000 × preço_input)
      + (tokens_output / 1.000.000 × preço_output)
```

**Quanto é 1 requisição?** Referências úteis:
- 1 token ≈ ¾ de palavra (inglês) / ~0,5 palavra (português).
- 1 página de texto ≈ 500–800 tokens.
- Uma pergunta curta + resposta média ≈ 500 input + 500 output = 1.000 tokens.

**Exemplo de 1 chamada** (500 input + 500 output) em cada modelo:

| Modelo | Conta | Custo por chamada |
|---|---|---|
| GPT-4o | (500/1M×$2,50)+(500/1M×$10) | ~$0,00625 |
| GPT-4o mini | (500/1M×$0,15)+(500/1M×$0,60) | ~$0,000375 |
| Claude 3.5 Sonnet | (500/1M×$3)+(500/1M×$15) | ~$0,009 |
| Llama 3.1 70B (hosted) | (500/1M×$0,60)+(500/1M×$0,80) | ~$0,0007 |
| Llama 3.1 8B (hosted) | (500/1M×$0,10)+(500/1M×$0,10) | ~$0,0001 |

> 💡 GPT-4o custa ~62x mais que Llama 8B por chamada. Isso NÃO significa que Llama
> 8B é a melhor escolha — significa que você deve usar o modelo certo para cada tarefa.

### Custo mensal por volume (modelo em API vs. self-hosted)

Cenário: cada requisição = 1.000 tokens (500 in + 500 out). Comparando o custo mensal.

| Requisições/mês | GPT-4o | GPT-4o mini | Llama 70B (API) | Self-hosted A100 (24/7) |
|---|---|---|---|---|
| 10 mil | ~$63 | ~$4 | ~$7 | ~$1.500 (fixo) |
| 100 mil | ~$625 | ~$38 | ~$70 | ~$1.500 (fixo) |
| 1 milhão | ~$6.250 | ~$375 | ~$700 | ~$1.500 (fixo) |
| 10 milhões | ~$62.500 | ~$3.750 | ~$7.000 | ~$1.500–3.000* |

\* Self-hosted vira uma **linha quase horizontal**: o custo é fixo (a GPU) até saturar;
para 10M req talvez precise de 2 GPUs (~$3.000). API é uma **reta crescente**: dobrar
o volume dobra o custo.

### O ponto de virada (break-even)

```
Custo
  │                                    ╱ API proprietária (GPT-4o)
  │                                  ╱
  │                                ╱
  │                              ╱
  │        ┌──────────────────╱───────────  Self-hosted (GPU fixa)
  │        │                ╱
  │        │              ╱   ← ponto de virada
  │        │            ╱
  │________|__________╱___________________________► Volume (req/mês)
           └── abaixo daqui: API é mais barata
               acima daqui: self-hosted é mais barato
```

**Regra prática:** se você gasta mais de ~$1.500–2.000/mês em API **e** o uso é
constante (não picos), simule o custo de uma GPU dedicada. Abaixo disso, API quase
sempre ganha em custo total (lembre que self-hosted tem custo escondido: pessoas
operando, monitoramento, downtime).

### Custos escondidos que a tabela não mostra

| Opção | Custo aparente | Custos escondidos |
|---|---|---|
| API proprietária | Por token | Rate limits (perda de vendas em pico), lock-in |
| Aberto hospedado | Por token | Latência variável, qualidade menor → mais retrabalho |
| Serverless GPU | Por hora | **Cold start** (pior UX), GPU ociosa acesa |
| Self-hosted | GPU/mês | **Equipe de ops**, energia, atualização, uptime, VRAM subdimensionada |

> 💡 O maior custo escondido do self-hosted não é a GPU — é o **tempo de engenharia**
> para manter tudo funcionando 24/7. Some ~1 pessoa parcial de DevOps/MLOps ao orçamento.

---

## Casos de uso mapeados por custo

### 💬 Chatbot de atendimento (alto volume, tarefas simples)
- **Perfil:** milhões de mensagens/mês, perguntas repetitivas, respostas curtas.
- **Custo dominante:** volume de tokens.
- **Recomendação:** GPT-4o mini / Gemini Flash / Llama 8B hospedado. Se o volume
  passar de ~1M req/mês constante, avaliar self-hosted de um modelo 8B–13B.
- **Truque de custo:** cache de respostas frequentes + modelo pequeno; escale para
  modelo grande só quando a pergunta for complexa (roteamento por dificuldade).

### 🧑‍💻 Assistente de código / análise complexa (baixo volume, alta qualidade)
- **Perfil:** poucas requisições, mas cada uma exige raciocínio forte.
- **Custo dominante:** qualidade do modelo (não o volume).
- **Recomendação:** Claude 3.5 Sonnet / GPT-4o via API proprietária. Volume é baixo,
  então o custo por chamada maior não pesa; a qualidade evita retrabalho caro.
- **Truque de custo:** não vale self-hosted aqui — o volume não justifica a GPU.

### 📄 Processamento de documentos em lote (batch, tolera latência)
- **Perfil:** milhares de documentos processados de madrugada, sem urgência.
- **Custo dominante:** volume alto, mas não em tempo real.
- **Recomendação:** APIs com **Batch API** (OpenAI/Anthropic dão ~50% de desconto em
  jobs assíncronos) ou serverless GPU (aceita cold start porque não é interativo).
- **Truque de custo:** Batch API corta o custo pela metade quando você tolera espera.

### 🔒 Dados sensíveis / regulados (saúde, jurídico, financeiro)
- **Perfil:** dados não podem sair da rede; volume variável.
- **Custo dominante:** conformidade > custo.
- **Recomendação:** Self-hosted (vLLM/TGI) mesmo que mais caro por token — o custo de
  um vazamento/multa supera qualquer economia de API. Modelo aberto 70B quantizado.
- **Truque de custo:** quantização (AWQ/GPTQ) reduz a GPU necessária pela metade.

### 📈 Startup em crescimento incerto (volume imprevisível)
- **Perfil:** pode explodir ou não; ninguém sabe ainda.
- **Custo dominante:** flexibilidade e risco.
- **Recomendação:** API proprietária + gateway (LiteLLM/OpenRouter). Custo cresce com
  o uso (você não paga GPU parada) e você troca de modelo sem retrabalho.
- **Truque de custo:** só migre para self-hosted quando o volume estabilizar alto —
  otimizar cedo demais desperdiça tempo de engenharia.

### ⚡ Aplicação sensível a latência (voz, tempo real)
- **Perfil:** resposta precisa ser quase instantânea.
- **Custo dominante:** latência (que impacta conversão/UX).
- **Recomendação:** Groq (LPU, latência mínima) hospedado; ou self-hosted com GPU
  dedicada perto do usuário (sem cold start).
- **Truque de custo:** evite serverless com scale-to-zero aqui — o cold start mata a UX.

### 🧪 Uso interno / desenvolvimento / experimentação
- **Perfil:** dev testando ideias, sem exposição ao usuário final.
- **Custo dominante:** quase zero se local.
- **Recomendação:** Ollama/LM Studio local (grátis, só energia) para o dia a dia;
  API proprietária pontual para validar a qualidade do modelo grande.
- **Truque de custo:** rode localmente enquanto prototipa, pague API só no que for pro.

### Resumo: qual otimizar por caso

| Caso de uso | Fator que domina o custo | Melhor opção de custo |
|---|---|---|
| Chatbot alto volume | Volume de tokens | Modelo pequeno + cache; self-host se ≥1M/mês |
| Assistente de código | Qualidade | API proprietária (volume baixo) |
| Batch de documentos | Volume não-interativo | Batch API (−50%) ou serverless |
| Dados sensíveis | Conformidade | Self-hosted (custo aceito) |
| Startup incerta | Flexibilidade/risco | API + gateway |
| Tempo real / voz | Latência | Groq ou GPU dedicada |
| Uso interno/dev | Praticamente nada | Ollama local |

---

## Padrão recomendado: abstraia o engine

Independentemente da opção, **não acople seu código a um único provedor**. Use um
gateway/camada de abstração para trocar de engine sem reescrever a aplicação:

- **LiteLLM** — proxy/biblioteca que fala com dezenas de provedores usando a mesma interface.
- **OpenRouter** — uma API única roteando para vários modelos/provedores.
- Ou simplesmente padronize na **API compatível com OpenAI** (a maioria dos engines suporta).

```python
# Mesmo código serve para OpenAI, Together, Ollama ou vLLM — só muda base_url/model.
client = OpenAI(base_url=BASE_URL, api_key=API_KEY)
client.chat.completions.create(model=MODEL, messages=[...])
```

Assim você pode começar barato/simples (Opção A ou B) e migrar para self-hosted
(Opção D) depois, sem retrabalho.

---

## Erros comuns nesta camada

- ❌ Pular direto para self-hosted "para economizar" com volume baixo → mais caro e complexo.
- ❌ Deixar GPU serverless ligada e ociosa → paga sem usar (cuide do scale-to-zero).
- ❌ Subestimar VRAM necessária → o modelo não carrega; use quantização.
- ❌ Acoplar o código a um provedor → migração dolorosa (use gateway/API padrão).
- ❌ Ignorar cold start em cargas com picos → latência ruim para o usuário.
- ❌ Escolher engine de produção (vLLM/TensorRT) para um protótipo → complexidade à toa.

---

*Detalhamento da seção 2 do guia principal. Preços e engines evoluem rápido —
confirme sempre nas páginas oficiais antes de decisões de arquitetura/orçamento.*
