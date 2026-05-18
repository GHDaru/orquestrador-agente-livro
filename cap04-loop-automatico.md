---
title: "Capítulo 4 — O loop automático"
layout: default
---

# Capítulo 4 — O loop automático

---

## A dor

No Capítulo 3 o código parava depois de executar uma ferramenta.

Tente fazer esta pergunta com o `tool_manual.py`:

```
Liste os arquivos desta pasta e depois me diga
quantas linhas tem o arquivo loop.py.
```

O modelo vai pedir `list_dir` — você executa, devolve.
O modelo vê o resultado e percebe que precisa ler o arquivo.
Ele pede uma segunda ferramenta — mas o código não tem como
continuar. O ciclo terminou depois da primeira execução.

Para resolver, o ciclo precisa se repetir automaticamente
até o modelo não pedir mais nenhuma ferramenta.

---

## O padrão ReAct em código

ReAct (Reason + Act — razão e ação) é o padrão
que estrutura o loop de um agente:

```
while True:
    chama o modelo                    ← Reason: modelo raciocina
    se o modelo pediu ferramentas:
        executa cada ferramenta       ← Act: você executa
        adiciona resultados           ← Observe: resultado volta
        continua o loop
    senão:
        encerrou → break
```

A condição de parada é simples e precisa:
se o modelo não pediu nenhuma ferramenta, ele terminou.
Se pediu, executa tudo e continua.

O `while True` não é um loop infinito perigoso —
a condição de saída é controlada pela resposta do modelo.
Nos próximos capítulos você vai adicionar um limite máximo
de turnos como proteção adicional.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta do capítulo:

```powershell
New-Item -ItemType Directory -Force -Path cap04
```

No VS Code, crie `cap04.ipynb` dentro de `cap04`
e selecione o kernel `Python (agente)`.

---

## As células do notebook

### Célula 1 — Imports, ferramentas e schema

Neste capítulo adicionamos `read_file` como segunda ferramenta.
O dicionário `TOOLS_FN` conecta o nome que o modelo retorna
no `function_call` à função Python correspondente.
Quando o modelo pede `list_dir`, o loop busca `TOOLS_FN["list_dir"]`
e chama com os argumentos fornecidos pelo modelo.

```python
import os
from google import genai
from google.genai import types

client = genai.Client()


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
        description=(
            "Lista arquivos e pastas em um diretório. "
            "Use quando o usuário perguntar sobre arquivos "
            "ou o conteúdo de uma pasta."
        ),
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={
                "path": types.Schema(type=types.Type.STRING,
                                     description="Caminho do diretório. Use '.' para o atual.")
            },
            required=["path"]
        )
    ),
    types.FunctionDeclaration(
        name="read_file",
        description=(
            "Lê o conteúdo de um arquivo de texto. "
            "Use quando precisar ver o que está dentro de um arquivo."
        ),
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={
                "path": types.Schema(type=types.Type.STRING,
                                     description="Caminho completo do arquivo a ler.")
            },
            required=["path"]
        )
    ),
])

print(f"Ferramentas disponíveis: {list(TOOLS_FN.keys())}")
```

---

### Célula 2 — O loop ReAct completo

```python
def rodar(prompt: str):
    historico = [
        types.Content(role="user", parts=[types.Part(text=prompt)])
    ]

    turno = 0

    while True:
        turno += 1
        print(f"\n[turno {turno}]")

        response = client.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        function_calls = [p for p in parts if p.function_call]

        for p in parts:
            if p.text:
                print(f"Gemini: {p.text}")

        if not function_calls:
            print("\n✓ concluído")
            break

        tool_results = []
        for p in function_calls:
            fc   = p.function_call
            args = dict(fc.args) if fc.args else {}

            print(f"→ {fc.name}({args})")

            fn        = TOOLS_FN.get(fc.name)
            resultado = fn(**args) if fn else f"Ferramenta '{fc.name}' não encontrada."

            preview = resultado[:100] + "..." if len(resultado) > 100 else resultado
            print(f"← {preview}")

            tool_results.append(
                types.Part(
                    function_response=types.FunctionResponse(
                        name=fc.name,
                        response={"result": resultado}
                    )
                )
            )

        historico.append(types.Content(role="tool", parts=tool_results))
```

---

### Célula 3 — Testando com uma ferramenta

```python
rodar("Quantos arquivos tem na pasta atual?")
```

---

### Célula 4 — Testando com duas ferramentas em sequência

Esta pergunta exige dois turnos: `list_dir` para descobrir
os arquivos e `read_file` para contar as linhas.
Observe o loop continuando automaticamente.

```python
rodar("Liste os arquivos desta pasta e me diga quantas linhas tem o maior arquivo Python.")
```

---

### Célula 5 — Observando o histórico crescer

