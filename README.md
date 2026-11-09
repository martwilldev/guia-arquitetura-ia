# 🧠 Guia de Arquitetura e Engenharia de IA — Do Fundamento à Produção

> Manual técnico e trilha de estudos sobre **LLMs, Orquestração de Agentes, RAG, Infraestrutura de Inferência, Governança (FinOps/Segurança) e Adoção Corporativa** — com arquitetura prática voltada para produção e nuvem (AWS Bedrock, Lambda, Serverless GPU).

---

## 🗺️ Estrutura do Repositório

Todos os capítulos estão organizados em **6 módulos numerados** dentro de [`docs/`](docs/), com arquivos numerados na ordem recomendada de leitura. Você pode seguir a trilha linear do Módulo 1 ao 6 ou ir direto ao seu objetivo na [navegação rápida](#-navegação-rápida-por-objetivo).

```text
guia-arquitetura-ia/
├── README.md                                  ◄── Índice geral e trilha de leitura
└── docs/
    ├── 01-fundamentos/                        O que é um LLM, tensores, inference engines e fine-tuning
    │   ├── 01-o-que-significa-aprender.md
    │   ├── 02-llm-deep-dive.md
    │   ├── 03-anatomia-tecnica-llm.md
    │   ├── 04-llm-inference-engines.md
    │   └── 05-treinamento-e-fine-tuning.md
    ├── 02-orquestracao/                       Como coordenar LLMs, tools, memória e agentes
    │   ├── 01-orquestradores-estudo-didatico.md
    │   ├── 02-orquestradores-casos-de-produto.md
    │   ├── 03-orquestradores-ia-lambda.md
    │   ├── 04-frameworks-orquestracao.md
    │   ├── 05-comparacao-frameworks.md
    │   ├── 06-roteamento-de-llms.md
    │   ├── 07-padroes-execucao/               (6 capítulos: loop ReAct → múltiplos orquestradores)
    │   └── 08-estudos-de-caso/                (6 arquiteturas reais comparadas em 3 abordagens cada)
    ├── 03-rag-dados/                          Bancos vetoriais, embeddings, busca híbrida e reranking
    │   └── 01-bancos-vetoriais-e-documentos.md
    ├── 04-infraestrutura/                     Onde e como hospedar/servir modelos (APIs a Serverless GPU)
    │   ├── 01-hospedagem-e-orquestracao-llm.md
    │   ├── 02-hospedagem-inference-engine-guia.md
    │   └── 03-serverless-gpu.md
    ├── 05-governanca-seguranca/               FinOps de tokens, gateways, segurança, DLP e observabilidade
    │   ├── 01-precificacao-tokens-credenciais-aws.md
    │   ├── 02-vazamento-dados-no-chat.md
    │   ├── 03-ferramentas-precificacao-controle-tokens.md
    │   ├── 04-posicionamento-controle-tokens.md
    │   ├── 05-observabilidade-ia-arquitetura.md
    │   └── 06-omniroute-ai-gateway.md
    └── 06-adocao-empresa/                     Estratégia prática para estruturar IA em uma organização
        └── 01-guia-ia-na-empresa.md
```

---

## 🧭 Trilha de leitura lógica

A trilha vai do **conceito** (o que é um modelo) até a **produção** (custos, segurança, adoção),
passando por **como construir** (orquestração, RAG) e **onde rodar** (infraestrutura).

```text
  ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
  │ 1. FUNDAMENTOS │──►│ 2. ORQUESTRAÇÃO│──►│  3. RAG/DADOS  │
  │  "o que é"     │   │  "como coordenar"  │  "como dar dados"│
  └────────────────┘   └────────────────┘   └────────┬───────┘
                                                     │
  ┌────────────────┐   ┌────────────────┐   ┌────────▼───────┐
  │ 6. ADOÇÃO      │◄──│ 5. GOVERNANÇA  │◄──│ 4. INFRAESTRUT.│
  │  "levar p/ empresa"│  "custo/segurança" │  "onde rodar"  │
  └────────────────┘   └────────────────┘   └────────────────┘
```

---

## 📖 [Módulo 1 — Fundamentos](docs/01-fundamentos/)

> Comece aqui se você quer entender **o que é um LLM e como ele funciona por dentro**.
> Nenhum pré-requisito.

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 1.1 | [O que significa "aprender"](docs/01-fundamentos/01-o-que-significa-aprender.md) | O que quer dizer "o modelo aprendeu": ajuste de pesos, treino, sem banco de dados interno. **Leitura mais leve — bom ponto de partida.** |
| 1.2 | [LLM Deep Dive](docs/01-fundamentos/02-llm-deep-dive.md) | Guia aprofundado do que é um LLM: previsão do próximo token, o que ele consegue fazer e por quê. |
| 1.3 | [Anatomia Técnica de um Modelo LLM](docs/01-fundamentos/03-anatomia-tecnica-llm.md) | O modelo "fisicamente": arquivos de pesos, tensores, tokenizer, config. Como um modelo é, de fato, um conjunto de arquivos. |
| 1.4 | [LLMs e Inference Engines](docs/01-fundamentos/04-llm-inference-engines.md) | Ponte entre teoria e execução: tokens, e o que é um *inference engine* (o que roda o modelo). Prepara para o módulo de infraestrutura. |
| 1.5 | [Treinamento de IA e Fine-Tuning](docs/01-fundamentos/05-treinamento-e-fine-tuning.md) | As 3 fases do treinamento (pré-treino → fine-tuning → alinhamento), quando fazer fine-tuning vs. RAG vs. prompt, técnicas (LoRA/QLoRA), fine-tuning na AWS (Bedrock/SageMaker) e custos. |

**Ordem sugerida:** 1.1 → 1.2 → 1.3 → 1.4 → 1.5

---

## 📖 [Módulo 2 — Orquestração](docs/02-orquestracao/)

> O coração da coletânea: como coordenar LLMs, ferramentas, memória e agentes.
> **Pré-requisito:** Módulo 1 (ou saber o básico de LLM).

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 2.1 | [Orquestradores de IA — Estudo Didático](docs/02-orquestracao/01-orquestradores-estudo-didatico.md) | Conceito de orquestrador (o "maestro"), por que existe, componentes. **Base conceitual do módulo.** |
| 2.2 | [Orquestradores como Produto — Casos de Uso](docs/02-orquestracao/02-orquestradores-casos-de-produto.md) | Casos reais de uso, comparações e tendências de mercado. Visão de produto/negócio. |
| 2.3 | [Orquestradores de IA com AWS Lambda](docs/02-orquestracao/03-orquestradores-ia-lambda.md) | Mão na massa: construir um orquestrador do zero com SDK puro (Bedrock + Lambda + DynamoDB), agente ReAct, Step Functions, streaming. |
| 2.4 | [Frameworks de Orquestração (mercado)](docs/02-orquestracao/04-frameworks-orquestracao.md) | Panorama dos frameworks (LangChain, LlamaIndex, CrewAI, AutoGen, Bedrock Agents, Strands, etc.) e encaixe no Lambda. |
| 2.5 | [Comparação: LangChain vs LlamaIndex vs Haystack vs Bedrock Agents](docs/02-orquestracao/05-comparacao-frameworks.md) | Aprofundamento dos 4 principais + onde rodar (Lambda/EC2/ECS/Fargate) + custos. |
| 2.6 | [Roteamento de LLMs](docs/02-orquestracao/06-roteamento-de-llms.md) | Componente **dentro** do orquestrador: como decidir qual modelo usar (por custo, complexidade, semântico, cascata, fallback). |
| 2.7 | [Padrões de Execução (série)](docs/02-orquestracao/07-padroes-execucao/) | Série de 6 documentos focados: loop de agente, agente único, sub-agentes, multi-agente, múltiplos orquestradores e a análise de **custo/latência/throughput** — com exemplos, ferramentas de mercado e opções self-hosted. |
| 2.8 | [Estudos de Caso (série)](docs/02-orquestracao/08-estudos-de-caso/) | 6 cenários reais (FAQ, RAG, agente com tools, multi-agente, batch, revenda multi-tenant) resolvidos de **3 formas cada** (framework/modelo/técnica/compute), com **custo estimado** e **complexidade** lado a lado. |

**Ordem sugerida:** 2.1 → 2.2 → 2.3 → 2.4 → 2.5 → 2.6 → 2.7 → 2.8

---

## 📖 [Módulo 3 — RAG e Dados Vetoriais](docs/03-rag-dados/)

> Como dar "conhecimento" ao orquestrador com seus próprios dados.
> **Pré-requisito:** Módulo 2 (entender que o orquestrador consulta uma base de dados).

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 3.1 | [Bancos Vetoriais e Documentos](docs/03-rag-dados/01-bancos-vetoriais-e-documentos.md) | Estudo completo: embeddings, chunking, índices ANN (HNSW/IVF), busca híbrida, reranking, e catálogo de opções (SaaS, AWS, self-hosted, on-premise) com análise de custo por volume. |

---

## 📖 [Módulo 4 — Infraestrutura (onde hospedar)](docs/04-infraestrutura/)

> Onde e como servir/rodar os modelos, das APIs prontas ao self-hosted.
> **Pré-requisito:** Módulo 1 (inference engine) ajuda bastante.

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 4.1 | [Como Hospedar e Consumir LLMs + Orquestração](docs/04-infraestrutura/01-hospedagem-e-orquestracao-llm.md) | Panorama do "plug-and-play" (API pronta) ao "faça você mesmo" (self-hosted), com preços e recomendações. **Comece por aqui neste módulo.** |
| 4.2 | [Hospedagem de LLM (Inference Engine) — Guia Completo](docs/04-infraestrutura/02-hospedagem-inference-engine-guia.md) | Aprofundamento: as 4 formas de consumo, onde hospedar de fato (AWS/Azure/GCP e plataformas especializadas), comparação de custos com contas reais. |
| 4.3 | [Serverless GPU](docs/04-infraestrutura/03-serverless-gpu.md) | GPU sob demanda que escala a zero: o que é, por que aparece em várias camadas (servir LLM, embeddings, reranking, ingestão), plataformas (Modal, RunPod, Replicate, SageMaker Serverless), custos e cold start. |

**Ordem sugerida:** 4.1 → 4.2 → 4.3

---

## 📖 [Módulo 5 — Governança e Segurança](docs/05-governanca-seguranca/)

> O que separa um protótipo de um sistema em produção: custo, credenciais, acesso, segurança.
> **Pré-requisito:** Módulos 2 e 4 (ter uma arquitetura para governar).

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 5.1 | [Precificação, Tokens, Credenciais e Acesso na AWS](docs/05-governanca-seguranca/01-precificacao-tokens-credenciais-aws.md) | Camada de FinOps + segurança: precificação do Bedrock, controle de tokens/orçamento por tenant, IAM/Secrets Manager, controle de acesso, multi-tenancy, guardrails, arquitetura de referência completa. |
| 5.2 | [Vazou Senha ou Dados Pessoais no Chat com IA — E Agora?](docs/05-governanca-seguranca/02-vazamento-dados-no-chat.md) | Perspectiva do usuário/risco: o que acontece com dados sensíveis colados num chat de IA e como agir. Leitura leve e prática. |
| 5.3 | [Ferramentas para Precificação e Controle de Tokens](docs/05-governanca-seguranca/03-ferramentas-precificacao-controle-tokens.md) | **Para quem revende/oferece IA aos seus usuários:** ferramentas, serviços e produtos para medir consumo, limitar, precificar e cobrar (gateways de LLM como LiteLLM/Portkey, billing como Stripe/Lago, observabilidade, nativo AWS e DIY), comparando complexidade e custo. |
| 5.4 | [Onde Fica o Controle de Tokens no Fluxo (Posicionamento)](docs/05-governanca-seguranca/04-posicionamento-controle-tokens.md) | **Decisão arquitetural:** em que ponto do fluxo o controle de tokens deve viver — no orquestrador, numa camada antes, num gateway dedicado ou distribuído. 5 opções com comparação lado a lado e análise de custo de cada posicionamento. |
| 5.5 | [Observabilidade de IA — Arquitetura](docs/05-governanca-seguranca/05-observabilidade-ia-arquitetura.md) | **Como enxergar o sistema de IA:** o que coletar (tokens, custo, latência, qualidade, traço), como instrumentar (SDK, gateway, OpenTelemetry, logs), onde armazenar (métricas/traces/logs/warehouse), como exibir (dashboards e alertas), análise de mercado (SaaS), opções self-hosted e análise de custo. |
| 5.6 | [OmniRoute — AI Gateway Gratuito](docs/05-governanca-seguranca/06-omniroute-ai-gateway.md) | **Estudo de uma ferramenta concreta de gateway:** o que é o OmniRoute, por que ele **não é um orquestrador** (é um AI Gateway, categoria do LiteLLM/OpenRouter), onde encaixa no fluxo (Ponto 3 do doc 5.4), como usar (npm/Docker/consumo OpenAI-compatible), comparação lado a lado com outros gateways e quando usar em dev vs. produção. |

**Ordem sugerida:** 5.1 → 5.2 → 5.3 → 5.4 → 5.5 → 5.6

---

## 📖 [Módulo 6 — Adoção na Empresa](docs/06-adocao-empresa/)

> Como transformar todo o conhecimento anterior em uma estratégia de IA para uma organização.
> **Pré-requisito:** visão geral dos módulos anteriores.

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 6.1 | [Guia: Montando uma Estrutura de IA na Empresa](docs/06-adocao-empresa/01-guia-ia-na-empresa.md) | Peças do quebra-cabeça (modelo, contexto, interface), decisões de ferramenta e passos práticos para adotar IA numa empresa. |

---

## 🎯 Navegação rápida por objetivo

Não quer seguir a trilha linear? Vá direto ao que precisa:

| Seu objetivo | Leia |
|--------------|------|
| Entender o que é um LLM, sem jargão | [1.1](docs/01-fundamentos/01-o-que-significa-aprender.md) → [1.2](docs/01-fundamentos/02-llm-deep-dive.md) |
| Saber como o modelo funciona por dentro | [1.3 Anatomia](docs/01-fundamentos/03-anatomia-tecnica-llm.md) |
| Entender treinamento e decidir fine-tuning vs. RAG vs. prompt | [1.5 Treinamento e Fine-Tuning](docs/01-fundamentos/05-treinamento-e-fine-tuning.md) |
| Construir meu primeiro orquestrador | [2.3 Orquestrador com Lambda](docs/02-orquestracao/03-orquestradores-ia-lambda.md) |
| Escolher um framework (LangChain, etc.) | [2.4](docs/02-orquestracao/04-frameworks-orquestracao.md) → [2.5](docs/02-orquestracao/05-comparacao-frameworks.md) |
| Reduzir custo escolhendo o modelo certo por request | [2.6 Roteamento](docs/02-orquestracao/06-roteamento-de-llms.md) |
| Decidir entre agente único, multi-agente ou vários orquestradores (custo/latência) | [2.7 Padrões de Agentes](docs/02-orquestracao/07-padroes-execucao/) |
| Ver arquiteturas completas resolvidas de 3 formas (custo + complexidade) | [2.8 Estudos de Caso](docs/02-orquestracao/08-estudos-de-caso/) |
| Fazer "chat com meus documentos" (RAG) | [3.1 Bancos Vetoriais](docs/03-rag-dados/01-bancos-vetoriais-e-documentos.md) |
| Decidir onde hospedar o modelo | [4.1](docs/04-infraestrutura/01-hospedagem-e-orquestracao-llm.md) → [4.2](docs/04-infraestrutura/02-hospedagem-inference-engine-guia.md) |
| Usar GPU sob demanda que escala a zero (custo variável) | [4.3 Serverless GPU](docs/04-infraestrutura/03-serverless-gpu.md) |
| Controlar custos e tokens em produção | [5.1 Precificação e Governança](docs/05-governanca-seguranca/01-precificacao-tokens-credenciais-aws.md) |
| Oferecer/revender IA para meus usuários (medir, limitar, cobrar) | [5.3 Ferramentas de Precificação e Controle](docs/05-governanca-seguranca/03-ferramentas-precificacao-controle-tokens.md) |
| Decidir ONDE colocar o controle de tokens no fluxo (orquestrador? camada antes? gateway?) | [5.4 Posicionamento do Controle de Tokens](docs/05-governanca-seguranca/04-posicionamento-controle-tokens.md) |
| Enxergar o sistema de IA (custo, latência, qualidade): o que coletar, armazenar e exibir | [5.5 Observabilidade de IA](docs/05-governanca-seguranca/05-observabilidade-ia-arquitetura.md) |
| Entender uma ferramenta pronta de gateway (é orquestrador? onde encaixa?) | [5.6 OmniRoute — AI Gateway](docs/05-governanca-seguranca/06-omniroute-ai-gateway.md) |
| Proteger dados sensíveis / segurança | [5.1](docs/05-governanca-seguranca/01-precificacao-tokens-credenciais-aws.md) + [5.2](docs/05-governanca-seguranca/02-vazamento-dados-no-chat.md) |
| Levar IA para minha empresa | [6.1 Guia Empresa](docs/06-adocao-empresa/01-guia-ia-na-empresa.md) |

---

## 🧩 Como os temas se conectam

```text
                     ┌─────────────────────────────────────────┐
                     │            USUÁRIO / APLICAÇÃO            │
                     └───────────────────┬───────────────────────┘
                                         │
                            ┌────────────▼────────────┐
             Módulo 2  ────►│       ORQUESTRADOR       │
                            │  (coordena tudo)         │
                            │                          │
                            │  ┌────────────────────┐  │
             Módulo 2.6 ───►│  │  Roteador de LLM   │  │
                            │  └─────────┬──────────┘  │
                            └────────────┼──────────────┘
                          ┌──────────────┼──────────────┐
                          ▼              ▼              ▼
                   ┌────────────┐ ┌────────────┐ ┌────────────┐
        Módulo 1 ─►│  MODELO    │ │  RAG /     │◄─ Módulo 3   │
        Módulo 4 ─►│ (LLM +     │ │  Vetores   │ │  Tools/    │
                   │  engine)   │ │            │ │  APIs      │
                   └────────────┘ └────────────┘ └────────────┘
                          │
                  ┌───────▼───────────────────────────────────┐
     Módulo 5 ───►│  GOVERNANÇA: custo, tokens, credenciais,   │
                  │  acesso, segurança (envolve tudo acima)    │
                  └────────────────────────────────────────────┘
     Módulo 6 ───► ADOÇÃO: como tudo isso vira estratégia na empresa
```

- **Módulo 1** explica o **modelo** (a peça central).
- **Módulo 4** explica **onde o modelo roda** (inference engine / hospedagem).
- **Módulo 2** explica o **orquestrador**, que coordena o modelo — e dentro dele vive o **roteador (2.6)**.
- **Módulo 3** é a **memória/conhecimento** (RAG) que o orquestrador consulta.
- **Módulo 5** é transversal: **governa custo e segurança** de toda a arquitetura.
- **Módulo 6** transforma tudo em **estratégia de empresa**.

---

## 📌 Notas

- **Foco AWS:** os documentos usam AWS (Bedrock, Lambda) como pano de fundo, mas os conceitos (orquestração, RAG, roteamento) são portáveis para outros provedores.
- **Custos:** todos os valores de custo nos documentos são **ordens de grandeza ilustrativas** para raciocínio comparativo. Confirme sempre na [Calculadora de Preços da AWS](https://calculator.aws) e nas páginas oficiais antes de decidir.

---

## 🗂️ Índice completo (lista plana)

<details>
<summary>Todos os documentos, em ordem de módulo</summary>

1. [`docs/01-fundamentos/01-o-que-significa-aprender.md`](docs/01-fundamentos/01-o-que-significa-aprender.md)
2. [`docs/01-fundamentos/02-llm-deep-dive.md`](docs/01-fundamentos/02-llm-deep-dive.md)
3. [`docs/01-fundamentos/03-anatomia-tecnica-llm.md`](docs/01-fundamentos/03-anatomia-tecnica-llm.md)
4. [`docs/01-fundamentos/04-llm-inference-engines.md`](docs/01-fundamentos/04-llm-inference-engines.md)
5. [`docs/01-fundamentos/05-treinamento-e-fine-tuning.md`](docs/01-fundamentos/05-treinamento-e-fine-tuning.md)
6. [`docs/02-orquestracao/01-orquestradores-estudo-didatico.md`](docs/02-orquestracao/01-orquestradores-estudo-didatico.md)
7. [`docs/02-orquestracao/02-orquestradores-casos-de-produto.md`](docs/02-orquestracao/02-orquestradores-casos-de-produto.md)
8. [`docs/02-orquestracao/03-orquestradores-ia-lambda.md`](docs/02-orquestracao/03-orquestradores-ia-lambda.md)
9. [`docs/02-orquestracao/04-frameworks-orquestracao.md`](docs/02-orquestracao/04-frameworks-orquestracao.md)
10. [`docs/02-orquestracao/05-comparacao-frameworks.md`](docs/02-orquestracao/05-comparacao-frameworks.md)
11. [`docs/02-orquestracao/06-roteamento-de-llms.md`](docs/02-orquestracao/06-roteamento-de-llms.md)
12. [`docs/02-orquestracao/07-padroes-execucao/`](docs/02-orquestracao/07-padroes-execucao/) (série: 01-loop, 02-agente-unico, 03-sub-agentes, 04-multi-agente, 05-multiplos-orquestradores, 06-custo-latencia-throughput)
13. [`docs/02-orquestracao/08-estudos-de-caso/`](docs/02-orquestracao/08-estudos-de-caso/) (série: 01-chatbot-faq, 02-rag-documentos, 03-agente-com-tools, 04-multi-agente-atendimento, 05-pipeline-batch, 06-revenda-ia-multitenant)
14. [`docs/03-rag-dados/01-bancos-vetoriais-e-documentos.md`](docs/03-rag-dados/01-bancos-vetoriais-e-documentos.md)
15. [`docs/04-infraestrutura/01-hospedagem-e-orquestracao-llm.md`](docs/04-infraestrutura/01-hospedagem-e-orquestracao-llm.md)
16. [`docs/04-infraestrutura/02-hospedagem-inference-engine-guia.md`](docs/04-infraestrutura/02-hospedagem-inference-engine-guia.md)
17. [`docs/04-infraestrutura/03-serverless-gpu.md`](docs/04-infraestrutura/03-serverless-gpu.md)
18. [`docs/05-governanca-seguranca/01-precificacao-tokens-credenciais-aws.md`](docs/05-governanca-seguranca/01-precificacao-tokens-credenciais-aws.md)
19. [`docs/05-governanca-seguranca/02-vazamento-dados-no-chat.md`](docs/05-governanca-seguranca/02-vazamento-dados-no-chat.md)
20. [`docs/05-governanca-seguranca/03-ferramentas-precificacao-controle-tokens.md`](docs/05-governanca-seguranca/03-ferramentas-precificacao-controle-tokens.md)
21. [`docs/05-governanca-seguranca/04-posicionamento-controle-tokens.md`](docs/05-governanca-seguranca/04-posicionamento-controle-tokens.md)
22. [`docs/05-governanca-seguranca/05-observabilidade-ia-arquitetura.md`](docs/05-governanca-seguranca/05-observabilidade-ia-arquitetura.md)
23. [`docs/05-governanca-seguranca/06-omniroute-ai-gateway.md`](docs/05-governanca-seguranca/06-omniroute-ai-gateway.md)
24. [`docs/06-adocao-empresa/01-guia-ia-na-empresa.md`](docs/06-adocao-empresa/01-guia-ia-na-empresa.md)

</details>

---

> **Resumo em uma frase:** comece pelos [fundamentos](docs/01-fundamentos/) para entender o que é um LLM,
> avance para [orquestração](docs/02-orquestracao/) e [RAG](docs/03-rag-dados/) para aprender a construir,
> veja [infraestrutura](docs/04-infraestrutura/) para saber onde rodar, e feche com
> [governança](docs/05-governanca-seguranca/) e [adoção](docs/06-adocao-empresa/) para levar à produção.
