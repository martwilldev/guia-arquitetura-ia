# OmniRoute — AI Gateway Gratuito (uso, encaixe e comparação)

> **OmniRoute** ([github.com/diegosouzapw/OmniRoute](https://github.com/diegosouzapw/OmniRoute))
> é um **AI Gateway / LLM Gateway** open-source (MIT) que expõe **um único endpoint
> compatível com OpenAI** (`http://localhost:20128/v1`) e roteia cada request para
> **um entre ~350+ provedores** (150+ com tier gratuito), com fallback automático,
> compressão de tokens e controle de custo/quota.
>
> Esta doc responde duas perguntas diretas: **(1) o OmniRoute é um orquestrador?**
> e **(2) ele entra na comparação dos outros documentos?** — e depois explica como usá-lo.

---

## Índice

1. [Resposta rápida: é orquestrador? entra na comparação?](#1-resposta-rápida)
2. [O que o OmniRoute é (e o que não é)](#2-o-que-o-omniroute-é-e-o-que-não-é)
3. [Onde ele encaixa no fluxo (a peça do módulo 5.4)](#3-onde-ele-encaixa-no-fluxo)
4. [Recursos principais](#4-recursos-principais)
5. [Como usar (instalação e uso)](#5-como-usar)
6. [OmniRoute vs LiteLLM/OpenRouter/Portkey (categoria de gateway)](#6-omniroute-vs-outros-gateways)
7. [OmniRoute encaixa em arquiteturas de produção multi-tenant?](#7-omniroute-encaixa-em-arquiteturas-de-produção-multi-tenant)
8. [Quando usar e quando NÃO usar](#8-quando-usar-e-quando-não-usar)
9. [Armadilhas e pontos de atenção](#9-armadilhas-e-pontos-de-atenção)

---

## 1. Resposta rápida

### ❓ O OmniRoute é um orquestrador?

**Não** — não no sentido que esta coletânea dá à palavra "orquestrador" (o "maestro"
descrito no [doc 2.1](../02-orquestracao/01-orquestradores-estudo-didatico.md) que coordena
raciocínio, tools, memória, agentes e decide *o que fazer* a cada passo).

O OmniRoute é um **AI Gateway** (ou *LLM Gateway* / proxy de roteamento). Ele **não
coordena um raciocínio de múltiplos passos** — ele recebe uma chamada já pronta e
decide **para qual provedor de modelo** mandar (por custo, quota, latência, saúde),
com fallback se um falhar. É a mesma categoria do **LiteLLM**, **OpenRouter** e
**Portkey** — que já aparecem nesta coletânea na
[Categoria A — Gateways de LLM do doc 5.3](03-ferramentas-precificacao-controle-tokens.md#5-categoria-a--gateways-de-llm-o-coração-da-solução).

> ⚠️ **Cuidado com o nome.** O OmniRoute tem alguns recursos que *tocam* o mundo de
> agentes (ele expõe um **servidor MCP** com 110 tools, um endpoint **A2A**, e combos
> `pipeline`/`fusion`). Mas isso é o gateway **oferecendo integração** para orquestradores/
> agentes o consumirem — não o gateway *sendo* o orquestrador. Quem decide a lógica de
> negócio continua sendo o seu orquestrador (Lambda, LangChain, Bedrock Agents…), que
> chama o OmniRoute como se fosse a "OpenAI".

### ❓ Ele entrou na comparação dos outros documentos?

**Não diretamente** (a coletânea foi escrita antes / com foco em AWS Bedrock), **mas
a categoria dele já está comparada.** O doc 5.3 compara **LiteLLM, Portkey, Helicone,
OpenRouter, Kong AI Gateway, Cloudflare AI Gateway** — todos gateways. O OmniRoute é
mais um item dessa mesma categoria (a seção [6](#6-omniroute-vs-outros-gateways) faz
a comparação lado a lado).

Curiosidade: o próprio OmniRoute **nasceu como fork do `9router`** e como port do
`CLIProxyAPI`, e **usa o dataset de preços do LiteLLM** para o cost-tracking — ou seja,
ele se posiciona explicitamente na mesma família do LiteLLM/OpenRouter.

---

## 2. O que o OmniRoute é (e o que não é)

| | OmniRoute (AI Gateway) | Orquestrador (doc 2.x) |
|---|---|---|
| **Papel** | Roteia a chamada para o provedor certo | Decide *o que fazer*: chama tools, faz loop, coordena agentes |
| **Entrada** | Um request `chat/completions` já montado | Uma tarefa/objetivo do usuário |
| **Decisão** | Qual provedor/modelo, fallback, compressão | Qual passo, qual tool, quando parar |
| **Estado** | Quota, saúde do provedor, cache | Histórico da conversa, memória, plano |
| **Analogia** | Central telefônica / balanceador | Maestro da orquestra |
| **Equivalentes** | LiteLLM, OpenRouter, Portkey, Kong AI GW | LangChain, CrewAI, Bedrock Agents, seu Lambda |

**Em uma frase:** o OmniRoute é a camada que fica **no caminho do modelo** — o
**Ponto 3** do diagrama do [doc 5.4](04-posicionamento-controle-tokens.md#1-o-fluxo-completo-e-os-pontos-possíveis).

---

## 3. Onde ele encaixa no fluxo

Reaproveitando o diagrama do [doc 5.4](04-posicionamento-controle-tokens.md):

```
      Usuário / App (Next.js, IDE, CLI)
        │
        ▼
 ┌───────────────┐   PONTO 1: EDGE / API GATEWAY
 │  API Gateway   │   (auth, rate limit por request)
 └───────┬───────┘
        │
        ▼
 ┌───────────────┐   PONTO 2: ORQUESTRADOR (seu Lambda / LangChain)
 │ Orquestrador   │   (decide passos, chama tools, monta o prompt)
 └───────┬───────┘
        │  fala "OpenAI-compatible"
        ▼
 ┌───────────────┐   PONTO 3: ◄── AQUI VIVE O OMNIROUTE
 │   OMNIROUTE    │   (mede token, escolhe provedor, fallback,
 │  (AI Gateway)  │    comprime, cacheia, aplica quota/budget)
 └───────┬───────┘
        │
        ▼
 ┌────────────────────────────┐   PONTO 4: PROVEDORES (350+)
 │ Bedrock/OpenAI/Claude/Gemini│   (retornam o `usage` real)
 │ Groq/DeepSeek/free tiers…   │
 └────────────────────────────┘
```

O OmniRoute é uma implementação concreta da **Opção 2** ("Gateway de LLM no caminho
do modelo") do doc 5.4 — na forma de **serviço autônomo** (não biblioteca).

---

## 4. Recursos principais

Segundo o repositório (v3.8.x):

- **~350+ provedores** por trás de um endpoint OpenAI-compatible; 150+ com tier gratuito
  (Groq, DeepSeek, Cerebras, OpenRouter `:free`, Pollinations sem chave, etc.).
- **Zero-config**: o modelo `auto` monta um "combo virtual" e escolhe o provedor ao vivo.
- **19 estratégias de roteamento** (`priority`, `cost-optimized`, `round-robin`,
  `least-used`, `headroom`, `fusion`, `pipeline`…) — combináveis por "combo".
- **Fallback resiliente** em 4 tiers (Assinatura → API Key → Barato → Gratuito) com
  **circuit breaker**, cooldown de chave e lockout por modelo.
- **Compressão de tokens** (pipeline de 12 engines: RTK + Caveman etc.) que reduz
  15–95% dos tokens de forma transparente — reduz custo direto.
- **Controle de custo/quota**: headers `X-OmniRoute-*` com custo/uso por request,
  budgets em USD por chave, telemetria de quota ao vivo.
- **Interfaces de integração**: REST `/v1/*`, **MCP** (stdio/HTTP/SSE, 110 tools),
  **A2A** (JSON-RPC 2.0), webhooks.
- **Local-first / privado**: roda na sua máquina/servidor, credenciais em AES-256-GCM,
  telemetria desligada por padrão, trilha de auditoria em SQLite local.
- **Onde roda**: npm global, Docker (AMD64/ARM64), Electron (desktop), Termux (Android),
  PWA, source. Stack Node.js + TypeScript + Next.js + SQLite.

> ⚠️ **RAM para coding agents**: o próprio README avisa que chamadas longas de
> `POST /v1/responses` (Claude Code, Codex) exigem heap grande (8–12 GB). Não é um
> processo "leve" quando serve agentes pesados.

---

## 5. Como usar

### 5.1 Instalar e subir (npm)

```bash
npm install -g omniroute
omniroute
# Dashboard: http://localhost:20128
# API:       http://localhost:20128/v1
```

### 5.2 Testar sem nenhuma chave (provedor free pré-configurado)

```bash
curl http://localhost:20128/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"auto","messages":[{"role":"user","content":"Olá!"}]}'
```

### 5.3 Docker (recomendado para servidor)

```bash
docker run -d --name omniroute --restart unless-stopped --stop-timeout 40 \
  -p 127.0.0.1:20128:20128 -v omniroute-data:/app/data \
  diegosouzapw/omniroute:latest
```

Para servir agentes pesados, subir o heap e a memória do container:

```bash
docker run -d --name omniroute \
  -e OMNIROUTE_MEMORY_MB=8192 --memory=10g \
  -p 127.0.0.1:20128:20128 -v omniroute-data:/app/data \
  diegosouzapw/omniroute:latest
```

### 5.4 Consumir a partir do seu orquestrador (como se fosse OpenAI)

Qualquer SDK OpenAI-compatible funciona — basta trocar `base_url`:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:20128/v1",
    api_key="SUA_CHAVE_DO_DASHBOARD",  # copiada do painel → Endpoints
)

resp = client.chat.completions.create(
    model="auto/cheap",   # deixa o gateway escolher o mais barato disponível
    messages=[{"role": "user", "content": "Resuma este texto..."}],
)
print(resp.choices[0].message.content)
```

Do ponto de vista do seu orquestrador (Lambda/LangChain), **o OmniRoute É a "OpenAI"** —
ele só troca o endereço. Toda a lógica de fallback/custo/compressão acontece sem
mudar o código do orquestrador.

### 5.5 Ligar um coding CLI direto

```bash
omniroute run claude   --model auto/coding    # Claude Code
omniroute run codex    --model auto/cheap      # Codex CLI
omniroute configure codex                      # escreve o config do próprio Codex
```

---

## 6. OmniRoute vs outros gateways

Mesma tabela/critérios da [Categoria A do doc 5.3](03-ferramentas-precificacao-controle-tokens.md#5-categoria-a--gateways-de-llm-o-coração-da-solução):

| Produto | Foco principal | Metering/custo | Quota/budget | Roteamento+fallback | Compressão de token | Open-source | Roda na AWS |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **OmniRoute** | Maximizar **free tiers** + resiliência multi-provedor, foco em dev/coding-agents | ✅ (headers, USD/chave) | ✅ (budget USD, quota) | ✅ (19 estratégias, 4 tiers) | ✅ (12 engines, 15–95%) | ✅ (MIT) | ✅ (Docker em ECS/EC2) |
| **LiteLLM** | Gateway unificado p/ 100+ modelos, metering por virtual key | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **OpenRouter** | Gateway **SaaS** agregador; compra crédito e revende | ✅ (por chave) | ✅ | ✅ | ❌ | ❌ | ✅ (via API) |
| **Portkey** | AI gateway com guardrails/cache/observabilidade | ✅ | ✅ | ✅ | Parcial (cache) | Parcial | ✅ |
| **Kong AI Gateway** | Plugins de IA sobre o Kong (edge) | ✅ | ✅ | ✅ | ❌ | ✅ (core) | ✅ |

### Onde o OmniRoute se diferencia

- **Foco em custo zero**: seu grande diferencial é agregar e alternar entre **tiers
  gratuitos** de dezenas de provedores. Isso é ótimo para uso **pessoal / dev / protótipo**.
- **Compressão de token embutida** (RTK/Caveman) — a maioria dos outros gateways não faz.
- **Voltado a coding agents/CLIs** (Claude Code, Codex, Cursor) mais do que a
  backend de produção multi-tenant.

### Onde o LiteLLM/Portkey tendem a ganhar para produção corporativa

- **Multi-tenancy e billing por cliente** mais maduros (virtual keys + budgets do
  LiteLLM, ver [doc 5.3](03-ferramentas-precificacao-controle-tokens.md#destaque-litellm-opção-open-source-popular)).
- **Integração AWS Bedrock** como cidadão de primeira classe (foco desta coletânea).
- Modelo operacional mais previsível (não depende de "caçar" free tiers que mudam).

> ⚠️ **Sobre depender de free tiers em produção:** o próprio README avisa que os
> números de tokens gratuitos **mudam nos dois sentidos** (provedores abrem e fecham
> tiers a qualquer momento). Isso é aceitável para uso pessoal, mas é **risco de
> disponibilidade** para um produto que promete SLA a clientes.

---

## 7. OmniRoute encaixa em arquiteturas de produção multi-tenant?

Em uma arquitetura de produção típica (como nos [Estudos de Caso do módulo 2.8](../02-orquestracao/08-estudos-de-caso/README.md)),
é comum combinar: 2 tiers de modelos (free/serverless GPU + pago por API), orquestrador Lambda,
RAG em pgvector, guardrails, controle de custo/tokens e auditoria.

O OmniRoute **poderia** ocupar o **Ponto 3 (gateway no caminho do modelo)** dessa
arquitetura — cobrindo o "modelo free vs pago" e o fallback entre eles com uma única
configuração de combo. Mas há ressalvas para um produto multi-tenant "de verdade":

| Necessidade da Arquitetura | OmniRoute cobre? | Observação |
|---|---|---|
| Alternar modelo free ↔ pago | ✅ | combos com tiers e `cost-optimized` |
| Fallback resiliente | ✅ | circuit breaker + 4 tiers |
| Controle de custo/token | ✅ (parcial) | budget por chave; billing por cliente é mais fraco que LiteLLM |
| Multi-tenancy forte / isolamento por cliente | ⚠️ | pensado mais p/ dev individual do que SaaS multi-tenant |
| Bedrock como 1ª classe | ⚠️ | possível, mas não é o foco; a coletânea prioriza Bedrock |
| SLA de disponibilidade | ⚠️ | depender de free tiers é risco; para produção use provedores pagos/estáveis |

**Conclusão:** ótimo para **MVP/protótipo** e para reduzir custo em
desenvolvimento. Para a fase de **produção multi-tenant** com SLA, o padrão da
coletânea (LiteLLM/Portkey + Bedrock, ver [doc 5.4](04-posicionamento-controle-tokens.md))
continua sendo a recomendação mais segura.

---

## 8. Quando usar e quando NÃO usar

**Use o OmniRoute quando:**
- Você quer **um endpoint único** para muitos provedores sem escrever adaptadores.
- Está em **dev/protótipo** e quer **rodar de graça** aproveitando free tiers.
- Usa **coding agents/CLIs** (Claude Code, Codex, Cursor) e quer fallback + economia de token.
- Quer **self-host local-first** com privacidade (nada sai para uma nuvem intermediária).

**Prefira LiteLLM/Portkey + Bedrock quando:**
- É um **produto multi-tenant** com billing por cliente e SLA.
- Precisa de **Bedrock como cidadão de primeira classe** e integração AWS profunda.
- Não pode depender da volatilidade de tiers gratuitos.

**Lembre-se:** OmniRoute **não substitui** seu orquestrador. Ele fica **abaixo** dele.
Você ainda precisa do orquestrador (módulo 2) para coordenar tools, RAG e passos.

---

## 9. Armadilhas e pontos de atenção

- **Não é orquestrador** — não coloque nele a lógica de negócio/agente.
- **Free tiers são voláteis** — bom para dev, arriscado para SLA de produção.
- **Consumo de RAM alto** ao servir coding agents (8–12 GB de heap recomendados).
- **Termos de provedores** — usar "web/cookie providers" ou contornar rate limits de
  serviços pagos pode violar os termos de uso deles; o próprio catálogo marca 13
  provedores como *avoid*. Avalie o risco jurídico antes de usar em produto comercial.
- **Superfície grande** — é um projeto muito amplo (compressão, MITM/TPROXY, proxy,
  stealth TLS). Em produção, habilite só o que precisa e revise segurança.

---

## Referências desta coletânea

- [5.3 Ferramentas de Precificação e Controle de Tokens](03-ferramentas-precificacao-controle-tokens.md) — categoria de gateways (LiteLLM, Portkey…).
- [5.4 Posicionamento do Controle de Tokens](04-posicionamento-controle-tokens.md) — onde o gateway vive no fluxo.
- [2.1 Orquestradores — Estudo Didático](../02-orquestracao/01-orquestradores-estudo-didatico.md) — o que é (de verdade) um orquestrador.
- [2.6 Roteamento de LLMs](../02-orquestracao/06-roteamento-de-llms.md) — o roteamento como conceito (o OmniRoute é uma implementação disso na camada de gateway).

> **Fonte primária:** [github.com/diegosouzapw/OmniRoute](https://github.com/diegosouzapw/OmniRoute) (README v3.8.x). Números de provedores/tokens/free-tiers mudam com frequência — confirme sempre no repositório.
