# Guia Didático: Como Hospedar e Consumir LLMs + Orquestração

> Do "plug-and-play" (API pronta) ao "faça você mesmo" (self-hosted), com preços,
> comparações e recomendações práticas.

Este guia usa como base o fluxo abaixo (o mesmo do diagrama):

```
┌─────────────────────┐
│  🧠 Modelo (LLM)     │   ← QUAL cérebro usar (GPT-4o, Llama, etc.)
│     GPT-4o           │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  ⚙️ Inference Engine │   ← COMO servir/executar esse cérebro
│    API do provedor   │
└──────────┬──────────┘
           │
           ▼
     (Orquestrador)        ← COMO coordenar tudo (prompts, RAG, ferramentas, memória)
```

Três perguntas separadas, muitas vezes confundidas:

1. **Modelo (LLM)** → *qual* modelo. Ex: GPT-4o, Claude, Llama 3.1, Mistral.
2. **Inference Engine** → *como* esse modelo roda e responde. Pode ser a API de um
   provedor (você não gerencia nada) ou um servidor que você mesmo sobe (vLLM, Ollama...).
3. **Orquestrador** → *como* você amarra o LLM ao seu produto: prompts, memória,
   busca em documentos (RAG), chamadas de ferramentas, múltiplos passos (agentes).

---

## 1. Conceitos rápidos (para não se perder)

| Termo | O que é | Analogia |
|---|---|---|
| **LLM** | O modelo treinado que gera texto | O cérebro |
| **Inference Engine** | Software que roda o modelo e responde requisições | O motor que dá vida ao cérebro |
| **Endpoint / API** | O "endereço" HTTP onde você manda o prompt | A tomada onde você pluga |
| **Token** | Pedaço de texto (~4 caracteres ou ~¾ de palavra) | A unidade de cobrança |
| **Orquestrador** | Framework que coordena LLM + dados + ferramentas | O maestro |
| **RAG** | Buscar informação externa e injetar no prompt | Dar uma "cola" ao modelo |

> 💡 **Regra de ouro dos custos**: quase todos os provedores cobram por **token**,
> separando *input* (o que você envia) de *output* (o que o modelo gera). Output
> costuma ser 3–5x mais caro que input.

---

## 2. Formas de HOSPEDAR / CONSUMIR o modelo (Inference Engine)

Existe um espectro do mais fácil (e menos controle) ao mais trabalhoso (e mais controle):

```
Mais fácil / menos controle                          Mais difícil / mais controle
├──────────────┬─────────────────┬──────────────────┬────────────────────────┤
 API proprietária   API de modelo     Serverless GPU /     Self-hosted
 (OpenAI, etc.)     aberto (hosted)   Inference endpoints  (sua GPU / seu servidor)
```

### 2.1. Opção A — API proprietária (mais simples)

Você chama o modelo pela API do dono. Não gerencia infraestrutura nenhuma.

**Exemplos:** OpenAI (GPT-4o), Anthropic (Claude), Google (Gemini).

- ✅ Zero infraestrutura, melhor qualidade de ponta, escala automática.
- ✅ Ideal para começar e para produtos que precisam do "melhor modelo".
- ❌ Custo por token pode escalar rápido em volume alto.
- ❌ Seus dados saem para o provedor (verifique política de retenção/treino).
- ❌ Lock-in: cada API tem particularidades.

### 2.2. Opção B — API de modelo aberto hospedado por terceiros

Modelos open-weight (Llama, Mistral, Qwen, DeepSeek) servidos por plataformas que
cuidam da infra por você. Você paga por token, mas usa modelo aberto.

**Exemplos:** Together AI, Fireworks AI, Groq, Replicate, AWS Bedrock, Azure AI, OpenRouter.

- ✅ Mais barato que os modelos proprietários de topo.
- ✅ Pode trocar de provedor mantendo o mesmo modelo (menos lock-in).
- ✅ Groq entrega latência absurdamente baixa (hardware LPU).
- ❌ Qualidade geralmente abaixo do GPT-4o/Claude Opus nos casos mais difíceis.

### 2.3. Opção C — Serverless GPU / Inference Endpoints

Você sobe *o seu* modelo (ou um aberto) e a plataforma cuida do autoscaling de GPU.
Paga por segundo/hora de GPU usada, geralmente com "scale to zero".

**Exemplos:** Modal, RunPod, Replicate, HuggingFace Inference Endpoints, AWS SageMaker.

- ✅ Controle do modelo + sem gerenciar cluster.
- ✅ Bom para modelos fine-tunados próprios.
- ❌ Cold start (demora para "acordar" a GPU).
- ❌ Você paga a GPU ligada, não por token — pode sair caro se ociosa.

### 2.4. Opção D — Self-hosted (máximo controle)

Você mesmo roda o inference engine na sua GPU (on-premise ou cloud alugada).

**Engines populares:**

