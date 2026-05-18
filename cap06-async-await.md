---
title: "Capítulo 6 — Tempo real: async/await"
layout: default
---

# Capítulo 6 — Tempo real: async/await

---

## A dor

Rode o loop do Capítulo 5 com uma pergunta complexa:

```
Liste todos os arquivos .py desta pasta, leia cada um,
e me diga qual tem o maior número de funções definidas.
```

O modelo vai usar várias ferramentas em sequência.
Cada chamada ao Gemini demora entre 2 e 8 segundos.
Durante todo esse tempo, o Python fica **completamente parado**:

- Não pode atender outro usuário
- Não pode enviar dados parciais para o frontend
- Não pode fazer absolutamente nada além de esperar

Para um script de terminal, isso é aceitável.
Para um servidor web atendendo várias pessoas ao mesmo tempo,
**cada usuário bloqueia todos os outros**.

E pior: quando você quiser exibir o texto do modelo aparecendo
palavra por palavra no chat (como ChatGPT), com código síncrono
**isso é impossível**. O Python só termina a chamada quando
recebe a resposta inteira.

A solução tem um nome: programação assíncrona.

---

## O modelo mental: garçom vs cozinheiro

Antes do código, uma analogia.

Imagine dois funcionários de um restaurante.

### O cozinheiro síncrono

Recebe um pedido. Prepara o prato inteiro do começo ao fim.
Só pega o próximo pedido quando termina o atual.
Se um prato leva 30 minutos, todos os outros clientes esperam.

```
[12:00] Recebe pedido 1
[12:30] Termina pedido 1
[12:30] Recebe pedido 2
[13:00] Termina pedido 2
```

Dois clientes em uma hora. Simples, mas devagar quando há fila.

### O garçom assíncrono

Recebe um pedido. Anota e leva para a cozinha.
Enquanto a cozinha prepara, ele atende outros clientes.
Quando um pedido fica pronto, ele entrega.

```
[12:00] Anota pedido 1, leva à cozinha
[12:01] Anota pedido 2, leva à cozinha
[12:02] Anota pedido 3, leva à cozinha
[12:30] Entrega pedido 1 (a cozinha avisou)
[12:31] Entrega pedido 2
[12:32] Entrega pedido 3
```

Três clientes em pouco mais de meia hora — porque o tempo de cozimento
é compartilhado. O garçom não fica parado esperando.

### O que isso tem a ver com Python

O Python "cozinheiro" é o código síncrono normal.
Cada `client.generate_content()` é como cozinhar um prato:
o Python para tudo, espera a resposta, depois continua.

O Python "garçom" é `async/await`. Quando o código pede uma
operação demorada (`await ...`), ele "anota o pedido" e libera
o processo para fazer outras coisas. Quando a resposta chega,
ele retoma de onde parou.

**Tempo de espera externa** (rede, disco, banco) é exatamente
quando `async/await` brilha. **Cálculo interno** (processar dados,
fazer contas) não — para isso, o cozinheiro é igual ou melhor.

---

## Os três conceitos

### async def — uma função que pode pausar

Uma função declarada com `async def` é chamada de **coroutine**
(corrotina — função que pode pausar a execução e retomá-la depois).

```python
async def chamar_modelo(prompt):
    response = await client.aio.models.generate_content(...)
    return response.text
```

A diferença mais estranha: **chamar uma coroutine não a executa**.
Em código normal:

```python
def soma(a, b):
    return a + b

resultado = soma(2, 3)   # roda a função e retorna 5
```

Com `async`:

```python
async def soma(a, b):
    return a + b

resultado = soma(2, 3)   # NÃO roda — retorna um "coroutine object"
print(resultado)          # <coroutine object soma at 0x...>
```

Para realmente executar uma coroutine, você precisa de duas coisas:
um event loop (loop de eventos — o coordenador) e o operador `await`.

### await — espera sem bloquear

`await` faz três coisas em uma:

1. Executa a coroutine
2. Pausa a função atual até o resultado chegar
3. Libera o event loop para fazer outras coisas enquanto espera

```python
async def main():
    resultado = await soma(2, 3)   # agora sim executa, retorna 5
    print(resultado)
```

`await` só pode ser usado **dentro de uma função async**.
Em código síncrono normal, dá erro de sintaxe.

### asyncio.run — o ponto de entrada

O event loop é o coordenador: ele rastreia todas as coroutines
pendentes, decide qual rodar quando, e gerencia os `await`s.

`asyncio.run()` cria o event loop, executa a coroutine principal,
e fecha o loop quando ela termina:

```python
import asyncio

async def main():
    print("começou")
    await asyncio.sleep(1)   # pausa por 1 segundo sem bloquear
    print("terminou")

asyncio.run(main())
```

Você só usa `asyncio.run()` uma vez, no ponto de entrada do programa.
Tudo dentro dele pode usar `await`.

---

## Um detalhe importante: a versão async do SDK

O SDK do Gemini tem duas versões da API:

```python
# Síncrono — bloqueia o Python
client.models.generate_content(...)

# Assíncrono — libera o Python enquanto espera
await client.aio.models.generate_content(...)
```

O caminho `client.aio.models` (de "async input/output")
é a versão que você usa em código assíncrono.
É a única diferença na assinatura — o resto é igual.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta do capítulo:

```powershell
New-Item -ItemType Directory -Force -Path cap06
```

No VS Code, crie `cap06.ipynb` dentro de `cap06`
e selecione o kernel `Python (agente)`.

> **Importante sobre notebooks:** o Jupyter já tem um event loop
> rodando automaticamente. Você não precisa usar `asyncio.run()` —
> pode usar `await` direto nas células. Em scripts `.py` normais,
> aí sim você precisa do `asyncio.run()`.

---

## As células do notebook

### Célula 1 — Imports

```python
import asyncio
import os
import time
from dataclasses import dataclass, field
from enum import Enum
from google import genai
from google.genai import types

client = genai.Client()
```

---

### Célula 2 — Comparação síncrono vs assíncrono

Vamos medir a diferença concreta. Três chamadas ao Gemini,
primeiro sequenciais (síncronas), depois simultâneas (assíncronas).

```python
async def chamada_async(pergunta: str):
    response = await client.aio.models.generate_content(
        model="gemini-2.5-flash",
        contents=pergunta
    )
    return response.text


def chamada_sync(pergunta: str):
    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=pergunta
    )
    return response.text


perguntas = [
    "Diga apenas: A",
    "Diga apenas: B",
    "Diga apenas: C",
]


print("Síncrono — uma de cada vez:")
inicio = time.time()
for p in perguntas:
    r = chamada_sync(p)
    print(f"  {r.strip()}")
print(f"Tempo total: {time.time() - inicio:.2f}s")
```

---

### Célula 3 — A mesma coisa com asyncio.gather

`asyncio.gather` roda múltiplas coroutines em paralelo.
O Python dispara as três chamadas, espera todas terminarem,
e retorna os resultados na ordem original.

```python
print("Assíncrono — todas ao mesmo tempo:")
inicio = time.time()

resultados = await asyncio.gather(*[chamada_async(p) for p in perguntas])

for r in resultados:
    print(f"  {r.strip()}")
print(f"Tempo total: {time.time() - inicio:.2f}s")
```

Compare os tempos. Síncrono soma os tempos individuais.
Assíncrono é limitado pela chamada mais lenta — porque elas
acontecem em paralelo.

---

### Célula 4 — Coroutine não é função normal

```python
async def diga_oi():
    return "oi"


resultado1 = diga_oi()
print(f"Sem await: {resultado1}")
print(f"Tipo:     {type(resultado1).__name__}")

resultado2 = await diga_oi()
print(f"\nCom await: {resultado2}")
print(f"Tipo:     {type(resultado2).__name__}")
```

