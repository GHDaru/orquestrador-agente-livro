---
title: "Capítulo 12 — Hooks: pontos de extensão"
layout: default
---

# Capítulo 12 — Hooks: pontos de extensão

---

## A dor

No final do Capítulo 11, listamos várias coisas que você
pode querer adicionar ao loop:

- Log de auditoria (gravar cada tool call)
- Aprovação de ações destrutivas (`delete_file`)
- Injetar contexto antes de cada prompt
- Telemetria de tempo e tokens
- Notificações de eventos importantes

Se cada uma dessas exigir mexer no `rodar()`, três problemas:

1. **Acoplamento**: o loop fica responsável por coisas externas
2. **Risco**: cada mudança pode quebrar o que funcionava
3. **Reuso**: o mesmo log pode servir 10 agentes diferentes

A solução é **inverter** a relação: em vez do loop conhecer
cada extensão, o loop expõe **pontos de extensão**, e qualquer
código se "pluga" neles externamente.

Esses pontos são chamados de **hooks** (ganchos).

---

## O conceito de hook

Hook é um ponto definido onde o loop diz: "neste momento,
quem quiser pode executar código". O loop emite o evento,
e qualquer handler (manipulador) registrado é chamado.

```
loop em ação                handlers registrados
   │
   │── BEFORE_PROMPT ─────→ injeta git status
   │── BEFORE_PROMPT ─────→ log de auditoria
   │
   │── chama o modelo
   │
   │── BEFORE_TOOL_CALL ──→ pede aprovação se for destrutiva
   │── BEFORE_TOOL_CALL ──→ log de auditoria
   │
   │── executa a tool
   │
   │── AFTER_TOOL_CALL ───→ telemetria
   │
   ... e assim por diante
```

O loop não sabe quais handlers existem. Só dispara o evento.
Adicionar uma nova feature é registrar um novo handler —
zero alterações no loop.

Esse é o padrão **Observer** (Observador), comum em interfaces
gráficas, sistemas reativos e arquiteturas event-driven.

---

## Lifecycle events do agente

O loop tem momentos bem definidos onde extensões fazem sentido:

```python
class HookEvent(str, Enum):
    BEFORE_PROMPT       = "before_prompt"       # antes de cada chamada ao modelo
    AFTER_RESPONSE      = "after_response"      # depois da resposta do modelo
    BEFORE_TOOL_CALL    = "before_tool_call"    # antes de executar uma tool
    AFTER_TOOL_CALL     = "after_tool_call"     # depois de executar uma tool
    SESSION_START       = "session_start"       # primeiro turno
    SESSION_END         = "session_end"         # último turno
    ERROR_OCCURRED      = "error_occurred"      # qualquer erro
```

Sete pontos cobrem praticamente tudo. Se uma feature não couber
em nenhum desses pontos, é sinal de que talvez não seja uma extensão —
seja parte do domínio.

---

## HookRegistry — onde os handlers moram

O `HookRegistry` (registro de hooks) mantém a lista de handlers
por evento. O loop pergunta ao registry: "para o evento X,
quem está escutando?", e chama cada handler.

```python
class HookRegistry:
    def __init__(self):
        self.handlers: dict[HookEvent, list[Callable]] = {}

    def on(self, event: HookEvent):
        def decorator(fn):
            self.handlers.setdefault(event, []).append(fn)
            return fn
        return decorator

    async def emit(self, event: HookEvent, context: dict):
        for handler in self.handlers.get(event, []):
            await handler(context)
```

A API é simples:

```python
hooks = HookRegistry()

@hooks.on(HookEvent.BEFORE_PROMPT)
async def logar(context):
    print(f"Antes do prompt: {context['prompt']}")
```

Registrar é decorar. Disparar é chamar `hooks.emit(...)`.

---

## MessageBus — comunicação assíncrona

Alguns handlers precisam **esperar** por algo externo antes do loop
continuar. Exemplo clássico: aprovação de tool destrutiva.

Quando o modelo pede `delete_file`, o handler de aprovação:
1. Envia uma pergunta ao frontend ("aprovar?")
2. **Espera** o usuário clicar Aprovar ou Rejeitar
3. Devolve a decisão ao loop

Para isso, hooks por si só não bastam. Precisa de um
canal de comunicação assíncrono — o **MessageBus**
(barramento de mensagens).

