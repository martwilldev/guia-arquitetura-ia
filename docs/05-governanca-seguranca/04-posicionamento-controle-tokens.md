# Onde Fica o Controle de Tokens no Fluxo de IA — Posicionamento Arquitetural

> As outras docs deste módulo respondem *com quais ferramentas* controlar tokens
> ([`03-ferramentas-precificacao-controle-tokens.md`](03-ferramentas-precificacao-controle-tokens.md))
> e *como implementar na AWS* ([`01-precificacao-tokens-credenciais-aws.md`](01-precificacao-tokens-credenciais-aws.md)).
>
> **Este documento responde uma pergunta diferente e anterior:** dentro de um fluxo
> completo de IA, **em que ponto** essa camada de controle deve viver? No orquestrador?
> Numa camada antes dele? Num gateway dedicado? Distribuída? Aqui estão as opções,
> as explicações, a comparação lado a lado e a **análise de custo** de cada escolha.

---

## Índice

1. [O fluxo completo e os pontos possíveis](#1-o-fluxo-completo-e-os-pontos-possíveis)
2. [O que "controle de tokens" precisa fazer em cada ponto](#2-o-que-controle-de-tokens-precisa-fazer-em-cada-ponto)
3. [As 5 opções de posicionamento](#3-as-5-opções-de-posicionamento)
4. [Comparação lado a lado](#4-comparação-lado-a-lado)
5. [Análise de custo por opção](#5-análise-de-custo-por-opção)
6. [Recomendação por estágio/cenário](#6-recomendação-por-estágiocenário)
7. [Padrão recomendado (defesa em profundidade)](#7-padrão-recomendado-defesa-em-profundidade)
8. [Armadilhas de posicionamento](#8-armadilhas-de-posicionamento)

---

## 1. O fluxo completo e os pontos possíveis

Um fluxo de IA em produção tem vários "pontos de passagem" onde o request pode ser
inspecionado, medido e barrado. O controle de tokens **pode** morar em qualquer um
deles — cada um com um trade-off diferente.

```
      Usuário
        │
        ▼
 ┌───────────────┐   ◄── PONTO 1: EDGE / API GATEWAY
 │  API Gateway   │       (auth, rate limit por request, quota grossa)
 └───────┬───────┘
        │
        ▼
 ┌───────────────┐   ◄── PONTO 2: DENTRO DO ORQUESTRADOR
 │ Orquestrador   │       (regra de negócio; pode medir/limitar embutido)
 │ (seu código)   │
 └───────┬───────┘
        │
        ▼
 ┌───────────────┐   ◄── PONTO 3: GATEWAY LLM, NO CAMINHO DO MODELO
 │  Gateway LLM   │       (entre orquestrador e provedor; mede token,
 │  (LiteLLM…)    │        orçamento, roteia, cache) — pode ser lib ou serviço
 └───────┬───────┘
        │
        ▼
 ┌───────────────┐   ◄── PONTO 4: PROVEDOR
 │ Bedrock/OpenAI │       (fonte da verdade: retorna `usage` real de tokens)
 └───────────────┘
```

Observação central que guia tudo:

> O **API Gateway conta REQUESTS** (não sabe quantos tokens um request gasta).
> O **número de tokens só é conhecido de verdade DEPOIS** que o provedor responde
> (campo `usage`). Portanto, o controle *fino* de token/custo tem que acontecer
> onde você **lê a resposta do provedor** — ou seja, no orquestrador ou num gateway
> de LLM que fica no caminho da resposta. O edge só faz o controle *grosso*.

---

## 2. O que "controle de tokens" precisa fazer em cada ponto

Antes de escolher o lugar, lembre das 4 funções (detalhadas na doc de ferramentas):

| Função | Precisa de quê | Onde consegue fazer bem |
|---|---|---|
| **Medir (metering)** | Ler o `usage` do provedor | Só depois da resposta → orquestrador ou gateway LLM |
| **Limitar (limiting)** | Saber consumo acumulado antes de chamar | Orquestrador/gateway (token) + edge (request) |
| **Precificar (pricing)** | Aplicar markup sobre o custo | Orquestrador/gateway |
| **Cobrar (billing)** | Enviar uso para um faturador | Fora do caminho quente (assíncrono) |

> Conclusão parcial: **medir e precificar** exigem estar no caminho da resposta do
> provedor. **Limitar** pode ser feito em dois níveis (request no edge, token no
> orquestrador). **Cobrar** é assíncrono e não precisa estar no fluxo.

---

## 3. As 5 opções de posicionamento

### Opção 1 — No API Gateway / Edge (controle grosso)

**O que é:** usar o próprio API Gateway (ex.: Amazon API Gateway Usage Plans) para
limitar por *número de requests* e autenticar.

```
Usuário ─► API Gateway (rate limit + quota por request) ─► Orquestrador ─► Provedor
```

- ✅ Barato, nativo, primeira linha contra abuso bruto (DDoS, flood).
- ✅ Não exige código extra — é configuração.
- ❌ **Não conhece tokens** — um request pode gastar 100 ou 100.000 tokens.
- ❌ Não protege custo: o usuário estoura o orçamento com poucos requests grandes.

**Papel real:** defesa de borda. **Nunca** é suficiente sozinho para controle de custo.

---

### Opção 2 — Gateway de LLM no caminho do modelo (entre orquestrador e provedor)

**O que é:** um componente dedicado (ex.: LiteLLM, Portkey, Kong AI Gateway) que
fica **no caminho da chamada ao LLM** — ou seja, **entre o orquestrador e o provedor**,
não antes do orquestrador. Toda chamada ao modelo passa por ele, que lê o `usage`,
mede, limita, roteia e cacheia.

> ⚠️ **Onde ele NÃO fica:** *antes* do orquestrador. Nesse ponto o gateway ainda não
> viu nenhuma chamada ao LLM, logo **não há token para medir** — o controle de token
> só existe onde a resposta do provedor é lida. Antes do orquestrador quem atua é o
> **API Gateway** (Opção 1), para auth e rate limit por request.

O gateway de LLM pode existir em **duas formas**:

**2a) Como biblioteca DENTRO do orquestrador** (ex.: SDK do LiteLLM):
```
Usuário ─► API GW ─► Orquestrador { usa lib de gateway } ─► Provedor
                                    mede/roteia/cacheia
                                    (mesmo processo, sem salto de rede)
```
- ✅ Sem serviço extra e sem latência de rede adicional.
- ✅ Ganha roteamento/cache/multi-provedor da lib sem operar um container.
- ❌ A lógica ainda vive dentro do orquestrador (acopla; duplica se houver vários apps).
- Na prática, essa forma é bem próxima da Opção 3 — a diferença é usar uma **lib pronta**
  em vez de escrever o controle à mão.

**2b) Como serviço separado DEPOIS do orquestrador** (proxy dedicado):
```
Usuário ─► API GW ─► Orquestrador ─► [Gateway LLM (serviço)] ─► Provedor
                                     mede token, orçamento,
                                     roteamento, cache
```
- ✅ Controle centralizado: **um único ponto** mede, limita e roteia todas as chamadas.
- ✅ Reutilizável por **vários** orquestradores/apps (não duplica lógica).
- ✅ Ganha cache, roteamento e multi-provedor "de graça".
- ✅ Troca de provedor sem mexer no orquestrador.
- ❌ Mais um serviço para operar (container em ECS/EC2) → custo e +1 salto de latência.

> 💡 **Gateway de LLM ≠ ter vários orquestradores.** São eixos independentes: o gateway
> responde *"por onde passam as chamadas ao modelo"*; o número de orquestradores responde
> *"quantos cérebros de coordenação você tem"*. Você pode ter **1 orquestrador + 1 gateway**
> (ex.: só para ganhar cache/roteamento/troca de provedor). O que muda é o *retorno*: com
> **vários** orquestradores, um gateway **compartilhado** passa a valer muito mais a pena,
> porque centraliza o controle em vez de duplicá-lo em cada um (é a Opção 5).

**Papel real:** o "lugar canônico" do controle de token via ferramenta pronta. Na forma
**2a** serve a um app com pouco atrito; na forma **2b** brilha quando há escala ou mais de
um app consumindo LLM.

---

### Opção 3 — DENTRO do orquestrador (embutido no código)

**O que é:** a própria lógica do orquestrador (seu Lambda/serviço) mede tokens,
consulta orçamento no DynamoDB, escolhe modelo e debita — sem um gateway separado.

```
Usuário ─► API GW ─► Orquestrador (mede + limita + roteia embutido) ─► Provedor
```

- ✅ Zero salto de rede extra, menor latência.
- ✅ Sem serviço adicional para operar → mais barato no início.
- ✅ Controle total do comportamento (regra de negócio custom por tenant).
- ❌ Lógica de controle **acoplada** ao orquestrador → se tiver vários apps, duplica.
- ❌ Você reimplementa o que um gateway já entrega (cache, dashboards, multi-provedor).

**Papel real:** ideal para MVP e para quem tem **um único** orquestrador. É a
abordagem do código de exemplo nas docs de AWS deste módulo.

---

### Opção 4 — Distribuído (edge + orquestrador) — defesa em profundidade

**O que é:** combinar o controle *grosso* no edge (Opção 1) com o controle *fino*
no orquestrador (Opção 3). Cada camada faz o que sabe melhor.

```
Usuário ─► API GW (rate limit/quota por request)     ◄── barra abuso grosso
             │
             ▼
        Orquestrador (mede token, orçamento, roteia)  ◄── controla custo real
             │
             ▼
          Provedor
```

- ✅ Robusto: abuso bruto morre no edge; custo por token controlado no orquestrador.
- ✅ Sem serviço dedicado extra (usa o que já existe).
- ✅ Melhor relação simplicidade/robustez para a maioria dos casos.
- ❌ Ainda duplica lógica se houver múltiplos orquestradores.

**Papel real:** o **padrão recomendado** para produção sem multi-app (ver seção 7).

---

### Opção 5 — Distribuído com gateway dedicado (edge + gateway LLM) — escala/multi-app

**O que é:** edge para abuso grosso + gateway de LLM dedicado para o controle fino,
compartilhado por todos os orquestradores/apps da organização.

```
Usuário ─► API GW ─► Orquestrador A ┐
Usuário ─► API GW ─► Orquestrador B ┼─► [Gateway LLM único] ─► Provedores
Usuário ─► API GW ─► Orquestrador C ┘   (mede, limita, roteia,
                                          cache, billing, dashboards)
```

- ✅ **Um único ponto de verdade** para todos os apps (governança consistente).
- ✅ Cache e roteamento compartilhados = economia agregada maior.
- ✅ Troca de provedor e políticas em um lugar só.
- ❌ Mais caro e complexo: operar o gateway em alta disponibilidade.
- ❌ Ponto único de falha se mal projetado (precisa de HA).

**Papel real:** organizações com **vários** produtos/times consumindo LLM.

---

## 4. Comparação lado a lado

| Critério | 1. Edge | 2. Gateway LLM | 3. No orquestrador | 4. Edge+Orq. | 5. Edge+Gateway |
|---|:---:|:---:|:---:|:---:|:---:|
| Controla **tokens** (não só request) | ❌ | ✅ | ✅ | ✅ | ✅ |
| Controla **abuso bruto** (flood) | ✅ | ➖ | ➖ | ✅ | ✅ |
| Latência adicional | Nenhuma | +1 salto (serviço) / 0 (lib) | Nenhuma | Nenhuma | +1 salto |
| Reutilizável por vários apps | ✅ | ✅ (2b) / ❌ (2a) | ❌ | ❌ | ✅ |
| Cache / roteamento prontos | ❌ | ✅ | Você faz | Você faz | ✅ |
| Esforço de implementação | Baixo | Médio | Médio | Médio | Alto |
| Serviço extra p/ operar | Não | **Sim (2b)** / Não (2a) | Não | Não | **Sim** |
| Custo operacional fixo | Baixo | Médio (2b) / Baixo (2a) | Baixo | Baixo | Médio-alto |
| Melhor para | Anti-abuso | Ferramenta pronta / escala | MVP / 1 app | **Produção comum** | **Multi-app** |

Legenda: ✅ faz bem · ➖ parcial · ❌ não faz

---

## 5. Análise de custo por opção

> ⚠️ Valores **ilustrativos** (ordem de grandeza) para raciocínio comparativo.
> Confirme sempre na [Calculadora AWS](https://calculator.aws). Premissa comum:
> a conta é dominada pelo **token do LLM** (60–90% do total), como detalhado em
> [`01-precificacao-tokens-credenciais-aws.md`](01-precificacao-tokens-credenciais-aws.md).

### 5.1 Componentes de custo de cada posicionamento

| Opção | Custo de infra adicional | Custo de engenharia | Impacto no custo de token |
|---|---|---|---|
| **1. Edge** | ~$0 (API GW já cobra por request) | Baixo (config) | **Nenhum** — não reduz token |
| **2. Gateway LLM** | 2a lib: ~$0 · 2b serviço 24/7 (ECS/EC2): ~$30–150/mês* | Médio (subir/operar) | **Reduz** via cache+roteamento |
| **3. No orquestrador** | ~$0 (usa o Lambda que já existe) | Médio (código) | **Reduz** se implementar cache/rota |
| **4. Edge+Orq.** | ~$0 além do que já existe | Médio | **Reduz** (roteamento no código) |
| **5. Edge+Gateway** | Gateway em HA (2+ containers): ~$100–400/mês* | Alto | **Reduz mais** (cache/rota agregados) |

\* Ordem de grandeza para um container pequeno/médio rodando contínuo. Um LiteLLM
Proxy em Fargate pequeno fica na casa de dezenas de dólares/mês; em HA, mais.

### 5.2 O ponto que importa: o custo da ferramenta é ruído perto do token

```
Exemplo ilustrativo — 1M requests/mês, ~US$ 30.000 em tokens (modelo médio)

  Custo de tokens .............................. US$ 30.000  ← 98–99% da conta
  Custo do gateway/orquestrador (infra) ........ US$   50–400
  ─────────────────────────────────────────────────────────
  A escolha do POSICIONAMENTO muda ~US$ 50–400/mês de infra,
  MAS pode mudar MILHARES em token, dependendo de:
     • ter cache (respostas repetidas = token zero)
     • ter roteamento (mandar simples p/ modelo barato)
```

**Conclusão de custo:** não escolha o posicionamento para economizar os ~$50–400/mês
de infra. Escolha pelo que **habilita a redução do token** (cache + roteamento) com o
menor atrito operacional. Um gateway que corta 30% do token em 1M req paga a si mesmo
centenas de vezes.

### 5.3 Quando o gateway dedicado (Opção 2/5) se paga

```
Economia de token com cache+roteamento centralizados  >  custo de operar o gateway

Regra prática:
  • 1 app, volume baixo/médio  → NÃO justifica gateway dedicado (use Opção 3 ou 4).
  • vários apps OU volume alto → o gateway se paga fácil (cache/rota agregados
    cortam token muito acima dos ~$100–400/mês de operá-lo).
```

### 5.4 Custo escondido de cada opção

| Opção | Custo escondido |
|---|---|
| 1. Edge | **Falsa sensação de controle** — não protege custo de token; prejuízo com requests grandes |
| 2. Gateway LLM | Forma 2b: operação/HA do serviço e +1 salto de latência (impacta UX). Forma 2a (lib): acopla ao orquestrador como a Opção 3 |
| 3. No orquestrador | Duplicação de lógica se surgirem novos apps; reimplementar cache/dashboards |
| 4. Edge+Orq. | Idem 3 quando escala para multi-app |
| 5. Edge+Gateway | Complexidade de HA; time para operar o gateway como serviço crítico |

---

## 6. Recomendação por estágio/cenário

| Cenário | Posicionamento recomendado | Por quê |
|---|---|---|
| **MVP / 1 app / validar** | **Opção 3** (dentro do orquestrador) | Menor custo e atrito; usa o Lambda que já existe; mede pelo `usage` |
| **Produção, 1 app** | **Opção 4** (edge + orquestrador) | Edge barra abuso; orquestrador controla custo de token — robusto e barato |
| **Vários apps/times** | **Opção 5** (edge + gateway LLM) | Governança e cache/rota centralizados; um ponto de verdade |
| **Só preciso barrar flood** | **Opção 1** (edge) — como complemento | Resolve abuso, mas some com controle de token em outra camada |
| **Quero trocar de provedor fácil** | **Opção 2/5** (gateway LLM) | Abstrai o provedor num ponto único |
| **Dados sensíveis / self-host** | **Opção 3 ou 5** dentro da sua rede | Controle no seu ambiente; gateway self-host (LiteLLM) se multi-app |

> Resumo: **comece na Opção 3**, evolua para **Opção 4** ao ir para produção, e só
> vá para **Opção 5** quando tiver mais de um app/time consumindo LLM.

---

## 7. Padrão recomendado (defesa em profundidade)

Para a maioria dos casos de produção, combine as camadas — cada uma faz o que sabe:

```
┌────────────────────────────────────────────────────────────────┐
│ CAMADA 1 — EDGE (API Gateway)                                    │
│  • auth (Cognito/JWT)                                            │
│  • rate limit por REQUEST (anti-flood)                           │
│  • quota grossa por plano (ex.: Free = 1.000 req/mês)            │
│  → barato, barra abuso bruto                                     │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────────┐
│ CAMADA 2 — CONTROLE DE TOKEN (orquestrador OU gateway LLM)       │
│  • lê `usage` do provedor (fonte da verdade)                     │
│  • orçamento por tenant (DynamoDB, incremento atômico)           │
│  • max_tokens de saída (teto por request)                        │
│  • roteamento por custo (simples → modelo barato)                │
│  • cache de respostas repetidas                                  │
│  • degradação graciosa (rebaixa modelo em vez de bloquear)       │
│  → controla o custo REAL (onde o dinheiro está)                  │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────────┐
│ CAMADA 3 — PROVEDOR (Bedrock/OpenAI)                             │
│  • retorna `usage` (input/output tokens) = fonte da verdade      │
└────────────────────────────┬───────────────────────────────────┘
                             ▼
┌────────────────────────────────────────────────────────────────┐
│ FORA DO CAMINHO QUENTE (assíncrono)                              │
│  • billing (Stripe/Lago) recebe o uso e fatura                   │
│  • observabilidade (Langfuse/CloudWatch) por tenant/feature      │
│  • Budgets/Anomaly Detection alertam picos                       │
└────────────────────────────────────────────────────────────────┘
```

Regras do padrão:
- **Edge limita request; camada de token limita custo.** São controles diferentes — use os dois.
- **Meça sempre pelo `usage` real** do provedor, nunca por estimativa.
- **Billing e observabilidade ficam fora do caminho quente** (assíncronos) para não adicionar latência.
- **Escolha entre orquestrador (Opção 4) e gateway (Opção 5) pelo nº de apps**, não pelo custo de infra.

---

## 8. Armadilhas de posicionamento

| Armadilha | Consequência | Correção |
|---|---|---|
| Controlar só no **edge** | Usuário estoura custo com poucos requests grandes (edge não vê token) | Adicione controle de token no orquestrador/gateway |
| Colocar **billing no caminho quente** | Latência alta, acoplamento, falha de billing derruba a inferência | Torne o billing assíncrono (fila/evento) |
| **Gateway dedicado cedo demais** (1 app, volume baixo) | Paga e opera um serviço sem retorno | Comece embutido no orquestrador (Opção 3) |
| **Duplicar lógica** em vários orquestradores | Regras divergem, difícil manter | Centralize num gateway LLM (Opção 5) |
| Medir por **estimativa** em vez do `usage` | Cobrança e orçamento furados | Leia o `usage` da resposta do provedor |
| Gateway **sem HA** como ponto único | Uma falha derruba toda a IA | Rode o gateway em alta disponibilidade |
| Esquecer o **cache/roteamento** | Perde a maior alavanca de economia de token | Ative na camada de token (Opção 2/3/4/5) |

---

## Resumo em uma frase

> **O controle de tokens deve viver onde a resposta do provedor é lida — no
> orquestrador (para 1 app) ou num gateway de LLM dedicado (para vários apps) —
> com o API Gateway na borda apenas para abuso por request; a escolha do
> posicionamento deve ser guiada pelo número de aplicações e pela facilidade de
> habilitar cache e roteamento (que reduzem o token, o custo que realmente importa),
> não pelos poucos dólares de infra da camada em si.**

---

### Documentos relacionados

- [`03-ferramentas-precificacao-controle-tokens.md`](03-ferramentas-precificacao-controle-tokens.md) — *com quais ferramentas* fazer o controle (gateways, billing, observabilidade).
- [`01-precificacao-tokens-credenciais-aws.md`](01-precificacao-tokens-credenciais-aws.md) — *como implementar na AWS* (código de orçamento por tenant, IAM, arquitetura).
- [`../02-orquestracao/06-roteamento-de-llms.md`](../02-orquestracao/06-roteamento-de-llms.md) — roteamento por custo, a maior alavanca de economia de token.
- [`../02-orquestracao/07-padroes-execucao/05-multiplos-orquestradores.md`](../02-orquestracao/07-padroes-execucao/05-multiplos-orquestradores.md) — quando faz sentido ter múltiplos orquestradores (impacta a Opção 5).
