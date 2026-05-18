---
title: "Capítulo 13 — Sub-agentes: o orquestrador delega"
layout: default
---

# Capítulo 13 — Sub-agentes: o orquestrador delega

---

## A dor

Imagine esta tarefa:

> "Analise os 50 arquivos Python deste projeto, identifique
> os 5 com pior qualidade de código, e proponha melhorias específicas
> para cada um."

Um único loop tentando fazer isso:

- Vai chamar `list_dir`, `read_file`, `read_file`, `read_file`...
- O histórico cresce monstruosamente — cada arquivo ocupa milhares
  de tokens
- A cada turno, o modelo recebe **todos** os arquivos já lidos
- Os tokens consumidos crescem em ordem quadrática
- Provavelmente atinge o limite de contexto antes de terminar

E mesmo se terminasse, o modelo precisaria manter na cabeça
50 arquivos simultaneamente para comparar.

A solução é **delegar**: o orquestrador identifica os arquivos,
e para cada análise específica cria um **sub-agente** com seu
próprio loop, seu próprio histórico, suas próprias ferramentas.

O sub-agente recebe uma tarefa, trabalha isoladamente, e devolve
apenas o resultado final ao orquestrador. O histórico monstro
do sub-agente é descartado depois.

---

## Por que isto é DDD — Isolamento de Bounded Context

Em DDD, um **Bounded Context** (contexto delimitado) é uma fronteira
dentro da qual os termos têm significado próprio e o estado é isolado.
"Cliente" no Bounded Context de Vendas é diferente de "Cliente"
no de Suporte — embora se refiram à mesma pessoa real.

Sub-agentes implementam Bounded Contexts:

- Cada sub-agente tem seu próprio `ToolRegistry`
- Cada sub-agente tem seu próprio histórico
- O orquestrador só vê o resultado, não os detalhes internos

Ao decompor uma tarefa em sub-agentes, você está **descobrindo
limites de contexto** — separando responsabilidades que antes
estavam misturadas no agente principal.

---

## LocalAgentDefinition

Antes de criar um sub-agente, definimos um *blueprint*:
qual sistema usar, que ferramentas, qual prompt:

```python
@dataclass
class LocalAgentDefinition:
    name:          str
    system_prompt: str
    tool_names:    list[str]   # nomes das tools que ele pode usar
    max_turns:     int = 10
```

Isso é uma **Value Object** em DDD: um objeto sem identidade própria,
definido apenas pelos seus valores. Dois `LocalAgentDefinition`
com os mesmos campos são equivalentes.

---

## LocalAgentExecutor

O executor instancia um sub-agente a partir da definição.
Cria um `ToolRegistry` filtrado (só as tools listadas),
um histórico vazio, um loop dedicado.

```python
class LocalAgentExecutor:
    def __init__(self, registry: dict, definition: LocalAgentDefinition):
        self.registry   = {name: registry[name] for name in definition.tool_names}
        self.schemas    = [...]   # schemas só das tools listadas
        self.definition = definition

    async def run(self, task: str) -> str:
        # Loop ReAct completo, mas isolado
        # Retorna apenas o texto final, não o histórico
        ...
```

O isolamento é total: o sub-agente não vê o histórico do
orquestrador, nem suas ferramentas extras. Trabalha como
se fosse um agente independente.

---

## A tool complete_task

Como o orquestrador sabe quando o sub-agente terminou?
O modelo natural seria: o sub-agente responde com texto, e isso
é considerado conclusão. Mas isso é frágil — o modelo pode
divagar, ou retornar um relatório intermediário.

A solução: dar ao sub-agente uma ferramenta especial chamada
`complete_task`. Quando ele chama essa tool, está sinalizando
explicitamente "minha tarefa terminou, este é o resultado":

```python
@tool(
    name="complete_task",
    description="Chame esta ferramenta quando completar sua tarefa. Passe o resultado final.",
    parameters={
        "properties": {"result": {"description": "Resultado final da tarefa"}},
        "required": ["result"]
    }
)
def complete_task(result: str) -> str:
    # Esta função é especial: o loop trata como sinal de fim
    return result
```

O `LocalAgentExecutor` detecta a chamada a `complete_task`
e encerra o loop, retornando o `result` ao orquestrador.

---

## AgentTerminateMode

Um sub-agente pode terminar por três motivos:

```python
class AgentTerminateMode(str, Enum):
    GOAL      = "goal"        # chamou complete_task
    MAX_TURNS = "max_turns"   # atingiu limite de turnos
    ERROR     = "error"       # erro na execução
```

