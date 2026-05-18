---
title: "Capítulo 7 — Produzindo eventos em tempo real"
layout: default
---

# Capítulo 7 — Produzindo eventos em tempo real

---

## A dor

O loop do Capítulo 6 está async. Roda em paralelo, atende múltiplos
usuários simultaneamente. Mas tem uma característica que vira problema
quando você quer mostrar progresso ao usuário:

```python
events = await rodar("liste e leia os arquivos")
# ← O Python fica aqui parado 15 segundos
#   antes de qualquer Event chegar
for ev in events:
    print(ev)   # tudo é impresso de uma vez no final
```

`rodar` retorna uma **lista**. Listas só ficam prontas quando todos
os elementos foram calculados. Para uma operação demorada, isso significa
silêncio total até o fim — e então uma enxurrada de Events.

Para o efeito "ChatGPT" (palavra por palavra aparecendo) você precisa
que o `rodar` **entregue cada Event no momento em que acontece**,
sem esperar o loop terminar.

A diferença está entre `return` e `yield`.

---

## O modelo mental: torneira vs balde

### Lista é balde

Você pede um balde de água.
A pessoa enche o balde inteiro.
Quando termina, te entrega o balde cheio.
Você só vê água quando o balde já está totalmente cheio.

```
[espera]
[espera]
[espera]
[15 segundos depois]
TODA A ÁGUA DE UMA VEZ
```

Isso é o que `return list[Event]` faz.

### Generator é torneira

Você abre a torneira.
A água começa a fluir imediatamente.
Cada gota que sai, você usa imediatamente.
Não precisa esperar a torneira "terminar".

```
[1s]  gota 1
[2s]  gota 2
[5s]  gota 3
[6s]  gota 4
...
```

Isso é o que `yield Event` faz.

---

## yield: o que é

`yield` é uma palavra-chave do Python que transforma uma função
comum em um **generator** (gerador — função que produz valores
um a um, pausando entre eles).

Comparação direta:

```python
# Função normal — retorna uma lista
def numeros():
    return [1, 2, 3]

resultado = numeros()
print(resultado)   # [1, 2, 3]
```

```python
# Generator — produz cada número conforme pedido
def numeros():
    yield 1
    yield 2
    yield 3

resultado = numeros()
print(resultado)   # <generator object numeros at 0x...>

for n in resultado:
    print(n)   # 1, depois 2, depois 3
```

A diferença crucial: o generator **pausa** em cada `yield`.
Quando o consumidor pede o próximo valor (com `for` ou `next()`),
a função retoma de onde parou, calcula o próximo valor,
e pausa de novo.

Para um loop que demora minutos, isso significa: cada Event
fica disponível **assim que é produzido**. O consumidor pode
processar (imprimir, enviar pela rede, atualizar UI) imediatamente.

---

## AsyncGenerator: yield + async

Combinar `async` com `yield` produz um AsyncGenerator
(gerador assíncrono — função que produz valores ao longo do tempo,
pausando tanto pelos yields quanto pelos awaits internos):

```python
async def rodar(prompt: str) -> AsyncGenerator[Event, None]:
    while True:
        response = await client.aio.models.generate_content(...)
        # ↑ async pausa aqui, espera o Gemini

        yield Event(type=EventType.CONTENT, payload={"text": "..."})
        # ↑ yield pausa aqui, entrega o Event ao consumidor
```

Esse tipo de função pode ter **vários pontos de pausa**:
um para esperar o modelo (`await`), outro para entregar
o evento ao consumidor (`yield`). O Python coordena tudo
automaticamente.

### A assinatura do tipo

`AsyncGenerator[Event, None]` significa:

- **Event** — tipo dos valores produzidos (o que sai pelos yields)
- **None** — tipo do valor de retorno final (não usamos)

Tem que importar de `typing`:

```python
from typing import AsyncGenerator
```

### Como consumir: async for

Generator síncrono usa `for`. Async generator usa `async for`:

```python
async for ev in rodar("alguma pergunta"):
    print(ev)   # roda assim que cada Event é produzido
```

A cada `yield` no `rodar`, o consumidor recebe um Event e processa.
Enquanto o consumidor processa, o produtor está pausado.
Quando o consumidor pede o próximo, o produtor retoma.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta do capítulo:

```powershell
New-Item -ItemType Directory -Force -Path cap07
```

No VS Code, crie `cap07.ipynb` dentro de `cap07`
e selecione o kernel `Python (agente)`.

---

## As células do notebook

### Célula 1 — Generator síncrono: o conceito básico

```python
def contagem_regressiva(n: int):
    print(f"  [generator] começando contagem")
    for i in range(n, 0, -1):
        print(f"  [generator] vai entregar {i}")
        yield i
        print(f"  [generator] retomou após yield")
    print(f"  [generator] terminei")


gen = contagem_regressiva(3)
print(f"Tipo de gen: {type(gen).__name__}")
print("Vou iterar:")

for valor in gen:
    print(f"[consumidor] recebi {valor}")
```

Observe a ordem dos prints. O generator e o consumidor
se alternam: produtor entrega um valor, consumidor processa,
produtor retoma, e assim por diante.

---

### Célula 2 — Lista vs generator: o experimento de tempo

```python
import time


def lista(n: int):
    print("Função lista: começando...")
    resultado = []
    for i in range(n):
        time.sleep(0.5)
        resultado.append(i)
    print("Função lista: terminou. Retornando tudo de uma vez.")
    return resultado


def generator(n: int):
    print("Função generator: começando...")
    for i in range(n):
        time.sleep(0.5)
        yield i
    print("Função generator: terminou.")


print("=== LISTA ===")
inicio = time.time()
itens = lista(3)
print(f"  [{time.time()-inicio:.1f}s] consumidor pegou os itens")
for i in itens:
    print(f"  [{time.time()-inicio:.1f}s] processando {i}")

print("\n=== GENERATOR ===")
inicio = time.time()
gen = generator(3)
for i in gen:
    print(f"  [{time.time()-inicio:.1f}s] processando {i}")
```

Compare. Na lista, todos os itens aparecem juntos depois
da espera total. No generator, cada item aparece logo após
ser produzido — o consumidor pode reagir em tempo real.

---

### Célula 3 — AsyncGenerator: async + yield

```python
import asyncio
from typing import AsyncGenerator


async def async_generator(n: int) -> AsyncGenerator[int, None]:
    for i in range(n):
        await asyncio.sleep(0.5)
        yield i


inicio = time.time()
async for valor in async_generator(3):
    print(f"[{time.time()-inicio:.1f}s] recebi {valor}")
```

A função `async_generator` tem dois pontos de pausa:
o `await asyncio.sleep` (espera não bloqueante)
e o `yield` (entrega ao consumidor).
O `async for` coordena ambos.

---

### Célula 4 — Imports e tipos do agente

```python
import os
from dataclasses import dataclass, field
from enum import Enum
from typing import AsyncGenerator
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
class ToolCall:
    id:   str
    name: str
    args: dict


@dataclass
class Event:
    type:    EventType
    payload: dict = field(default_factory=dict)


def list_dir(path: str) -> str:
    try:
        return "\n".join(sorted(os.listdir(path)))
    except Exception as e:
        return f"Erro: {e}"


def read_file(path: str) -> str:
    try:
        with open(path, "r", encoding="utf-8") as f:
            return f.read()
    except Exception as e:
        return f"Erro: {e}"


TOOLS_FN = {"list_dir": list_dir, "read_file": read_file}

schema = types.Tool(function_declarations=[
    types.FunctionDeclaration(
        name="list_dir",
        description="Lista arquivos e pastas em um diretório.",
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={"path": types.Schema(type=types.Type.STRING, description="Caminho")},
            required=["path"]
        )
    ),
    types.FunctionDeclaration(
        name="read_file",
        description="Lê o conteúdo de um arquivo de texto.",
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={"path": types.Schema(type=types.Type.STRING, description="Caminho")},
            required=["path"]
        )
    ),
])
```

