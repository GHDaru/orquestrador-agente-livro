---
title: "Capítulo 11 — Sessão: o agente que lembra"
layout: default
---

# Capítulo 11 — Sessão: o agente que lembra

---

## A dor

O sistema do Capítulo 10 funciona — mas se você reiniciar
o servidor, perde tudo. Cada conversa começa do zero.

Pior: se você abrir o chat hoje, conversar, fechar o browser,
e abrir amanhã, é como nunca ter conversado. Nenhuma continuidade.

Para um chat real, isso não funciona. O usuário espera que
o agente lembre — pelo menos da conversa anterior, idealmente
de tudo que aconteceu antes.

A solução tem duas camadas:

1. **Identificar conversas**: cada chat tem um `session_id`
2. **Persistir o estado**: salvar o histórico em disco

E aqui surge um problema arquitetural: como salvamos o estado?
Em arquivos JSON? Em banco de dados? Em Redis? Em memória?
A resposta não é "uma das opções" — é "depende do ambiente,
e o loop não deveria precisar saber qual".

Esta é a primeira vez no livro que precisamos de um conceito
do DDD (Domain-Driven Design — Design Orientado ao Domínio).

---

## Por que isto é DDD — Port e Adapter

DDD propõe separar **o que** seu domínio faz de **como**
a infraestrutura faz. No nosso caso:

- **O que**: salvar e recuperar sessões
- **Como**: arquivos JSON, banco SQL, Redis, etc.

A separação se concretiza em dois conceitos:

### Port (porta)

Um Port é uma **interface** (contrato abstrato) que descreve
**o que** o domínio precisa, sem dizer **como**:

```python
class SessionPort:
    async def save(self, session_id: str, history: list) -> None: ...
    async def load(self, session_id: str) -> list | None: ...
```

O loop conhece apenas a Port. Ele chama `port.save(...)` sem
saber se isso vai escrever em arquivo, em banco, em rede.

### Adapter (adaptador)

Um Adapter é uma **implementação concreta** da Port:

```python
class FileSessionStore(SessionPort):
    async def save(self, session_id, history):
        # escreve em ~/.agent/sessions/{session_id}.json
        ...

    async def load(self, session_id):
        # lê de ~/.agent/sessions/{session_id}.json
        ...
```

Você pode ter múltiplos adapters: `FileSessionStore`,
`SqliteSessionStore`, `RedisSessionStore`. O loop não muda
quando você troca um pelo outro.

### Por que isso importa

1. **Testes**: você cria um `MemorySessionStore` para testes
   rodarem rápido, sem tocar em disco.
2. **Evolução**: começa com arquivos, migra para banco quando
   o sistema crescer, sem reescrever o loop.
3. **Clareza**: o código do loop não fica poluído com detalhes
   de I/O (Input/Output — entrada e saída de dados).

Este padrão tem nomes equivalentes em outros contextos:
*Hexagonal Architecture*, *Dependency Inversion*, *Clean Architecture*.
Todos descrevem a mesma ideia: domínio define o contrato,
infraestrutura implementa.

---

## A entidade Session

Antes da Port, definimos o que é uma "sessão" no nosso domínio.
Isso é uma **Entidade** em DDD — algo identificado por um ID
e com estado próprio:

```python
@dataclass
class Session:
    id:         str
    history:    list[types.Content]
    created_at: str
    updated_at: str
```

Quatro campos. Identidade, conteúdo, e timestamps.

---

## AGENT.md — contexto persistente do projeto

Outro conceito que entra aqui: o arquivo `AGENT.md` na raiz do projeto.
É um arquivo de texto livre onde você escreve instruções persistentes
para o agente — como um briefing que ele lê antes de cada conversa.

```markdown
# Sobre este projeto

Estou construindo um sistema de notas em Python.
Use sempre type hints. Prefira f-strings.
Quando criar funções, escreva docstrings curtas.
```

O loop carrega esse arquivo e injeta no `system_instruction`.
O agente passa a "saber" o contexto sem você repetir a cada conversa.

---

## LoopDetectionService — quando o agente trava