O orquestrador usa essa informação para decidir o que fazer:
se foi GOAL, usa o resultado. Se foi MAX_TURNS ou ERROR,
pode tentar de novo ou reportar.

---

## AgentScheduler

Para o orquestrador conseguir invocar sub-agentes, criamos uma
ferramenta especial: `delegate_to_agent`. Por trás, há um
**scheduler** (agendador) que sabe quais sub-agentes existem
e como executá-los:

```python
class AgentScheduler:
    def __init__(self):
        self.agents: dict[str, LocalAgentDefinition] = {}

    def register(self, definition: LocalAgentDefinition):
        self.agents[definition.name] = definition

    async def dispatch(self, agent_name: str, task: str) -> str:
        definition = self.agents[agent_name]
        executor   = LocalAgentExecutor(GLOBAL_TOOLS, definition)
        return await executor.run(task)
```

O orquestrador vê isso como qualquer outra tool — não sabe
que por trás está um loop ReAct inteiro.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta:

```powershell
New-Item -ItemType Directory -Force -Path cap13
```

No VS Code, crie `cap13.ipynb` dentro de `cap13`.

---

## As células do notebook

### Célula 1 — Definitions e enums

```python
import os
from dataclasses import dataclass, field
from enum import Enum
from typing import AsyncGenerator, Callable
from google import genai
from google.genai import types

client = genai.Client()


class EventType(str, Enum):
    CONTENT      = "content"
    TOOL_REQUEST = "tool_request"
    TOOL_RESULT  = "tool_result"
    FINISHED     = "finished"
    ERROR        = "error"
    SUB_AGENT    = "sub_agent"


class AgentTerminateMode(str, Enum):
    GOAL      = "goal"
    MAX_TURNS = "max_turns"
    ERROR     = "error"


@dataclass
class LocalAgentDefinition:
    name:          str
    system_prompt: str
    tool_names:    list[str]
    max_turns:     int = 10


@dataclass
class AgentResult:
    mode:   AgentTerminateMode
    output: str


@dataclass
class Event:
    type:    EventType
    payload: dict = field(default_factory=dict)


print("Tipos definidos.")
```

---

### Célula 2 — Registro global de ferramentas

```python
GLOBAL_TOOLS:   dict[str, Callable] = {}
GLOBAL_SCHEMAS: dict[str, types.FunctionDeclaration] = {}


def tool(name: str, description: str, parameters: dict):
    def decorator(fn):
        GLOBAL_TOOLS[name] = fn
        GLOBAL_SCHEMAS[name] = types.FunctionDeclaration(
            name=name, description=description,
            parameters=types.Schema(
                type=types.Type.OBJECT,
                properties={
                    k: types.Schema(type=types.Type.STRING, description=v.get("description", ""))
                    for k, v in parameters.get("properties", {}).items()
                },
                required=parameters.get("required", [])
            )
        )
        return fn
    return decorator


@tool(name="list_dir", description="Lista arquivos.",
      parameters={"properties": {"path": {"description": "Caminho"}}, "required": ["path"]})
def list_dir(path: str) -> str:
    try:
        return "\n".join(sorted(os.listdir(path)))
    except Exception as e:
        return f"Erro: {e}"


@tool(name="read_file", description="Lê arquivo.",
      parameters={"properties": {"path": {"description": "Caminho"}}, "required": ["path"]})
def read_file(path: str) -> str:
    try:
        with open(path, "r", encoding="utf-8") as f:
            return f.read()
    except Exception as e:
        return f"Erro: {e}"


@tool(
    name="complete_task",
    description=(
        "Chame esta ferramenta quando completar sua tarefa. "
        "Passe o resultado final no parâmetro 'result'."
    ),
    parameters={
        "properties": {"result": {"description": "Resultado final da tarefa"}},
        "required": ["result"]
    }
)
def complete_task(result: str) -> str:
    return result


print(f"Tools globais: {list(GLOBAL_TOOLS.keys())}")
```

---

### Célula 3 — LocalAgentExecutor

