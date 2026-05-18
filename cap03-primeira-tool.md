---
title: "Capítulo 3 — O modelo não consegue agir"
layout: default
---

# Capítulo 3 — O modelo não consegue agir

---

## A dor

No Capítulo 2 você construiu um chat com memória.
Agora rode o `chat.py` e tente:

```
Você: Liste os arquivos da minha pasta atual
```

```
Gemini: Desculpe, não tenho acesso direto ao seu sistema
de arquivos. Sou um modelo de linguagem e não posso executar
comandos no seu computador ou acessar seus arquivos locais.
```

Ele sabe *como* fazer — mas não *consegue* fazer.
Dá a instrução em vez de executar.

A limitação é fundamental: modelos de linguagem processam texto
e produzem texto. Raciocinam — mas não agem.

Para que o modelo possa agir, você precisa declarar uma ferramenta.
E você executa essa ferramenta no lugar dele.

---

## A separação central: planejador e executor

Esta é a arquitetura central de todo sistema agentico:

```
┌─────────────────────────────────────────┐
│  MODELO (planejador)                    │
│  - raciocina sobre o que precisa        │
│  - decide qual ferramenta usar          │
│  - formula os argumentos               │
│  - NÃO executa nada                    │
└──────────────────┬──────────────────────┘
                   │ "quero chamar list_dir com path='.'"
                   ▼
┌─────────────────────────────────────────┐
│  SEU CÓDIGO (executor)                  │
│  - recebe o pedido estruturado          │
│  - executa a função Python real         │
│  - captura o resultado                  │
│  - devolve ao modelo                   │
└─────────────────────────────────────────┘
```

O modelo nunca toca no sistema de arquivos, nunca roda comandos.
Ele produz um pedido estruturado — e você executa.
Isso é deliberado: você mantém controle total sobre o que é executado.

---

## O que é uma FunctionDeclaration

`FunctionDeclaration` é o **contrato** entre você e o modelo.
O modelo não vê o código da sua função Python —
ele só lê a declaração: nome, descrição e parâmetros.
Com base nisso, decide quando e como chamá-la.

A declaração segue o formato JSON Schema
(JavaScript Object Notation Schema — padrão aberto para
descrever a estrutura de dados em formato de texto legível).

### Os campos da declaração

**`name`** — o identificador da ferramenta, em snake_case.
O modelo retorna exatamente este nome no pedido de execução.

**`description`** — o campo mais importante.
O modelo lê apenas este texto para decidir se deve usar a ferramenta
e em que situações. Uma descrição ruim resulta em modelo confuso
ou que nunca usa a ferramenta, ou que a usa com argumentos errados.

**`parameters`** — define os argumentos que o modelo pode passar,
usando `types.Schema` com tipos específicos.

### Tipos suportados em types.Schema

O campo `type` dentro de `types.Schema` define o tipo de dado
que o modelo deve passar para o argumento:

| Tipo | Constante | Exemplo de uso |
|---|---|---|
| Texto | `types.Type.STRING` | nomes, caminhos, comandos |
| Inteiro | `types.Type.INTEGER` | contagens, limites, índices |
| Decimal | `types.Type.NUMBER` | temperaturas, preços, coordenadas |
| Booleano | `types.Type.BOOLEAN` | flags, switches, incluir/excluir |
| Lista | `types.Type.ARRAY` | múltiplos valores do mesmo tipo |
| Objeto | `types.Type.OBJECT` | estruturas compostas com sub-campos |

Tipos podem ser aninhados:
- `ARRAY` de `STRING`: lista de extensões de arquivo
- `OBJECT` dentro de `OBJECT`: estrutura hierárquica com sub-campos
- `ARRAY` de `OBJECT`: lista de registros compostos

### O que é `types` e de onde vem

`from google.genai import types`

`types` é o módulo do SDK do Gemini que contém todas as classes
de dados usadas para estruturar requisições e respostas da API.
As principais classes:

| Classe | Para que serve |
|---|---|
| `types.Tool` | agrupa uma ou mais FunctionDeclarations |
| `types.FunctionDeclaration` | declara uma ferramenta ao modelo |
| `types.Schema` | define a estrutura de dados em JSON Schema |
| `types.Type` | enum com os tipos suportados |
| `types.Content` | uma mensagem na conversa |
| `types.Part` | unidade de conteúdo dentro de uma mensagem |
| `types.FunctionResponse` | resultado de uma ferramenta |
| `types.GenerateContentConfig` | configurações da chamada |

Documentação completa: [googleapis.github.io/python-genai](https://googleapis.github.io/python-genai/)
Referência de function calling: [ai.google.dev/gemini-api/docs/function-calling](https://ai.google.dev/gemini-api/docs/function-calling)

### Qualquer modelo do Gemini suporta function calling?

Não. Function calling é uma capacidade específica.
Os modelos da família **Gemini 2.5 e 3.x** suportam plenamente.
Para os fins deste livro, `gemini-2.5-flash` suporta
function calling sem restrições.

### Outros LLMs usam a mesma estrutura?

Não — cada provedor tem seu próprio formato.
O **conceito** é o mesmo (planejador pede, executor executa),
mas a implementação é diferente:

| Provedor | LLM | Formato da declaração | Nome do pedido |
|---|---|---|---|
| Google | Gemini | `types.FunctionDeclaration` + `types.Schema` | `function_call` |
| Anthropic | Claude | dicionário com `input_schema` (JSON Schema) | `tool_use` |
| OpenAI | GPT | dicionário com `parameters` (JSON Schema) | `tool_calls` |

JSON Schema é um padrão aberto — as definições de parâmetros
são praticamente idênticas entre os três. O que muda é a classe
Python que você usa para embrulhá-las.

O loop ReAct que você vai construir nos próximos capítulos
tem a mesma lógica para qualquer LLM —
só a parte de montar e interpretar as mensagens muda.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta do capítulo:

```powershell
New-Item -ItemType Directory -Force -Path cap03
```

No VS Code, crie `cap03.ipynb` dentro de `cap03`
e selecione o kernel `Python (agente)`.

---

## As células do notebook

### Célula 1 — Imports

```python
import os
from google import genai
from google.genai import types

client = genai.Client()
```

---

### Célula 2 — A função Python (a ferramenta real)

Execute esta célula para ver a função funcionar diretamente,
sem nenhum modelo envolvido. É esta função que o modelo
vai *pedir* para você executar — ele nunca a chama diretamente.

```python
def list_dir(path: str) -> str:
    try:
        itens = os.listdir(path)
        return "\n".join(sorted(itens))
    except Exception as e:
        return f"Erro: {e}"

print(list_dir("."))
```

---

### Célula 3 — Anatomia de uma FunctionDeclaration

Observe cada campo: `name` em snake_case, `description` detalhada,
`parameters` com `types.Schema`, `required` listando os obrigatórios.

```python
declaracao = types.FunctionDeclaration(
    name="list_dir",
    description=(
        "Lista arquivos e pastas em um diretório. "
        "Use quando o usuário perguntar sobre arquivos, "
        "conteúdo de pastas, ou quiser saber o que existe "
        "em um caminho específico."
    ),
    parameters=types.Schema(
        type=types.Type.OBJECT,
        properties={
            "path": types.Schema(
                type=types.Type.STRING,
                description="Caminho do diretório a listar. Use '.' para o atual."
            )
        },
        required=["path"]
    )
)

print(f"nome: {declaracao.name}")
print(f"descrição: {declaracao.description[:60]}...")
```

---

### Célula 4 — Exemplos de cada tipo Schema

Execute para ver como cada tipo é construído e como
tipos aninhados (ARRAY de STRING, OBJECT com sub-campos)
funcionam na prática.

```python
schema_string  = types.Schema(type=types.Type.STRING,  description="Nome do arquivo")
schema_integer = types.Schema(type=types.Type.INTEGER, description="Número máximo de resultados")
schema_number  = types.Schema(type=types.Type.NUMBER,  description="Temperatura em Celsius")
schema_boolean = types.Schema(type=types.Type.BOOLEAN, description="Incluir arquivos ocultos?")

schema_array_string = types.Schema(
    type=types.Type.ARRAY,
    items=types.Schema(type=types.Type.STRING),
    description="Lista de extensões: ['.py', '.txt']"
)

schema_object = types.Schema(
    type=types.Type.OBJECT,
    properties={
        "host": types.Schema(type=types.Type.STRING,  description="Endereço do servidor"),
        "port": types.Schema(type=types.Type.INTEGER, description="Porta de conexão"),
        "ssl":  types.Schema(type=types.Type.BOOLEAN, description="Usar SSL?"),
    },
    required=["host", "port"]
)

declaracao_rica = types.FunctionDeclaration(
    name="buscar_arquivos",
    description="Busca arquivos com filtros opcionais.",
    parameters=types.Schema(
        type=types.Type.OBJECT,
        properties={
            "pasta":     types.Schema(type=types.Type.STRING, description="Pasta onde buscar"),
            "extensoes": schema_array_string,
            "max_itens": schema_integer,
            "ocultos":   schema_boolean,
        },
        required=["pasta"]
    )
)

for t in ["STRING", "INTEGER", "NUMBER", "BOOLEAN", "ARRAY", "OBJECT"]:
    print(f"  types.Type.{t}")

print()
print(f"Parâmetros obrigatórios: {declaracao_rica.parameters.required}")
print(f"Parâmetros opcionais: extensoes, max_itens, ocultos")
```

---

### Célula 5 — O schema completo pronto para uso

```python
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
                "path": types.Schema(
                    type=types.Type.STRING,
                    description="Caminho do diretório. Use '.' para o atual."
                )
            },
            required=["path"]
        )
    )
])

print(f"Ferramentas: {len(schema.function_declarations)}")
print(f"Nome: {schema.function_declarations[0].name}")
```

---

### Célula 6 — Primeira chamada: vendo o pedido bruto

Observe que o modelo não respondeu com texto — retornou
um `function_call` com o nome da ferramenta e os argumentos.

```python
pergunta = "Quantos arquivos tem na pasta atual?"
print(f"Você: {pergunta}\n")

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=pergunta,
    config=types.GenerateContentConfig(tools=[schema])
)

for part in response.candidates[0].content.parts:
    if part.text:
        print(f"  TEXTO: {part.text}")
    elif part.function_call:
        fc = part.function_call
        print(f"  FUNCTION CALL: {fc.name}({dict(fc.args)})")
```

---

### Célula 7 — Executando e devolvendo o resultado

O histórico precisa de três partes em ordem:
a pergunta, a resposta do modelo com o `function_call`,
e o resultado da ferramenta com `role="tool"`.

```python
for part in response.candidates[0].content.parts:
    if part.function_call:
        fc   = part.function_call
        args = dict(fc.args)

        print(f"→ {fc.name}({args})")
        resultado = list_dir(args.get("path", "."))
        print(f"← {resultado}\n")

        historico = [
            types.Content(role="user",  parts=[types.Part(text=pergunta)]),
            response.candidates[0].content,
            types.Content(
                role="tool",
                parts=[types.Part(
                    function_response=types.FunctionResponse(
                        name=fc.name,
                        response={"result": resultado}
                    )
                )]
            ),
        ]

        response2 = client.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        print(f"Gemini: {response2.text}")
```

---

### Célula 8 — Experimento: descrição ruim vs boa

```python
schema_ruim = types.Tool(function_declarations=[
    types.FunctionDeclaration(
        name="list_dir",
        description="Faz coisas com pastas",
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={"path": types.Schema(type=types.Type.STRING, description="caminho")},
            required=["path"]
        )
    )
])

response_ruim = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Quantos arquivos tem na pasta atual?",
    config=types.GenerateContentConfig(tools=[schema_ruim])
)

for part in response_ruim.candidates[0].content.parts:
    if part.text:
        print(f"Texto (inventado): {part.text[:100]}...")
    elif part.function_call:
        print(f"Pediu a ferramenta: {part.function_call.name}({dict(part.function_call.args)})")
```

---

### Célula 9 — Experimento: pergunta sem necessidade de tool

```python
response_sem_tool = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Qual é a capital da França?",
    config=types.GenerateContentConfig(tools=[schema])
)

for part in response_sem_tool.candidates[0].content.parts:
    if part.text:
        print(f"Texto: {part.text}")
    elif part.function_call:
        print(f"Pediu tool: {part.function_call.name}")
```

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| planejador | o modelo — decide o que fazer | separação de responsabilidades |
| executor | seu código — faz acontecer | você controla o que é executado |
| `FunctionDeclaration` | contrato que descreve a ferramenta | o modelo usa para decidir quando e como chamar |
| `description` | texto que explica a ferramenta | campo mais importante — guia o comportamento |
| `types.Schema` | estrutura de dados em JSON Schema | define os parâmetros da ferramenta |
| `types.Type` | enum de tipos suportados | STRING, INTEGER, NUMBER, BOOLEAN, ARRAY, OBJECT |
| `types.Tool` | agrupa FunctionDeclarations | passado no config da chamada |
| `function_call` | pedido do modelo para usar uma ferramenta | como o modelo "age" sem agir diretamente |
| `role="tool"` | tipo de mensagem para resultados | terceiro role além de user e model |
| `FunctionResponse` | estrutura para devolver o resultado | formato que o Gemini espera |
| JSON Schema | padrão aberto para descrever dados | usado por Gemini, Claude e GPT |

---

## A próxima dor

O código do Capítulo 3 funciona — mas para depois de uma ferramenta.

Tente:
```
"Liste os arquivos da pasta atual e me diga
quantas linhas tem o maior arquivo."
```

O modelo vai pedir `list_dir` — você executa, devolve.
O modelo vai precisar de uma segunda ferramenta para ler o arquivo —
mas o ciclo não se repete. O código trava.

Essa limitação leva ao Capítulo 4: o `while True`.

---

## Arquivo final do capítulo

Quando terminar o notebook, crie o arquivo `cap03\tool_manual.py`:

```python
import os
from google import genai
from google.genai import types

client = genai.Client()


def list_dir(path: str) -> str:
    try:
        itens = os.listdir(path)
        return "\n".join(sorted(itens))
    except Exception as e:
        return f"Erro: {e}"


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
                "path": types.Schema(
                    type=types.Type.STRING,
                    description="Caminho do diretório. Use '.' para o atual."
                )
            },
            required=["path"]
        )
    )
])


pergunta = input("Você: ").strip()

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=pergunta,
    config=types.GenerateContentConfig(tools=[schema])
)

for part in response.candidates[0].content.parts:
    if part.text:
        print(f"Gemini: {part.text}")

    elif part.function_call:
        fc   = part.function_call
        args = dict(fc.args)

        print(f"→ {fc.name}({args})")
        resultado = list_dir(args.get("path", "."))
        print(f"← {resultado}\n")

        historico = [
            types.Content(role="user",  parts=[types.Part(text=pergunta)]),
            response.candidates[0].content,
            types.Content(
                role="tool",
                parts=[types.Part(
                    function_response=types.FunctionResponse(
                        name=fc.name,
                        response={"result": resultado}
                    )
                )]
            ),
        ]

        response2 = client.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(tools=[schema])
        )

        print(f"Gemini: {response2.text}")
```
