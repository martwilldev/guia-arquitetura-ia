# Observabilidade de IA/LLM — Arquitetura: O Que Coletar, Como, Onde Armazenar e Como Exibir

> Guia arquitetural da camada de **observabilidade** de um sistema de IA: quais sinais
> coletar, como coletá-los (instrumentação), onde armazená-los, como exibi-los, o que o
> mercado oferece, opções self-hosted e a análise de custo de cada escolha.
>
> Complementa a doc [`04-posicionamento-controle-tokens.md`](04-posicionamento-controle-tokens.md),
> que trata do *controle* de tokens no fluxo. Aqui o foco é **enxergar** o sistema:
> custo, qualidade, latência, uso e erros — por request, por usuário e por feature.

> ⚠️ Valores de custo são **ilustrativos** (ordem de grandeza) para raciocínio.
> Confirme sempre nas páginas oficiais e na [Calculadora AWS](https://calculator.aws).

---

## Índice

1. [Por que observabilidade de IA é diferente](#1-por-que-observabilidade-de-ia-é-diferente)
2. [O que coletar (os sinais)](#2-o-que-coletar-os-sinais)
3. [Como coletar (instrumentação)](#3-como-coletar-instrumentação)
4. [Onde armazenar (camada de dados)](#4-onde-armazenar-camada-de-dados)
5. [Como exibir (visualização e alertas)](#5-como-exibir-visualização-e-alertas)
6. [Arquitetura de referência](#6-arquitetura-de-referência)
7. [Análise de mercado (SaaS)](#7-análise-de-mercado-saas)
8. [Opções self-hosted](#8-opções-self-hosted)
9. [Análise de custo](#9-análise-de-custo)
10. [Recomendações por estágio](#10-recomendações-por-estágio)
11. [Armadilhas e boas práticas](#11-armadilhas-e-boas-práticas)
12. [Glossário](#12-glossário)

---

## 1. Por que observabilidade de IA é diferente

Observabilidade clássica (APM) mede latência, erros e throughput. Um sistema de LLM
adiciona sinais que o APM tradicional **não** captura:

| Dimensão | APM clássico | Observabilidade de IA |
|---|---|---|
| Latência/erros | ✅ | ✅ |
| **Custo por request** | ❌ | ✅ (tokens × preço) |
| **Tokens (in/out)** | ❌ | ✅ |
| **Qualidade da resposta** | ❌ | ✅ (avaliação, feedback) |
| **Prompt/resposta (conteúdo)** | ❌ | ✅ (com cuidado de PII) |
| **Traço de agente/RAG** | ❌ | ✅ (passos, tools, chunks recuperados) |
| **Alucinação / relevância** | ❌ | ✅ (grounding, scoring) |

> A diferença central: em IA, **custo e qualidade são variáveis de runtime**. Você
> precisa observar quanto cada request custou e se a resposta foi boa — não só se foi rápida.

Três "pilares clássicos" (logs, métricas, traces) ganham um quarto no mundo LLM:

```
   LOGS        MÉTRICAS      TRACES        + AVALIAÇÃO (evals)
 (eventos)   (agregados)  (encadeamento)   (qualidade da saída)
```

---

## 2. O que coletar (os sinais)

Organize o que coletar em 5 grupos. Nem tudo é obrigatório — comece pelo essencial (⭐).

### 2.1 Custo e uso (FinOps)
| Sinal | Por quê | Prioridade |
|---|---|---|
| Tokens de entrada / saída (do `usage`) | Base de custo e cobrança | ⭐ essencial |
| Custo estimado por request (tokens × preço) | FinOps, chargeback | ⭐ essencial |
| Modelo usado | Atribuir custo, avaliar roteamento | ⭐ essencial |
| Identidade: `tenant_id`, `user_id`, `feature` | Custo por cliente/feature | ⭐ essencial |
| Cache hit/miss | Medir economia do cache | recomendado |

### 2.2 Desempenho (latência)
| Sinal | Por quê | Prioridade |
|---|---|---|
| Latência total | UX | ⭐ essencial |
| **TTFT** (time to first token) | UX de streaming | recomendado |
| Tokens/segundo (throughput) | Capacidade | recomendado |
| Latência por etapa (RAG, tool, LLM) | Achar gargalo | recomendado |
| Tempo em fila / cold start | Diagnóstico de infra | opcional |

### 2.3 Qualidade (evals)
| Sinal | Por quê | Prioridade |
|---|---|---|
| Feedback do usuário (👍/👎, nota) | Sinal de qualidade real | ⭐ essencial |
| Score de avaliação automática (LLM-as-judge) | Qualidade em escala | recomendado |
| Grounding/relevância do RAG | Detectar alucinação | recomendado |
| Taxa de recusa / fallback | Saúde do sistema | recomendado |

### 2.4 Traço (debugging)
| Sinal | Por quê | Prioridade |
|---|---|---|
| Prompt final enviado (com PII tratada) | Debug e auditoria | ⭐ essencial |
| Resposta do modelo | Debug e evals | ⭐ essencial |
| Passos do agente (ReAct) e tools chamadas | Entender decisões | recomendado |
| Chunks recuperados no RAG (ids + score) | Diagnóstico de RAG | recomendado |
| Erros e exceções (com stack) | Confiabilidade | ⭐ essencial |

### 2.5 Segurança / compliance
| Sinal | Por quê | Prioridade |
|---|---|---|
| Bloqueios de guardrail (PII, tópico) | Compliance | recomendado |
| Tentativas de prompt injection detectadas | Segurança | recomendado |
| Quem chamou o quê (auditoria) | Trilha de auditoria | ⭐ essencial |

> ⚠️ **PII em prompts/respostas**: conteúdo é ouro para debug mas é dado sensível.
> Mascare/anonimize antes de armazenar, ou guarde em store separado com acesso restrito
> e retenção curta (ver seção 11).

---

## 3. Como coletar (instrumentação)

Há 4 abordagens, do mais fácil ao mais controlado. Podem coexistir.

### 3.1 SDK/wrapper da ferramenta de observabilidade
Envolve suas chamadas ao LLM com um decorator/SDK (Langfuse, Helicone, etc.).
```python
# Exemplo conceitual (Langfuse)
from langfuse.decorators import observe

@observe()  # captura entrada, saída, latência, tokens automaticamente
def responder(pergunta: str, tenant_id: str):
    resp = bedrock.converse(modelId=MODELO, messages=[...])
    return resp
```
- ✅ Rápido, captura traço/tokens/latência automaticamente.
- ❌ Acopla à ferramenta; algum overhead.

### 3.2 Via gateway de LLM (instrumentação centralizada)
Se você já usa um gateway (LiteLLM/Portkey), ele **já emite** telemetria de toda
chamada — sem instrumentar cada app.
```
Orquestrador ─► Gateway LLM ─► Provedor
                    │  (emite: tokens, custo, latência, modelo por request)
                    └──► backend de observabilidade
```
- ✅ Um ponto instrumenta tudo; consistente entre apps.
- ✅ Combina com o controle de tokens (mesma camada).
- ❌ Só vê o que passa pelo gateway (não vê etapas internas do agente).

### 3.3 OpenTelemetry (padrão aberto, sem lock-in)
Instrumente com **OTel** (traces/metrics/logs) e exporte para qualquer backend.
Há convenções semânticas emergentes para GenAI (atributos `gen_ai.*`).
```
App/Orquestrador ──(OTLP)──► OTel Collector ──► [Tempo/Jaeger | Prometheus | backend SaaS]
```
- ✅ **Sem lock-in** — troca o backend sem reinstrumentar.
- ✅ Unifica IA com o resto do sistema (mesmo padrão).
- ❌ Mais setup inicial; convenções GenAI ainda evoluindo.

### 3.4 Logs estruturados + pipeline próprio (DIY)
Emita um evento JSON por request e processe depois.
```python
logger.info(json.dumps({
    "event": "llm_call", "tenant": tenant_id, "feature": "chat",
    "model": modelo, "tokens_in": u["inputTokens"], "tokens_out": u["outputTokens"],
    "cost_usd": custo, "latency_ms": lat, "cache_hit": False,
}))
```
- ✅ Simples, sem dependência, você controla tudo.
- ❌ Você constrói agregação/dashboards/alertas (reinventa a roda).

### Comparação de instrumentação
| Abordagem | Setup | Lock-in | Captura traço de agente | Melhor para |
|---|---|---|---|---|
| SDK da ferramenta | Baixo | Alto | ✅ | Começar rápido |
| Via gateway | Baixo (se já tem gateway) | Médio | ➖ | Múltiplos apps |
| OpenTelemetry | Médio | **Nenhum** | ✅ | Padronização/escala |
| Logs DIY | Baixo | Nenhum | Você faz | Volume baixo/controle |

---

## 4. Onde armazenar (camada de dados)

Cada tipo de sinal pede um tipo de store diferente. Misturar tudo num só é erro comum.

| Tipo de dado | Store adequado | Por quê |
|---|---|---|
| **Métricas** (séries temporais) | TSDB: Prometheus, Amazon Managed Prometheus, CloudWatch, InfluxDB | Agregação rápida, retenção por resolução |
| **Traces** (encadeamento) | Tempo, Jaeger, X-Ray, backend do SaaS | Consulta por trace_id, waterfall |
| **Logs/eventos** (por request) | OpenSearch/Elastic, CloudWatch Logs, Loki, ClickHouse | Busca textual, filtros |
| **Analítico** (custo por tenant no tempo) | Data warehouse: ClickHouse, Athena+S3, BigQuery, Redshift | Agregações pesadas, BI |
| **Conteúdo (prompt/resposta)** | Store separado com acesso restrito (S3+KMS, tabela dedicada) | PII, retenção curta |
| **Feedback/evals** | Banco relacional (Postgres) | Relacionar com request, análise |

### Padrão de retenção em camadas (economiza muito)
```
Quente (0–30 dias)   ─► store rápido/caro (OpenSearch/Prometheus)  ← debug diário
Morno (30–180 dias)  ─► warehouse/colunar (ClickHouse/Athena)      ← análise/BI
Frio (>180 dias)     ─► S3 (Parquet, comprimido)                   ← auditoria barata
```
> 💡 Métricas agregadas podem viver anos baratas; **logs de conteúdo** (prompt/resposta)
> devem ter retenção curta por custo **e** por compliance.

### Granularidade: guarde bruto ou agregado?
- **Alto volume**: guarde **agregados** (por minuto/hora/tenant) para dashboards + uma
  **amostra** do bruto para debug. Guardar 100% do bruto em store quente explode o custo.
- **Baixo volume**: pode guardar tudo bruto sem dor.

---

## 5. Como exibir (visualização e alertas)

### 5.1 O que cada público quer ver
| Público | Pergunta que faz | Painel |
|---|---|---|
| Engenharia | "Onde está o gargalo/erro?" | Traces, latência por etapa, taxa de erro |
| FinOps/produto | "Quanto custa e quem gasta?" | Custo por tenant/feature/modelo no tempo |
| Qualidade/PM | "As respostas são boas?" | Feedback 👍/👎, scores de eval, alucinação |
| Segurança | "Houve abuso/vazamento?" | Guardrails acionados, anomalias, auditoria |

### 5.2 Dashboards essenciais (mínimo viável)
1. **Custo**: US$/dia total, por tenant, por modelo, por feature. Projeção de mês.
2. **Uso**: requests/min, tokens in/out, cache hit rate.
3. **Latência**: p50/p95/p99 total e TTFT.
4. **Qualidade**: % 👍, score médio de eval, taxa de fallback.
5. **Erros**: taxa por tipo, top erros, guardrails acionados.

### 5.3 Ferramentas de exibição
| Ferramenta | Perfil |
|---|---|
| **Grafana** | Padrão open-source; conecta a Prometheus, Loki, Tempo, ClickHouse, CloudWatch |
| **Amazon Managed Grafana / CloudWatch Dashboards** | Nativo AWS |
| **Dashboards do SaaS** (Langfuse, Helicone, Datadog) | Prontos, focados em LLM |
| **Metabase / Superset** | BI sobre o warehouse (custo por tenant, relatórios) |

### 5.4 Alertas (não só dashboards)
Dashboard ninguém olha 24h — **alerte** no que importa:
- Custo do dia > X% da média (anomalia de gasto).
- p95 de latência acima do SLA.
- Taxa de erro / fallback subindo.
- Queda brusca de qualidade (👎 em alta).
- Pico de tokens de um tenant (possível abuso/bug).

```
Métrica cruza limite ─► Alertmanager/CloudWatch Alarm ─► Slack/PagerDuty/SNS
                                                        └► (opcional) ação automática
                                                           (ex.: modo econômico)
```

---

## 6. Arquitetura de referência

```
┌───────────────┐
│ App / Orquestr.│  instrumentação (SDK / OTel / logs)
└───────┬───────┘
        │ emite: tokens, custo, latência, traço, feedback
        ▼
┌───────────────────────────────┐
│  COLETA                        │  OTel Collector / agente / SDK do SaaS
│  (buffer, sampling, roteamento)│  ← amostra o bruto, agrega, mascara PII
└───────┬───────────────┬───────┘
        │               │
        ▼               ▼
┌───────────────┐  ┌───────────────────────────────────────────────┐
│  QUENTE        │  │  ANALÍTICO / FRIO                              │
│  Métricas(TSDB)│  │  Warehouse (ClickHouse/Athena+S3)             │
│  Traces        │  │  + S3 Parquet (auditoria, retenção longa)     │
│  Logs (busca)  │  │  + store de conteúdo (S3+KMS, acesso restrito)│
└───────┬───────┘  └───────────────────┬───────────────────────────┘
        │                              │
        ▼                              ▼
┌───────────────────────────────────────────────────────────────────┐
│  EXIBIÇÃO E ALERTAS                                                  │
│  Grafana / dashboards SaaS  ·  Metabase/Superset (BI)               │
│  Alertmanager / CloudWatch Alarms ─► Slack / PagerDuty / SNS        │
└───────────────────────────────────────────────────────────────────┘
```

- **Coleta** faz o trabalho pesado: amostragem, agregação e **mascaramento de PII**
  antes de persistir. É o ponto onde você controla volume (=custo).
- **Quente vs analítico/frio** separa "debug de agora" de "análise histórica barata".
- **Conteúdo** (prompt/resposta) vai para um store isolado por PII.

---

## 7. Análise de mercado (SaaS)

### 7.1 Especializadas em LLM
| Produto | Foco | Custo/tokens por req | Traço agente/RAG | Evals | Open-source |
|---|---|:---:|:---:|:---:|:---:|
| **Langfuse** | Observabilidade LLM (tracing, custo, evals) | ✅ | ✅ | ✅ | ✅ (self-host) |
| **Helicone** | Observabilidade + gateway (proxy) | ✅ | ➖ | ➖ | ✅ |
| **LangSmith** | Ecossistema LangChain (trace + eval) | ✅ | ✅ | ✅ | ❌ |
| **Arize Phoenix** | Observabilidade + evals de LLM/ML | ✅ | ✅ | ✅ | ✅ |
| **Portkey** | Gateway + observabilidade | ✅ | ➖ | ➖ | Parcial |
| **HoneyHive / Traceloop** | Observabilidade GenAI (OTel-nativa) | ✅ | ✅ | ✅ | Parcial |

### 7.2 Observabilidade geral com módulo LLM
| Produto | Nota |
|---|---|
| **Datadog LLM Observability** | Integra IA ao APM que a empresa já usa; potente e caro |
| **New Relic AI Monitoring** | Idem, dentro do New Relic |
| **Grafana Cloud** | Stack LGTM gerenciada (Loki/Grafana/Tempo/Mimir) |
| **AWS CloudWatch + X-Ray** | Nativo AWS; Bedrock já emite métricas/invocation logging |

> **Padrão comum:** ferramenta LLM-específica (Langfuse/Helicone) para *custo+traço+eval*
> **+** o APM que a empresa já tem (Datadog/CloudWatch) para infra. Ou tudo em OTel
> exportando para os dois.

---

## 8. Opções self-hosted

Rodar você mesmo dá controle de dados (PII não sai) e pode custar menos em escala.

### 8.1 Stacks open-source populares
| Camada | Opções self-hosted |
|---|---|
| **LLM-específico** | **Langfuse** (self-host), **Arize Phoenix**, **Helicone** (self-host) |
| **Métricas** | Prometheus / VictoriaMetrics / Mimir |
| **Traces** | Grafana Tempo / Jaeger |
| **Logs** | Loki / OpenSearch / ClickHouse |
| **Warehouse analítico** | **ClickHouse** (excelente p/ eventos de alto volume) |
| **Exibição** | Grafana / Metabase / Superset |
| **Coleta** | OpenTelemetry Collector |

### 8.2 Duas montagens típicas
**A) "LLM turnkey" self-hosted:** Langfuse (Docker) + Postgres + ClickHouse.
- ✅ Pronto para LLM (custo, traço, eval) sem montar cada peça.
- ✅ PII fica no seu ambiente.
- ❌ Você opera o Langfuse + bancos.

**B) "Stack OTel completa":** OTel Collector → Prometheus + Tempo + Loki → Grafana
(a "stack LGTM"), com ClickHouse para analítico de custo.
- ✅ Sem lock-in, unifica IA + infra, muito flexível.
- ❌ Mais peças para operar; curva de aprendizado.

### 8.3 Onde rodar na AWS
- **ECS/Fargate ou EKS** para os serviços (Langfuse, Grafana, Collector).
- **ClickHouse** em EC2 (com EBS) ou gerenciado (ClickHouse Cloud).
- **Amazon Managed Prometheus + Managed Grafana** = meio-termo (você não opera, mas fica na AWS).
- **S3** para retenção fria (Parquet) + **Athena** para consultar barato.

---

## 9. Análise de custo

> ⚠️ Ordens de grandeza ilustrativas. O custo real depende de **volume de eventos**,
> **retenção** e **cardinalidade** (nº de séries/labels distintos).

### 9.1 De onde vem o custo da observabilidade
```
Custo ≈ Ingestão (quantos eventos/GB entram)
      + Armazenamento (quanto × por quanto tempo)
      + Consulta/dashboards (algumas ferramentas cobram)
      + (SaaS) por evento/trace/usuário/host
```
Os dois maiores vilões: **alto volume de eventos brutos** e **retenção longa em store quente**.

### 9.2 SaaS vs Self-hosted (ilustrativo)
Cenário: 1M requests/mês, 1 evento de trace por request + métricas.

| Abordagem | Custo aprox./mês | Comentário |
|---|---|---|
| **SaaS LLM (Langfuse Cloud/Helicone)** | dezenas–centenas de US$ por volume | Tiers por eventos; barato no início |
| **Datadog LLM Obs.** | centenas–milhares US$ | Cobra por host/eventos/traces; escala caro |
| **Self-host Langfuse (ECS+ClickHouse)** | ~US$ 100–400 (infra) | Custo ~fixo; ganha em escala; você opera |
| **Stack OTel (LGTM) self-host** | ~US$ 150–600 (infra) | Mais peças; máximo controle |
| **CloudWatch (nativo)** | por GB ingerido + métricas | Simples, mas logs volumosos ficam caros |

### 9.3 Padrão de custo: SaaS cresce com volume, self-host é ~fixo
```
Custo
  │                         ╱ SaaS (por evento/trace)
  │                       ╱
  │        ┌────────────╱──────────  Self-hosted (infra ~fixa)
  │        │          ╱
  │        │        ╱  ← ponto de virada
  │________|______╱________________________► Volume (eventos/mês)
```
Mesma lógica do inference engine: **volume baixo → SaaS**; **volume alto/constante →
self-host** tende a compensar (e mantém PII em casa).

### 9.4 Como cortar custo de observabilidade (as maiores alavancas)
| Técnica | Efeito |
|---|---|
| **Amostragem (sampling)** de traces (ex.: 100% de erros, 5–10% de sucessos) | Corta ingestão drasticamente |
| **Agregar** métricas na coleta (não guardar cada evento cru quente) | Menos volume quente |
| **Retenção em camadas** (quente→warehouse→S3) | Store frio é ~grátis vs quente |
| **Baixar cardinalidade** (não use user_id como label de métrica) | Evita explosão de séries |
| **Separar conteúdo** (prompt/resposta) com retenção curta | Menos storage e menos risco PII |
| **Parquet + Athena** para histórico | Consulta barata sob demanda |

> 💡 **Cardinalidade** é o assassino silencioso de custo em métricas: usar identificadores
> únicos (user_id, request_id) como *labels* cria milhões de séries. Use-os em **logs/traces**
> (indexados), não em métricas. Em métricas, agregue por dimensões de baixa cardinalidade
> (modelo, feature, tenant se forem poucos).

---

## 10. Recomendações por estágio

| Estágio | Coleta | Armazenamento | Exibição | Custo-alvo |
|---|---|---|---|---|
| **MVP / 1 app** | SDK LLM (Langfuse Cloud) ou logs no CloudWatch | Gerenciado do SaaS / CloudWatch | Dashboards do SaaS | Baixo, por evento |
| **Produção 1 app** | Langfuse (self-host) **ou** SaaS + CloudWatch p/ infra | Postgres + ClickHouse (ou SaaS) | Grafana + dashboards SaaS | Médio, ~fixo |
| **Escala / multi-app** | **OTel** no gateway + apps → Collector | LGTM/ClickHouse self-host + S3 frio | Grafana + Metabase (BI custo) | Otimizar com sampling |
| **Dados sensíveis** | Self-host (Langfuse/OTel) na sua VPC | Tudo no seu ambiente, PII isolada+KMS | Grafana self-host | Aceita custo por privacidade |

> Regra: **comece com SDK/SaaS** (rápido), migre para **OTel + self-host** quando o
> volume/custo justificar ou quando PII exigir manter dados em casa.

---

## 11. Armadilhas e boas práticas

### Boas práticas
- **Meça custo pelo `usage` real** do provedor (fonte da verdade), não estime.
- **Correlacione tudo por um `trace_id`** (request → etapas → custo → feedback).
- **Amostre traces**, mas **guarde 100% dos erros**.
- **Padronize atributos** (`tenant`, `feature`, `model`, `env`) desde o dia 1.
- **Separe conteúdo (PII) do restante** da telemetria; retenção curta e acesso restrito.
- **Alertas > dashboards**: o que importa deve avisar sozinho.
- **Feche o loop de qualidade**: capture feedback do usuário e ligue ao request.

### Armadilhas
| Armadilha | Consequência | Correção |
|---|---|---|
| Logar prompt/resposta com PII sem tratar | Vazamento, multa (LGPD) | Mascarar na coleta; store isolado |
| Guardar 100% do bruto em store quente | Custo explode | Sampling + retenção em camadas |
| user_id/request_id como **label de métrica** | Explosão de cardinalidade → conta gigante | Usar em logs/traces, não em métricas |
| Só dashboards, sem alertas | Ninguém percebe o incêndio | Configurar alertas nos KPIs |
| Observabilidade só de infra (sem custo/qualidade) | Cego para o que importa em IA | Coletar tokens, custo, feedback, eval |
| Instrumentar acoplado a 1 SaaS | Lock-in, migração dolorosa | OpenTelemetry como camada de coleta |
| Retenção longa de logs de conteúdo | Custo + risco de compliance | Retenção curta; agregados para histórico |

---

## 12. Glossário

| Termo | Significado |
|---|---|
| **Observabilidade** | Capacidade de entender o estado interno do sistema pelos sinais que ele emite |
| **Logs / Métricas / Traces** | Os três pilares: eventos, agregados numéricos, encadeamento de operações |
| **Eval** | Avaliação da qualidade da saída do LLM (humana ou automática) |
| **LLM-as-judge** | Usar um LLM para pontuar a resposta de outro |
| **TTFT** | Time To First Token — latência até o primeiro token (streaming) |
| **Trace / span** | Um trace é a jornada de um request; spans são as etapas dentro dele |
| **trace_id** | Identificador que correlaciona todas as etapas de um request |
| **Cardinalidade** | Número de valores distintos de um label; alta cardinalidade encarece métricas |
| **Sampling** | Guardar só uma fração dos eventos para reduzir volume/custo |
| **TSDB** | Time-Series Database (Prometheus, etc.), otimizado para métricas |
| **OpenTelemetry (OTel)** | Padrão aberto de instrumentação (traces/metrics/logs) |
| **OTLP** | Protocolo do OpenTelemetry para exportar telemetria |
| **LGTM** | Stack Grafana: Loki, Grafana, Tempo, Mimir |
| **Retenção em camadas** | Mover dados de store quente→morno→frio conforme envelhecem |
| **Grounding** | Aderência da resposta ao contexto/RAG (mede alucinação) |

---

## Resumo em uma frase

> **Observe IA coletando os sinais que o APM não vê — tokens, custo, qualidade e traço
> por request/tenant/feature —, instrumentando de forma padronizada (idealmente OpenTelemetry
> ou um SDK LLM como Langfuse), armazenando cada tipo no store certo com retenção em camadas
> (quente→warehouse→S3) e o conteúdo com PII isolado, exibindo em dashboards de custo/latência/qualidade
> com alertas ativos; comece com SaaS para velocidade e migre para self-host quando volume, custo
> ou privacidade justificarem — controlando o custo da própria telemetria com sampling e baixa cardinalidade.**

---

### Documentos relacionados

- [`04-posicionamento-controle-tokens.md`](04-posicionamento-controle-tokens.md) — onde o controle de tokens vive (observabilidade fica "fora do caminho quente").
- [`03-ferramentas-precificacao-controle-tokens.md`](03-ferramentas-precificacao-controle-tokens.md) — ferramentas de metering/billing (Langfuse/Helicone aparecem lá também).
- [`01-precificacao-tokens-credenciais-aws.md`](01-precificacao-tokens-credenciais-aws.md) — métricas de tokens no CloudWatch, tags de custo, FinOps na AWS.
- [`../02-orquestracao/06-roteamento-de-llms.md`](../02-orquestracao/06-roteamento-de-llms.md) — decisões de roteamento que a observabilidade ajuda a validar.
