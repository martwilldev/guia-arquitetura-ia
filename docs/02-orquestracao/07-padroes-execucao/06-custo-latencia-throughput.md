# Custo, Latência e Throughput — Análise de Desempenho dos Padrões de Execução

> **Arquivo técnico de referência da série.** Os demais documentos remetem a este para os números.
> Aqui consolidamos a **análise de desempenho transversal** — custo, latência e throughput — de
> todos os padrões de execução de agentes (chamada simples, agente único, sub-agentes, multi-agente,
> múltiplos orquestradores), no contexto **AWS Bedrock + Lambda**.

> ⚠️ **Todos os números de custo e latência são ILUSTRATIVOS** — ordens de grandeza para raciocínio
> comparativo, **não** cotações. Sempre confirme na
> [página de preços do Amazon Bedrock](https://aws.amazon.com/bedrock/pricing/) e na
> [Calculadora de Preços da AWS](https://calculator.aws) antes de decidir.

---

## Índice

1. [A "física" do desempenho: o triângulo](#1-a-física-do-desempenho-o-triângulo)
2. [As leis fundamentais](#2-as-leis-fundamentais)
3. [Análise de custo por padrão](#3-análise-de-custo-por-padrão)
4. [Análise de latência por padrão](#4-análise-de-latência-por-padrão)
5. [Análise de throughput](#5-análise-de-throughput)
6. [Tabela comparativa geral](#6-tabela-comparativa-geral)
7. [Onde cada otimização ataca](#7-onde-cada-otimização-ataca)
8. [Ferramentas de medição e observabilidade](#8-ferramentas-de-medição-e-observabilidade)
9. [Árvore de decisão custo × latência](#9-árvore-de-decisão-custo--latência)
10. [Armadilhas](#10-armadilhas)
11. [Glossário](#11-glossário)
12. [Resumo em uma frase](#12-resumo-em-uma-frase)

---

## 1. A "física" do desempenho: o triângulo

Custo, latência e throughput **não são independentes**. Mexer em um quase sempre mexe nos outros.
Pense neles como três vértices de um triângulo que se puxam:

```
                        CUSTO
                     (US$ / request)
                        /\
                       /  \
                      /    \
                     /      \
                    /        \
                   /  você    \
                  /  escolhe   \
                 /  o ponto     \
                /                \
     LATÊNCIA ──────────────────── THROUGHPUT
   (tempo p/ 1 req)              (reqs / segundo)
```

Exemplos de como um vértice puxa o outro:

- **Quero menos latência** → paralelizo chamadas → gasto **mais compute/custo** ao mesmo tempo.
- **Quero mais throughput** → uso modelo menor/mais rápido → **menor custo por request**, mas talvez **menos qualidade**.
- **Quero menor custo** → uso menos chamadas/contexto menor → pode **aumentar latência** (mais idas e voltas) ou reduzir qualidade.
- **Latência longa por request** → cada worker fica ocupado mais tempo → **derruba o throughput** com a mesma frota.

### Definições precisas

| Métrica | O que mede | Unidade típica | Do que depende |
|---------|-----------|----------------|----------------|
| **Custo** | Quanto se paga por resolver 1 tarefa | US$ / request | Nº de **tokens** (entrada + saída) × preço do modelo. Compute (Lambda) é a menor parte. |
| **Latência** | Tempo até a tarefa ficar pronta | segundos | Nº de **chamadas LLM em série** × tempo por chamada; **TTFT** (time to first token) para respostas em streaming. |
| **Throughput** | Quantas tarefas o sistema atende por unidade de tempo | requests / segundo (req/s) | Concorrência disponível ÷ tempo por request; limitado por **quotas do Bedrock** e concorrência do Lambda. |

> **Custo** é sobre *tokens*. **Latência** é sobre *chamadas em série*. **Throughput** é sobre
> *concorrência ÷ tempo por request*. Guarde estas três frases — elas explicam quase tudo.

---

## 2. As leis fundamentais

Sete "leis" que valem para qualquer padrão:

1. **Cada chamada LLM soma custo E tempo.** Toda ida ao modelo custa tokens (dinheiro) e demora (segundos). Reduzir o número de chamadas ataca os dois de uma vez.

2. **Série soma latência.** Chamadas encadeadas (uma depende da anterior) somam o tempo:
   `latência ≈ nº_chamadas_em_série × tempo_por_chamada`.

   ```
   [call A]──►[call B]──►[call C]      latência = tA + tB + tC   (soma)
   ```

3. **Paralelo soma custo/compute, não latência.** Chamadas independentes rodam ao mesmo tempo: a latência é a **da mais lenta**, mas você paga **todas**.

   ```
   [call A] ┐
   [call B] ├──► latência = max(tA,tB,tC)   custo = cA + cB + cC   (paga todas)
   [call C] ┘
   ```

4. **O contexto do loop cresce ~quadraticamente.** No loop de agente, cada iteração reenvia todo o histórico. Se cada passo adiciona contexto, o total de tokens processados cresce como a **soma 1+2+3+...+N ≈ N²/2**. Loops longos ficam caros muito rápido.

   ```
   iter 1: [P]                    → contexto 1
   iter 2: [P + r1]               → contexto 2
   iter 3: [P + r1 + r2]          → contexto 3
   ...
   iter N: [P + r1 + ... + r(N-1)]→ contexto N
   TOTAL de tokens ≈ 1+2+...+N ≈ N²/2   (quadrático)
   ```

5. **Latência longa derruba throughput.** Se cada request ocupa um worker por muito tempo, a mesma frota atende menos requests/segundo. Throughput e latência-por-request são inversamente ligados.

6. **Tokens dominam o custo; compute é a menor parte.** Numa arquitetura Bedrock+Lambda típica, o custo dos tokens do modelo costuma ser **ordens de grandeza maior** que o custo de Lambda/DynamoDB. Otimize tokens primeiro.

7. **Paralelizar melhora latência à custa de custo/concorrência.** É a troca mais comum: você "compra" tempo gastando compute simultâneo e consumindo quota mais rápido.

---

## 3. Análise de custo por padrão

Custo relativo em **número de chamadas LLM** (proxy do custo de tokens), com **base = 1 chamada simples**.

> ⚠️ **ILUSTRATIVO.** O custo real depende dos tokens de entrada/saída e do modelo escolhido.
> Confirme na [preços do Bedrock](https://aws.amazon.com/bedrock/pricing/) e na [Calculadora AWS](https://calculator.aws).

| Padrão | Chamadas LLM (aprox.) | Custo relativo (base=1) | Observação |
|--------|----------------------|-------------------------|------------|
| **Chamada simples** | 1 | **1×** | Baseline: prompt → resposta. |
| **Agente único — 3 iterações** | ~3 + reprocesso de contexto | **~4–5×** | Cada iteração reenvia o histórico (efeito quadrático leve). |
| **Agente único — 6 iterações** | ~6 + reprocesso de contexto | **~10–14×** | Contexto cresce ~N²; custo dispara com iterações. |
| **Sub-agentes em série** | 1 (supervisor) + N (sub-agentes) | **~(N+1)×** + overhead | Cada sub-agente é um mini-loop; some seus custos. |
| **Sub-agentes em paralelo** | 1 + N (simultâneos) | **~(N+1)×** | Mesmo custo total do série (paga todos), mas menor latência. |
| **Multi-agente supervisor** | supervisor + Σ(agentes) + rodadas de coordenação | **~8–20×+** | Supervisor "conversa" com cada agente; várias rodadas. |
| **Multi-agente em rede** | conversas N-para-N entre agentes | **~15×–∞** | Sem limite claro; agentes podem "conversar" indefinidamente. **Maior risco de custo.** |

```
CUSTO RELATIVO (ilustrativo, escala log-ish)
simples          █ 1×
agente 3 iter    ████ ~4–5×
agente 6 iter    ██████████ ~10–14×
sub-agentes (N=3)████ ~4× +overhead
multi supervisor ████████████████ ~8–20×
multi em rede    ████████████████████████ ~15×–∞  ⚠️
```

**Lição:** o custo escala com **quantas chamadas** e **quão grande fica o contorno de contexto**.
Multi-agente em rede é o mais imprevisível — sempre coloque limites (max de rodadas, budget de tokens).

---

## 4. Análise de latência por padrão

Latência escala com o número de **chamadas em série** (as que dependem umas das outras).

| Padrão | Chamadas em série | Latência relativa (base=1) | Por quê |
|--------|-------------------|----------------------------|---------|
| **Chamada simples** | 1 | **1×** | Uma ida ao modelo. |
| **Agente único — 3 iter** | 3 | **~3×** | Cada passo espera o anterior. |
| **Agente único — 6 iter** | 6 | **~6×** | Latência ∝ nº de iterações. |
| **Sub-agentes em série** | 1 + N | **~(N+1)×** | Supervisor espera cada sub-agente terminar. |
| **Sub-agentes em paralelo** | 1 + 1 (o mais lento) | **~2×** | Sub-agentes rodam juntos; espera o mais lento. |
| **Multi-agente supervisor** | várias rodadas em série | **alta (~5–10×)** | Coordenação sequencial entre agentes. |
| **Multi-agente em rede** | imprevisível | **muito alta** | Depende de quantas trocas ocorrem. |

```
LATÊNCIA (série soma; paralelo colapsa)

SÉRIE:     [S]──►[A]──►[B]──►[C]        =  tS + tA + tB + tC
PARALELO:  [S]──►┬─[A]─┐
                 ├─[B]─┼──►                =  tS + max(tA,tB,tC)
                 └─[C]─┘
```

### Táticas para reduzir latência

| Tática | Como ajuda | Custo colateral |
|--------|-----------|-----------------|
| **Paralelizar** chamadas independentes | Colapsa N chamadas em ~1 (a mais lenta) | Paga todas (mais custo/concorrência) |
| **Streaming (TTFT)** | Usuário vê o texto surgir; melhora a *latência percebida* | Nenhum relevante |
| **Modelo mais rápido** (ex.: um "haiku"/"lite") | Menos tempo por chamada | Possível perda de qualidade |
| **Cache** (prompt/resposta) | Elimina a chamada inteira quando há acerto | Complexidade de invalidação |
| **Menos saltos** (menos iterações/agentes) | Reduz chamadas em série | Pode limitar a autonomia do agente |
| **Provisioned Concurrency (Lambda)** | Elimina cold start | Custo fixo por instância provisionada |

> **TTFT (time to first token)** é a métrica que o usuário sente num chat. Otimizar TTFT com
> streaming melhora a *experiência* mesmo sem reduzir a latência total.

---

## 5. Análise de throughput

Throughput é quantos requests/segundo o sistema sustenta. Fórmula mental:

```
                 C  (concorrência: nº de workers/execuções simultâneas)
  throughput ≈  ───
                 T  (tempo médio por request, em segundos)
```

- **Mais workers (C↑)** → mais throughput.
- **Request mais rápido (T↓)** → mais throughput.
- **Loop longo (T↑)** → **derruba** o throughput: cada execução prende um worker por mais tempo.

Exemplo ilustrativo: com **100** execuções simultâneas e **5 s** por request →
`100 / 5 = 20 req/s`. Se o loop cresce para **20 s**/request → `100 / 20 = 5 req/s` (¼ do throughput).

### O gargalo real: quota do Bedrock, não o Lambda

Um erro comum é escalar o Lambda e esbarrar num teto invisível. **Na prática, o limite costuma ser
a quota de tokens/requests por minuto do Bedrock (TPM/RPM), não a concorrência do Lambda.**

```
          [Muitos usuários]
                 │
                 ▼
        ┌─────────────────┐   Lambda escala fácil (1.000+ simultâneos)
        │  Lambda (C alto)│   ◄── raramente é o gargalo
        └────────┬────────┘
                 │
                 ▼
        ┌─────────────────┐   ⚠️ AQUI está o teto:
        │  Amazon Bedrock │   quota TPM/RPM por modelo/região
        └─────────────────┘   ◄── GARGALO comum (throttling / 429)
```

### Fatores AWS que limitam throughput

| Fator | Limite típico (ilustrativo) | Impacto | Como aliviar |
|-------|----------------------------|---------|--------------|
| **Lambda — concorrência** | 1.000 simultâneos por conta/região (padrão, ajustável) | Teto de workers | Pedir aumento; reserved/provisioned concurrency |
| **Bedrock — quota TPM/RPM** | Varia por modelo/região | **Gargalo mais comum** (throttling) | Service Quotas: pedir aumento; **Provisioned Throughput** |
| **Lambda — timeout 15 min** | 900 s máximo | Loops muito longos estouram | Quebrar em etapas; Step Functions; async |
| **DynamoDB — throughput** | On-demand ou provisionado | Estado/histórico pode limitar | On-demand; particionar bem a chave |
| **API Gateway / limites de payload** | limites de tempo/tamanho | Respostas longas síncronas | Streaming; async + polling |

### Padrões para alto throughput

| Padrão | Ideia | Ganha |
|--------|-------|-------|
| **Assíncrono + SQS** | Enfileira requests; workers consomem no seu ritmo | Absorve picos; não perde requests no throttling |
| **Desacoplar tempo-real de batch** | Chat = síncrono; relatórios/ingestão = fila/batch | Protege a experiência interativa |
| **Aumentar quota** (Service Quotas) | Pedir mais TPM/RPM à AWS | Ergue o teto do gargalo real |
| **Cache** | Respostas repetidas não vão ao modelo | Menos consumo de quota e custo |
| **Provisioned Throughput (Bedrock)** | Capacidade dedicada (model units) | Throughput garantido e previsível para alto volume |

---

## 6. Tabela comparativa geral

Visão consolidada. Ícones: 🟢 favorável · 🟡 moderado · 🔴 caro/lento/complexo.

| Padrão | Custo | Latência | Throughput | Complexidade |
|--------|:-----:|:--------:|:----------:|:------------:|
| **Chamada simples** | 🟢 | 🟢 | 🟢 | 🟢 |
| **Agente único (poucas iter.)** | 🟡 | 🟡 | 🟡 | 🟢 |
| **Agente único (muitas iter.)** | 🔴 | 🔴 | 🔴 | 🟡 |
| **Sub-agentes em série** | 🟡 | 🔴 | 🟡 | 🟡 |
| **Sub-agentes em paralelo** | 🔴 | 🟢 | 🟡 | 🟡 |
| **Multi-agente supervisor** | 🔴 | 🔴 | 🟡 | 🔴 |
| **Multi-agente em rede** | 🔴 | 🔴 | 🔴 | 🔴 |
| **Múltiplos orquestradores** | 🟡 | 🟡 | 🟢¹ | 🔴 |

> ¹ Múltiplos orquestradores podem **melhorar** throughput ao isolar domínios e escalar cada um
> independentemente — ao custo de mais complexidade operacional. Ver
> [05-multiplos-orquestradores.md](05-multiplos-orquestradores.md).

**Leitura rápida:** quanto mais "inteligente/autônomo" o padrão (mais iterações, mais agentes,
mais conversas), mais 🔴 ele acumula. A arte é usar o padrão **mais simples que resolve**.

---

## 7. Onde cada otimização ataca

Nem toda otimização melhora tudo. Este mapa mostra **onde cada uma bate**:

| Otimização | Custo | Latência | Throughput | Nota |
|------------|:-----:|:--------:|:----------:|------|
| **Roteamento** (modelo certo p/ cada request) | 🟢↓ | 🟢↓ | 🟢↑ | Manda o barato/rápido resolver o simples |
| **Cache** (prompt/resposta) | 🟢↓ | 🟢↓ | 🟢↑ | Acerto = zero chamada; ajuda os três |
| **Paralelizar** chamadas independentes | 🔴↑ | 🟢↓ | 🟡 | Baixa latência, **sobe** custo/concorrência |
| **Limitar iterações** do loop | 🟢↓ | 🟢↓ | 🟢↑ | Corta a cauda quadrática de contexto |
| **Truncar/resumir contexto** | 🟢↓ | 🟢↓ | 🟢↑ | Menos tokens por chamada |
| **Streaming** | ⚪ | 🟢↓ (percebida) | ⚪ | Melhora TTFT; não muda custo/throughput |
| **Async + fila (SQS)** | ⚪ | 🔴↑ (por req) | 🟢↑ | Aceita latência maior para absorver volume |
| **Provisioned Concurrency (Lambda)** | 🔴↑ | 🟢↓ (sem cold start) | 🟢↑ | Custo fixo; elimina cold start |
| **Provisioned Throughput (Bedrock)** | 🔴↑ | 🟢↓ | 🟢↑ | Capacidade dedicada; alto volume previsível |

Legenda: ↓ melhora · ↑ piora (ou aumenta) · ⚪ neutro.

> **Regra de ouro:** comece por **roteamento + cache + limitar iterações + truncar contexto** —
> são as que melhoram os **três** eixos ao mesmo tempo e têm o melhor retorno.

---

## 8. Ferramentas de medição e observabilidade

Não dá para otimizar o que você não mede. Ferramentas comuns para custo e latência:

| Ferramenta | Tipo | Mede | Modelo |
|------------|------|------|--------|
| **Langfuse** | Observabilidade de LLM | Tokens, custo, latência por chamada, traces de agente | Self-hosted **ou** SaaS |
| **Helicone** | Observabilidade/proxy de LLM | Custo, latência, logs de request | SaaS (self-hosted possível) |
| **Amazon CloudWatch** | Nativo AWS | Métricas/logs de Lambda, duração, erros, invocações | Gerenciado AWS |
| **AWS X-Ray** | Nativo AWS | Tracing distribuído (Lambda → Bedrock → DynamoDB), gargalos de latência | Gerenciado AWS |
| **LiteLLM** (gateway) | Gateway de LLM | Custo/tokens por chave, roteamento, rate limit | Self-hosted (open source) |
| **Portkey** (gateway) | Gateway de LLM | Observabilidade, cache, roteamento, custo | SaaS (self-hosted possível) |

```
   App/Orquestrador
        │
        ▼
   ┌─ Gateway (LiteLLM/Portkey) ─┐   ◄── mede custo/tokens/roteia ANTES do modelo
   │        │                    │
   │        ▼                    │
   │    Bedrock ─────────────────┘
   │
   └─► CloudWatch + X-Ray (Lambda/infra)  +  Langfuse/Helicone (traces de LLM)
```

**Como escolher:**
- **Já na AWS, quer o básico rápido:** CloudWatch + X-Ray (custo de infra e latência ponta a ponta).
- **Quer traces ricos de agente + custo por chamada:** Langfuse (self-hosted p/ dados sensíveis) ou Helicone/Portkey (SaaS, mais rápido de subir).
- **Quer também roteirizar e limitar tokens:** um gateway (LiteLLM self-hosted / Portkey SaaS).

---

## 9. Árvore de decisão custo × latência

```
                    ┌───────────────────────────────┐
                    │  A tarefa é interativa (chat)? │
                    └───────────────┬───────────────┘
                        SIM         │        NÃO (batch/assíncrono)
              ┌─────────────────────┘         └───────────────────────┐
              ▼                                                        ▼
   ┌─────────────────────┐                             ┌────────────────────────────┐
   │ Latência importa MUITO│                            │ Custo importa mais que tempo │
   └──────────┬───────────┘                            └──────────────┬───────────────┘
              ▼                                                        ▼
   • Streaming (TTFT)                                     • Async + SQS (absorve volume)
   • Modelo rápido + roteamento                           • Modelo barato + batch
   • Paralelizar sub-tarefas                              • Cache agressivo
   • Provisioned Concurrency                              • Limitar iterações do loop
   • Menos saltos/iterações                               • Truncar/resumir contexto
              │                                                        │
              ▼                                                        ▼
   ┌───────────────────────────┐               ┌───────────────────────────────────┐
   │ Volume alto e constante?  │               │ Volume alto e previsível?         │
   └────────────┬──────────────┘               └───────────────┬───────────────────┘
        SIM     │                                        SIM    │
                ▼                                               ▼
   Provisioned Throughput (Bedrock)               Provisioned Throughput +
   + aumentar quota TPM/RPM                        pedir aumento de quota
```

**Atalho:** *interativo* → otimize **latência** (streaming, modelo rápido, paralelizar).
*Batch* → otimize **custo** (async, modelo barato, cache, menos iterações).

---

## 10. Armadilhas

- **Escalar o Lambda achando que resolve throughput.** O teto costuma ser a **quota TPM/RPM do Bedrock** — vai bater em `429/throttling` mesmo com Lambda sobrando.
- **Ignorar o custo quadrático do loop.** Um agente que "só" faz mais algumas iterações pode multiplicar o custo por reprocessar todo o contexto a cada passo.
- **Multi-agente em rede sem limite.** Agentes conversando N-para-N sem teto de rodadas = custo imprevisível. Sempre defina `max_rounds` e budget de tokens.
- **Confundir latência total com TTFT.** Streaming melhora a percepção, mas o custo e a latência *total* seguem iguais.
- **Paralelizar tudo.** Reduz latência, mas **soma custo** e consome quota mais rápido — pode piorar throughput global sob carga.
- **Otimizar compute antes de tokens.** Lambda/DynamoDB costumam ser centavos perto do custo dos tokens. Ataque tokens primeiro.
- **Não medir.** Sem observabilidade (Langfuse/CloudWatch/X-Ray) você otimiza no escuro.
- **Timeout de 15 min do Lambda em loops longos.** Loops ou multi-agente demorados estouram; use Step Functions ou async.

---

## 11. Glossário

| Termo | Significado |
|-------|-------------|
| **Token** | Unidade de texto que o modelo processa; base da cobrança (entrada + saída). |
| **TTFT** (time to first token) | Tempo até o primeiro token da resposta aparecer; principal métrica de latência percebida em chat. |
| **Latência** | Tempo total para completar uma tarefa. |
| **Throughput** | Requests atendidos por unidade de tempo (req/s). |
| **Concorrência** | Nº de execuções rodando ao mesmo tempo (workers). |
| **TPM / RPM** | Tokens por minuto / Requests por minuto — quotas do Bedrock. |
| **Throttling** | Bloqueio/limitação (HTTP 429) quando estoura a quota. |
| **Provisioned Concurrency** | Instâncias Lambda pré-aquecidas (sem cold start), custo fixo. |
| **Provisioned Throughput** | Capacidade dedicada no Bedrock (model units) para alto volume previsível. |
| **Cold start** | Atraso na primeira invocação de uma função Lambda "fria". |
| **Cache** | Reuso de resultados anteriores para evitar chamadas repetidas ao modelo. |
| **Roteamento** | Escolher o modelo/rota certo para cada request (por custo/complexidade). |
| **Custo quadrático do loop** | Crescimento ~N² dos tokens processados quando o contexto é reenviado a cada iteração. |

---

## 12. Resumo em uma frase

> **Custo** é sobre *tokens*, **latência** é sobre *chamadas em série*, e **throughput** é sobre
> *concorrência ÷ tempo por request* — e como os três se puxam, escolha o padrão mais simples que
> resolve, otimize tokens antes de compute, e lembre que o gargalo real costuma ser a **quota do
> Bedrock**, não o Lambda.

---

## Navegação da série

- ⬅️ **Anterior na série:** [05-multiplos-orquestradores.md](05-multiplos-orquestradores.md)
- 🏠 **Volte ao índice da série:** [README.md](README.md)
