# Estudos de Caso — Arquiteturas de IA Resolvidas de Ponta a Ponta

> Cenários reais de IA na AWS, cada um resolvido de **3 formas diferentes** (Opção A/B/C),
> variando **framework + modelo + técnica + compute** — com **custo estimado mensal**,
> **nível de complexidade** (🟢🟡🔴) e trade-offs lado a lado.
> Serve para ver, na prática, como cada decisão muda o resultado.

---

## Como ler estes estudos

Cada estudo de caso segue a mesma estrutura:

1. **Problema e requisitos** (volume, latência, qualidade).
2. **Arquitetura** (diagrama).
3. **3 variações de decisão** (A/B/C) — cada uma com código/config, **custo estimado** (com a conta) e **complexidade**.
4. **Tabela comparativa** das 3 opções.
5. **Recomendação por perfil** (MVP / crescimento / escala).
6. **Armadilhas** e resumo.

> ⚠️ Todos os custos são **ordens de grandeza ilustrativas** para raciocínio comparativo.
> Confirme na [Calculadora AWS](https://calculator.aws) e nos [preços do Bedrock](https://aws.amazon.com/bedrock/pricing/).

---

## 📚 Os cenários

| # | Estudo de caso | O que ele ensina | Volume exemplo |
|---|----------------|------------------|----------------|
| 01 | [Chatbot de FAQ](01-chatbot-faq.md) | Como técnica (FAQ no prompt vs RAG vs Knowledge Base) e modelo mudam custo/complexidade | 100k msg/mês |
| 02 | [Chat com Documentos (RAG)](02-rag-documentos.md) | Impacto da escolha do vector store e de técnicas (rerank/híbrido) no custo | 5M vetores, 50k queries/mês |
| 03 | [Agente com Tools](03-agente-com-tools.md) | Como nº de iterações e roteamento de modelo afetam o custo de um agente | 200k requests/mês, 3 tools |
| 04 | [Atendimento Multi-Agente](04-multi-agente-atendimento.md) | O custo real da coordenação multi-agente (3–10×) e quando NÃO usar | 50k conversas/mês |
| 05 | [Pipeline Batch de Documentos](05-pipeline-batch.md) | Economia do Bedrock Batch e do modelo aberto em volume; trade-off de complexidade | 500k documentos/mês |
| 06 | [Revenda de IA Multi-Tenant](06-revenda-ia-multitenant.md) | Custo do controle, margem, e DIY vs gateway vs SaaS para cobrar usuários | 500 tenants, 2M requests/mês |

---

## 🧭 O padrão que se repete em TODOS os casos

Ao ler os 6, um padrão fica claro:

```
🟢 Opção A (mais simples)   → geralmente SDK puro / gerenciado + modelo barato + técnica simples
                              menor complexidade, ótimo para MVP

🟡 Opção B (equilíbrio)     → framework + técnica intermediária (RAG, roteamento)
                              melhor relação custo/qualidade para crescimento

🔴 Opção C (mais poderosa)  → multi-agente / self-hosted / GPU própria / modelo forte
                              maior qualidade ou controle, mais cara/complexa — só em escala
```

E três lições transversais:

1. **Os tokens do LLM quase sempre dominam o custo** — otimizar modelo/contexto/roteamento economiza mais que trocar o compute.
2. **Comece simples (Opção A) e só suba quando a dor justificar** — muitas vezes a Opção A resolve por muito menos.
3. **"Gerenciado" troca custo/lock-in por menos complexidade operacional** — bom no início; reavalie na escala.

---

## 🔗 Base conceitual (leia se algum termo não fizer sentido)

- Orquestrador do zero: [`../03-orquestradores-ia-lambda.md`](../03-orquestradores-ia-lambda.md)
- Frameworks: [`../04-frameworks-orquestracao.md`](../04-frameworks-orquestracao.md) e [`../05-comparacao-frameworks.md`](../05-comparacao-frameworks.md)
- Roteamento de modelo: [`../06-roteamento-de-llms.md`](../06-roteamento-de-llms.md)
- Padrões de agente (loop, multi-agente, etc.): [`../07-padroes-execucao/`](../07-padroes-execucao/)
- RAG / vetores: [`../../03-rag-dados/01-bancos-vetoriais-e-documentos.md`](../../03-rag-dados/01-bancos-vetoriais-e-documentos.md)
- Serverless / GPU / custos: [`../../04-infraestrutura/03-serverless-gpu.md`](../../04-infraestrutura/03-serverless-gpu.md)
- Custo/tokens/revenda: [`../../05-governanca-seguranca/03-ferramentas-precificacao-controle-tokens.md`](../../05-governanca-seguranca/03-ferramentas-precificacao-controle-tokens.md)

---

> **Resumo em uma frase:** estes 6 estudos mostram o mesmo cenário resolvido em 3 níveis de
> sofisticação cada — provando que a decisão certa depende do volume, da qualidade exigida e do
> quanto de complexidade você aguenta operar, mas que **começar simples com modelo barato quase
> sempre é o caminho de melhor custo-benefício até a dor justificar subir de nível.**