Antes de salvar tudo, vale adicionar uma proteção: detectar quando
o agente entra em loop. Por exemplo, chamando a mesma ferramenta
com os mesmos argumentos três vezes seguidas sem progresso.

```python
class LoopDetectionService:
    def __init__(self, max_repetitions: int = 3):
        self.max = max_repetitions
        self.recent_calls: list[str] = []

    def check(self, name: str, args: dict) -> bool:
        signature = f"{name}:{json.dumps(args, sort_keys=True)}"
        self.recent_calls.append(signature)
        self.recent_calls = self.recent_calls[-10:]

        return self.recent_calls.count(signature) >= self.max
```

Se a mesma chamada aparece 3 vezes nas últimas 10, o loop encerra
com um Event de erro indicando o problema.

Em DDD, isso é um **Domain Service** — uma lógica de negócio
que não pertence a uma entidade específica, mas opera sobre
o domínio.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta:

```powershell
New-Item -ItemType Directory -Force -Path cap11
Copy-Item cap08\loop.py cap11\loop_base.py
```

No VS Code, crie `cap11.ipynb` dentro de `cap11`.

---

## As células do notebook

### Célula 1 — A entidade Session

```python
import json
import os
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path
from google.genai import types


@dataclass
class Session:
    id:         str
    history:    list = field(default_factory=list)
    created_at: str  = field(default_factory=lambda: datetime.now().isoformat())
    updated_at: str  = field(default_factory=lambda: datetime.now().isoformat())


s = Session(id="conversa-001")
print(s)
```

---

### Célula 2 — A Port (interface)

```python
from abc import ABC, abstractmethod


class SessionPort(ABC):
    @abstractmethod
    async def save(self, session: Session) -> None: ...

    @abstractmethod
    async def load(self, session_id: str) -> Session | None: ...

    @abstractmethod
    async def list_all(self) -> list[str]: ...


print("Port definida.")
print(f"Métodos: {[m for m in dir(SessionPort) if not m.startswith('_')]}")
```

`ABC` (Abstract Base Class — Classe Base Abstrata) e `@abstractmethod`
forçam que qualquer subclasse implemente os métodos.
Se você esquecer um, o Python recusa instanciar a classe.

---

### Célula 3 — O Adapter para arquivos JSON

```python
class FileSessionStore(SessionPort):
    def __init__(self, base_dir: Path = Path.home() / ".agent" / "sessions"):
        self.base_dir = base_dir
        self.base_dir.mkdir(parents=True, exist_ok=True)

    def _path(self, session_id: str) -> Path:
        return self.base_dir / f"{session_id}.json"

    def _serialize_history(self, history: list) -> list:
        result = []
        for content in history:
            parts = []
            for part in content.parts:
                if part.text:
                    parts.append({"text": part.text})
                elif hasattr(part, 'function_call') and part.function_call:
                    parts.append({
                        "function_call": {
                            "name": part.function_call.name,
                            "args": dict(part.function_call.args) if part.function_call.args else {}
                        }
                    })
                elif hasattr(part, 'function_response') and part.function_response:
                    parts.append({
                        "function_response": {
                            "name":     part.function_response.name,
                            "response": dict(part.function_response.response)
                        }
                    })
            result.append({"role": content.role, "parts": parts})
        return result

    def _deserialize_history(self, data: list) -> list:
        result = []
        for item in data:
            parts = []
            for p in item["parts"]:
                if "text" in p:
                    parts.append(types.Part(text=p["text"]))
                elif "function_call" in p:
                    fc = p["function_call"]
                    parts.append(types.Part(function_call=types.FunctionCall(
                        name=fc["name"], args=fc["args"]
                    )))
                elif "function_response" in p:
                    fr = p["function_response"]
                    parts.append(types.Part(function_response=types.FunctionResponse(
                        name=fr["name"], response=fr["response"]
                    )))
            result.append(types.Content(role=item["role"], parts=parts))
        return result

    async def save(self, session: Session) -> None:
        data = {
            "id":         session.id,
            "created_at": session.created_at,
            "updated_at": datetime.now().isoformat(),
            "history":    self._serialize_history(session.history),
        }
        self._path(session.id).write_text(
            json.dumps(data, indent=2, ensure_ascii=False),
            encoding="utf-8"
        )

    async def load(self, session_id: str) -> Session | None:
        path = self._path(session_id)
        if not path.exists():
            return None
        data = json.loads(path.read_text(encoding="utf-8"))
        return Session(
            id=data["id"],
            history=self._deserialize_history(data["history"]),
            created_at=data["created_at"],
            updated_at=data["updated_at"],
        )

    async def list_all(self) -> list[str]:
        return [p.stem for p in self.base_dir.glob("*.json")]


store = FileSessionStore()
print(f"Sessions armazenadas em: {store.base_dir}")
```

