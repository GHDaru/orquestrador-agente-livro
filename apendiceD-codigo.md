---
title: "Apêndice D — Código completo por fase"
layout: default
---

# Apêndice D — Código completo por fase

Arquivos finais de cada capítulo, limpos, sem comentários
pedagógicos. Para copiar e rodar direto, sem precisar voltar
às explicações.

---

## Como usar

Cada capítulo tem seu arquivo final descrito no próprio capítulo,
na seção "Arquivo final do capítulo". Este apêndice apenas
agrupa todos para consulta rápida.

**Os arquivos estão localizados em:**

```
agente/
├── cap01/call.py
├── cap02/chat.py
├── cap03/tool_manual.py
├── cap04/loop.py
├── cap05/typed_loop.py
├── cap06/async_loop.py
├── cap07/generator_loop.py
├── cap08/loop.py          ← o loop ReAct completo
├── cap09/server.py        ← servidor FastAPI + SSE
├── cap10/index.html       ← chat no browser
├── cap11/session_loop.py  ← com persistência
├── cap12/hooks_loop.py    ← com hooks
├── cap13/subagents_loop.py ← com sub-agentes
└── cap14/                  ← MCP + Skills
```

---

## O loop canônico (Cap 8)

Este é o "loop de referência" que serve de base para tudo
a partir do Capítulo 9. Outras versões nos capítulos seguintes
são extensões desta:

```python
# cap08/loop.py
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
    parameters={"properties": {"path": {"description": "Caminho do diretório."}},
                "required": ["path"]}
)
def list_dir(path: str) -> str:
    try:
        return "\n".join(sorted(os.listdir(path)))
    except Exception as e:
        return f"Erro: {e}"


@tool(
    name="read_file",
    description="Lê o conteúdo de um arquivo de texto.",
    parameters={"properties": {"path": {"description": "Caminho do arquivo."}},
                "required": ["path"]}
)
def read_file(path: str) -> str:
    try:
        with open(path, "r", encoding="utf-8") as f:
            return f.read()
    except Exception as e:
        return f"Erro: {e}"


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

## Servidor com SSE (Cap 9)

```python
# cap09/server.py
import json
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from loop import rodar

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)


class RunRequest(BaseModel):
    prompt: str


@app.post("/stream")
async def stream(request: RunRequest):
    async def event_stream():
        async for ev in rodar(request.prompt):
            data = {"type": ev.type.value, "payload": ev.payload}
            yield f"data: {json.dumps(data)}\n\n"

    return StreamingResponse(event_stream(), media_type="text/event-stream")


@app.get("/health")
async def health():
    return {"status": "ok", "version": "0.1.0"}
```

---

## Frontend (Cap 10)

Veja `cap10/index.html` no Capítulo 10. É um arquivo único
com HTML, CSS e JavaScript inline. Não há versão "limpa" —
o arquivo já é apenas código de produção.

---

## Versões evolutivas

A partir do Cap 11, o loop ganha adições:

- **Cap 11** (`session_loop.py`): adiciona `Session`, `SessionPort`,
  `FileSessionStore`, `LoopDetectionService`. O loop principal
  agora recebe `session_id` e persiste estado.

- **Cap 12** (`hooks_loop.py`): adiciona `HookEvent`, `HookRegistry`,
  `MessageBus`. O loop emite eventos em pontos do lifecycle.

- **Cap 13** (`subagents_loop.py`): adiciona `LocalAgentDefinition`,
  `LocalAgentExecutor`, `AgentScheduler`, `complete_task`,
  `delegate_to_agent`. O orquestrador pode delegar a sub-agentes.

- **Cap 14**: adiciona `BaseTool`, `McpClientAdapter`,
  `McpToolProxy`, `SkillRegistry`, `activate_skill`. Tools
  externas e conhecimento on-demand.

Cada um inclui o código completo no próprio capítulo.

---

## pyproject.toml final

Para reproduzir o ambiente completo:

```toml
[project]
name = "agente"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "google-genai>=1.0.0",
    "fastapi>=0.115.0",
    "uvicorn>=0.30.0",
    "jupyter>=1.0.0",
    "ipykernel>=6.0.0",
    "mcp>=0.5.0",
]
```

Comando para criar do zero:

```powershell
uv init agente
cd agente
uv add google-genai fastapi uvicorn jupyter ipykernel mcp
```


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="apendiceC-glossario">← Apêndice C</a></div>
  <div><a href="index">Sumário</a></div>
  <div><span></span></div>
</div>