```python
class MessageBus:
    def __init__(self):
        self.subscribers: dict[str, list[asyncio.Queue]] = {}

    def subscribe(self, topic: str) -> asyncio.Queue:
        queue = asyncio.Queue()
        self.subscribers.setdefault(topic, []).append(queue)
        return queue

    async def publish(self, topic: str, message: dict):
        for queue in self.subscribers.get(topic, []):
            await queue.put(message)
```

Padrão publish/subscribe (publicação/assinatura).
Quem quer ouvir um tópico se inscreve; quem tem algo a dizer publica.
O bus distribui.

---

## Por que isto é DDD — Domain Service e Application Service

Em DDD, hooks tipicamente moram na camada de **Application Service**
(Serviço de Aplicação) — orquestração que não pertence ao núcleo
do domínio mas coordena suas operações.

- O **domínio** é o loop ReAct: `Event`, `ToolCall`, `Session`
- O `HookRegistry` é **infraestrutura**: tecnicamente neutra
- Os handlers que você registra são **Application Services**:
  orquestram comportamento sem alterar o domínio

A regra de ouro: o domínio define **o que** acontece;
os hooks decidem **o que mais** acontece além disso.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta:

```powershell
New-Item -ItemType Directory -Force -Path cap12
```

No VS Code, crie `cap12.ipynb` dentro de `cap12`.

---

## As células do notebook

### Célula 1 — HookEvent enum

```python
import asyncio
from enum import Enum
from typing import Callable


class HookEvent(str, Enum):
    BEFORE_PROMPT    = "before_prompt"
    AFTER_RESPONSE   = "after_response"
    BEFORE_TOOL_CALL = "before_tool_call"
    AFTER_TOOL_CALL  = "after_tool_call"
    SESSION_START    = "session_start"
    SESSION_END      = "session_end"
    ERROR_OCCURRED   = "error_occurred"


for e in HookEvent:
    print(f"  {e.name} = {e.value}")
```

---

### Célula 2 — HookRegistry com decorator

```python
class HookRegistry:
    def __init__(self):
        self.handlers: dict[HookEvent, list[Callable]] = {}

    def on(self, event: HookEvent):
        def decorator(fn):
            self.handlers.setdefault(event, []).append(fn)
            return fn
        return decorator

    async def emit(self, event: HookEvent, context: dict):
        for handler in self.handlers.get(event, []):
            try:
                if asyncio.iscoroutinefunction(handler):
                    await handler(context)
                else:
                    handler(context)
            except Exception as e:
                print(f"Erro em handler de {event}: {e}")


hooks = HookRegistry()
print("HookRegistry criado.")
```

---

### Célula 3 — Registrando handlers

Três handlers como demonstração: um log simples, uma telemetria
que mede tempo, e um auditor que escreve em arquivo.

```python
import time
from pathlib import Path

audit_file = Path("audit.log")
tool_timings = {}


@hooks.on(HookEvent.BEFORE_PROMPT)
def log_prompt(ctx):
    print(f"[log] prompt: {ctx['prompt'][:60]}")


@hooks.on(HookEvent.BEFORE_TOOL_CALL)
def start_timer(ctx):
    tool_timings[ctx["call_id"]] = time.time()


@hooks.on(HookEvent.AFTER_TOOL_CALL)
def end_timer(ctx):
    inicio = tool_timings.pop(ctx["call_id"], None)
    if inicio:
        ms = (time.time() - inicio) * 1000
        print(f"[telemetria] {ctx['name']} levou {ms:.1f}ms")


@hooks.on(HookEvent.AFTER_TOOL_CALL)
async def auditar(ctx):
    linha = f"{time.time()}: {ctx['name']}({ctx['args']}) → {ctx['output'][:50]}\n"
    audit_file.write_text(audit_file.read_text() + linha if audit_file.exists() else linha)


print(f"Handlers registrados em BEFORE_PROMPT:    {len(hooks.handlers.get(HookEvent.BEFORE_PROMPT, []))}")
print(f"Handlers registrados em AFTER_TOOL_CALL:  {len(hooks.handlers.get(HookEvent.AFTER_TOOL_CALL, []))}")
```

---

### Célula 4 — Disparando os eventos manualmente