---

### Célula 4 — Testando save e load

```python
s = Session(
    id="teste-001",
    history=[
        types.Content(role="user",  parts=[types.Part(text="Olá")]),
        types.Content(role="model", parts=[types.Part(text="Oi! Em que posso ajudar?")]),
    ]
)

await store.save(s)
print(f"Salvo: {store._path(s.id)}")

s2 = await store.load("teste-001")
print(f"Carregado: id={s2.id}")
print(f"Mensagens: {len(s2.history)}")
print(f"Primeira: {s2.history[0].parts[0].text}")
```

---

### Célula 5 — Listando sessões

```python
sessoes = await store.list_all()
print(f"Total de sessões: {len(sessoes)}")
for sid in sessoes:
    print(f"  {sid}")
```

---

### Célula 6 — Carregando AGENT.md

```python
def load_agent_md(project_root: Path = Path.cwd()) -> str:
    agent_md = project_root / "AGENT.md"
    if not agent_md.exists():
        return ""
    return agent_md.read_text(encoding="utf-8")


contexto = load_agent_md()
if contexto:
    print(f"AGENT.md carregado ({len(contexto)} chars):")
    print(contexto[:200])
else:
    print("AGENT.md não encontrado ou vazio")
```

---

### Célula 7 — LoopDetectionService

```python
class LoopDetectionService:
    def __init__(self, max_repetitions: int = 3, window: int = 10):
        self.max    = max_repetitions
        self.window = window
        self.recent_calls: list[str] = []

    def check(self, name: str, args: dict) -> bool:
        signature = f"{name}:{json.dumps(args, sort_keys=True)}"
        self.recent_calls.append(signature)
        self.recent_calls = self.recent_calls[-self.window:]
        return self.recent_calls.count(signature) >= self.max


detector = LoopDetectionService(max_repetitions=3)

for i in range(5):
    em_loop = detector.check("list_dir", {"path": "."})
    print(f"Chamada {i+1}: em loop? {em_loop}")
```

A terceira chamada idêntica em sequência dispara a detecção.

---

### Célula 8 — Loop ReAct com sessão

A função `rodar` agora recebe `session_id`. Ela carrega o histórico
existente (se houver), continua a conversa, salva no fim.