Sem `await`, você recebe um objeto coroutine — a função não executou.
Com `await`, a função executou e retornou o valor.

---

### Célula 5 — Os tipos do Capítulo 5 (reuso)

```python
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

### Célula 6 — O loop ReAct em versão assíncrona

A única mudança em relação ao Capítulo 5:
`async def rodar` em vez de `def rodar`, e `await client.aio.models`
em vez de `client.models`.

Toda a lógica do loop continua idêntica. async/await não muda
a estrutura — só permite que o Python faça outras coisas
durante as esperas externas.

```python
async def rodar(prompt: str) -> list[Event]:
    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]
    events: list[Event] = []

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
                events.append(Event(type=EventType.CONTENT, payload={"text": part.text}))
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


events = await rodar("Liste os arquivos desta pasta.")

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
```

---

### Célula 7 — Atendendo múltiplas perguntas em paralelo

Aqui está o ganho concreto: três perguntas independentes
rodando ao mesmo tempo. Cada uma usa o loop ReAct completo.

```python
async def rodar_e_imprimir(prompt: str, label: str):
    print(f"[{label}] começando...")
    events = await rodar(prompt)
    
    texto_final = ""
    for ev in events:
        if ev.type == EventType.CONTENT:
            texto_final += ev.payload['text']
    
    print(f"[{label}] terminou: {texto_final[:60]}...")
    return events


inicio = time.time()

await asyncio.gather(
    rodar_e_imprimir("Liste os arquivos desta pasta", "A"),
    rodar_e_imprimir("Quanto é 17 vezes 23?",          "B"),
    rodar_e_imprimir("Qual a capital da Suécia?",      "C"),
)

print(f"\nTempo total: {time.time() - inicio:.2f}s")
```

Observe que os logs `[A] começando...`, `[B] começando...`,
`[C] começando...` aparecem quase simultaneamente. E os
"terminou" aparecem conforme cada um conclui.

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| concorrência | múltiplas tarefas em progresso ao mesmo tempo | servir vários usuários, não esperar à toa |
| `async def` | declara uma coroutine | função que pode pausar sem bloquear |
| coroutine | função que pode ser pausada e retomada | unidade fundamental do async |
| `await` | executa e espera uma coroutine | sem bloquear o event loop |
| event loop | coordenador de coroutines | gerencia o que roda quando |
| `asyncio.run()` | ponto de entrada para código async | só em scripts, não em notebooks |
| `asyncio.gather()` | roda múltiplas coroutines em paralelo | speedup de N para 1 |
| `client.aio.models` | versão async do SDK do Gemini | use com `await` |

---

## A próxima dor

O loop está async — mas ainda retorna uma **lista** de Events.
Isso significa que o consumidor só recebe alguma coisa quando
o loop inteiro termina. Para uma tarefa de 30 segundos,
30 segundos de silêncio antes de qualquer Event chegar.

O que você realmente quer é o efeito ChatGPT: o texto aparecendo
palavra por palavra, tool calls aparecendo conforme acontecem.
Para isso, o loop precisa **produzir** Events à medida que
eles ocorrem — não acumular tudo numa lista e retornar no final.

Essa é a diferença entre `return list[Event]` e `yield Event`.
É o tema do Capítulo 7.

---

## Arquivo final do capítulo

Quando terminar o notebook, crie o arquivo `cap06\async_loop.py`:

```python
import asyncio
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


async def rodar(prompt: str) -> list[Event]:
    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]
    events: list[Event] = []

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
                events.append(Event(type=EventType.CONTENT, payload={"text": part.text}))
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


async def main():
    prompt = input("Você: ").strip() or "Liste os arquivos desta pasta."
    
    for ev in await rodar(prompt):
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
  <div><a href="cap05-tipos-enum">← Cap 5 — Tipos</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap07-generator">Cap 7 — Generator →</a></div>
</div>
