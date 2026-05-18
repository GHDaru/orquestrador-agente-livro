---
title: "Capítulo 8 — O decorator @tool"
layout: default
---

# Capítulo 8 — O decorator @tool

---

## A dor

No Capítulo 7, adicionar uma nova ferramenta exige mexer em três lugares:

1. A função Python que faz o trabalho
2. O dicionário `TOOLS_FN` que mapeia nome → função
3. O `schema` com a `FunctionDeclaration` completa

Para duas ferramentas é tolerável. Para dez é um desastre.
Você esquece um lugar, e a ferramenta aparece no schema mas não
é executável — ou vice-versa. O erro só aparece quando o modelo
tenta usar a ferramenta.

A solução é declarar tudo num lugar só:

```python
@tool(
    name="list_dir",
    description="Lista arquivos em um diretório.",
    parameters={...}
)
def list_dir(path: str) -> str:
    ...
```

Um decorator (`@tool`) registra automaticamente a função
no dicionário e gera o schema. Você escreve a ferramenta uma vez,
em um lugar só. Adicionar uma nova é trivial.

Mas para isso, você precisa entender como decorators funcionam por dentro.

---

## Decorators: o conceito

Decorator é uma função que **recebe outra função e retorna uma função**.
É um padrão de Python para "embrulhar" comportamento ao redor
de uma função existente, sem modificar o código original.

A sintaxe `@decorator` é só açúcar sintático.
Estes dois trechos são equivalentes:

```python
@meu_decorator
def minha_funcao():
    return 42

# É exatamente igual a:

def minha_funcao():
    return 42

minha_funcao = meu_decorator(minha_funcao)
```

O decorator é chamado **uma vez**, no momento em que o Python
processa a definição da função. O nome `minha_funcao` é
reatribuído ao resultado do decorator.

---

## Os três níveis de complexidade

### Nível 1 — Decorator simples

Função que recebe função e retorna função:

```python
def logar(fn):
    def wrapper(*args, **kwargs):
        print(f"Chamando {fn.__name__}")
        resultado = fn(*args, **kwargs)
        print(f"Retornou: {resultado}")
        return resultado
    return wrapper


@logar
def soma(a, b):
    return a + b


soma(2, 3)
# Chamando soma
# Retornou: 5
```

### Nível 2 — Decorator com parâmetros (fábrica de decorators)

Quando você precisa configurar o decorator, adiciona mais um nível:

```python
def repetir(n):                      # ← recebe configuração
    def decorator(fn):                # ← recebe a função
        def wrapper(*args, **kwargs):  # ← embrulha a chamada
            for _ in range(n):
                fn(*args, **kwargs)
        return wrapper
    return decorator


@repetir(3)
def diga_oi():
    print("oi")


diga_oi()
# oi
# oi
# oi
```

`repetir(3)` é chamada primeiro — ela retorna um decorator.
Esse decorator é então aplicado a `diga_oi`.

### Nível 3 — Decorator que registra sem modificar

Para nosso caso (registrar ferramentas), não precisamos
modificar a função — só registrá-la em um dicionário global
e devolver a função original intacta:

```python
_REGISTRO = {}

def registrar(nome):
    def decorator(fn):
        _REGISTRO[nome] = fn
        return fn   # devolve a função sem alterar
    return decorator


@registrar("saudacao")
def diga_oi():
    return "oi"


print(_REGISTRO)        # {'saudacao': <function diga_oi>}
print(diga_oi())        # oi  (função funciona normalmente)
```

Esse é o padrão exato que vamos usar para o `@tool`.

---

## MAX_TURNS: o invariante de segurança

O `while True` do loop assume que o modelo eventualmente vai
parar de pedir ferramentas. Mas se o modelo entrar em loop
(chamando a mesma ferramenta com os mesmos argumentos infinitamente),
não há proteção — o código roda até consumir toda a cota da API.

A solução é simples: trocar `while True` por um loop com limite máximo.

```python
MAX_TURNS = 10

for turno in range(MAX_TURNS):
    # ... lógica do loop ...
    if not tool_calls:
        return
# Se chegou aqui, atingiu o limite
yield Event(type=EventType.ERROR, payload={"message": "Limite atingido"})
```

Dez turnos é generoso para tarefas reais. Tarefas que precisam
de mais provavelmente deveriam ser quebradas em sub-tarefas
(que aparecem no Capítulo 13).

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta do capítulo:

```powershell
New-Item -ItemType Directory -Force -Path cap08
```

