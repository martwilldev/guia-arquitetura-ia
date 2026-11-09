# Bancos Vetoriais e Vetorização de Documentos para RAG

> Estudo didático e aprofundado sobre a **camada de dados vetoriais** que um orquestrador de IA (Bedrock/Lambda) consulta em uma arquitetura RAG. Complementa os documentos existentes sobre orquestradores em AWS e RAG com LlamaIndex/Haystack.

> ⚠️ **Aviso sobre custos:** Todos os valores monetários neste documento são **ordens de grandeza ilustrativas** para raciocínio arquitetural. Preços reais mudam com frequência, variam por região e dependem de descontos, reservas e negociações. **Sempre confirme nas páginas de preço oficiais de cada fornecedor e na [Calculadora de Preços da AWS](https://calculator.aws/).**

---

## Índice

1. [Fundamentos](#1-fundamentos)
   - [1.1 O que é um embedding](#11-o-que-é-um-embedding)
   - [1.2 O que é um banco vetorial](#12-o-que-é-um-banco-vetorial)
   - [1.3 Métricas de similaridade](#13-métricas-de-similaridade)
   - [1.4 Índices ANN](#14-índices-ann-approximate-nearest-neighbor)
   - [1.5 Trade-off recall × latência × memória](#15-trade-off-recall--latência--memória)
   - [1.6 Fluxo completo (diagrama)](#16-fluxo-completo-documento--busca)
2. [Pipeline de vetorização de documentos](#2-pipeline-de-vetorização-de-documentos)
   - [2.1 Ingestão / loaders](#21-ingestão--loaders)
   - [2.2 Limpeza e normalização](#22-limpeza-e-normalização)
   - [2.3 Chunking](#23-chunking-o-coração-do-pipeline)
   - [2.4 Escolha do modelo de embedding](#24-escolha-do-modelo-de-embedding)
   - [2.5 Metadados](#25-metadados)
   - [2.6 Atualização e reindexação](#26-atualização-e-reindexação)
   - [2.7 Deduplicação](#27-deduplicação)
3. [Estratégias de busca](#3-estratégias-de-busca)
4. [Catálogo de opções de banco vetorial](#4-catálogo-de-opções-de-banco-vetorial)
   - [4.a SaaS / proprietários](#4a-serviços-gerenciados--proprietários-saas)
   - [4.b Nativos AWS / gerenciados na nuvem](#4b-nativos-aws--gerenciados-na-nuvem)
   - [4.c Self-hosted](#4c-self-hosted-você-opera)
   - [4.d On-premise / local / edge](#4d-on-premise--local--edge)
5. [Tabela comparativa grande](#5-tabela-comparativa-grande)
6. [Integração com o orquestrador](#6-integração-com-o-orquestrador)
7. [Análise de custo detalhada](#7-análise-de-custo-detalhada)
8. [Custo oculto do self-hosted / on-prem](#8-custo-oculto-do-self-hosted--on-prem)
9. [Segurança](#9-segurança)
10. [Árvore de decisão e boas práticas](#10-árvore-de-decisão-e-boas-práticas)
11. [Glossário](#11-glossário)
12. [Resumo em uma frase](#12-resumo-em-uma-frase)

---

## 1. Fundamentos

### 1.1 O que é um embedding

Um **embedding** é um vetor de números de ponto flutuante (ex.: 384, 768, 1024 ou 1536 dimensões) que representa o **significado semântico** de um pedaço de texto (ou imagem, áudio, etc.) em um espaço matemático. Textos com significado parecido ficam **próximos** nesse espaço; textos sem relação ficam distantes.

```
"cachorro" ->  [ 0.021, -0.334,  0.887, ...,  0.045 ]   (1024 números)
"cão"      ->  [ 0.019, -0.340,  0.881, ...,  0.050 ]   (bem próximo de "cachorro")
"contabilidade" -> [-0.512, 0.223, -0.041, ..., 0.778 ] (bem distante)
```

Propriedades importantes:

- **Dimensionalidade fixa:** um dado modelo sempre gera vetores do mesmo tamanho. Você não pode misturar vetores de modelos diferentes no mesmo índice.
- **Denso vs esparso:** embeddings de modelos neurais são **densos** (quase todo valor é diferente de zero). Representações tipo BM25/TF-IDF são **esparsas** (a maioria dos valores é zero). Busca híbrida combina os dois.
- **Determinístico por modelo:** o mesmo texto no mesmo modelo gera sempre o mesmo vetor. Trocar de modelo exige **reindexar tudo**.

### 1.2 O que é um banco vetorial

Um **banco vetorial** (vector database / vector store) é um sistema de armazenamento otimizado para:

1. **Guardar** milhões/bilhões de embeddings + seus metadados.
2. **Indexar** esses vetores com estruturas ANN (ver 1.4).
3. **Buscar** os *k* vizinhos mais próximos de um vetor de consulta em milissegundos (top-*k* similarity search).
4. Opcionalmente: **filtrar** por metadados, fazer **busca híbrida**, gerenciar **multi-tenant**, replicar, escalar.

Diferença central para um banco relacional: o índice não é B-tree para igualdade/range, e sim uma estrutura de **vizinhança aproximada** no espaço vetorial.

> Um "banco vetorial" pode ser um **produto dedicado** (Pinecone, Qdrant, Weaviate, Milvus) ou uma **extensão/capacidade** adicionada a um banco existente (pgvector no Postgres, k-NN no OpenSearch, vector search no MongoDB/Redis/DocumentDB).

### 1.3 Métricas de similaridade

A "proximidade" entre dois vetores é medida por uma **métrica de distância/similaridade**. As três mais usadas:

| Métrica | Fórmula (intuição) | O que mede | Quando usar |
|---|---|---|---|
| **Cosseno** | `cos(θ) = (A·B) / (‖A‖·‖B‖)` | Ângulo entre vetores; ignora magnitude | Padrão para texto. A maioria dos modelos de embedding é treinada/normalizada para cosseno |
| **Dot product** (produto interno) | `A·B = Σ aᵢbᵢ` | Alinhamento + magnitude | Quando os vetores **já estão normalizados** (aí equivale a cosseno) ou quando a magnitude carrega sinal útil |
| **Euclidiana (L2)** | `√(Σ (aᵢ−bᵢ)²)` | Distância "em linha reta" | Alguns modelos e casos de imagem; menos comum em texto |

Pontos práticos:

- **Se os vetores forem normalizados** (norma = 1), cosseno, dot product e L2 produzem o **mesmo ranking**. Muitos pipelines normalizam na ingestão e usam dot product por ser mais barato de computar.
- **Use a métrica que o modelo recomenda.** Titan, Cohere, OpenAI e a maioria dos modelos open-source de texto assumem **cosseno**. Usar a métrica errada degrada o recall silenciosamente.
- No banco, a métrica é configurada no momento da criação do índice (`cosine`, `dotproduct`, `l2`/`euclidean`) e geralmente **não pode mudar** sem reindexar.

### 1.4 Índices ANN (Approximate Nearest Neighbor)

Buscar o vizinho **exato** (k-NN exato / brute force) exige comparar a consulta contra **todos** os vetores — O(N). Isso é inviável em milhões de vetores por consulta em tempo real. Os índices **ANN** trocam um pouco de precisão (recall) por **enorme ganho de velocidade**.

#### HNSW (Hierarchical Navigable Small World)
Grafo em camadas. Camadas superiores são esparsas (saltos longos), inferiores densas (saltos curtos). A busca "desce" das camadas superiores para as inferiores navegando para vizinhos cada vez mais próximos.

- **Prós:** recall alto, latência baixíssima, ótimo para consultas em tempo real. É o índice padrão da maioria dos bancos vetoriais modernos.
- **Contras:** consome **muita RAM** (o grafo fica em memória), construção mais lenta, updates custam.
- **Parâmetros:** `M` (nº de conexões por nó — mais = melhor recall, mais memória), `efConstruction` (esforço na construção), `efSearch`/`ef` (esforço na busca — mais = melhor recall, mais latência).

#### IVF (Inverted File Index)
Particiona o espaço em `nlist` clusters (via k-means). Na busca, examina apenas os `nprobe` clusters mais próximos da consulta.

- **Prós:** menos memória que HNSW, boa para grandes volumes.
- **Contras:** recall depende de `nprobe`; precisa de etapa de **treino** (k-means) sobre uma amostra; menos bom para dados que mudam muito.
- **Parâmetros:** `nlist` (nº de clusters), `nprobe` (quantos clusters visitar na busca).

#### IVFPQ (IVF + Product Quantization)
Adiciona **quantização de produto** ao IVF: comprime cada vetor em códigos curtos (ex.: 1536 floats → 96 bytes). Reduz drasticamente a memória.

- **Prós:** economia enorme de memória/armazenamento; viabiliza **bilhões** de vetores.
- **Contras:** perda de precisão (aproximação da aproximação); normalmente exige um passo de **rerank exato** sobre os candidatos para recuperar recall.
- **Uso típico:** escala massiva onde a RAM do HNSW puro seria proibitiva.

#### ScaNN (Scalable Nearest Neighbors — Google)
Combina particionamento, quantização "anisotrópica" e rescoring. Muito eficiente em recall/latência para grandes escalas. Usado internamente pelo Google e disponível como biblioteca.

- **Prós:** excelente relação recall/velocidade em larga escala.
- **Contras:** menos "plug-and-play" que HNSW nos bancos comerciais; mais presente no ecossistema Google/TF.

#### Resumo dos índices

| Índice | Memória | Recall | Latência | Escala típica | Updates |
|---|---|---|---|---|---|
| Flat (exato) | Alta | 100% | Alta (O(N)) | < ~100k | Fácil |
| HNSW | Muito alta | Muito alto | Muito baixa | Milhões | Médio |
| IVF | Média | Alto (ajustável) | Baixa | Milhões–dezenas de M | Precisa retreino |
| IVFPQ | Baixa | Médio-alto (com rerank) | Baixa | Centenas de M–bilhões | Precisa retreino |
| ScaNN | Baixa-média | Muito alto | Muito baixa | Milhões–bilhões | Médio |

### 1.5 Trade-off recall × latência × memória

Todo índice ANN vive num triângulo de tensão:

```
             RECALL (qualidade)
                   /\
                  /  \
                 /    \
                /      \
   LATÊNCIA <--+--------+--> MEMÓRIA / CUSTO
   (velocidade)          (RAM, armazenamento)
```

- Aumentar `efSearch`/`nprobe` → **melhora recall**, **piora latência**.
- Usar HNSW puro → **recall e latência ótimos**, **custo de RAM alto**.
- Usar IVFPQ → **memória baixa**, **recall menor** (mitigado com rerank).
- **Regra prática:** meça recall@k contra um ground-truth exato numa amostra e ajuste os parâmetros até bater sua meta (ex.: recall@10 ≥ 0,95) com a menor latência/custo possível.

### 1.6 Fluxo completo (documento → busca)

```
  INGESTÃO (offline / batch)                          CONSULTA (online / tempo real)
  ============================                        ===============================

  +-------------+                                     +---------------------+
  |  Documento  |  (PDF, HTML, DOCX, MD, ...)         |  Pergunta do usuário |
  +------+------+                                     +----------+----------+
         |                                                        |
         v  loader/parser                                         v  mesmo modelo!
  +------+------+                                     +-----------+---------+
  |  Texto bruto |                                    |  Modelo de embedding |
  +------+------+                                     |  (Titan/Cohere/bge)  |
         |  limpeza/normalização                      +-----------+---------+
         v                                                         |
  +------+------+                                                  v
  |  Chunks     |  (fixo/sentença/parágrafo/semântico + overlap)   [vetor da query]
  +------+------+                                                  |
         |                                                         |
         v  modelo de embedding                                    |
  +------+------+                                                  |
  |  Embeddings | + metadados (fonte, página, tenant, data)        |
  +------+------+                                                  |
         |                                                         |
         v  upsert                                                 v
  +------+---------------------------------------------------------+------+
  |                      BANCO VETORIAL (índice ANN)                      |
  |   [vec1|meta] [vec2|meta] [vec3|meta] ... [vecN|meta]                 |
  +------+---------------------------------------------------------+------+
         |                                                         |
         |                            top-k similarity + filtros    |
         |                                                         v
         |                                            +------------+-----------+
         |                                            | top-k chunks relevantes |
         |                                            +------------+-----------+
         |                                                         | (opcional)
         |                                                         v  reranking
         |                                            +------------+-----------+
         |                                            |  contexto -> LLM (RAG)  |
         |                                            +------------------------+

  Separe SEMPRE ingestão (batch/Fargate) de consulta (Lambda). Ver seção 6.
```

---

## 2. Pipeline de vetorização de documentos

O pipeline transforma documentos crus em vetores buscáveis. Cada etapa tem impacto direto na qualidade final do RAG. "Garbage in, garbage out" vale duplamente aqui.

### 2.1 Ingestão / loaders

Ler o conteúdo de diferentes formatos e extrair texto (e estrutura) utilizável.

- **PDF:** o mais problemático. PDFs podem ser texto nativo, digitalizados (precisam OCR), ter colunas, tabelas, cabeçalhos/rodapés repetidos. Ferramentas: `PyMuPDF`/`fitz`, `pdfplumber`, `unstructured`, Amazon **Textract** (OCR gerenciado, ótimo para digitalizados e tabelas/formulários).
- **HTML/web:** limpar tags, remover nav/footer/anúncios. `BeautifulSoup`, `trafilatura`, `readability`.
- **DOCX/PPTX/XLSX:** `python-docx`, `python-pptx`, `openpyxl`, ou `unstructured`.
- **Markdown/txt/código:** leitura direta, mas preservar estrutura (headers, blocos de código).
- **Bancos/APIs:** extrair registros e serializar em texto coerente.

No ecossistema RAG, **LlamaIndex** (`SimpleDirectoryReader`, readers específicos) e **Haystack** (`Converters`) oferecem loaders prontos — alinhado aos docs existentes do projeto.

### 2.2 Limpeza e normalização

- Remover cabeçalhos/rodapés/números de página repetidos.
- Normalizar espaços em branco, quebras de linha, hifenização de fim de linha.
- Corrigir encoding (mojibake), remover caracteres de controle.
- Decidir sobre lowercasing (geralmente **não** para embeddings modernos — eles lidam bem com caixa).
- Preservar estrutura semântica útil (títulos, listas, tabelas) — ela ajuda o chunking.
- **Cuidado com PII:** este é o ponto natural para mascarar/tokenizar dados sensíveis antes de gerar embeddings (ver seção 9).

### 2.3 Chunking (o coração do pipeline)

Modelos de embedding têm limite de tokens e, mais importante, **um chunk deve conter uma ideia coerente**. Chunk grande demais dilui o significado (o vetor vira uma "média" de vários assuntos); pequeno demais perde contexto.

#### Estratégias

| Estratégia | Como funciona | Prós | Contras | Quando usar |
|---|---|---|---|---|
| **Tamanho fixo** (por tokens/caracteres) | Corta a cada N tokens (ex.: 512) | Simples, previsível, rápido | Corta no meio de frases/ideias | Baseline, textos homogêneos |
| **Por sentença** | Agrupa frases até ~N tokens | Respeita fronteiras naturais | Frases longas/curtas variam muito | Texto corrido, FAQ |
| **Por parágrafo** | Cada parágrafo (ou grupo) vira um chunk | Boa unidade semântica | Parágrafos muito longos estouram limite | Artigos, documentação |
| **Estrutural / recursivo** | Divide por hierarquia (título → seção → parágrafo → sentença) até caber | Mantém contexto estrutural | Precisa parser de estrutura | Docs técnicos, manuais, Markdown |
| **Semântico** | Usa embeddings para detectar mudança de assunto e cortar ali | Chunks coesos por significado | Mais caro (gera embeddings extras na ingestão) | Documentos heterogêneos, alta qualidade |

#### Overlap (sobreposição)

Repetir os últimos ~10–20% de tokens de um chunk no início do próximo. Evita que uma resposta seja "cortada ao meio" na fronteira entre chunks.

```
Doc:  [.....sentença A.....][.....sentença B.....][.....sentença C.....]
Chunk1: A + B
Chunk2:     B + C     <- B se repete (overlap) para não perder o contexto de fronteira
```

#### Como escolher

- **Comece com:** chunking recursivo/estrutural, ~256–512 tokens, overlap de ~50–80 tokens.
- **Ajuste pela avaliação:** meça a qualidade de recuperação (recall@k, e depois a qualidade das respostas do LLM). Reduza o chunk se as respostas trazem "lixo" junto; aumente se faltam detalhes.
- **Respeite o modelo:** o chunk deve caber com folga no limite de tokens do embedding.
- **Considere o custo:** chunking semântico gera embeddings extras (ver custo de ingestão na seção 7).
- **Guarde o texto original do chunk** nos metadados — é o que vai para o LLM depois.

### 2.4 Escolha do modelo de embedding

O modelo de embedding define **dimensão**, **qualidade**, **custo** e **onde roda** (API gerenciada vs open-source self-hosted). Trocar de modelo depois exige **reindexar tudo**, então escolha com cuidado.

| Modelo | Dimensões | Tipo | Nota de custo | Observações |
|---|---|---|---|---|
| **Amazon Titan Embeddings v2** (Bedrock) | 1024 / 512 / 256 (configurável) | API gerenciada | Por token de entrada | Nativo AWS, integra direto com Bedrock/Lambda; suporta dimensões reduzidas p/ economizar armazenamento |
| **Cohere Embed v3** (Bedrock ou API Cohere) | 1024 | API gerenciada | Por token | Ótimo multilíngue; modos `search_document` vs `search_query` |
| **OpenAI text-embedding-3-small/large** | 1536 / 3072 (redutível) | API gerenciada | Por token | Forte qualidade; fora da AWS (egress/latência/lock-in externo) |
| **bge** (BAAI, ex.: `bge-m3`, `bge-large`) | 384–1024 | Open-source | "Grátis" (roda em GPU/CPU sua) | Top em benchmarks; `bge-m3` faz denso+esparso+multilíngue |
| **e5** (Microsoft, `multilingual-e5-*`) | 384–1024 | Open-source | Custo de compute | Bom multilíngue; usa prefixos `query:`/`passage:` |
| **gte** (Alibaba, `gte-*`) | 384–1024 | Open-source | Custo de compute | Boa qualidade geral |

Critérios de decisão:

- **Qualidade:** consulte o **MTEB leaderboard** (Massive Text Embedding Benchmark), preferindo tarefas de *retrieval* e o **idioma** (português!).
- **Dimensão:** mais dimensões ≈ mais qualidade, mas **mais armazenamento e RAM** e queries mais caras. 1024 é um bom equilíbrio. Titan e OpenAI-3 permitem **truncar** dimensões (Matryoshka) para economizar.
- **Simetria query/documento:** alguns modelos usam prompts/prefixos diferentes para documento e consulta — use corretamente ou o recall cai.
- **Gerenciado vs open-source:** API (Titan/Cohere/OpenAI) = zero ops, custo por token, ideal para volumes baixos/médios. Open-source (bge/e5/gte) = sem custo por token, mas você opera GPU/CPU; compensa em **alto volume** ou requisito de **dados nunca saírem do ambiente**.
- **Consistência absoluta:** o **mesmo modelo e mesma versão** devem ser usados na ingestão e na consulta.

### 2.5 Metadados

Cada vetor deve carregar metadados para filtragem, rastreabilidade e citação:

```json
{
  "id": "doc123_chunk045",
  "text": "texto original do chunk (para enviar ao LLM)",
  "source": "manual_produto_v3.pdf",
  "page": 45,
  "section": "Instalação",
  "tenant_id": "cliente_A",
  "lang": "pt-BR",
  "created_at": "2026-09-25",
  "hash": "sha256:..."
}
```

- **Filtros:** `tenant_id`, `lang`, `data`, `tipo_documento` permitem restringir a busca (ver 3 e multi-tenant).
- **Citação:** `source` + `page`/`section` dão rastreabilidade nas respostas do LLM.
- **Governança:** `created_at`, `hash`, `version` habilitam atualização/dedup/expurgo.

### 2.6 Atualização e reindexação

- **Upsert incremental:** ao mudar um documento, reprocessar só os chunks afetados e fazer `upsert` (usar IDs determinísticos, ex.: `hash` do conteúdo).
- **Expurgo:** deletar por `source`/`tenant_id` (compliance/LGPD). Verifique se o banco suporta delete por filtro eficiente.
- **Reindexação total (obrigatória) quando:** trocar o **modelo de embedding**, mudar a **dimensão**, ou mudar a **métrica de similaridade**. Faça com **índice paralelo** (blue/green): construa o novo, valide recall, aponte a consulta para ele, descarte o antigo.
- **Versão do índice** nos metadados evita misturar vetores de modelos diferentes.

### 2.7 Deduplicação

- **Exata:** hash (sha256) do texto normalizado do chunk; descarta idênticos.
- **Quase-duplicatas:** MinHash/SimHash ou similaridade de embedding acima de um limiar (ex.: cosseno > 0,97) → mantém um representante.
- **Por que importa:** duplicatas inflam custo de armazenamento/queries e **poluem o top-k** (o LLM recebe a mesma informação repetida, desperdiçando janela de contexto).

---

## 3. Estratégias de busca

### 3.1 Busca puramente vetorial (dense)
Embeda a query e busca os *k* vizinhos mais próximos por similaridade. Excelente para **similaridade semântica** ("como cancelo meu plano?" acha "encerramento de assinatura"). Fraca para **termos exatos**, códigos, SKUs, nomes próprios raros, siglas.

### 3.2 Busca híbrida (vetorial + keyword/BM25)
Combina o score **denso** (embedding) com o score **esparso/lexical** (BM25/TF-IDF ou embeddings esparsos tipo SPLADE). Fusão típica: **RRF (Reciprocal Rank Fusion)** ou soma ponderada de scores.

```
query --> [busca vetorial] --> lista A (ranqueada)
      \-> [busca BM25]      --> lista B (ranqueada)
                                     |
                              RRF / soma ponderada
                                     v
                             lista final combinada
```

- **Ganho:** recupera tanto por significado quanto por termo exato. É o **estado da arte prático** para RAG de qualidade.
- **Suporte nativo:** OpenSearch, Weaviate, Qdrant, Vespa, Elasticsearch, Milvus (via combinação), MongoDB Atlas ($search + $vectorSearch). pgvector puro **não** faz híbrido sozinho — combine com o full-text search (`tsvector`) do Postgres.

### 3.3 Reranking (cross-encoder)
Depois de recuperar, digamos, top-50 candidatos, um **cross-encoder** (que lê query + candidato **juntos**, não vetores separados) reordena por relevância fina, e você fica com o top-5.

- **Prós:** salto grande de precisão; corrige erros do retriever.
- **Contras:** caro (uma passada de modelo por candidato) — por isso só sobre um subconjunto. Latência adicional.
- **Opções:** Cohere Rerank (API, disponível via Bedrock/Cohere), cross-encoders open-source (`bge-reranker`, `ms-marco-MiniLM`), Amazon Bedrock (rerank em Knowledge Bases).

```
query + [50 candidatos do retriever] --> cross-encoder --> top-5 reordenado --> LLM
```

### 3.4 Filtros por metadados
Restringir a busca vetorial a um subconjunto (`tenant_id = "A"`, `lang = "pt-BR"`, `data > X`).

- **Pré-filtro** (filtra antes/durante o ANN): mais correto, mas pode degradar o índice se o filtro for muito seletivo. Qdrant, Weaviate, OpenSearch lidam bem.
- **Pós-filtro** (busca k grande e filtra depois): simples, mas pode devolver menos que *k* após filtrar. Aumente o *k* de busca para compensar.

### 3.5 Multi-tenant
Isolar dados de clientes diferentes:

- **Namespace/coleção por tenant:** isolamento forte (Pinecone namespaces, coleções separadas). Melhor para poucos tenants grandes.
- **Metadado `tenant_id` + filtro:** um índice compartilhado, filtra por tenant. Escala para muitos tenants, mas exige disciplina de filtro (nunca esquecer o filtro = vazamento de dados!).
- **Índice físico por tenant:** máximo isolamento, custo operacional/piso mensal por tenant. Só para requisitos rígidos.

---

## 4. Catálogo de opções de banco vetorial

> Cada opção traz: **o que é**, **prós**, **contras**, **quando usar** e uma **nota de custo** (modelo de cobrança, ilustrativo). Preços exatos: sempre confira as páginas oficiais.

### 4.a Serviços gerenciados / proprietários (SaaS)

#### Pinecone
- **O que é:** banco vetorial gerenciado puro, pioneiro do mercado. Serverless (pay-per-use) e por pods.
- **Prós:** operação zero, escala transparente, baixa latência, namespaces para multi-tenant, híbrido (sparse-dense). Muito fácil de começar.
- **Contras:** proprietário (lock-in), dados fora da sua conta AWS (a menos que use private link), custo pode crescer rápido em volume alto. Sem controle fino de infra.
- **Quando usar:** time pequeno, quer velocidade de entrega, não quer operar nada, volume pequeno/médio.
- **Custo:** serverless cobra por **armazenamento (GB-mês)** + **operações de leitura/escrita** (read/write units); modelo por **pod** cobra por hora de pod (tamanho × réplicas). Tem tier grátis pequeno.

#### Weaviate Cloud
- **O que é:** versão gerenciada do Weaviate (open-source). Vetorial + híbrido + módulos de vetorização integrados.
- **Prós:** híbrido nativo (BM25 + vetor) muito bom, GraphQL/REST, multi-tenancy nativo, pode auto-vetorizar. Também dá para self-hostar (sem lock-in de dados/formato).
- **Contras:** curva de aprendizado do modelo de dados/schema; custo gerenciado.
- **Quando usar:** quer híbrido forte e multi-tenant, com opção futura de trazer para casa.
- **Custo:** por dimensão-vetor armazenada / SLA tier; cobrança baseada em nº de vetores × dimensões e tier de disponibilidade.

#### Qdrant Cloud
- **O que é:** Qdrant (open-source, escrito em Rust) gerenciado. Rápido, filtros ricos, quantização embutida.
- **Prós:** excelente performance/custo, filtros por payload muito poderosos, quantização (scalar/product/binary) reduz RAM/custo, híbrido. Self-host trivial (sem lock-in).
- **Contras:** ecossistema um pouco menor que Pinecone; você configura mais coisas.
- **Quando usar:** quer performance e controle de custo com filtros ricos; caminho fácil SaaS→self-hosted.
- **Custo:** por recursos do cluster (RAM/CPU/disco) por hora; tier grátis pequeno.

#### Zilliz Cloud (Milvus gerenciado)
- **O que é:** versão gerenciada do **Milvus** (open-source), voltada a **escala massiva** (bilhões de vetores).
- **Prós:** escala altíssima, múltiplos tipos de índice (HNSW, IVF, IVFPQ, DiskANN), separação compute/armazenamento, GPU.
- **Contras:** complexidade conceitual maior; overkill para volumes pequenos.
- **Quando usar:** bilhões de vetores, requisitos de escala/throughput extremos.
- **Custo:** por unidades de compute/armazenamento (CU) do cluster; tier serverless com piso baixo.

#### MongoDB Atlas Vector Search
- **O que é:** capacidade de vetor **dentro** do MongoDB Atlas. Um índice vetorial ao lado dos seus documentos.
- **Prós:** se você **já usa MongoDB**, dados operacionais e vetores no mesmo lugar (menos ETL), híbrido ($search + $vectorSearch), gerenciado.
- **Contras:** não é especialista em vetores como Pinecone/Qdrant; custo do Atlas; escala vetorial limitada frente a Milvus.
- **Quando usar:** já é casa MongoDB e quer RAG sem novo sistema.
- **Custo:** incluso no cluster Atlas (tier de instância/hora) + Search Nodes dedicados opcionais.

#### Redis Enterprise / Redis Cloud (vector)
- **O que é:** busca vetorial no Redis (módulo RediSearch), em memória.
- **Prós:** latência **ultrabaixa** (in-memory), híbrido, ótimo quando você já usa Redis para cache/sessão.
- **Contras:** tudo em RAM = **caro por GB** em grande escala; persistência/HA exigem cuidado.
- **Quando usar:** baixa latência crítica, volume moderado, cache + vetor juntos.
- **Custo:** por memória do cluster (GB-hora); RAM é o fator dominante.

### 4.b Nativos AWS / gerenciados na nuvem

Estes são os mais relevantes para o projeto (orquestrador em Bedrock/Lambda), pois vivem **dentro da conta AWS**, na mesma VPC, com IAM.

#### Amazon OpenSearch Serverless (coleção do tipo Vector Search, k-NN)
- **O que é:** OpenSearch gerenciado e serverless com engine k-NN (HNSW/FAISS/nmslib) e busca lexical (BM25) no mesmo motor.
- **Prós:** **híbrido nativo** (vetor + BM25 + filtros), serverless (escala por OCU), integração IAM/VPC, é o **store padrão** de muitas configurações de Bedrock Knowledge Bases.
- **Contras:** tem um **piso mensal** relevante por causa do mínimo de OCUs (mesmo ocioso custa); afinar k-NN exige conhecimento.
- **Quando usar:** quer híbrido gerenciado 100% AWS, integração direta com Bedrock KB.
- **Custo:** por **OCU** (OpenSearch Compute Unit) por hora, separadas para indexação e busca, + armazenamento. **Atenção ao mínimo de OCUs** (piso mensal não trivial).

#### Amazon Aurora PostgreSQL + pgvector
- **O que é:** Postgres gerenciado (Aurora) com a extensão **pgvector** (índices HNSW e IVFFlat).
- **Prós:** SQL + vetor no mesmo banco (JOIN com dados relacionais!), transacional, familiar, Aurora Serverless v2 escala por ACU, integra com Bedrock KB. **Sem lock-in de formato** (é Postgres).
- **Contras:** pgvector escala bem até milhões (não bilhões); híbrido exige combinar com `tsvector`; HNSW em RAM limita volume por instância.
- **Quando usar:** volume pequeno/médio, quer relacional + vetor juntos, prioriza simplicidade e portabilidade.
- **Custo:** Aurora Serverless v2 por **ACU-hora** (mín. configurável, pode escalar a zero em cenários recentes) + armazenamento + I/O. Provisionado: por hora de instância.

#### Amazon RDS PostgreSQL + pgvector
- **O que é:** igual acima, mas RDS "clássico" (instância fixa) em vez de Aurora.
- **Prós:** mais barato/simples que Aurora para cargas estáveis; mesmo pgvector.
- **Contras:** sem o auto-scaling fino do Aurora Serverless; você dimensiona a instância.
- **Quando usar:** carga previsível, quer o menor custo de Postgres gerenciado com vetor.
- **Custo:** por **hora de instância** (classe) + armazenamento EBS. Reserved Instances reduzem bastante.

#### Amazon MemoryDB / ElastiCache (vector search)
- **O que é:** busca vetorial em memória sobre MemoryDB (Redis durável) ou ElastiCache.
- **Prós:** latência **ultrabaixa**, in-memory, durável (MemoryDB), 100% AWS.
- **Contras:** RAM cara em escala; melhor para volumes moderados com requisito de latência.
- **Quando usar:** latência crítica + volume moderado, já usando Redis na stack.
- **Custo:** por **hora de nó** (RAM é o driver) + (MemoryDB) custo de log de dados.

#### Amazon Kendra
- **O que é:** serviço de **busca empresarial gerenciada** (semântica + conectores prontos para SharePoint, S3, Salesforce, etc.). Abstrai embeddings/índice.
- **Prós:** conectores nativos, ranking semântico pronto, ACL/permissões por documento, quase zero engenharia de pipeline.
- **Contras:** **caro** (piso mensal alto na edição Enterprise), menos flexível que montar seu próprio store, é uma "caixa" (menos controle).
- **Quando usar:** busca corporativa com muitas fontes e controle de acesso, orçamento confortável, pouca vontade de engenharia.
- **Custo:** por **hora de índice** (Developer/Enterprise edition), com **piso mensal alto**; + custo de conectores/scans.

#### Amazon Bedrock Knowledge Bases
- **O que é:** **RAG gerenciado**. Você aponta um bucket S3; ele faz chunking, embeddings (Titan/Cohere) e grava em um **vector store por baixo** (OpenSearch Serverless, Aurora pgvector, Pinecone, Redis Enterprise, MongoDB Atlas, Neptune Analytics). Expõe `Retrieve` e `RetrieveAndGenerate`.
- **Prós:** monta o pipeline inteiro para você; integra nativamente com agentes/LLMs do Bedrock; troca o store por config. **Menos código** no orquestrador.
- **Contras:** você paga o store escolhido **+** o processamento; menos controle fino sobre chunking/rerank (embora crescente); ainda depende do piso do store (ex.: OCUs do OpenSearch).
- **Quando usar:** quer RAG rápido, 100% AWS, sem construir o pipeline manualmente.
- **Custo:** **embeddings por token** (ingestão/consulta) + **custo do vector store subjacente** + (RetrieveAndGenerate) tokens do LLM.

#### Amazon DocumentDB (vector search)
- **O que é:** DocumentDB (compatível com MongoDB) com índice vetorial (HNSW/IVFFlat).
- **Prós:** se já usa DocumentDB, vetor no mesmo banco; gerenciado; VPC/IAM.
- **Contras:** especialização vetorial menor; escala vetorial moderada.
- **Quando usar:** já é casa DocumentDB e quer adicionar RAG sem novo sistema.
- **Custo:** por **hora de instância** + armazenamento/IO.

#### Amazon Neptune Analytics
- **O que é:** engine analítico de **grafos** com **busca vetorial integrada**. Combina relações de grafo + similaridade vetorial (GraphRAG).
- **Prós:** ideal para **GraphRAG** — unir conhecimento estruturado (grafo) com semântico (vetor); pode ser store de Bedrock KB.
- **Contras:** nicho; overkill se você não precisa de grafo.
- **Quando usar:** casos de conhecimento altamente relacionado onde grafo + vetor somam.
- **Custo:** por **m-NCU** (capacidade) por hora + armazenamento.

### 4.c Self-hosted (você opera)

Você roda em **EC2 / ECS / EKS**. Sem custo de licença (open-source), mas com **custo de infra + operação** (ver seção 8).

#### Qdrant (self-hosted)
- **O que é:** o mesmo Qdrant do Cloud, rodando em container seu.
- **Prós:** performance excelente (Rust), quantização para reduzir RAM, filtros ricos, fácil de containerizar, snapshots. Ótima relação custo/benefício.
- **Contras:** você opera HA/backup/upgrades.
- **Quando usar:** quer controle e custo baixo com boa performance; caminho natural vindo do Qdrant Cloud.
- **Custo:** custo de EC2/EBS + operação. Licença: **zero** (Apache 2.0).

#### Weaviate (self-hosted)
- **O que é:** Weaviate open-source em container.
- **Prós:** híbrido nativo forte, multi-tenancy, módulos. Mesmo formato do Cloud.
- **Contras:** consumo de RAM (HNSW); operação por sua conta.
- **Quando usar:** híbrido/multi-tenant com controle total.
- **Custo:** infra + operação. Licença: **zero**.

#### Milvus (self-hosted)
- **O que é:** banco vetorial de **escala massiva**, arquitetura distribuída (várias componentes: proxy, query nodes, data nodes, etcd, object storage).
- **Prós:** bilhões de vetores, múltiplos índices (HNSW/IVF/IVFPQ/DiskANN), GPU, separa compute/armazenamento.
- **Contras:** **complexo de operar** (muitos componentes) — considere EKS + operador. Overkill para volumes pequenos.
- **Quando usar:** escala massiva com equipe de plataforma.
- **Custo:** infra significativa (múltiplos nós + object storage) + operação. Licença: **zero**.

#### Chroma (self-hosted)
- **O que é:** banco vetorial simples, dev-friendly, popular para protótipos e apps pequenos.
- **Prós:** trivial de subir, API amigável, boa DX, integra com LlamaIndex/LangChain.
- **Contras:** não pensado para altíssima escala/HA robusto; recursos de produção mais enxutos.
- **Quando usar:** protótipos, POCs, apps de volume pequeno.
- **Custo:** infra mínima + operação. Licença: **zero**.

#### pgvector (Postgres self-managed)
- **O que é:** Postgres com pgvector rodando em EC2/container seu (em vez de RDS/Aurora).
- **Prós:** SQL + vetor, sem custo de serviço gerenciado, controle total.
- **Contras:** você opera Postgres (backup, réplica, tuning); escala vetorial moderada.
- **Quando usar:** já opera Postgres, volume pequeno/médio, quer custo mínimo.
- **Custo:** EC2/EBS + operação. Licença: **zero**.

#### Vespa
- **O que é:** engine de busca/serving de larga escala (Yahoo/Verizon), vetor + lexical + ranking ML no mesmo motor.
- **Prós:** híbrido + ranking sofisticado, escala altíssima, tempo real.
- **Contras:** curva de aprendizado íngreme; operação complexa.
- **Quando usar:** busca/recomendação de larga escala com ranking ML avançado.
- **Custo:** infra + operação (especializada). Licença: **zero**.

#### Elasticsearch / OpenSearch (self-managed)
- **O que é:** o motor de busca clássico com k-NN vetorial, rodando por sua conta.
- **Prós:** híbrido (BM25 + vetor) maduro, ecossistema enorme, filtros/agregações.
- **Contras:** RAM/tuning de cluster; operação de search cluster não é trivial.
- **Quando usar:** já opera ES/OpenSearch e quer somar vetor; híbrido com controle total.
- **Custo:** infra do cluster + operação. Licença: **zero** (OpenSearch Apache 2.0).

#### FAISS (biblioteca — **não** é um banco)
- **O que é:** biblioteca C++/Python do Meta para busca de similaridade (HNSW, IVF, IVFPQ, GPU). É o **motor** que muitos bancos usam por baixo.
- **Prós:** rapidíssima, muitos índices, suporte a GPU, controle total.
- **Contras:** **não tem** persistência/API/filtros/HA prontos — você constrói tudo em volta (servidor, storage, metadados, updates).
- **Quando usar:** você quer embutir busca vetorial num app/serviço próprio e controlar cada detalhe; ou como componente interno.
- **Custo:** só infra. Licença: **zero** (MIT).

#### LanceDB
- **O que é:** banco vetorial **embutido** (embedded) baseado no formato de arquivo colunar **Lance**, orientado a disco (não precisa tudo em RAM).
- **Prós:** roda embutido (como SQLite dos vetores), armazenamento em disco/objeto (S3) barato, bom para grandes volumes com custo baixo, versionamento de dados.
- **Contras:** ecossistema mais novo; menos recursos de servidor multiusuário robusto (embora existam ofertas).
- **Quando usar:** app que quer vetor local/serverless barato, dados no S3, sem servidor dedicado.
- **Custo:** só armazenamento (disco/S3) + compute do app. Licença: **zero** (Apache 2.0).

### 4.d On-premise / local / edge

Roda **no seu data center**, na máquina do usuário, ou em dispositivos edge. Motivadores: **dados que não podem sair** (compliance/LGPD/setor regulado), **offline**, **latência local**.

#### FAISS (local)
- **O que é:** a biblioteca (acima) usada localmente/embarcada.
- **Prós:** sem dependência de rede, rápido, roda em CPU/GPU local.
- **Contras:** você constrói persistência/atualização; índice em memória.
- **Quando usar:** app desktop/servidor local, batch analítico, incorporado a outro serviço.

#### Chroma (local)
- **O que é:** Chroma em modo embutido/local, persistindo em disco.
- **Prós:** simplicidade máxima para apps locais e POCs.
- **Contras:** escala/HA limitados.
- **Quando usar:** protótipo local, app pequeno on-prem.

#### LanceDB (local)
- **O que é:** embutido, persiste em arquivos Lance no disco local.
- **Prós:** disco-orientado (não estoura RAM), sem servidor, barato, versionado.
- **Quando usar:** grande volume local/edge com pouca RAM; app "serverless" no dispositivo.

#### SQLite-vss / sqlite-vec
- **O que é:** extensões que adicionam busca vetorial ao **SQLite**. `sqlite-vec` é a evolução mais recente/leve.
- **Prós:** um único arquivo, zero servidor, embarcável em qualquer lugar (mobile, edge, desktop), SQL familiar.
- **Contras:** escala pequena (milhares a poucos milhões), recursos ANN limitados.
- **Quando usar:** RAG **local/offline**, apps móveis/edge, volume pequeno.

#### pgvector (on-prem)
- **O que é:** Postgres + pgvector no seu data center.
- **Prós:** SQL + vetor, maduro, controle total, dados nunca saem.
- **Contras:** você opera tudo; escala vetorial moderada.
- **Quando usar:** já tem Postgres on-prem e requisito de dados locais.

#### Milvus (on-prem)
- **O que é:** Milvus no seu cluster (Kubernetes on-prem).
- **Prós:** escala massiva sem sair do data center.
- **Contras:** operação complexa (equipe de plataforma).
- **Quando usar:** grande escala + exigência on-prem estrita.

---

## 5. Tabela comparativa grande

Legenda de facilidade operacional: 🟢 baixa carga / 🟡 média / 🔴 alta. Lock-in: 🟢 baixo / 🟡 médio / 🔴 alto.

| Opção | Tipo | Modelo de custo | Escala típica | Híbrida (vetor+BM25) | Facilidade op. | Lock-in | Encaixe Lambda/Bedrock |
|---|---|---|---|---|---|---|---|
| **Pinecone** | SaaS | GB-mês + read/write units (ou pod/hora) | Milhões–centenas de M | Sim (sparse-dense) | 🟢 | 🔴 | Bom (SDK; store de Bedrock KB) |
| **Weaviate Cloud** | SaaS | Vetores×dim + tier SLA | Milhões | Sim (nativo) | 🟢 | 🟡 (open-source) | Bom (API); store de KB |
| **Qdrant Cloud** | SaaS | Recursos do cluster/hora | Milhões | Sim | 🟢 | 🟢 (open-source) | Bom (API) |
| **Zilliz (Milvus)** | SaaS | CU compute/armazenamento | Bilhões | Parcial (combinar) | 🟢 | 🟡 (open-source) | Bom (API) |
| **MongoDB Atlas VS** | SaaS | Cluster/hora + Search Nodes | Milhões | Sim ($search) | 🟢 | 🟡 | Bom se já usa Mongo; store de KB |
| **Redis Cloud (vector)** | SaaS | Memória (GB-hora) | Milhões (RAM-bound) | Sim | 🟢 | 🟡 | Bom; store de KB |
| **OpenSearch Serverless** | AWS | OCU/hora + storage (**piso**) | Milhões–dezenas de M | **Sim (nativo)** | 🟢 | 🟡 | **Ótimo** (VPC/IAM; store padrão de KB) |
| **Aurora PG + pgvector** | AWS | ACU-hora + storage/IO | Milhões | Combinar c/ tsvector | 🟡 | 🟢 (Postgres) | **Ótimo** (VPC/IAM; store de KB) |
| **RDS PG + pgvector** | AWS | Instância/hora + EBS | Milhões | Combinar c/ tsvector | 🟡 | 🟢 (Postgres) | Ótimo (VPC/IAM) |
| **MemoryDB/ElastiCache** | AWS | Nó/hora (RAM) | Milhões (RAM-bound) | Sim | 🟡 | 🟡 | Ótimo (latência baixa) |
| **Amazon Kendra** | AWS | Índice/hora (**piso alto**) | Empresarial (docs) | Semântico gerenciado | 🟢 | 🔴 | Bom (busca pronta) |
| **Bedrock Knowledge Bases** | AWS | Tokens + store subjacente | Depende do store | Depende do store | 🟢 | 🟡 | **Nativo** (Retrieve/RAG) |
| **DocumentDB (vector)** | AWS | Instância/hora + storage | Milhões | Não nativo | 🟡 | 🟡 | Bom se já usa DocDB |
| **Neptune Analytics** | AWS | m-NCU/hora + storage | Milhões (GraphRAG) | Grafo + vetor | 🟡 | 🟡 | Bom (GraphRAG; store de KB) |
| **Qdrant self-hosted** | Self | EC2/EBS + operação | Milhões–centenas de M | Sim | 🔴 | 🟢 | Bom (endpoint na VPC) |
| **Weaviate self-hosted** | Self | EC2/EBS + operação | Milhões | Sim | 🔴 | 🟢 | Bom |
| **Milvus self-hosted** | Self | Cluster (multi-nó) + operação | **Bilhões** | Parcial | 🔴 | 🟢 | Bom (escala) |
| **Chroma self-hosted** | Self | EC2 pequeno + operação | Pequeno–médio | Limitado | 🟡 | 🟢 | Ok (POC) |
| **pgvector self-managed** | Self | EC2/EBS + operação | Milhões | tsvector | 🔴 | 🟢 | Ok |
| **Vespa** | Self | Cluster + operação | Bilhões | **Sim (avançado)** | 🔴 | 🟢 | Ok (busca/rank) |
| **ES/OpenSearch self-mgd** | Self | Cluster + operação | Dezenas de M | **Sim (maduro)** | 🔴 | 🟢 | Ok |
| **FAISS (lib)** | Self/On-prem | Só infra (você constrói tudo) | Milhões–bilhões | Não (você faz) | 🔴 | 🟢 | Precisa envelope de serviço |
| **LanceDB** | Self/On-prem | Storage (disco/S3) + compute | Milhões (disco) | Parcial | 🟡 | 🟢 | Bom (embutido/serverless) |
| **FAISS local** | On-prem | Infra local | Milhões | Não | 🔴 | 🟢 | N/A (local) |
| **Chroma local** | On-prem | Infra local | Pequeno | Limitado | 🟢 | 🟢 | N/A (local) |
| **sqlite-vec / vss** | On-prem | ~zero (arquivo) | Pequeno (≤ ~M) | Não | 🟢 | 🟢 | N/A (edge/mobile) |
| **pgvector on-prem** | On-prem | Hardware + operação | Milhões | tsvector | 🔴 | 🟢 | N/A (local) |
| **Milvus on-prem** | On-prem | Cluster + operação | Bilhões | Parcial | 🔴 | 🟢 | N/A (local) |

**Leitura rápida da tabela:**
- **100% AWS + híbrido gerenciado:** OpenSearch Serverless.
- **100% AWS + simples/relacional:** Aurora/RDS pgvector.
- **RAG pronto sem código de pipeline:** Bedrock Knowledge Bases.
- **Menor custo em escala com controle:** Qdrant/Milvus self-hosted.
- **Dados não podem sair:** on-prem (pgvector/Milvus) ou edge (sqlite-vec/LanceDB).

---

## 6. Integração com o orquestrador

### 6.1 O padrão central: separar INGESTÃO da CONSULTA

Este é o princípio arquitetural mais importante — e reforça o que os docs de orquestrador do projeto já pregam.

```
  INGESTÃO (batch, pesada, esporádica)          CONSULTA (online, leve, frequente)
  ====================================          ===================================
  S3 (docs) --> EventBridge/Step Functions      API Gateway --> Lambda (orquestrador)
        --> Fargate/Batch (loader+chunk           --> Bedrock (embedding da query)
            +embedding em lote)                   --> Vector DB (top-k + filtros)
        --> upsert no Vector DB                   --> [rerank opcional]
                                                  --> Bedrock LLM (RAG)
```

**Por que separar:**
- **Ingestão** é CPU/IO-intensiva, longa, em lote, tolera minutos. **Fargate/AWS Batch/Step Functions** são ideais (sem limite de 15 min do Lambda, escala horizontal, controle de custo).
- **Consulta** precisa ser rápida e barata por chamada. **Lambda** é perfeito: só embeda a query e faz um top-k. Mantenha o Lambda "magro" (sem carregar modelos pesados; use Bedrock para embeddings).
- **Nunca** faça reindexação/ingestão massiva dentro do Lambda de consulta.

**Regras para o Lambda de consulta:**
- Reuse conexões (client fora do handler) — pool de conexão para pgvector, cliente HTTP keep-alive para OpenSearch/Pinecone.
- Coloque o Lambda na **mesma VPC/subnets** do banco (Aurora/OpenSearch/Qdrant self-hosted) — use VPC endpoints para Bedrock/S3.
- Trate timeouts/retries; use secrets do **Secrets Manager** (nunca hardcode).

### 6.2 Exemplos de código (consulta) com Bedrock embeddings

Todos geram o embedding da query com **Titan** no Bedrock e depois consultam o store. Trechos curtos e ilustrativos.

**Helper comum — embedding via Bedrock:**
```python
import boto3, json

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

def embed_query(text: str) -> list[float]:
    resp = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps({"inputText": text, "dimensions": 1024, "normalize": True}),
    )
    return json.loads(resp["body"].read())["embedding"]
```

**a) pgvector (Aurora/RDS PostgreSQL):**
```python
import psycopg
# conexão criada FORA do handler (reuso entre invocações)
conn = psycopg.connect(host=DB_HOST, dbname="rag", user=DB_USER, password=DB_PWD)

def search_pgvector(question: str, tenant_id: str, k: int = 5):
    qvec = embed_query(question)
    # '<=>' = distância de cosseno no pgvector; filtro por metadado no WHERE
    sql = """
        SELECT id, text, source, page, 1 - (embedding <=> %s::vector) AS score
        FROM chunks
        WHERE tenant_id = %s
        ORDER BY embedding <=> %s::vector
        LIMIT %s;
    """
    with conn.cursor() as cur:
        cur.execute(sql, (qvec, tenant_id, qvec, k))
        return cur.fetchall()
```

**b) Amazon OpenSearch (Serverless / k-NN) — busca híbrida:**
```python
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
import boto3

region = "us-east-1"
creds = boto3.Session().get_credentials()
auth = AWS4Auth(creds.access_key, creds.secret_key, region, "aoss",
                session_token=creds.token)
client = OpenSearch(hosts=[{"host": OS_ENDPOINT, "port": 443}],
                    http_auth=auth, use_ssl=True, verify_certs=True,
                    connection_class=RequestsHttpConnection)

def search_opensearch(question: str, tenant_id: str, k: int = 5):
    qvec = embed_query(question)
    body = {
        "size": k,
        "query": {
            "bool": {
                "filter": [{"term": {"tenant_id": tenant_id}}],
                "should": [
                    {"knn": {"embedding": {"vector": qvec, "k": k}}},   # denso
                    {"match": {"text": question}}                        # BM25 (híbrido)
                ]
            }
        }
    }
    res = client.search(index="chunks", body=body)
    return [(h["_source"]["text"], h["_score"]) for h in res["hits"]["hits"]]
```

**c) Pinecone (consulta):**
```python
from pinecone import Pinecone

pc = Pinecone(api_key=PINECONE_API_KEY)         # criado fora do handler
index = pc.Index("rag-index")

def search_pinecone(question: str, tenant_id: str, k: int = 5):
    qvec = embed_query(question)
    res = index.query(
        vector=qvec,
        top_k=k,
        namespace=tenant_id,          # multi-tenant por namespace
        filter={"lang": {"$eq": "pt-BR"}},
        include_metadata=True,
    )
    return [(m["metadata"]["text"], m["score"]) for m in res["matches"]]
```

**d) Bedrock Knowledge Bases (RAG gerenciado — sem gerenciar store diretamente):**
```python
agent = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

def rag_answer(question: str):
    return agent.retrieve_and_generate(
        input={"text": question},
        retrieveAndGenerateConfiguration={
            "type": "KNOWLEDGE_BASE",
            "knowledgeBaseConfiguration": {
                "knowledgeBaseId": KB_ID,
                "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20240620-v1:0",
            },
        },
    )["output"]["text"]
```

> Observação: em todos os casos, o **mesmo modelo/dimensão** usado na ingestão deve ser usado na consulta. Em produção, adicione rerank (Cohere Rerank / cross-encoder) sobre o top-k antes de mandar ao LLM quando a precisão for crítica.

---

## 7. Análise de custo detalhada

> ⚠️ **Repetindo o aviso:** os números abaixo são **ordens de grandeza ilustrativas** (padrões de 2024–2025, região tipo `us-east-1`), para raciocínio comparativo. **Confirme sempre** nas páginas de preço oficiais e na [Calculadora AWS](https://calculator.aws/). Descontos, reservas, região e negociação mudam tudo.

### 7.1 Componentes de custo (o que você paga)

| Componente | O que é | Onde pesa mais |
|---|---|---|
| **Armazenamento (por vetor/GB)** | Guardar embeddings + metadados + índice | Escala grande; HNSW infla (grafo em RAM) |
| **Compute / queries** | CPU/RAM/OCU/ACU para servir buscas e indexar | Alto QPS; índices em memória |
| **Embeddings de ingestão** | Tokens processados pelo modelo ao vetorizar docs | Ingestão inicial e reindexações |
| **Embeddings de consulta** | Tokens da query a cada busca | Alto volume de perguntas |
| **Egress / rede** | Dados saindo (entre AZ/região/internet) | Store fora da AWS; cross-AZ |
| **Piso mensal mínimo** | Custo fixo mesmo ocioso (mín. de OCU/ACU/pod/índice) | SaaS/serverless com mínimos altos (OpenSearch, Kendra) |
| **Operação (self-hosted)** | Pessoas, backup, HA, upgrades | Self-hosted/on-prem (ver seção 8) |

### 7.2 Tamanho de um vetor (base do cálculo de armazenamento)

Um vetor **float32** de dimensão *d* ocupa `d × 4 bytes` (bruto). Índices e metadados adicionam overhead (HNSW ~1,5–2×; metadados variam).

| Dimensão | Bytes/vetor (bruto float32) | + HNSW/metadados (~2×) |
|---|---|---|
| 384 | ~1,5 KB | ~3 KB |
| 768 | ~3 KB | ~6 KB |
| 1024 | ~4 KB | ~8 KB |
| 1536 | ~6 KB | ~12 KB |

**Estimativas de armazenamento bruto (dim 1024, ~8 KB/vetor com overhead):**
- 100k vetores → ~0,8 GB
- 10M vetores → ~80 GB
- 500M vetores → ~4 TB

> **Quantização** (scalar int8, PQ, binário) pode reduzir isso 4×–32×, ao custo de um pouco de recall — decisivo em escala grande (Qdrant, Milvus, IVFPQ).

### 7.3 Custo de embeddings de ingestão (por tokens)

Ilustrativo: modelos de embedding gerenciados custam na casa de **~US$ 0,02 por 1M de tokens** (ordem de grandeza; Titan v2/OpenAI-3-small são baratos; large custa mais).

Suponha ~1 chunk = ~512 tokens:

| Cenário | Vetores | Tokens (~512/chunk) | Custo ingestão (~$0,02/1M tok) |
|---|---|---|---|
| Pequeno | 100k | ~51M tokens | **~US$ 1** (uma vez) |
| Médio | 10M | ~5,1 bi tokens | **~US$ 100** (uma vez) |
| Grande | 500M | ~256 bi tokens | **~US$ 5.000** (uma vez) |

> A ingestão é **custo pontual** (paga na vez inicial e em reindexações). Modelos **open-source** (bge/e5) eliminam esse custo por token, trocando por custo de GPU/tempo. **Reindexar** (trocar de modelo) repaga esse valor — mais um motivo para escolher o modelo com cuidado.

### 7.4 Cenário PEQUENO — 100k vetores (~0,8 GB)

| Opção | Custo mensal (ordem de grandeza) | Comentário |
|---|---|---|
| **pgvector local / sqlite-vec** | **~US$ 0** | Cabe em um t3.small ou até no próprio app |
| **RDS pgvector (t3/t4g pequeno)** | **~US$ 15–50** | Barato, relacional + vetor |
| **Aurora Serverless v2 (mín. baixo)** | ~US$ 40–90 | ACUs mínimas; escala a zero em cenários ociosos |
| **Qdrant/Chroma self-hosted (t3.small)** | ~US$ 15–40 | + operação |
| **Pinecone serverless** | ~US$ 0–25 | Tier grátis pode cobrir; barato nesse volume |
| **OpenSearch Serverless** | **~US$ 300+** | ❗ **Piso de OCUs** domina — caro para volume pequeno |
| **Kendra** | **~US$ 800+** | ❗ Piso mensal alto; overkill aqui |

**Vencedor típico:** pgvector (RDS pequeno) ou Pinecone serverless / sqlite-vec/local. **Evite** OpenSearch Serverless e Kendra por causa do piso.

### 7.5 Cenário MÉDIO — 10M vetores (~80 GB)

| Opção | Custo mensal (ordem de grandeza) | Comentário |
|---|---|---|
| **Qdrant self-hosted** (r6g.xlarge + EBS) | **~US$ 200–400** | Ótimo custo/benefício; quantização ajuda |
| **Aurora pgvector** (instância média) | ~US$ 300–600 | pgvector ainda confortável nesse volume |
| **OpenSearch Serverless** | ~US$ 500–1.500 | Piso dilui; híbrido gerenciado incluso |
| **Pinecone serverless** | ~US$ 300–800 | Depende de QPS (read units) |
| **Milvus self-hosted** | ~US$ 300–700 | Começa a brilhar; + operação |
| **Redis/MemoryDB** | **~US$ 1.000+** | RAM cara para 80 GB |

**Vencedor típico:** Qdrant/Milvus self-hosted (se você tolera operar) ou OpenSearch Serverless / Pinecone (se quer gerenciado). Redis fica caro (in-memory).

### 7.6 Cenário GRANDE — 500M vetores (~4 TB, ou muito menos com quantização)

| Opção | Custo mensal (ordem de grandeza) | Comentário |
|---|---|---|
| **Milvus self-hosted** (cluster + object storage + PQ) | **~US$ 2.000–6.000** | Projetado para isso; quantização derruba RAM/custo |
| **Qdrant self-hosted** (cluster + quantização) | ~US$ 2.500–7.000 | Viável com quantização/disco |
| **LanceDB (S3 + compute)** | **~US$ 1.000–4.000** | Disco/S3-orientado; barato em armazenamento |
| **OpenSearch Serverless** | ~US$ 5.000–20.000+ | Escala, mas OCUs crescem muito |
| **Pinecone** | ~US$ 5.000–20.000+ | Simples, porém caro nessa escala |
| **Redis/MemoryDB** | **proibitivo** | 4 TB em RAM = dezenas de milhares/mês |

**Vencedor típico:** self-hosted com **quantização** (Milvus/Qdrant) ou **LanceDB** (disco/S3). Gerenciados puros ficam caros; Redis é inviável.

### 7.7 Modelos de cobrança lado a lado

| Modelo | Como cobra | Implicação |
|---|---|---|
| **Pinecone (serverless)** | GB-mês armazenados + read/write units | Paga pelo uso; barato ocioso, sobe com QPS/volume |
| **Pinecone (pod)** | Pod/hora (tamanho × réplicas) | Custo fixo previsível; paga ocioso |
| **OpenSearch Serverless** | OCU/hora (indexação + busca) + storage | **Piso de OCUs** → caro em volume pequeno |
| **Aurora Serverless v2** | ACU-hora + storage + IO | Escala fino; pode reduzir muito ocioso |
| **RDS** | Instância/hora + EBS | Previsível; Reserved Instances economizam |
| **Kendra** | Índice/hora (Developer/Enterprise) | **Piso mensal alto** |
| **Self-hosted (EC2)** | Instância/hora + EBS/S3 | **Licença zero**, mas + operação (seção 8) |
| **On-prem** | CAPEX hardware + OPEX operação | Sem conta de nuvem; custo de gente/infra local |

### 7.8 Regra de bolso por faixa

```
Volume/complexidade  ->  Opção que TENDE a ser mais econômica
-------------------------------------------------------------
POC / < 1M           ->  pgvector (RDS pequeno), sqlite-vec, Pinecone serverless
1M – 50M gerenciado  ->  OpenSearch Serverless (se quer híbrido AWS) ou Pinecone
1M – 50M com equipe  ->  Qdrant self-hosted (melhor custo/controle)
> 100M               ->  Milvus/Qdrant self-hosted + quantização, ou LanceDB (S3)
Dados não saem       ->  pgvector/Milvus on-prem; sqlite-vec/LanceDB no edge
RAG sem código       ->  Bedrock Knowledge Bases (paga store por baixo + tokens)
```

---

## 8. Custo oculto do self-hosted / on-prem

"Licença zero" **não** é "custo zero". O open-source elimina a taxa de software, mas você assume tudo abaixo:

| Custo oculto | Detalhe | Impacto |
|---|---|---|
| **Operação / SRE** | Monitoramento, alertas, plantão, tuning de índice (M, ef, nprobe) | Horas de pessoas caras, contínuo |
| **Backup / restore** | Snapshots, testes de restauração, retenção | Storage + processo + risco se falhar |
| **Alta disponibilidade (HA)** | Réplicas, multi-AZ, failover, quorum (etcd no Milvus) | Multiplica infra (2×–3×) |
| **Upgrades / patching** | Versões do banco, SO, CVEs de segurança | Janelas de manutenção, risco de regressão |
| **Escalabilidade** | Resharding, rebalanceamento, migração de dados ao crescer | Projetos inteiros de migração |
| **Capacity planning** | Dimensionar RAM/disco antes de estourar | Sobre-provisiona (desperdício) ou falta (incidente) |
| **Segurança** | TLS, rotação de credenciais, hardening, auditoria | Contínuo; exige expertise |
| **Onboarding / bus factor** | Conhecimento concentrado em poucas pessoas | Risco operacional |

### Quando self-hosted vale a pena

**Vale a pena quando:**
- **Volume alto e sustentado** (dezenas/centenas de milhões+): a economia de infra supera o custo de operação.
- Você **já tem** equipe de plataforma/SRE e maturidade em containers/K8s.
- Requisito de **dados on-prem** (compliance/regulatório) sem alternativa gerenciada.
- Necessidade de **controle fino** (índices custom, quantização específica, hardware GPU).

**Não vale (prefira gerenciado) quando:**
- Time pequeno, foco no produto, não em infra.
- Volume pequeno/médio (o piso do self-hosted em pessoas supera o serviço gerenciado).
- Você quer **time-to-market** rápido.
- **Regra prática:** só assuma operação de banco vetorial quando a economia mensal projetada superar, com folga, o custo totalmente carregado de 0,25–1 engenheiro dedicado.

---

## 9. Segurança

### 9.1 Criptografia
- **Em repouso:** habilite encryption at rest (KMS na AWS: RDS/Aurora/OpenSearch/etc.; disk encryption no self-hosted). Vetores e metadados podem conter informação sensível derivada.
- **Em trânsito:** TLS obrigatório entre orquestrador ↔ banco ↔ Bedrock. Em VPC, ainda assim use TLS.
- **Chaves:** gerencie com **KMS** (AWS) ou HSM/cofre on-prem; rotacione. Segredos de conexão no **Secrets Manager**, nunca em código/variáveis expostas.

### 9.2 Rede / isolamento
- Coloque o banco em **subnets privadas**; sem IP público.
- Lambda/orquestrador na **mesma VPC**; use **VPC endpoints (PrivateLink)** para Bedrock e S3 (tráfego não sai para a internet).
- Security groups mínimos (só a porta do banco, só da origem do orquestrador).
- SaaS externo (Pinecone/OpenAI): avalie **egress** e exija **PrivateLink** quando disponível; caso contrário, os dados trafegam pela internet — pesa em compliance.

### 9.3 Isolamento multi-tenant
- **Nunca** confie só no filtro de aplicação. Reforce o `tenant_id` no nível do retriever e valide sempre (um filtro esquecido = **vazamento entre clientes**).
- Isolamento forte → namespace/coleção/índice por tenant. Escala com muitos tenants → filtro + testes automatizados que garantem que consultas sem `tenant_id` falham.
- Registre auditoria de acesso por tenant.

### 9.4 PII em embeddings
- **Embeddings não são anônimos.** É possível, em certos casos, **reconstruir/inferir** conteúdo sensível a partir de vetores (inversion attacks). Trate embeddings de dados pessoais como **dados pessoais** (LGPD/GDPR).
- **Minimize/mascare PII antes de vetorizar** (redija CPF, nomes, e-mails quando não forem necessários à busca).
- **Direito ao esquecimento:** garanta delete por `source`/`tenant_id`/`subject` — inclusive nos backups e réplicas.
- **Controle de acesso a nível de documento** (ex.: ACLs do Kendra, filtros por permissão) para não expor conteúdo restrito no RAG.
- Evite logar o texto dos chunks/queries com PII em CloudWatch sem mascaramento.

---

## 10. Árvore de decisão e boas práticas

### 10.1 Árvore de decisão

```
                    ┌─ Dados PODEM sair do ambiente? ─┐
                    NÃO                                SIM
                     │                                  │
        ┌─ Edge/pequeno? ─┐               ┌─ Quer operar infra (equipe SRE)? ─┐
        SIM              NÃO              NÃO                                 SIM
         │                │                │                                   │
   sqlite-vec /     pgvector /       ┌─ 100% AWS? ─┐                    ┌─ Volume? ─┐
   LanceDB local    Milvus on-prem   SIM          NÃO                   <50M       >100M
                                      │             │                     │           │
                          ┌─ Precisa híbrido? ─┐  Pinecone/         Qdrant       Milvus/Qdrant
                          SIM               NÃO  Weaviate/Qdrant    self-hosted   + quantização
                           │                 │   Cloud             (EC2/EKS)     ou LanceDB(S3)
              ┌─ RAG sem código? ─┐    ┌─ Relacional junto? ─┐
              SIM             NÃO      SIM                NÃO
               │               │        │                  │
        Bedrock KB        OpenSearch  Aurora/RDS        OpenSearch
        (store=OpenS.)    Serverless  pgvector          Serverless
```

### 10.2 Boas práticas
- **Separe ingestão (Fargate/Batch) da consulta (Lambda).** Repita como mantra.
- **Fixe modelo + dimensão + métrica** e versione; reindexe em blue/green ao trocar.
- **Comece com HNSW + cosseno**; só vá para IVFPQ/quantização quando a memória doer.
- **Meça recall@k** contra ground-truth antes de otimizar latência/custo.
- **Use busca híbrida + rerank** quando a qualidade importar (termos exatos + semântica).
- **Metadados ricos** para filtro, citação e expurgo (LGPD).
- **Deduplique** para não poluir o top-k nem inflar custo.
- **Quantize** em escala grande (economia de 4×–32× de RAM/armazenamento).
- **Comece gerenciado**; migre para self-hosted só quando a conta e a maturidade justificarem.
- **VPC privada + KMS + Secrets Manager + filtro de tenant à prova de falha.**
- **Monitore:** latência p95, recall amostral, custo/1000 queries, uso de RAM/OCU/ACU.

---

## 11. Glossário

| Termo | Significado |
|---|---|
| **Embedding** | Vetor numérico que representa o significado de um dado (texto/imagem) |
| **ANN** | Approximate Nearest Neighbor — busca de vizinhos próximos aproximada (rápida) |
| **HNSW** | Índice em grafo hierárquico; recall alto, baixa latência, muita RAM |
| **IVF** | Índice por clusters (inverted file); visita só os clusters próximos |
| **IVFPQ** | IVF + Product Quantization; comprime vetores, economiza memória |
| **PQ / quantização** | Comprimir vetores em códigos curtos (perde um pouco de recall) |
| **ScaNN** | Índice do Google, ótima relação recall/velocidade em escala |
| **Recall@k** | Fração dos k verdadeiros vizinhos que o índice ANN de fato retornou |
| **Cosseno / dot / L2** | Métricas de similaridade/distância entre vetores |
| **Chunking** | Dividir documentos em pedaços coerentes para vetorizar |
| **Overlap** | Sobreposição de tokens entre chunks vizinhos |
| **BM25** | Ranking lexical clássico (palavra-chave); base da busca esparsa |
| **Híbrida** | Combinação de busca vetorial (densa) + lexical (BM25/esparsa) |
| **RRF** | Reciprocal Rank Fusion — funde rankings de buscas diferentes |
| **Rerank** | Reordenar candidatos com cross-encoder para precisão fina |
| **Cross-encoder** | Modelo que lê query+candidato juntos (mais preciso, mais caro) |
| **Upsert** | Inserir ou atualizar um vetor por ID |
| **Multi-tenant** | Isolar dados de clientes diferentes no mesmo sistema |
| **OCU** | OpenSearch Compute Unit — unidade de cobrança do OpenSearch Serverless |
| **ACU** | Aurora Capacity Unit — unidade de cobrança do Aurora Serverless v2 |
| **pgvector** | Extensão de vetores do PostgreSQL |
| **RAG** | Retrieval-Augmented Generation — LLM respondendo com contexto recuperado |
| **Knowledge Base (Bedrock)** | RAG gerenciado da AWS sobre um vector store |
| **Lock-in** | Grau de dependência/dificuldade de sair de um fornecedor |
| **MTEB** | Benchmark para comparar qualidade de modelos de embedding |

---

## 12. Resumo em uma frase

> **Comece com um store gerenciado 100% AWS (OpenSearch Serverless para híbrido, ou Aurora/RDS pgvector para simplicidade relacional) alimentado por Bedrock Titan Embeddings, separando a ingestão em Fargate da consulta em Lambda; migre para self-hosted (Qdrant/Milvus + quantização) apenas quando o volume e a maturidade da equipe justificarem a economia — e trate embeddings de dados pessoais como dados pessoais.**