```python
class LocalAgentExecutor:
    def __init__(self, definition: LocalAgentDefinition):
        self.definition = definition

        self.tools = {
            name: GLOBAL_TOOLS[name]
            for name in definition.tool_names
            if name in GLOBAL_TOOLS
        }
        self.tools["complete_task"] = complete_task

        self.schemas = [
            GLOBAL_SCHEMAS[name]
            for name in self.tools.keys()
            if name in GLOBAL_SCHEMAS
        ]

    async def run(self, task: str) -> AgentResult:
        historico = [types.Content(role="user", parts=[types.Part(text=task)])]

        for turno in range(self.definition.max_turns):
            response = await client.aio.models.generate_content(
                model="gemini-2.5-flash",
                contents=historico,
                config=types.GenerateContentConfig(
                    system_instruction=self.definition.system_prompt,
                    tools=[types.Tool(function_declarations=self.schemas)],
                    temperature=0.7,
                )
            )

            if not response.candidates:
                return AgentResult(mode=AgentTerminateMode.ERROR, output="Sem resposta do modelo")

            parts = response.candidates[0].content.parts
            historico.append(response.candidates[0].content)

            tool_calls = [p.function_call for p in parts if p.function_call]

            if not tool_calls:
                texto = "".join(p.text for p in parts if p.text)
                return AgentResult(mode=AgentTerminateMode.GOAL, output=texto)

            tool_result_parts = []
            for fc in tool_calls:
                name = fc.name
                args = dict(fc.args) if fc.args else {}

                if name == "complete_task":
                    return AgentResult(
                        mode=AgentTerminateMode.GOAL,
                        output=args.get("result", "")
                    )

                fn        = self.tools.get(name)
                resultado = fn(**args) if fn else f"Tool '{name}' não disponível"

                tool_result_parts.append(
                    types.Part(function_response=types.FunctionResponse(
                        name=name, response={"result": resultado}
                    ))
                )

            historico.append(types.Content(role="tool", parts=tool_result_parts))

        return AgentResult(mode=AgentTerminateMode.MAX_TURNS, output="Limite de turnos atingido")
```

---

### Célula 4 — Testando um sub-agente isolado

```python
analista = LocalAgentDefinition(
    name="analista_arquivo",
    system_prompt=(
        "Você é um analista de código. Você recebe um caminho de arquivo, "
        "lê seu conteúdo, e descreve em uma frase o que ele faz. "
        "Quando terminar, chame complete_task com sua análise."
    ),
    tool_names=["read_file"],
    max_turns=5,
)

executor = LocalAgentExecutor(analista)
resultado = await executor.run("Analise o arquivo cap08\\loop.py")

print(f"Modo: {resultado.mode.value}")
print(f"Saída: {resultado.output}")
```

---

### Célula 5 — AgentScheduler

```python
class AgentScheduler:
    def __init__(self):
        self.agents: dict[str, LocalAgentDefinition] = {}

    def register(self, definition: LocalAgentDefinition):
        self.agents[definition.name] = definition

    async def dispatch(self, agent_name: str, task: str) -> AgentResult:
        if agent_name not in self.agents:
            return AgentResult(mode=AgentTerminateMode.ERROR,
                               output=f"Sub-agente '{agent_name}' não registrado")

        executor = LocalAgentExecutor(self.agents[agent_name])
        return await executor.run(task)


scheduler = AgentScheduler()
scheduler.register(analista)
print(f"Sub-agentes registrados: {list(scheduler.agents.keys())}")
```

---

### Célula 6 — Tool delegate_to_agent

```python
@tool(
    name="delegate_to_agent",
    description=(
        "Delega uma tarefa específica para um sub-agente especializado. "
        "Use quando a tarefa for grande e puder ser decomposta. "
        "Sub-agentes disponíveis: 'analista_arquivo' (analisa um arquivo)."
    ),
    parameters={
        "properties": {
            "agent_name": {"description": "Nome do sub-agente (ex: 'analista_arquivo')"},
            "task":       {"description": "Tarefa a ser executada pelo sub-agente"},
        },
        "required": ["agent_name", "task"]
    }
)
async def delegate_to_agent(agent_name: str, task: str) -> str:
    resultado = await scheduler.dispatch(agent_name, task)
    return f"[{resultado.mode.value}] {resultado.output}"


print(f"Tool delegate_to_agent registrada.")
print(f"Tools globais agora: {list(GLOBAL_TOOLS.keys())}")
```

---

### Célula 7 — O loop do orquestrador

O loop do orquestrador é praticamente o mesmo do Capítulo 8.
A diferença: agora ele tem acesso à tool `delegate_to_agent`,
e pode decidir delegar tarefas grandes para sub-agentes.

