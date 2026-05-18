---
title: "Capítulo 5 — Nomeando as coisas"
layout: default
---

# Capítulo 5 — Nomeando as coisas

---

## A dor

O loop do Capítulo 4 funciona. Mas tem fragilidades que ainda
não apareceram porque o sistema é pequeno.

Imagine que você está mexendo no código e digita:

```python
if event_type == "tool_requst":   # ← typo invisível
    executar_tool(event)
```

Python não avisa. O `if` simplesmente nunca é verdadeiro
porque `"tool_requst"` nunca vai bater com `"tool_request"`.
O código roda, mas a tool nunca é executada — e você passa
horas debugando algo que poderia ter sido apontado pelo editor.

Outro problema: as ferramentas hoje são funções soltas
no escopo global. O dicionário `TOOLS_FN` mistura responsabilidades.
Quando você quiser adicionar um novo evento ao loop, ou
um novo tipo de mensagem, vai espalhar strings literais por todo o código.

A solução é **dar nomes precisos às coisas**.
Não nomes em comentários ou documentação — nomes que o Python
conhece e verifica.

---

## Tipos como linguagem

Quando você escreve um livro, certas palavras viram termos técnicos:
"loop", "turno", "ferramenta", "evento". Cada uma significa algo
específico no contexto do livro. Se você usar "turno" para duas
coisas diferentes, o leitor se perde.

Em código é igual. Os tipos são os termos técnicos do seu sistema.
Quando você define `EventType`, está dizendo:
"existem exatamente estes tipos de evento, nenhum outro vale".

### Enum — conjunto fechado de valores

Um Enum (enumeração — conjunto finito e nomeado de valores)
declara explicitamente quais valores são válidos.
O Python passa a verificar isso:

```python
# Sem Enum
event_type = "tool_requst"   # typo passa despercebido

# Com Enum
event_type = EventType.TOOL_REQUEST   # autocompletar, sem typo possível
```

Se você digitar `EventType.TOOL_REQUST`, o editor sublinha
imediatamente. O Python, quando o código roda, dá `AttributeError`
na hora — não passa silenciosamente como string.

### @dataclass — estrutura com campos definidos

Um dataclass (classe de dados — classe Python cuja função principal
é armazenar dados estruturados) substitui dicionários soltos por
estruturas com campos nomeados e tipados.

```python
# Sem dataclass — dicionário
event = {"type": "content", "paylod": {"text": "..."}}
#                              ↑ typo — KeyError só no runtime

# Com dataclass
event = Event(type=EventType.CONTENT, payload={"text": "..."})
event.payload   # autocompletar; erro imediato se campo não existe
```

O `@dataclass` é um decorator (você verá decorators em detalhe
no Capítulo 8). Por ora, basta saber que ele gera automaticamente
o `__init__`, o `__repr__` e o `__eq__` da classe — você só declara
os campos.

### Type hints — documentação que o Python lê

Type hints (anotações de tipo) são marcações que indicam o tipo
esperado de uma variável, parâmetro ou retorno de função:

```python
def rodar(prompt: str) -> list[Event]:
#               ↑          ↑
#         parâmetro    retorno
```

Type hints não impedem o Python de aceitar tipos errados —
mas seu editor (VS Code com Pylance) sublinha imediatamente
se você passar `42` onde espera `str`. E ferramentas como `mypy`
verificam o código inteiro antes mesmo de rodar.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta do capítulo:

```powershell
New-Item -ItemType Directory -Force -Path cap05
```

No VS Code, crie `cap05.ipynb` dentro de `cap05`
e selecione o kernel `Python (agente)`.

---

## As células do notebook

### Célula 1 — Imports

```python
import os
from dataclasses import dataclass, field
from enum import Enum
from google import genai
from google.genai import types

client = genai.Client()
```

---

### Célula 2 — O Enum dos tipos de evento

`EventType` é um Enum que herda de `str`. Isso permite usar os valores
como strings normais (`EventType.CONTENT == "content"` é verdadeiro)
mas com a segurança do Enum: autocompletar, validação e erro
imediato em caso de typo.

