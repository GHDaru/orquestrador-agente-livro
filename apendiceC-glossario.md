---
title: "Apêndice C — Glossário da linguagem ubíqua"
layout: default
---

# Apêndice C — Glossário da linguagem ubíqua

Todos os termos técnicos introduzidos no livro,
em ordem alfabética. Para cada um: significado curto
e onde apareceu pela primeira vez.

---

## A

**Adapter** — implementação concreta de uma Port. Traduz entre
o domínio e tecnologia específica. (Cap 11)

**AGENT.md** — arquivo de contexto persistente na raiz do projeto,
lido pelo agente em cada execução. (Cap 11)

**AgentScheduler** — registro de sub-agentes que sabe despachar
tarefas para o sub-agente correto. (Cap 13)

**AgentTerminateMode** — enum com motivos pelos quais um sub-agente
encerra: GOAL, MAX_TURNS, ERROR. (Cap 13)

**Anti-Corruption Layer** — camada que traduz entre o domínio
e sistemas externos. (Cap 14)

**API** (Application Programming Interface) — interface que permite
programas se comunicarem pela internet ou entre si. (Cap 1)

**Async generator** — função `async def` com `yield`, produz
valores em tempo real, podendo aguardar entre eles. (Cap 7)

**async/await** — palavras-chave do Python para programação
assíncrona: função que pode pausar sem bloquear o processo. (Cap 6)

**asyncio.run()** — ponto de entrada de programas async, cria
o event loop e executa a coroutine principal. (Cap 6)

**ASGI** (Asynchronous Server Gateway Interface) — interface para
servidores web async em Python. Uvicorn implementa. (Cap 9)

---

## B

**BaseTool** — interface comum para ferramentas internas e MCP. (Cap 14)

**Bounded Context** — fronteira do domínio onde termos têm
significado próprio. (Cap 13)

---

## C

**Client** (`genai.Client`) — objeto que gerencia a conexão com
a API do Gemini. (Cap 1)

**complete_task** — tool especial que sub-agentes chamam para
sinalizar conclusão da tarefa. (Cap 13)

**Content** (`types.Content`) — estrutura de uma mensagem na conversa,
com `role` e `parts`. (Cap 2)

**Coroutine** — função que pode pausar e retomar. Resultado
de chamar uma função `async def`. (Cap 6)

**CORS** (Cross-Origin Resource Sharing) — mecanismo de segurança
que controla quais origens podem acessar um servidor. (Cap 9)

---

## D

**Dataclass** (`@dataclass`) — decorator que gera `__init__`,
`__repr__`, `__eq__` automaticamente. (Cap 5)

**DDD** (Domain-Driven Design) — abordagem de design de software
focada em modelar o domínio do problema. (Cap 11)

**Decorator** — função que recebe outra função e retorna uma função,
modificando ou registrando comportamento. (Cap 8)

**delegate_to_agent** — tool que delega tarefa a sub-agente. (Cap 13)

**description** — campo da `FunctionDeclaration` que descreve
a ferramenta para o modelo. Campo mais importante. (Cap 3)

**DOM** (Document Object Model) — representação em memória dos
elementos de uma página HTML. (Introdução)

**Domain Service** — lógica de negócio que não pertence a uma
entidade específica. (Cap 11)

---

## E

**Entidade** — objeto com identidade única que persiste no tempo. (Cap 11)

**Enum** — conjunto fechado e nomeado de valores válidos. (Cap 5)

**Event** — unidade de comunicação do loop. Contém tipo e payload. (Cap 5)

**EventSource** — API nativa do browser para consumir SSE em
endpoints GET. (Cap 10)

**EventType** — enum com os tipos de Event: CONTENT, TOOL_REQUEST,
TOOL_RESULT, FINISHED, ERROR. (Cap 5)

**Event loop** — coordenador de coroutines no async Python. (Cap 6)

**Executor** — código que executa as ferramentas pedidas pelo modelo.
Você é o executor. (Cap 3)

---

## F

**FastAPI** — framework Python para servidores HTTP com suporte
async nativo. (Cap 9)

**fetch** — API JavaScript para requisições HTTP, suporta POST
e streaming. (Cap 10)

**FileSessionStore** — adapter que implementa `SessionPort`
salvando sessões em arquivos JSON. (Cap 11)

**FunctionCall** — pedido do modelo para executar uma ferramenta,
com `name` e `args`. (Cap 3)

**FunctionDeclaration** — contrato que descreve uma ferramenta
para o modelo: nome, descrição, parâmetros. (Cap 3)

**FunctionResponse** — estrutura para devolver o resultado de
uma ferramenta ao modelo. (Cap 3)

---

## G

**Gemini** — família de modelos de linguagem do Google. (Cap 1)

**Generator** — função com `yield` que produz valores aos poucos. (Cap 7)

---

## H

**Hexagonal Architecture** — arquitetura que separa domínio de
infraestrutura via portas e adapters. (Apêndice B)

**HookEvent** — enum com pontos de extensão no lifecycle do loop:
BEFORE_PROMPT, AFTER_RESPONSE, etc. (Cap 12)

**HookRegistry** — objeto que armazena e dispara handlers
para hooks. (Cap 12)