O `system_instruction` agora combina o prompt padrão com
o conteúdo do `AGENT.md`.

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import AsyncGenerator, Callable
from google import genai

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
    def decorator(fn):
        _TOOLS[name] = fn
        _SCHEMAS.append(
            types.FunctionDeclaration(
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
        )
        return fn
    return decorator


@tool(name="list_dir", description="Lista arquivos.",
      parameters={"properties": {"path": {"description": "Caminho."}}, "required": ["path"]})
def list_dir(path: str) -> str:
    try:
        return "\n".join(sorted(os.listdir(path)))
    except Exception as e:
        return f"Erro: {e}"


@tool(name="read_file", description="Lê arquivo.",
      parameters={"properties": {"path": {"description": "Caminho."}}, "required": ["path"]})
def read_file(path: str) -> str:
    try:
        with open(path, "r", encoding="utf-8") as f:
            return f.read()
    except Exception as e:
        return f"Erro: {e}"


async def rodar_com_sessao(
    prompt: str,
    session_id: str,
    session_store: SessionPort,
) -> AsyncGenerator[Event, None]:

    session = await session_store.load(session_id)
    if session is None:
        session = Session(id=session_id)

    session.history.append(
        types.Content(role="user", parts=[types.Part(text=prompt)])
    )

    agent_context = load_agent_md()
    system_prompt = (
        "Você é um assistente útil. Responda em português."
        + (f"\n\nContexto do projeto:\n{agent_context}" if agent_context else "")
    )

    detector = LoopDetectionService()

    for turno in range(MAX_TURNS):
        response = await client.aio.models.generate_content(
            model="gemini-2.5-flash",
            contents=session.history,
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
        session.history.append(response.candidates[0].content)

        tool_calls = []

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
            await session_store.save(session)
            return

        tool_result_parts = []
        for call in tool_calls:
            if detector.check(call.name, call.args):
                yield Event(
                    type=EventType.ERROR,
                    payload={"message": f"Loop detectado em {call.name}. Encerrando."}
                )
                await session_store.save(session)
                return

            fn        = _TOOLS.get(call.name)
            resultado = fn(**call.args) if fn else f"Tool '{call.name}' não existe."
            yield Event(type=EventType.TOOL_RESULT,
                        payload={"name": call.name, "output": resultado})
            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=call.name, response={"result": resultado}
                ))
            )

        session.history.append(types.Content(role="tool", parts=tool_result_parts))

    yield Event(type=EventType.ERROR,
                payload={"message": f"Limite de {MAX_TURNS} turnos."})
    await session_store.save(session)
```

---

### Célula 9 — Conversa em duas execuções

A primeira execução conta algo. A segunda, com o mesmo `session_id`,
deve lembrar.

```python
SESSION_ID = "demo-001"

async for ev in rodar_com_sessao("Meu nome é João.", SESSION_ID, store):
    if ev.type == EventType.CONTENT:
        print(f"Gemini: {ev.payload['text']}")
    elif ev.type == EventType.FINISHED:
        print("✓\n")
```

```python
async for ev in rodar_com_sessao("Qual é o meu nome?", SESSION_ID, store):
    if ev.type == EventType.CONTENT:
        print(f"Gemini: {ev.payload['text']}")
    elif ev.type == EventType.FINISHED:
        print("✓")
```

O modelo deve responder "João". A memória ficou em disco
entre as duas execuções.

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| Entidade (DDD) | objeto com identidade e estado | `Session` tem `id` único |
| Port (DDD) | interface abstrata do domínio | `SessionPort` define contrato |
| Adapter (DDD) | implementação concreta da Port | `FileSessionStore` em JSON |
| Domain Service (DDD) | lógica que não pertence a uma entidade | `LoopDetectionService` |
| `ABC` + `@abstractmethod` | contratos formais em Python | força implementação |
| serialização | converter objeto em string | salvar em disco |
| `pathlib.Path` | manipulação de caminhos | mais limpo que `os.path` |
| `AGENT.md` | briefing persistente do projeto | contexto que o agente sempre vê |

---

## A próxima dor

Você tem sessão. O agente lembra entre execuções.

Mas o `rodar_com_sessao` está ficando complexo. Imagine adicionar:

- Log de auditoria (gravar cada tool call em arquivo)
- Aprovação de tools destrutivas pelo usuário
- Injetar `git status` no prompt antes de cada turno
- Telemetria (medir tempo de cada chamada)

Tudo isso, hoje, exige mexer no código do loop. E cada vez que mexe,
arrisca quebrar o que funcionava. O loop fica responsável por
coisas que não são "rodar o agente" — são extensões.

O Capítulo 12 resolve isso com **hooks**: pontos de extensão
declarados no loop, onde código externo pode se plugar
sem alterar o núcleo.

---

## Arquivo final do capítulo

Junte as células no arquivo `cap11\session_loop.py`.
A versão final tem todas as classes e funções deste capítulo.


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap10-chat-browser">← Cap 10 — Chat</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap12-hooks">Cap 12 — Hooks →</a></div>
</div>