```python
class EventType(str, Enum):
    CONTENT      = "content"
    TOOL_REQUEST = "tool_request"
    TOOL_RESULT  = "tool_result"
    FINISHED     = "finished"
    ERROR        = "error"

print(EventType.TOOL_REQUEST)
print(EventType.TOOL_REQUEST.value)
print(EventType.TOOL_REQUEST == "tool_request")
```

---

### Célula 3 — Tente quebrar o Enum

Tente acessar um valor que não existe. O Python detecta na hora,
ao contrário de uma string solta que passaria silenciosamente.

```python
try:
    e = EventType.TOOL_REQUST
except AttributeError as err:
    print(f"Erro detectado imediatamente: {err}")
```

---

### Célula 4 — Os dataclasses

`@dataclass` gera automaticamente o construtor, a representação
em texto e a comparação por valor.

Observe `field(default_factory=dict)` no campo `payload`.
Sem ele, se você escrevesse `payload: dict = {}`, todos os Events
criados sem payload compartilhariam o **mesmo** dicionário —
modificar um afetaria todos. Esse é o bug clássico do "mutable default".

```python
@dataclass
class ToolCall:
    id:   str
    name: str
    args: dict


@dataclass
class Event:
    type:    EventType
    payload: dict = field(default_factory=dict)


tc = ToolCall(id="call_1", name="list_dir", args={"path": "."})
ev = Event(type=EventType.TOOL_REQUEST, payload={"name": "list_dir"})

print(tc)
print(ev)
print(f"tc.name = {tc.name}")
print(f"ev.type = {ev.type}")
```

---

### Célula 5 — Demonstrando o bug do mutable default

```python
@dataclass
class EventBugado:
    type:    EventType
    payload: dict = field(default_factory=lambda: {})


@dataclass
class EventErrado:
    type:    EventType
    payload: dict = None


a = EventErrado(type=EventType.FINISHED)
b = EventErrado(type=EventType.FINISHED)

print(f"a.payload = {a.payload}")
print(f"b.payload = {b.payload}")
print(f"a é b? {a.payload is b.payload}")
```

Sem `default_factory`, dataclasses não aceitam `dict = {}`
como default — o Python protege contra esse bug.
Por isso usamos `field(default_factory=dict)`.

---

### Célula 6 — Tools e schema (como no Capítulo 4)

```python
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


TOOLS_FN = {
    "list_dir":  list_dir,
    "read_file": read_file,
}

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

### Célula 7 — O loop agora produz Events tipados

Em vez de retornar uma lista bruta de dicionários ou imprimir direto,
o loop agora retorna uma lista de `Event` — objetos tipados
que o consumidor pode inspecionar com segurança.

Observe os type hints na assinatura da função:
`-> list[Event]` deixa explícito o que sai. Seu editor passa a
oferecer autocompletar quando você itera sobre o retorno.

```python
def rodar(prompt: str) -> list[Event]:
    historico = [
        types.Content(role="user", parts=[types.Part(text=prompt)])
    ]
    events: list[Event] = []

    while True:
        response = client.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        tool_calls: list[ToolCall] = []

        for part in parts:
            if part.text:
                events.append(Event(
                    type=EventType.CONTENT,
                    payload={"text": part.text}
                ))

            elif part.function_call:
                fc = part.function_call
                call = ToolCall(
                    id=fc.name,
                    name=fc.name,
                    args=dict(fc.args) if fc.args else {}
                )
                tool_calls.append(call)
                events.append(Event(
                    type=EventType.TOOL_REQUEST,
                    payload={"name": call.name, "args": call.args}
                ))

        if not tool_calls:
            events.append(Event(type=EventType.FINISHED))
            return events

        tool_result_parts = []
        for call in tool_calls:
            fn        = TOOLS_FN.get(call.name)
            resultado = fn(**call.args) if fn else f"Tool '{call.name}' não encontrada."

            events.append(Event(
                type=EventType.TOOL_RESULT,
                payload={"name": call.name, "output": resultado}
            ))

            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=call.name, response={"result": resultado}
                ))
            )

        historico.append(types.Content(role="tool", parts=tool_result_parts))
