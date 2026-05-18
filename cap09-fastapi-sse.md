---
title: "Capítulo 9 — FastAPI e SSE: o loop vira servidor"
layout: default
---

# Capítulo 9 — FastAPI e SSE: o loop vira servidor

---

## A dor

O loop do Capítulo 8 funciona — mas só no terminal.
Para qualquer outra interface (browser, app móvel, outro programa)
usar o agente, ele precisa estar disponível pela rede.

A solução comum é HTTP: o frontend faz uma requisição,
o backend roda o loop, devolve a resposta.

```
POST /run { "prompt": "liste arquivos" }
        ↓
   [loop roda por 15 segundos]
        ↓
{ "events": [Event, Event, ...] }
```

Mas há um problema: para tarefas longas, o frontend fica
15 segundos sem feedback nenhum. O usuário vê uma tela
parada e pensa que travou.

Para resolver isso, em vez de devolver tudo no final,
o servidor envia cada Event **conforme acontece**.
A tecnologia para isso é SSE (Server-Sent Events —
Eventos Enviados pelo Servidor).

---

## O que é HTTP, FastAPI e SSE

### HTTP em uma frase

HTTP (HyperText Transfer Protocol) é o protocolo onde o cliente
faz uma requisição (`GET /pagina` ou `POST /api/dados`)
e o servidor devolve uma resposta. Cada interação é independente.

No modelo HTTP tradicional, a resposta vem **inteira de uma vez**:
o cliente espera, o servidor processa, manda tudo. Conexão fecha.

### FastAPI

FastAPI é um framework Python (estrutura de desenvolvimento)
para criar servidores HTTP. Comparado ao Flask, mais antigo,
o FastAPI tem três vantagens importantes para nosso caso:

1. **Suporte nativo a async** — combina com nosso loop assíncrono
2. **Tipagem automática** — usa type hints para validação
3. **Documentação automática** — gera a doc interativa da API

Em FastAPI, um endpoint é uma função:

```python
from fastapi import FastAPI

app = FastAPI()

@app.post("/run")
async def run_endpoint(request: RunRequest):
    return {"resultado": "..."}
```

### SSE em uma frase

SSE é um modo especial de HTTP onde a conexão fica aberta
e o servidor pode enviar várias mensagens ao longo do tempo,
em vez de uma resposta única no final.

```
Cliente: GET /stream
Servidor: data: evento 1
          
          data: evento 2
          
          data: evento 3
          
          [conexão fecha quando o servidor decide]
```

Cada mensagem é uma linha começando com `data:` seguida do conteúdo
e duas quebras de linha. O formato é simples e funciona em qualquer
browser sem bibliotecas externas.

SSE é unidirecional (só servidor → cliente). Para comunicação
nos dois sentidos, existe WebSocket — mas para streaming de
eventos do agente, SSE é mais simples e suficiente.

### StreamingResponse

No FastAPI, SSE é implementado pela classe `StreamingResponse`,
que recebe um async generator e envia cada item conforme produzido:

```python
from fastapi.responses import StreamingResponse

async def gerador_de_eventos():
    yield "data: evento 1\n\n"
    await asyncio.sleep(1)
    yield "data: evento 2\n\n"

@app.get("/stream")
async def stream_endpoint():
    return StreamingResponse(
        gerador_de_eventos(),
        media_type="text/event-stream"
    )
```

Note como combina perfeitamente com nosso `rodar()` que já é
um AsyncGenerator. É só formatar cada Event como `data: {json}\n\n`.

---

## CORS — por que o browser bloqueia

CORS (Cross-Origin Resource Sharing — Compartilhamento de Recursos
entre Origens) é um mecanismo de segurança dos browsers.
Por padrão, uma página servida em `http://localhost:5500`
**não pode** fazer requisições para `http://localhost:8000`.
O browser bloqueia automaticamente.

Para desenvolvimento, o backend precisa explicitamente liberar
quais origens podem acessá-lo, via `CORSMiddleware`:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],   # libera tudo (só para dev!)
    allow_methods=["*"],
    allow_headers=["*"],
)
```

Em produção, `allow_origins` deve listar os domínios específicos
que podem acessar o servidor — nunca `*`.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta do capítulo:

```powershell
New-Item -ItemType Directory -Force -Path cap09
```

Copie o `loop.py` do Capítulo 8 para esta pasta:

```powershell
Copy-Item cap08\loop.py cap09\loop.py
```

No VS Code, crie `cap09.ipynb` dentro de `cap09`.

---

## As células do notebook

### Célula 1 — Servidor mínimo, sem streaming

Antes de SSE, vamos fazer o servidor mais simples possível:
endpoint POST que roda o loop e devolve todos os Events de uma vez.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from loop import rodar, EventType

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)


class RunRequest(BaseModel):
    prompt: str


@app.post("/run")
async def run(request: RunRequest):
    events = []
    async for ev in rodar(request.prompt):
        events.append({
            "type":    ev.type.value,
            "payload": ev.payload,
        })
    return {"events": events}
```

Note `BaseModel` do Pydantic (biblioteca de validação de dados):
ele converte o JSON da requisição em um objeto Python com tipos
verificados. Se o cliente mandar algo sem `prompt`, FastAPI retorna
erro 422 (Unprocessable Entity) automaticamente.

