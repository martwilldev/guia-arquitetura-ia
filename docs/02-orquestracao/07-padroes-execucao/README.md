# Padrões de Execução de Orquestradores de IA — Série

> Do mais simples ao mais complexo: **loop de agente → agente único → sub-agentes →
> multi-agente → múltiplos orquestradores**, e a **análise de desempenho** (custo,
> latência, throughput) que amarra tudo.
> Cada arquivo é focado em **um** conceito, com exemplos de código, ferramentas de
> mercado e opções self-hosted.

---

## Por que uma série (e não um arquivo só)

Estes conceitos são frequentemente confundidos. Separá-los deixa claro **o que é cada um**,
**quando subir de nível** e **quanto custa** (em dinheiro, tempo e complexidade).

```
 SIMPLES ───────────────────────────────────────────────► COMPLEXO

 Loop ──► Agente ──► Sub-agentes ──► Multi-agente ──► Múltiplos
          único                                       orquestradores
   │        │            │               │                 │
   └────────┴────────────┴───────────────┴─────────────────┘
                          │
             06 · Custo / Latência / Throughput
                (análise transversal a todos)
```

> **Regra de ouro:** comece no nível mais baixo que resolve o problema e **só suba quando a dor
> real justificar** (qualidade insuficiente, tarefa que não cabe, SLAs conflitantes).
> Cada nível acima custa mais e é mais lento.

---

## 📚 A série

| # | Documento | O que você aprende | Nível |
|---|-----------|--------------------|-------|
| 01 | [O Loop de Agente (ReAct)](01-loop-de-agente.md) | O ciclo pensar→agir→observar, por que o contexto cresce, `MAX_ITERACOES`, exemplo Bedrock e ferramentas que fazem o loop por você. | 🟢 Base |
| 02 | [Agente Único](02-agente-unico.md) | 1 LLM + 1 loop + tools. Quando basta, exemplo com DynamoDB, versão com framework, como escalar antes de partir para multi-agente. | 🟢 Base |
| 03 | [Sub-tarefas e Sub-agentes](03-sub-agentes.md) | Decompor uma tarefa; série vs paralelo (fan-out/fan-in); exemplos com `asyncio`/Step Functions/SQS; consolidação. | 🟡 Intermediário |
| 04 | [Multi-Agente](04-multi-agente.md) | Especialistas colaborando: supervisor, rede, pipeline. Custo da coordenação. Ferramentas de mercado + self-hosted (LangGraph, CrewAI, AutoGen, Bedrock MAC…). | 🟠 Avançado |
| 05 | [Múltiplos Orquestradores](05-multiplos-orquestradores.md) | Quando um orquestrador não basta: domínios/SLAs/escalas distintos; arquitetura hierárquica; como implementar na AWS. | 🔴 Avançado |
| 06 | [Custo, Latência e Throughput](06-custo-latencia-throughput.md) | A análise de desempenho consolidada: leis fundamentais, tabelas por padrão, gargalo de quota do Bedrock, onde cada otimização ataca, observabilidade. | 📊 Transversal |

**Ordem sugerida:** 01 → 02 → 03 → 04 → 05, lendo o **06** ao final (ou consultando-o a cada nível para entender o impacto de desempenho).

---

## 🎯 Atalhos por objetivo

| Seu objetivo | Vá para |
|--------------|---------|
| Entender como um agente "decide" e usa tools | [01 Loop](01-loop-de-agente.md) |
| Construir meu primeiro agente com tools | [02 Agente Único](02-agente-unico.md) |
| Acelerar uma tarefa dividindo em partes | [03 Sub-agentes](03-sub-agentes.md) |
| Coordenar especialistas (pesquisa, código, revisão) | [04 Multi-Agente](04-multi-agente.md) |
| Separar domínios/SLAs (tempo-real vs batch) | [05 Múltiplos Orquestradores](05-multiplos-orquestradores.md) |
| Estimar/otimizar custo, latência e throughput | [06 Desempenho](06-custo-latencia-throughput.md) |
| Ver ferramentas de mercado e self-hosted | Seções "Ferramentas" em [04](04-multi-agente.md) e [05](05-multiplos-orquestradores.md) |

---

## 🧩 Como isso se conecta ao resto do estudo

- **Roteamento** (qual modelo/agente usar) tem doc próprio: [`../06-roteamento-de-llms.md`](../06-roteamento-de-llms.md) — a maior alavanca de custo.
- **Frameworks** (LangChain, CrewAI, etc.): [`../04-frameworks-orquestracao.md`](../04-frameworks-orquestracao.md).
- **Construir o orquestrador base** (loop, Step Functions): [`../03-orquestradores-ia-lambda.md`](../03-orquestradores-ia-lambda.md).
- **Custo/tokens em produção**: [`../../05-governanca-seguranca/01-precificacao-tokens-credenciais-aws.md`](../../05-governanca-seguranca/01-precificacao-tokens-credenciais-aws.md).

---

> **Resumo em uma frase:** suba a escada de complexidade — loop → agente único → sub-agentes →
> multi-agente → múltiplos orquestradores — apenas quando a tarefa ou os SLAs exigirem, sempre
> consultando o documento de [custo, latência e throughput](06-custo-latencia-throughput.md) para
> saber o preço de cada degrau.