---

### Célula 5 — O loop ReAct como AsyncGenerator

A diferença em relação ao Capítulo 6: cada Event é entregue
no momento em que acontece, com `yield`. O retorno final
é simplesmente `return` (sem valor) — porque o tipo de retorno
é `AsyncGenerator[Event, None]`.

```python
async def rodar(prompt: str) -> AsyncGenerator[Event, None]:
    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]

    while True:
        response = await client.aio.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        tool_calls: list[ToolCall] = []

        for part in parts:
            if part.text:
                yield Event(type=EventType.CONTENT, payload={"text": part.text})

            elif part.function_call:
                fc = part.function_call
                call = ToolCall(id=fc.name, name=fc.name,
                                args=dict(fc.args) if fc.args else {})
                tool_calls.append(call)
                yield Event(type=EventType.TOOL_REQUEST,
                            payload={"name": call.name, "args": call.args})

        if not tool_calls:
            yield Event(type=EventType.FINISHED)
            return

        tool_result_parts = []
        for call in tool_calls:
            fn        = TOOLS_FN.get(call.name)
            resultado = fn(**call.args) if fn else f"Tool '{call.name}' não encontrada."

            yield Event(type=EventType.TOOL_RESULT,
                        payload={"name": call.name, "output": resultado})

            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=call.name, response={"result": resultado}
                ))
            )

        historico.append(types.Content(role="tool", parts=tool_result_parts))
```

---

### Célula 6 — Consumindo com async for

Observe que os prints aparecem **conforme o agente trabalha**,
não todos no final. Para tarefas multi-turno, você vê o progresso
em tempo real.

```python
import time

inicio = time.time()

async for ev in rodar("Liste os arquivos e leia o primeiro .py que encontrar."):
    elapsed = time.time() - inicio

    if ev.type == EventType.CONTENT:
        print(f"[{elapsed:.1f}s] Gemini: {ev.payload['text']}")
    elif ev.type == EventType.TOOL_REQUEST:
        print(f"[{elapsed:.1f}s] → {ev.payload['name']}({ev.payload['args']})")
    elif ev.type == EventType.TOOL_RESULT:
        out = ev.payload['output']
        print(f"[{elapsed:.1f}s] ← {out[:60]}{'...' if len(out) > 60 else ''}")
    elif ev.type == EventType.FINISHED:
        print(f"[{elapsed:.1f}s] ✓ concluído")
```

---

### Célula 7 — Comparando: lista vs generator no loop

```python
async def rodar_lista(prompt: str) -> list[Event]:
    events = []
    async for ev in rodar(prompt):
        events.append(ev)
    return events


print("=== COMO LISTA (espera tudo) ===")
inicio = time.time()
events = await rodar_lista("Liste os arquivos desta pasta")
print(f"[{time.time()-inicio:.1f}s] recebi {len(events)} events de uma vez")
for ev in events:
    print(f"   {ev.type.value}")

print("\n=== COMO GENERATOR (em tempo real) ===")
inicio = time.time()
async for ev in rodar("Liste os arquivos desta pasta"):
    print(f"[{time.time()-inicio:.1f}s] {ev.type.value}")
```

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| generator | função que produz valores aos poucos | base para entrega em tempo real |
| `yield` | entrega um valor e pausa a função | substituto do `return` quando você quer streaming |
| `for` em generator | itera valores produzidos | consumidor síncrono |
| AsyncGenerator | generator com `await` interno | combina async com yield |
| `async for` | itera AsyncGenerator | consumidor assíncrono |
| `from typing import AsyncGenerator` | import do tipo | para type hints |
| streaming | entrega contínua conforme produção | base do efeito "ChatGPT" |

---

## A próxima dor

O loop produz Events em tempo real. Cada turno do modelo
entrega seus eventos imediatamente. Mas ainda temos um problema
organizacional: adicionar uma nova ferramenta exige mexer em
**três lugares**:

1. Escrever a função Python
2. Registrar no dicionário `TOOLS_FN`
3. Adicionar no `schema` com a FunctionDeclaration completa

Se você quiser ter dez ferramentas, são trinta pontos de manutenção.
Errar um faz a tool aparecer no schema mas não ser executável,
ou vice-versa — silenciosamente.

Além disso, se o modelo entrar em loop infinito (chamando a mesma
ferramenta repetidamente), não há proteção. O `while True` continua
indefinidamente, consumindo cota da API.

Essas duas dores — organização e segurança — levam ao Capítulo 8:
o decorator `@tool` e o `MAX_TURNS`.

---

## Arquivo final do capítulo

Quando terminar o notebook, crie o arquivo `cap07\generator_loop.py`:

```python
import asyncio
import os
from dataclasses import dataclass, field
from enum import Enum
from typing import AsyncGenerator
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
class ToolCall:
    id:   str
    name: str
    args: dict


@dataclass
class Event:
    type:    EventType
    payload: dict = field(default_factory=dict)


def list_dir(path: str) -> str:
    try:
        return "\n".join(sorted(os.listdir(path)))
    except Exception as e:
        return f"Erro: {e}"


def read_file(path: str) -> str:
    try:
        with open(path, "r", encoding="utf-8") as f:
            return f.read()
    except Exception as e:
        return f"Erro: {e}"


TOOLS_FN = {"list_dir": list_dir, "read_file": read_file}

schema = types.Tool(function_declarations=[
    types.FunctionDeclaration(
        name="list_dir",
        description="Lista arquivos e pastas em um diretório.",
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={"path": types.Schema(type=types.Type.STRING, description="Caminho")},
            required=["path"]
        )
    ),
    types.FunctionDeclaration(
        name="read_file",
        description="Lê o conteúdo de um arquivo de texto.",
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={"path": types.Schema(type=types.Type.STRING, description="Caminho")},
            required=["path"]
        )
    ),
])


async def rodar(prompt: str) -> AsyncGenerator[Event, None]:
    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]

    while True:
        response = await client.aio.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        tool_calls: list[ToolCall] = []

        for part in parts:
            if part.text:
                yield Event(type=EventType.CONTENT, payload={"text": part.text})
            elif part.function_call:
                fc = part.function_call
                call = ToolCall(id=fc.name, name=fc.name,
                                args=dict(fc.args) if fc.args else {})
                tool_calls.append(call)
                yield Event(type=EventType.TOOL_REQUEST,
                            payload={"name": call.name, "args": call.args})

        if not tool_calls:
            yield Event(type=EventType.FINISHED)
            return

        tool_result_parts = []
        for call in tool_calls:
            fn        = TOOLS_FN.get(call.name)
            resultado = fn(**call.args) if fn else f"Tool '{call.name}' não encontrada."
            yield Event(type=EventType.TOOL_RESULT,
                        payload={"name": call.name, "output": resultado})
            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=call.name, response={"result": resultado}
                ))
            )

        historico.append(types.Content(role="tool", parts=tool_result_parts))


async def main():
    prompt = input("Você: ").strip() or "Liste os arquivos desta pasta."

    async for ev in rodar(prompt):
        if ev.type == EventType.CONTENT:
            print(f"Gemini: {ev.payload['text']}")
        elif ev.type == EventType.TOOL_REQUEST:
            print(f"→ {ev.payload['name']}({ev.payload['args']})")
        elif ev.type == EventType.TOOL_RESULT:
            out = ev.payload['output']
            print(f"← {out[:80]}{'...' if len(out) > 80 else ''}")
        elif ev.type == EventType.FINISHED:
            print("✓ concluído")


if __name__ == "__main__":
    asyncio.run(main())
```


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap06-async-await">← Cap 6 — async</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap08-decorator-tool">Cap 8 — Decorator →</a></div>
</div>