| Engine | Para quê | Perfil |
|---|---|---|
| **Ollama** | Rodar local no laptop/servidor, fácil | Dev/protótipo/local |
| **llama.cpp** | CPU ou GPU modesta, modelos quantizados | Edge / hardware limitado |
| **vLLM** | Produção, alta vazão (throughput) | Produção séria |
| **TGI** (HF) | Produção, integrado ao ecossistema HF | Produção |
| **LM Studio** | GUI amigável no desktop | Desktop/experimentação |
| **SGLang** | Alta performance, features avançadas | Produção avançada |

- ✅ Custo previsível, privacidade total, sem lock-in, sem limite de taxa.
- ✅ Melhor custo em volume MUITO alto e constante.
- ❌ Você opera tudo: GPU, disponibilidade, atualizações, escala.
- ❌ GPU boa é cara (comprar ou alugar). Requer time com conhecimento.

> 💡 **Quantização** (GGUF Q4/Q5, AWQ, GPTQ) reduz o tamanho do modelo para caber
> em GPUs menores, com pequena perda de qualidade. Essencial para self-hosted barato.

---

## 3. Preços — visão comparativa

> ⚠️ **Preços mudam com frequência.** Os valores abaixo são ordens de grandeza para
> comparação (USD por **1 milhão de tokens**, aproximados, meados de 2025).
> **Sempre confirme na página oficial de pricing antes de decidir.**

### 3.1. APIs proprietárias (por 1M tokens)

| Modelo | Input (~) | Output (~) | Observação |
|---|---|---|---|
| GPT-4o | $2,50 | $10,00 | Topo de linha OpenAI |
| GPT-4o mini | $0,15 | $0,60 | Ótimo custo-benefício |
| Claude 3.5 Sonnet | $3,00 | $15,00 | Forte em código/raciocínio |
| Claude 3.5 Haiku | $0,80 | $4,00 | Rápido e barato |
| Gemini 1.5 Pro | $1,25 | $5,00 | Contexto gigante (1M+ tokens) |
| Gemini 1.5 Flash | $0,075 | $0,30 | Muito barato |

### 3.2. Modelos abertos hospedados (por 1M tokens)

| Modelo (hosted) | Input (~) | Output (~) | Observação |
|---|---|---|---|
| Llama 3.1 8B | $0,10 | $0,10 | Barato, bom p/ tarefas simples |
| Llama 3.1 70B | $0,60 | $0,80 | Bom equilíbrio |
| Llama 3.1 405B | $3,00 | $3,00 | Competitivo com topo |
| Mistral / Mixtral | $0,20–$0,60 | similar | Eficiente |
| DeepSeek V3 | ~$0,30 | ~$1,20 | Excelente custo-benefício |

### 3.3. Self-hosted (por GPU, não por token)

Aqui você paga **infraestrutura**, não tokens.

| Recurso | Custo aproximado | Comentário |
|---|---|---|
| GPU cloud (A100 80GB) alugada | ~$1,50–$3,50 / hora | ~$1.000–$2.500/mês se 24/7 |
| GPU cloud (H100) alugada | ~$2,50–$8,00 / hora | Mais rápida, mais cara |
| GPU consumer (RTX 4090) comprar | ~$1.800 (uma vez) | Roda modelos até ~30B quantizados |
| Ollama no seu laptop (M-series) | Custo de energia | Modelos 7B–13B rodam bem |

> 💡 **Quando self-hosted compensa?** Quando o volume é alto e constante. Regra prática:
> se você gasta mais de ~$1.500–$2.000/mês em API e tem uso contínuo, vale simular o
> custo de uma GPU dedicada. Abaixo disso, API quase sempre é mais barata *e* mais fácil.

---

## 4. O ORQUESTRADOR (a camada que amarra tudo)

O modelo sozinho só "completa texto". O orquestrador dá a ele memória, acesso a dados
e ferramentas. Também há um espectro do "pronto" ao "faça você mesmo".

### 4.1. Frameworks de orquestração (você codifica, roda onde quiser)

| Framework | Forte em | Perfil |
|---|---|---|
| **LangChain** | Ecossistema amplo, integrações | Genérico, muito popular |
| **LlamaIndex** | RAG / indexação de documentos | Busca em dados |
| **LangGraph** | Fluxos com estado, agentes | Agentes complexos |
| **Haystack** | Pipelines de busca/RAG em produção | Empresas |
| **Semantic Kernel** | Ecossistema Microsoft/.NET | Stack MS |
| **DSPy** | Otimização automática de prompts | Pesquisa/avançado |

- ✅ Flexibilidade total, sem custo de licença, você controla o deploy.
- ❌ Você mantém o código e a infra.

### 4.2. Plataformas de orquestração gerenciadas (menos código)

| Plataforma | O que oferece |
|---|---|
| **AWS Bedrock (Agents/KB)** | Modelos + RAG + agentes gerenciados na AWS |
| **Azure AI Studio** | Orquestração + modelos no ecossistema Azure |
| **Google Vertex AI** | Modelos Gemini + pipelines gerenciados |
| **OpenAI Assistants API** | Memória, ferramentas e RAG geridos pela OpenAI |
| **Dify / Flowise / n8n** | Orquestração visual (low-code / no-code) |