No VS Code, crie `cap08.ipynb` dentro de `cap08`
e selecione o kernel `Python (agente)`.

---

## As células do notebook

### Célula 1 — Decorator mais simples possível

```python
def logar(fn):
    def wrapper(*args, **kwargs):
        print(f"→ chamando {fn.__name__}({args}, {kwargs})")
        resultado = fn(*args, **kwargs)
        print(f"← retornou {resultado}")
        return resultado
    return wrapper


@logar
def soma(a, b):
    return a + b


soma(2, 3)
```

---

### Célula 2 — Decorator com parâmetros

```python
def repetir(n):
    def decorator(fn):
        def wrapper(*args, **kwargs):
            for _ in range(n):
                fn(*args, **kwargs)
        return wrapper
    return decorator


@repetir(3)
def diga_oi(nome):
    print(f"oi, {nome}")


diga_oi("João")
```

---

### Célula 3 — Decorator de registro (sem modificar a função)

```python
_REGISTRO = {}


def registrar(nome):
    def decorator(fn):
        _REGISTRO[nome] = fn
        return fn
    return decorator


@registrar("saudacao")
def diga_oi():
    return "oi"


@registrar("despedida")
def diga_tchau():
    return "tchau"


print(f"Funções registradas: {list(_REGISTRO.keys())}")
print(f"Chamando saudacao: {_REGISTRO['saudacao']()}")
print(f"Chamando despedida: {_REGISTRO['despedida']()}")
```

Este é o padrão exato do `@tool`: registrar em estruturas
globais sem alterar a função em si.

---

### Célula 4 — Imports do loop

```python
import asyncio
import os
from dataclasses import dataclass, field
from enum import Enum
from typing import AsyncGenerator, Callable
from google import genai
from google.genai import types

client = genai.Client()

MAX_TURNS = 10


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
```

---

### Célula 5 — O decorator @tool

O decorator recebe três configurações (nome, descrição, parâmetros)
e registra a função em dois lugares: `_TOOLS` para o despacho
de execução, e `_SCHEMAS` para a chamada à API.

Note o detalhe importante: o decorator retorna `fn` (a função original).
A função fica intacta — você ainda pode chamá-la diretamente como
qualquer função Python normal.

```python
_TOOLS:   dict[str, Callable] = {}
_SCHEMAS: list[types.FunctionDeclaration] = []


def tool(name: str, description: str, parameters: dict):
    def decorator(fn: Callable) -> Callable:
        _TOOLS[name] = fn

        _SCHEMAS.append(
            types.FunctionDeclaration(
                name=name,
                description=description,
                parameters=types.Schema(
                    type=types.Type.OBJECT,
                    properties={
                        k: types.Schema(
                            type=types.Type.STRING,
                            description=v.get("description", "")
                        )
                        for k, v in parameters.get("properties", {}).items()
                    },
                    required=parameters.get("required", [])
                )
            )
        )

        return fn

    return decorator
```

---

### Célula 6 — Registrando ferramentas com o decorator

Compare com o Capítulo 7: aqui tudo de uma ferramenta fica
em um único bloco. Adicionar uma nova é só um novo bloco —
nenhuma alteração em outros lugares.

```python
@tool(
    name="list_dir",
    description="Lista arquivos e pastas em um diretório.",
    parameters={
        "properties": {"path": {"description": "Caminho. Use '.' para o atual."}},
        "required": ["path"]
    }
)
def list_dir(path: str) -> str:
    try:
        return "\n".join(sorted(os.listdir(path)))
    except Exception as e:
        return f"Erro: {e}"


@tool(
    name="read_file",
    description="Lê o conteúdo de um arquivo de texto.",
    parameters={
        "properties": {"path": {"description": "Caminho do arquivo a ler."}},
        "required": ["path"]
    }
)
def read_file(path: str) -> str:
    try:
        with open(path, "r", encoding="utf-8") as f:
            return f.read()
    except Exception as e:
        return f"Erro: {e}"


@tool(
    name="echo",
    description="Repete uma mensagem. Use para confirmar entendimento.",
    parameters={
        "properties": {"message": {"description": "Mensagem a repetir."}},
        "required": ["message"]
    }
)
def echo(message: str) -> str:
    return message


print(f"Ferramentas registradas: {list(_TOOLS.keys())}")
print(f"Schemas no Gemini: {len(_SCHEMAS)}")
```

---

### Célula 7 — Testando as funções diretamente

As funções decoradas continuam funcionando normalmente.
O decorator não as alterou — só as registrou.