```python
await hooks.emit(HookEvent.BEFORE_PROMPT, {"prompt": "Liste os arquivos desta pasta..."})

await hooks.emit(HookEvent.BEFORE_TOOL_CALL, {
    "call_id": "abc-123",
    "name":    "list_dir",
    "args":    {"path": "."}
})

await asyncio.sleep(0.05)

await hooks.emit(HookEvent.AFTER_TOOL_CALL, {
    "call_id": "abc-123",
    "name":    "list_dir",
    "args":    {"path": "."},
    "output":  "file1\nfile2\nfile3"
})
```

Múltiplos handlers no mesmo evento todos executam.

---

### Célula 5 — MessageBus para comunicação assíncrona

```python
class MessageBus:
    def __init__(self):
        self.subscribers: dict[str, list[asyncio.Queue]] = {}

    def subscribe(self, topic: str) -> asyncio.Queue:
        queue = asyncio.Queue()
        self.subscribers.setdefault(topic, []).append(queue)
        return queue

    async def publish(self, topic: str, message: dict):
        for queue in self.subscribers.get(topic, []):
            await queue.put(message)


bus = MessageBus()
print("MessageBus criado.")
```

---

### Célula 6 — Aprovação de tool destrutiva

Cenário: quando o modelo pede uma tool destrutiva como
`delete_file`, o handler publica uma pergunta no bus
e espera a resposta antes do loop continuar.

```python
DESTRUCTIVE_TOOLS = {"delete_file", "write_file", "execute_command"}


@hooks.on(HookEvent.BEFORE_TOOL_CALL)
async def aprovar_destructive(ctx):
    if ctx["name"] not in DESTRUCTIVE_TOOLS:
        return

    print(f"\n⚠ Tool destrutiva detectada: {ctx['name']}({ctx['args']})")

    queue = bus.subscribe("approval_response")

    await bus.publish("approval_request", {
        "call_id": ctx["call_id"],
        "name":    ctx["name"],
        "args":    ctx["args"],
    })

    response = await queue.get()

    if not response.get("approved"):
        raise PermissionError(f"Tool {ctx['name']} rejeitada pelo usuário")

    print(f"✓ Aprovado por {response.get('user', 'usuário')}")


async def simular_aprovacao_do_frontend(approved: bool):
    await asyncio.sleep(0.1)
    await bus.publish("approval_response", {"approved": approved, "user": "demo"})


async def teste_aprovacao(approved: bool):
    print(f"\n=== Teste com approved={approved} ===")
    asyncio.create_task(simular_aprovacao_do_frontend(approved))
    try:
        await hooks.emit(HookEvent.BEFORE_TOOL_CALL, {
            "call_id": "del-1",
            "name":    "delete_file",
            "args":    {"path": "arquivo.txt"}
        })
        print("Tool seria executada agora")
    except PermissionError as e:
        print(f"✗ Rejeitada: {e}")


await teste_aprovacao(approved=True)
await teste_aprovacao(approved=False)
```

Observe: o loop não sabe o que é uma "tool destrutiva".
Essa decisão está no handler. O loop só dispara o evento.

---

### Célula 7 — Loop com hooks integrados

Versão simplificada do loop, com `hooks.emit()` nos pontos certos.
O código do loop não cresceu muito — só adicionou os emits.