- ✅ Menos infra para gerenciar, rápido para chegar ao produto.
- ❌ Lock-in maior, menos flexibilidade fina, custo do serviço gerenciado.

### 4.3. Roteadores / gateways de LLM

Camada extra útil: um "gateway" que decide para qual modelo/provedor mandar cada
requisição (por custo, latência ou fallback).

- **OpenRouter** — uma API única para dezenas de modelos.
- **LiteLLM** — biblioteca/proxy que unifica APIs (OpenAI, Anthropic, Bedrock...).
- ✅ Troca de modelo sem reescrever código; fallback se um provedor cair.

---

## 5. Comparação lado a lado (resumo)

| Critério | API proprietária | Aberto hospedado | Serverless GPU | Self-hosted |
|---|---|---|---|---|
| Facilidade de começar | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| Custo em volume baixo | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐ |
| Custo em volume alto | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Privacidade dos dados | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Qualidade máxima | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Controle / customização | ⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Esforço operacional | Nenhum | Baixo | Médio | Alto |

---

## 6. Recomendações por cenário

### 🚀 Estou começando / protótipo / MVP
- **Modelo:** GPT-4o mini ou Gemini Flash (baratos e bons).
- **Inference:** API proprietária (Opção A).
- **Orquestrador:** LangChain ou LlamaIndex se precisar de RAG; senão, chamada direta.
- **Por quê:** velocidade máxima para validar a ideia, custo baixo, zero infra.

### 📈 Produto em produção com volume médio
- **Modelo:** GPT-4o / Claude 3.5 Sonnet para tarefas difíceis; modelo menor para o resto.
- **Inference:** API + um gateway (LiteLLM/OpenRouter) para fallback e troca fácil.
- **Orquestrador:** LangGraph/LlamaIndex, ou Bedrock/Vertex se já estiver na cloud.
- **Por quê:** confiabilidade e flexibilidade sem assumir operação de GPU.

### 🔒 Dados sensíveis / exigência de privacidade
- **Modelo:** Llama 3.1 / Mistral / Qwen (open-weight).
- **Inference:** Self-hosted com vLLM ou TGI (Opção D), dentro da sua rede.
- **Orquestrador:** Frameworks open-source (LangChain/Haystack) auto-hospedados.
- **Por quê:** os dados nunca saem do seu ambiente.

### 💰 Volume MUITO alto e constante (otimizar custo)
- **Modelo:** Llama 3.1 70B ou DeepSeek quantizado.
- **Inference:** Self-hosted com vLLM em GPU dedicada (comprada ou alugada 24/7).
- **Orquestrador:** open-source + LiteLLM.
- **Por quê:** custo por token cai drasticamente quando a GPU fica sempre ocupada.

### 🧑‍💻 Uso local / desenvolvedor individual
- **Modelo:** Llama 3.1 8B, Mistral 7B, Qwen (quantizados).
- **Inference:** Ollama ou LM Studio no próprio computador.
- **Orquestrador:** LangChain local apontando para o Ollama.
- **Por quê:** grátis (fora energia), privado, ótimo para aprender e prototipar.

---

## 7. Checklist de decisão

1. **Preciso da máxima qualidade?** → API proprietária (GPT-4o/Claude).
2. **Meus dados podem sair da minha rede?** → Não → self-hosted. Sim → pode usar API.
3. **Meu volume é alto e constante?** → Sim → simule custo de GPU dedicada.
4. **Tenho time para operar GPU/infra?** → Não → fique em API/gerenciado.
5. **Quero evitar lock-in?** → Use modelo aberto + gateway (LiteLLM/OpenRouter).
6. **Latência é crítica?** → Considere Groq (hosted) ou GPU dedicada próxima ao usuário.

---

## 8. Erros comuns a evitar

- ❌ Ir direto para self-hosted "para economizar" com volume baixo → sai mais caro e complexo.
- ❌ Esquecer que **output custa mais que input** ao estimar orçamento.
- ❌ Não medir tokens reais antes de escolher — faça um piloto e meça o consumo.
- ❌ Colar em um único provedor sem camada de abstração (gateway) → migração dolorosa.
- ❌ Ignorar política de retenção/treino de dados do provedor em casos sensíveis.
- ❌ Escolher o modelo mais caro para *todas* as tarefas — use modelo menor onde der.

---

## 9. Glossário rápido

- **Token**: unidade de texto usada para cobrança (~¾ de palavra em inglês).
- **Context window**: quantos tokens o modelo "lembra" por requisição.
- **Latência**: tempo até a primeira resposta / tempo total.
- **Throughput**: quantas requisições/tokens por segundo o sistema aguenta.
- **Cold start**: atraso ao "ligar" uma GPU que estava desligada (serverless).
- **Quantização**: comprimir o modelo (ex: 16 bits → 4 bits) para caber em menos memória.
- **RAG**: Retrieval-Augmented Generation — buscar dados e injetar no prompt.
- **Lock-in**: dependência de um fornecedor que dificulta a troca.

---

*Documento de referência. Preços e modelos evoluem rápido — confirme sempre nas páginas
oficiais de pricing antes de tomar decisões de arquitetura ou orçamento.*