```

---

### Célula 8 — Consumindo os events tipados

O consumidor faz pattern matching no campo `type` — usando o Enum,
não strings literais. Se você escrever `EventType.TOOL_REQUST`,
o editor avisa. Se um novo tipo de evento for adicionado ao Enum
e você esquecer de tratá-lo, dá pra detectar com ferramentas
de análise estática.

```python
events = rodar("Liste os arquivos desta pasta.")

for ev in events:
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
        print(f"✗ {ev.payload.get('message')}")
```

---

### Célula 9 — Inspecionando a lista de events

```python
events = rodar("Liste arquivos e leia o primeiro .py")

print(f"Total: {len(events)} eventos\n")
for i, ev in enumerate(events):
    print(f"[{i}] {ev.type.value}: {str(ev.payload)[:60]}")
```

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| `Enum` | conjunto fechado de valores nomeados | elimina typo silencioso em strings |
| `class EventType(str, Enum)` | Enum que herda de str | comparável com strings normais |
| `@dataclass` | gera __init__ e __repr__ automaticamente | substitui dicionários soltos por estrutura |
| `field(default_factory=dict)` | factory para valor default mutável | evita o bug do mutable default |
| type hints | anotações de tipo | documentação que o editor verifica |
| `list[Event]` | tipo genérico | lista especificamente de Events |
| pattern matching | comparação por tipo | substitui strings espalhadas |

---

## A próxima dor

O loop está mais sólido — tipos protegem contra erros,
estrutura clara separa responsabilidades.

Mas o `rodar()` retorna uma **lista** de Events.
Isso significa que ele só termina quando o loop acaba —
todos os Events ficam armazenados até o fim, e só então
são entregues ao consumidor.

Para um script de terminal, ok. Mas tente imaginar:
você quer mostrar no chat o texto chegando palavra por palavra,
em tempo real. Ou um servidor servindo múltiplos usuários
ao mesmo tempo. Com lista, nada disso é possível —
o Python fica bloqueado esperando o Gemini responder,
e o consumidor espera todos os Events de uma vez.

Essa limitação leva ao Capítulo 6: async/await.

---

## Arquivo final do capítulo

Quando terminar o notebook, crie o arquivo `cap05\typed_loop.py`:

```python
import os
from dataclasses import dataclass, field
from enum import Enum
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


def rodar(prompt: str) -> list[Event]:
    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]
    events: list[Event] = []

    while True:
        response = client.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        tool_calls: list[ToolCall] = []

        for part in parts:
            if part.text:
                events.append(Event(type=EventType.CONTENT,
                                    payload={"text": part.text}))
            elif part.function_call:
                fc = part.function_call
                call = ToolCall(id=fc.name, name=fc.name,
                                args=dict(fc.args) if fc.args else {})
                tool_calls.append(call)
                events.append(Event(type=EventType.TOOL_REQUEST,
                                    payload={"name": call.name, "args": call.args}))

        if not tool_calls:
            events.append(Event(type=EventType.FINISHED))
            return events

        tool_result_parts = []
        for call in tool_calls:
            fn        = TOOLS_FN.get(call.name)
            resultado = fn(**call.args) if fn else f"Tool '{call.name}' não encontrada."

            events.append(Event(type=EventType.TOOL_RESULT,
                                payload={"name": call.name, "output": resultado}))

            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=call.name, response={"result": resultado}
                ))
            )

        historico.append(types.Content(role="tool", parts=tool_result_parts))


if __name__ == "__main__":
    prompt = input("Você: ").strip() or "Liste os arquivos desta pasta."

    for ev in rodar(prompt):
        if ev.type == EventType.CONTENT:
            print(f"Gemini: {ev.payload['text']}")
        elif ev.type == EventType.TOOL_REQUEST:
            print(f"→ {ev.payload['name']}({ev.payload['args']})")
        elif ev.type == EventType.TOOL_RESULT:
            out = ev.payload['output']
            print(f"← {out[:80]}{'...' if len(out) > 80 else ''}")
        elif ev.type == EventType.FINISHED:
            print("✓ concluído")
```


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap04-loop-automatico">← Cap 4 — Loop</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap06-async-await">Cap 6 — async →</a></div>
</div>