**HTTP** (HyperText Transfer Protocol) — protocolo de comunicação
da internet, base de toda API web. (Cap 1)

---

## I

**Isolamento** — sub-agentes têm seu próprio estado e ferramentas,
sem contaminar o orquestrador. (Cap 13)

---

## J

**JSON Schema** — padrão aberto para descrever estrutura de dados.
Usado por Gemini, Claude, GPT. (Cap 3)

**Jupyter Notebook** — formato de documento (`.ipynb`) que mistura
células de texto e código executável. (Cap 0)

---

## L

**Lifecycle events** — momentos definidos no ciclo de vida do loop
onde hooks podem se plugar. (Cap 12)

**LLM** (Large Language Model) — modelo de linguagem de grande
escala. Gemini, Claude e GPT são LLMs. (Cap 3)

**LocalAgentDefinition** — blueprint de um sub-agente: nome,
prompt, ferramentas. (Cap 13)

**LocalAgentExecutor** — executa um sub-agente isolado. (Cap 13)

**LoopDetectionService** — detecta quando o agente entra em loop
chamando a mesma tool repetidamente. (Cap 11)

---

## M

**MAX_TURNS** — limite máximo de turnos do loop, protege contra
loop infinito. (Cap 8)

**McpClientAdapter** — cliente que conecta a servidor MCP e
expõe suas tools localmente. (Cap 14)

**McpToolProxy** — proxy que faz uma tool MCP parecer uma tool
local. (Cap 14)

**MCP** (Model Context Protocol) — protocolo aberto para conectar
ferramentas externas a agentes. (Cap 14)

**MessageBus** — barramento pub/sub para comunicação assíncrona
entre handlers. (Cap 12)

---

## P

**Part** (`types.Part`) — unidade de conteúdo dentro de uma `Content`.
Pode ser texto, function_call ou function_response. (Cap 2)

**pnpm** — gerenciador de pacotes JS que usa links simbólicos
em vez de copiar arquivos. (Cap 0)

**Port** — interface abstrata que descreve **o que** o domínio
precisa, sem dizer **como**. (Cap 11)

**Planejador** — o modelo, que decide o que fazer. (Cap 3)

**Proxy pattern** — objeto que se parece com outro mas internamente
delega para algum outro lugar. (Cap 14)

**Pydantic** — biblioteca de validação de dados via type hints. (Cap 9)

---

## R

**ReAct** (Reason + Act) — padrão de loop agentico: razão, ação,
observação, repetição. (Introdução)

**ReadableStream** — API JS para ler resposta HTTP em chunks. (Cap 10)

**Role** — campo de `Content` que identifica quem produziu a
mensagem: user, model, tool. (Cap 2)

---

## S

**Schema** (`types.Schema`) — estrutura de dados em JSON Schema
para descrever parâmetros. (Cap 3)

**SDK** (Software Development Kit) — conjunto de ferramentas e
bibliotecas para desenvolvimento numa plataforma. (Cap 1)

**Session** — entidade que representa uma conversa com estado. (Cap 11)

**SessionPort** — interface para persistência de sessões. (Cap 11)

**Skill** — pacote de conhecimento especializado carregado on-demand. (Cap 14)

**SkillRegistry** — registry que descobre skills no disco. (Cap 14)

**SSE** (Server-Sent Events) — mecanismo HTTP onde o servidor
envia mensagens ao cliente ao longo do tempo. (Cap 9)

**Stateless** — sem estado entre requisições. Cada chamada
independente. (Cap 1)

**Streaming** — entrega contínua de dados, conforme produzidos. (Cap 7)

**StreamingResponse** — classe do FastAPI que envia async generator
como SSE. (Cap 9)

**system_instruction** — prompt permanente que define a personalidade
do agente, enviado a cada chamada. (Cap 2)

---

## T

**Token** — unidade de processamento dos LLMs, pedaço de texto
de 3-5 caracteres. (Cap 1)

**Tool** (ferramenta) — função Python que o modelo pode pedir
para executar. (Cap 3)

**@tool** — decorator que registra uma função como ferramenta. (Cap 8)

**ToolCall** — pedido do modelo: id, name, args. (Cap 5)

**Tool** (`types.Tool`) — agrupa uma ou mais `FunctionDeclaration`. (Cap 3)

**TOOLS_FN** — dicionário que mapeia nome de ferramenta para
função Python. (Cap 4)

**Type hint** — anotação de tipo em Python, verificada pelo editor. (Cap 5)

---

## U

**uv** — gerenciador moderno de projetos Python, substitui pip
e virtualenv. (Cap 0)

**uvicorn** — servidor ASGI, roda aplicações FastAPI. (Cap 9)

---

## V

**Value Object** — objeto definido apenas por seus valores,
sem identidade. (Cap 5, formalizado no 13)

---

## W

**while True** — estrutura do loop ReAct, sai quando o modelo
não pede mais ferramentas. (Cap 4)

---

## Y

**yield** — palavra-chave que produz um valor e pausa a função. (Cap 7)


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="apendiceB-ddd">← Apêndice B</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="apendiceD-codigo">Apêndice D →</a></div>
</div>