```python
print(list_dir("."))
print()
print(echo("teste"))
```

---

### Célula 8 — O loop com MAX_TURNS

A mudança em relação ao Capítulo 7: `while True` virou
`for turno in range(MAX_TURNS)`. Se o loop atingir o limite
sem o modelo encerrar, é emitido um Event de ERROR.

O loop também usa `_SCHEMAS` (preenchido pelo decorator)
em vez do schema declarado manualmente.

```python
async def rodar(
    prompt: str,
    system_prompt: str = "Você é um assistente útil. Responda em português.",
) -> AsyncGenerator[Event, None]:
    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]

    for turno in range(MAX_TURNS):
        response = await client.aio.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(
                system_instruction=system_prompt,
                tools=[types.Tool(function_declarations=_SCHEMAS)],
                temperature=0.7,
            )
        )

        if not response.candidates:
            yield Event(type=EventType.ERROR, payload={"message": "Sem resposta do modelo."})
            return

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        tool_calls: list[ToolCall] = []

        for part in parts:
            if part.text:
                yield Event(type=EventType.CONTENT, payload={"text": part.text})
            elif part.function_call:
                fc   = part.function_call
                call = ToolCall(id=fc.name, name=fc.name,
                                args=dict(fc.args) if fc.args else {})
                tool_calls.append(call)
                yield Event(type=EventType.TOOL_REQUEST,
                            payload={"id": call.id, "name": call.name, "args": call.args})

        if not tool_calls:
            yield Event(type=EventType.FINISHED)
            return

        tool_result_parts = []
        for call in tool_calls:
            fn        = _TOOLS.get(call.name)
            resultado = fn(**call.args) if fn else f"Tool '{call.name}' não existe."

            yield Event(type=EventType.TOOL_RESULT,
                        payload={"id": call.id, "name": call.name, "output": resultado})

            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=call.name, response={"result": resultado}
                ))
            )

        historico.append(types.Content(role="tool", parts=tool_result_parts))

    yield Event(type=EventType.ERROR,
                payload={"message": f"Limite de {MAX_TURNS} turnos atingido."})
```

---

### Célula 9 — Executando

```python
async for ev in rodar("Liste os arquivos desta pasta e me diga quantos são."):
    if ev.type == EventType.CONTENT:
        print(f"Gemini: {ev.payload['text']}")
    elif ev.type == EventType.TOOL_REQUEST:
        print(f"→ {ev.payload['name']}({ev.payload['args']})")
    elif ev.type == EventType.TOOL_RESULT:
        out = ev.payload['output']
        print(f"← {out[:80]}{'...' if len(out) > 80 else ''}")
    elif ev.type == EventType.FINISHED:
        print("✓ concluído")
    elif ev.type == EventType.ERROR:
        print(f"✗ {ev.payload['message']}")
```

---

### Célula 10 — Adicionando uma quarta ferramenta — em um lugar só

```python
@tool(
    name="current_dir",
    description="Retorna o caminho do diretório atual de trabalho.",
    parameters={
        "properties": {},
        "required": []
    }
)
def current_dir() -> str:
    return os.getcwd()


print(f"Total de ferramentas agora: {list(_TOOLS.keys())}")

async for ev in rodar("Em que pasta você está rodando?"):
    if ev.type == EventType.CONTENT:
        print(f"Gemini: {ev.payload['text']}")
    elif ev.type == EventType.TOOL_REQUEST:
        print(f"→ {ev.payload['name']}({ev.payload['args']})")
    elif ev.type == EventType.TOOL_RESULT:
        print(f"← {ev.payload['output']}")
    elif ev.type == EventType.FINISHED:
        print("✓ concluído")
```

Nenhuma outra linha precisou mudar.

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| decorator | função que recebe e retorna função | base para registro automático |
| `@decorator` | açúcar sintático para chamar o decorator | substitui `f = decorator(f)` |
| fábrica de decorator | função que recebe config e retorna decorator | `@tool(name=..., ...)` |
| registro global | dicionários `_TOOLS` e `_SCHEMAS` | desacopla declaração de uso |
| `MAX_TURNS` | limite máximo de turnos | proteção contra loop infinito |
| `for turno in range(MAX_TURNS)` | substitui `while True` | invariante de segurança |
| `Callable` | tipo para funções | type hint mais preciso |

---

## O loop está completo

Ao final deste capítulo, você tem:

- Loop ReAct funcional
- Tipos para tudo
- Async/await com geração em tempo real
- Decorator que centraliza declaração de ferramentas
- Limite de turnos para segurança