---

### Célula 2 — Rodando o servidor

Para rodar o servidor, salve a célula 1 como `server.py` na pasta `cap09\`
e execute no PowerShell:

```powershell
uv run uvicorn server:app --reload --port 8000
```

`uvicorn` é o servidor ASGI (Asynchronous Server Gateway Interface —
interface para servidores web async em Python). `--reload` faz o servidor
reiniciar quando você muda o código.

Você vai ver:

```
INFO:     Uvicorn running on http://127.0.0.1:8000
INFO:     Started reloader process
INFO:     Application startup complete.
```

---

### Célula 3 — Testando com requests

```python
import requests

resp = requests.post(
    "http://localhost:8000/run",
    json={"prompt": "Liste os arquivos desta pasta"}
)

data = resp.json()
print(f"Status: {resp.status_code}")
print(f"Total de eventos: {len(data['events'])}")
print()
for ev in data['events']:
    print(f"  {ev['type']}: {str(ev['payload'])[:60]}")
```

Você verá todos os eventos chegando de uma vez — depois de
o loop inteiro terminar. Isso confirma o problema:
o cliente fica esperando todo o tempo sem feedback.

---

### Célula 4 — A versão com SSE

Agora a versão que entrega eventos em tempo real.
A função `event_stream` é um async generator que formata
cada Event no protocolo SSE: `data: {json}\n\n`.

`StreamingResponse` consome esse generator e envia cada item
imediatamente ao cliente. O cliente vai recebendo os eventos
um a um, conforme o agente trabalha.

Substitua o conteúdo de `server.py` por isto:

```python
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
            data = {
                "type":    ev.type.value,
                "payload": ev.payload,
            }
            yield f"data: {json.dumps(data)}\n\n"

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream"
    )
```

Salve. O `--reload` do uvicorn vai recarregar automaticamente.

---

### Célula 5 — Testando SSE com requests (stream=True)

```python
import requests
import json

resp = requests.post(
    "http://localhost:8000/stream",
    json={"prompt": "Liste os arquivos e leia o primeiro .py"},
    stream=True,
)

import time
inicio = time.time()

for linha in resp.iter_lines(decode_unicode=True):
    if linha and linha.startswith("data: "):
        data = json.loads(linha[6:])
        elapsed = time.time() - inicio
        print(f"[{elapsed:.1f}s] {data['type']}: {str(data['payload'])[:60]}")
```

Compare com a célula 3: agora cada evento chega no momento
em que é produzido pelo loop. O cliente pode reagir em tempo real.

---

### Célula 6 — Testando direto com curl

No PowerShell (numa janela separada), você também pode testar
sem código Python:

```powershell
curl -N -X POST http://localhost:8000/stream `
  -H "Content-Type: application/json" `
  -d '{\"prompt\": \"Liste os arquivos\"}'
```

A flag `-N` desabilita o buffer e mostra cada linha conforme chega.
O acento grave (`) é a continuação de linha do PowerShell.

---

### Célula 7 — Health check

É boa prática ter um endpoint simples que confirma se o servidor
está vivo. Útil para monitoramento e debugging.

```python
@app.get("/health")
async def health():
    return {"status": "ok", "version": "0.1.0"}
```

Adicione ao `server.py`. Teste:

```python
import requests
print(requests.get("http://localhost:8000/health").json())
```

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| HTTP | protocolo cliente-servidor da internet | base de toda comunicação web |
| FastAPI | framework Python para servidores HTTP | suporte async nativo |
| `@app.post("/run")` | decorator que registra endpoint | converte função em rota HTTP |
| Pydantic / `BaseModel` | validação automática de JSON | type-safe nas requisições |
| `uvicorn` | servidor ASGI | roda aplicações FastAPI |
| SSE | streaming HTTP do servidor para o cliente | eventos em tempo real |
| `StreamingResponse` | envia generator como SSE | conecta nosso async generator ao HTTP |
| `text/event-stream` | media type do SSE | sinaliza ao browser que é stream |
| CORS | controle de origens permitidas | resolve bloqueio do browser |
| `CORSMiddleware` | libera origens específicas | necessário para frontend acessar |

---

## A próxima dor

O servidor funciona. Você pode testar com `requests` ou `curl`
e ver os eventos chegando em tempo real. Mas isso ainda é só
backend — o usuário não usa `curl` para conversar com um agente.

Falta o frontend: uma página HTML que conecta no `/stream`,
lê os eventos conforme chegam, e renderiza cada tipo
de forma diferente (texto fluindo, tool calls em destaque,
resultados em outra cor).

Esse é o Capítulo 10.

---

## Arquivo final do capítulo

Quando terminar o notebook, deixe `cap09\server.py` com:

```python
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
            data = {
                "type":    ev.type.value,
                "payload": ev.payload,
            }
            yield f"data: {json.dumps(data)}\n\n"

    return StreamingResponse(event_stream(), media_type="text/event-stream")


@app.get("/health")
async def health():
    return {"status": "ok", "version": "0.1.0"}
```

Rodar:

```powershell
cd cap09
uv run uvicorn server:app --reload --port 8000
```


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap08-decorator-tool">← Cap 8 — Decorator</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap10-chat-browser">Cap 10 — Chat →</a></div>
</div>