```python
MAX_TURNS = 15


async def rodar_orquestrador(prompt: str) -> AsyncGenerator[Event, None]:
    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]

    schemas_disponiveis = [
        GLOBAL_SCHEMAS[name]
        for name in ["list_dir", "read_file", "delegate_to_agent"]
    ]

    for turno in range(MAX_TURNS):
        response = await client.aio.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(
                system_instruction=(
                    "Você é um orquestrador. Para tarefas grandes que envolvem "
                    "ler vários arquivos, prefira delegar a análise de cada um "
                    "ao sub-agente 'analista_arquivo'. Isso evita acumular "
                    "todo o conteúdo no seu próprio contexto."
                ),
                tools=[types.Tool(function_declarations=schemas_disponiveis)],
                temperature=0.7,
            )
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        tool_calls = []
        for part in parts:
            if part.text:
                yield Event(type=EventType.CONTENT, payload={"text": part.text})
            elif part.function_call:
                fc = part.function_call
                tool_calls.append((fc.name, dict(fc.args) if fc.args else {}))

        if not tool_calls:
            yield Event(type=EventType.FINISHED)
            return

        tool_result_parts = []
        for name, args in tool_calls:
            if name == "delegate_to_agent":
                yield Event(type=EventType.SUB_AGENT,
                            payload={"agent": args.get("agent_name"),
                                     "task":  args.get("task")})
                resultado = await delegate_to_agent(**args)
            else:
                yield Event(type=EventType.TOOL_REQUEST,
                            payload={"name": name, "args": args})

                fn = GLOBAL_TOOLS.get(name)
                resultado = fn(**args) if fn else f"Tool '{name}' não existe"

            yield Event(type=EventType.TOOL_RESULT,
                        payload={"name": name, "output": resultado})

            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=name, response={"result": resultado}
                ))
            )

        historico.append(types.Content(role="tool", parts=tool_result_parts))


async for ev in rodar_orquestrador(
    "Liste os arquivos .py do diretório cap08 e peça ao analista_arquivo para descrever cada um."
):
    if ev.type == EventType.CONTENT:
        print(f"\n[Orquestrador]: {ev.payload['text']}")
    elif ev.type == EventType.TOOL_REQUEST:
        print(f"  → {ev.payload['name']}({ev.payload['args']})")
    elif ev.type == EventType.SUB_AGENT:
        print(f"  └── [delegando para '{ev.payload['agent']}']: {ev.payload['task'][:60]}")
    elif ev.type == EventType.TOOL_RESULT:
        print(f"  ← {ev.payload['output'][:80]}")
    elif ev.type == EventType.FINISHED:
        print("\n✓ concluído")
```

Observe como o histórico do orquestrador permanece pequeno —
ele recebe **só o resumo** de cada análise, não o conteúdo
dos arquivos. Os sub-agentes carregaram esse peso e descartaram.

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| Bounded Context (DDD) | fronteira de significado e estado | base do isolamento de sub-agentes |
| Value Object (DDD) | objeto definido apenas por valores | `LocalAgentDefinition` |
| isolamento de tools | cada agente vê só suas próprias | sub-agente não acessa tudo |
| `complete_task` | sinaliza fim explícito | substituí "fim implícito por texto" |
| `AgentTerminateMode` | motivo da terminação | GOAL, MAX_TURNS, ERROR |
| `LocalAgentExecutor` | instancia e roda um sub-agente | encapsula o loop interno |
| `AgentScheduler` | registry de sub-agentes | conecta nomes a definitions |
| `delegate_to_agent` | tool de delegação | orquestrador "vê" sub-agentes como tools |

---

## A próxima dor

Você tem um sistema poderoso:

- Loop ReAct robusto
- Sessões persistentes
- Hooks extensíveis
- Sub-agentes isolados

Mas tudo isso são ferramentas **internas** ao seu projeto.
Se outro desenvolvedor quiser adicionar uma ferramenta ao seu agente,
ele precisa modificar seu código, adicionar arquivos no seu projeto,
seguir suas convenções.

Existe um padrão aberto que resolve isso: MCP (Model Context Protocol —
Protocolo de Contexto do Modelo). Ele define como ferramentas
externas (rodando em outro processo, em outra máquina, em outra linguagem)
podem ser plugadas em qualquer agente que fale o protocolo.

E mais: o agente pode descobrir conhecimento especializado on-demand —
ler "skills" do disco quando a tarefa exigir.

É o Capítulo 14: extensibilidade.

---

## Arquivo final

`cap13\subagents_loop.py` — junte as células 1 a 7.


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap12-hooks">← Cap 12 — Hooks</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap14-mcp-skills">Cap 14 — MCP →</a></div>
</div>