A Parte II termina aqui. O `loop.py` é o coração do sistema —
ele não vai mudar muito a partir de agora.
O que vem a seguir é colocar esse loop em um servidor (Parte III)
e adicionar infraestrutura (Parte IV).

---

## A próxima dor

O agente roda no terminal. Funciona, mas para usá-lo de qualquer
outro lugar — uma página web, um aplicativo móvel, outro programa —
você precisa expô-lo via HTTP.

E mais: para o frontend mostrar texto chegando em tempo real
(efeito ChatGPT), HTTP precisa transmitir eventos conforme acontecem.
A tecnologia para isso é SSE (Server-Sent Events).

Esses dois temas — colocar o loop em FastAPI e expor via SSE —
são o Capítulo 9.

---

## Arquivo final do capítulo

Quando terminar o notebook, crie o arquivo `cap08\loop.py`:

```python
import asyncio
import os
from dataclasses import dataclass, field
from enum import Enum
from typing import AsyncGenerator, Callable
from google import genai
from google.genai import types

client = genai.Client()
MAX_TURNS = 10


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


_TOOLS:   dict[str, Callable] = {}
_SCHEMAS: list[types.FunctionDeclaration] = []


def tool(name: str, description: str, parameters: dict):
    def decorator(fn: Callable) -> Callable:
        _TOOLS[name] = fn
        _SCHEMAS.append(
            types.FunctionDeclaration(
                name=name,
                description=description,
                parameters=types.Schema(
                    type=types.Type.OBJECT,
                    properties={
                        k: types.Schema(type=types.Type.STRING,
                                        description=v.get("description", ""))
                        for k, v in parameters.get("properties", {}).items()
                    },
                    required=parameters.get("required", [])
                )
            )
        )
        return fn
    return decorator


@tool(
    name="list_dir",
    description="Lista arquivos e pastas em um diretório.",
    parameters={"properties": {"path": {"description": "Caminho do diretório."}}, "required": ["path"]}
)
def list_dir(path: str) -> str:
    try:
        return "\n".join(sorted(os.listdir(path)))
    except Exception as e:
        return f"Erro: {e}"


@tool(
    name="read_file",
    description="Lê o conteúdo de um arquivo de texto.",
    parameters={"properties": {"path": {"description": "Caminho do arquivo."}}, "required": ["path"]}
)
def read_file(path: str) -> str:
    try:
        with open(path, "r", encoding="utf-8") as f:
            return f.read()
    except Exception as e:
        return f"Erro: {e}"


async def rodar(prompt: str,
                system_prompt: str = "Você é um assistente útil. Responda em português.",
                ) -> AsyncGenerator[Event, None]:
    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]

    for turno in range(MAX_TURNS):
        response = await client.aio.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(
                system_instruction=system_prompt,
                tools=[types.Tool(function_declarations=_SCHEMAS)],
                temperature=0.7,
            )
        )

        if not response.candidates:
            yield Event(type=EventType.ERROR, payload={"message": "Sem resposta."})
            return

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        tool_calls: list[ToolCall] = []

        for part in parts:
            if part.text:
                yield Event(type=EventType.CONTENT, payload={"text": part.text})
            elif part.function_call:
                fc   = part.function_call
                call = ToolCall(id=fc.name, name=fc.name,
                                args=dict(fc.args) if fc.args else {})
                tool_calls.append(call)
                yield Event(type=EventType.TOOL_REQUEST,
                            payload={"id": call.id, "name": call.name, "args": call.args})

        if not tool_calls:
            yield Event(type=EventType.FINISHED)
            return

        tool_result_parts = []
        for call in tool_calls:
            fn        = _TOOLS.get(call.name)
            resultado = fn(**call.args) if fn else f"Tool '{call.name}' não existe."
            yield Event(type=EventType.TOOL_RESULT,
                        payload={"id": call.id, "name": call.name, "output": resultado})
            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=call.name, response={"result": resultado}
                ))
            )

        historico.append(types.Content(role="tool", parts=tool_result_parts))

    yield Event(type=EventType.ERROR,
                payload={"message": f"Limite de {MAX_TURNS} turnos atingido."})


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
        elif ev.type == EventType.ERROR:
            print(f"✗ {ev.payload['message']}")


if __name__ == "__main__":
    asyncio.run(main())
```


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap07-generator">← Cap 7 — Generator</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap09-fastapi-sse">Cap 9 — FastAPI →</a></div>
</div>