```python
import os
from dataclasses import dataclass, field
from google import genai
from google.genai import types

client = genai.Client()


class EventType(str, Enum):
    CONTENT      = "content"
    TOOL_REQUEST = "tool_request"
    TOOL_RESULT  = "tool_result"
    FINISHED     = "finished"
    ERROR        = "error"


@dataclass
class Event:
    type:    EventType
    payload: dict = field(default_factory=dict)


_TOOLS = {}
_SCHEMAS = []


def tool(name, description, parameters):
    def decorator(fn):
        _TOOLS[name] = fn
        _SCHEMAS.append(types.FunctionDeclaration(
            name=name, description=description,
            parameters=types.Schema(
                type=types.Type.OBJECT,
                properties={k: types.Schema(type=types.Type.STRING, description=v.get("description", ""))
                            for k, v in parameters.get("properties", {}).items()},
                required=parameters.get("required", [])
            )
        ))
        return fn
    return decorator


@tool(name="list_dir", description="Lista arquivos.",
      parameters={"properties": {"path": {"description": "caminho"}}, "required": ["path"]})
def list_dir(path):
    try:
        return "\n".join(sorted(os.listdir(path)))
    except Exception as e:
        return f"Erro: {e}"


async def rodar_com_hooks(prompt, hooks: HookRegistry):
    await hooks.emit(HookEvent.SESSION_START, {"prompt": prompt})

    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]

    for turno in range(10):
        await hooks.emit(HookEvent.BEFORE_PROMPT, {"prompt": prompt, "turno": turno})

        try:
            response = await client.aio.models.generate_content(
                model="gemini-2.5-flash",
                contents=historico,
                config=types.GenerateContentConfig(
                    tools=[types.Tool(function_declarations=_SCHEMAS)]
                )
            )
        except Exception as e:
            await hooks.emit(HookEvent.ERROR_OCCURRED, {"error": str(e)})
            yield Event(type=EventType.ERROR, payload={"message": str(e)})
            return

        await hooks.emit(HookEvent.AFTER_RESPONSE, {"response": response})

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
            await hooks.emit(HookEvent.SESSION_END, {})
            yield Event(type=EventType.FINISHED)
            return

        tool_result_parts = []
        for name, args in tool_calls:
            call_id = f"{name}-{turno}"

            try:
                await hooks.emit(HookEvent.BEFORE_TOOL_CALL, {
                    "call_id": call_id, "name": name, "args": args
                })
            except PermissionError as e:
                yield Event(type=EventType.ERROR, payload={"message": str(e)})
                return

            yield Event(type=EventType.TOOL_REQUEST,
                        payload={"name": name, "args": args})

            fn        = _TOOLS.get(name)
            resultado = fn(**args) if fn else f"Tool '{name}' não existe."

            await hooks.emit(HookEvent.AFTER_TOOL_CALL, {
                "call_id": call_id, "name": name, "args": args, "output": resultado
            })

            yield Event(type=EventType.TOOL_RESULT,
                        payload={"name": name, "output": resultado})

            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=name, response={"result": resultado}
                ))
            )

        historico.append(types.Content(role="tool", parts=tool_result_parts))


async for ev in rodar_com_hooks("Liste os arquivos desta pasta.", hooks):
    if ev.type == EventType.CONTENT:
        print(f"Gemini: {ev.payload['text']}")
    elif ev.type == EventType.TOOL_REQUEST:
        print(f"→ {ev.payload['name']}")
    elif ev.type == EventType.TOOL_RESULT:
        print(f"← {ev.payload['output'][:50]}")
    elif ev.type == EventType.FINISHED:
        print("✓")
```

Os handlers registrados na Célula 3 todos executam automaticamente.
Você verá os logs `[log]` e `[telemetria]` intercalados
no fluxo normal.

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| Observer pattern | inversão da dependência (quem chama quem) | base dos hooks |
| `HookEvent` | enum de pontos de extensão | linguagem ubíqua dos lifecycle events |
| `HookRegistry` | armazena handlers por evento | infraestrutura de extensão |
| `@hooks.on(...)` | decorator de registro | API limpa para o consumidor |
| `emit()` | dispara um evento | o loop fala sem saber quem ouve |
| `MessageBus` | comunicação async pub/sub | para handlers que esperam externo |
| Application Service (DDD) | orquestração fora do domínio | onde hooks vivem |
| ações destrutivas | conceito de domínio mas decisão externa | hook decide, loop só executa |

---

## A próxima dor

Hooks permitem estender o loop sem alterá-lo. Mas todas as
ferramentas ainda compartilham o mesmo `_TOOLS` global —
o mesmo conjunto de capacidades para qualquer tarefa.

Imagine: você quer um agente especializado em arquivos
e outro especializado em queries SQL. Eles deveriam ter
conjuntos diferentes de ferramentas. E quando a tarefa é grande
demais, o orquestrador principal deveria poder **delegar**
para um sub-agente que trabalha isoladamente e devolve só o resultado.

Isso exige **isolamento**: cada agente com seu próprio registro
de ferramentas, sua própria sessão, seu próprio loop.

É o Capítulo 13.

---

## Arquivo final do capítulo

Junte tudo em `cap12\hooks_loop.py`.


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap11-sessao">← Cap 11 — Sessão</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap13-subagentes">Cap 13 — Sub-agentes →</a></div>
</div>