```python
def rodar_verbose(prompt: str):
    historico = [
        types.Content(role="user", parts=[types.Part(text=prompt)])
    ]

    turno = 0

    while True:
        turno += 1
        print(f"\n[turno {turno}] histórico: {len(historico)} mensagens")

        response = client.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        function_calls = [p for p in parts if p.function_call]

        for p in parts:
            if p.text:
                print(f"  Gemini: {p.text[:80]}...")

        if not function_calls:
            print(f"\n✓ concluído em {turno} turnos")
            print(f"  histórico final: {len(historico)} mensagens")
            break

        tool_results = []
        for p in function_calls:
            fc   = p.function_call
            args = dict(fc.args) if fc.args else {}

            print(f"  → {fc.name}({args})")

            fn        = TOOLS_FN.get(fc.name)
            resultado = fn(**args) if fn else f"Ferramenta '{fc.name}' não encontrada."
            print(f"  ← {resultado[:60]}...")

            tool_results.append(
                types.Part(
                    function_response=types.FunctionResponse(
                        name=fc.name,
                        response={"result": resultado}
                    )
                )
            )

        historico.append(types.Content(role="tool", parts=tool_results))


rodar_verbose("Liste os arquivos e leia o conteúdo do primeiro arquivo .py que encontrar.")
```

---

### Célula 6 — Experimento: pergunta sem ferramenta

```python
rodar("Qual é a fórmula da área de um círculo?")
```

---

### Célula 7 — Experimento: ferramenta inexistente

```python
TOOLS_FN_LIMITADO = {"list_dir": list_dir}


def rodar_limitado(prompt: str):
    historico = [
        types.Content(role="user", parts=[types.Part(text=prompt)])
    ]

    while True:
        response = client.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        function_calls = [p for p in parts if p.function_call]

        for p in parts:
            if p.text:
                print(f"Gemini: {p.text}")

        if not function_calls:
            break

        tool_results = []
        for p in function_calls:
            fc   = p.function_call
            args = dict(fc.args) if fc.args else {}

            fn        = TOOLS_FN_LIMITADO.get(fc.name)
            resultado = fn(**args) if fn else f"Ferramenta '{fc.name}' não disponível."

            print(f"→ {fc.name} → {resultado[:80]}")

            tool_results.append(
                types.Part(
                    function_response=types.FunctionResponse(
                        name=fc.name,
                        response={"result": resultado}
                    )
                )
            )

        historico.append(types.Content(role="tool", parts=tool_results))


rodar_limitado("Liste os arquivos e leia o conteúdo do primeiro.")
```

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| ReAct | padrão Reason + Act do loop agentico | estrutura que sustenta o `while True` |
| `while True` | ciclo que repete até o modelo encerrar | base do loop automático |
| `if not function_calls: break` | condição de parada | o modelo não pediu tools — terminou |
| `TOOLS_FN` | dicionário nome → função | conecta o pedido do modelo à função Python |
| `fn(**args)` | chamada dinâmica da ferramenta | executa qualquer função pelo nome |
| múltiplos turnos | loop com mais de uma chamada ao modelo | o modelo decide quantos turnos precisa |
| acumulação no histórico | cada turno adiciona mensagens | permite que o modelo veja resultados anteriores |

---

## A próxima dor

O loop funciona — mas tudo está misturado num único lugar:
ferramentas, schemas, loop e lógica de execução.

Quando você tentar adicionar uma quinta ou sexta ferramenta,
vai sentir: mexer em três lugares para cada adição.
Além disso, um erro de digitação num nome de evento
passa despercebido em tempo de execução.

Se o modelo entrar em loop — chamando a mesma ferramenta
repetidamente sem progresso — não há proteção.

Essas dores — organização e segurança — levam ao Capítulo 5:
tipos, Enum e a estrutura que vai sustentar o sistema.

---

## Arquivo final do capítulo

Quando terminar o notebook, crie o arquivo `cap04\loop.py`:

```python
import os
from google import genai
from google.genai import types

client = genai.Client()


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
        description=(
            "Lista arquivos e pastas em um diretório. "
            "Use quando o usuário perguntar sobre arquivos "
            "ou o conteúdo de uma pasta."
        ),
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={
                "path": types.Schema(type=types.Type.STRING,
                                     description="Caminho do diretório. Use '.' para o atual.")
            },
            required=["path"]
        )
    ),
    types.FunctionDeclaration(
        name="read_file",
        description=(
            "Lê o conteúdo de um arquivo de texto. "
            "Use quando precisar ver o que está dentro de um arquivo."
        ),
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={
                "path": types.Schema(type=types.Type.STRING,
                                     description="Caminho completo do arquivo a ler.")
            },
            required=["path"]
        )
    ),
])


def rodar(prompt: str):
    historico = [
        types.Content(role="user", parts=[types.Part(text=prompt)])
    ]

    turno = 0

    while True:
        turno += 1

        response = client.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        function_calls = [p for p in parts if p.function_call]

        for p in parts:
            if p.text:
                print(f"Gemini: {p.text}")

        if not function_calls:
            break

        tool_results = []
        for p in function_calls:
            fc   = p.function_call
            args = dict(fc.args) if fc.args else {}

            print(f"→ {fc.name}({args})")

            fn        = TOOLS_FN.get(fc.name)
            resultado = fn(**args) if fn else f"Ferramenta '{fc.name}' não encontrada."

            preview = resultado[:100] + "..." if len(resultado) > 100 else resultado
            print(f"← {preview}")

            tool_results.append(
                types.Part(
                    function_response=types.FunctionResponse(
                        name=fc.name,
                        response={"result": resultado}
                    )
                )
            )

        historico.append(types.Content(role="tool", parts=tool_results))


if __name__ == "__main__":
    prompt = input("Você: ").strip()
    if not prompt:
        prompt = "Liste os arquivos desta pasta."
    rodar(prompt)
```
