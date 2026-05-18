---
title: "Capítulo 2 — A conversa com memória"
layout: default
---

# Capítulo 2 — A conversa com memória

---

## A dor

No Capítulo 1 você viu o problema: cada chamada a
`generate_content()` é independente. O modelo não sabe
o que foi dito antes.

Na Célula 5 você montou o histórico manualmente e mandou
tudo de uma vez. Funcionou — mas foi trabalhoso.
Você escreveu o histórico na mão para uma conversa de três linhas.

Imagina para uma conversa de vinte turnos.

A solução é automatizar: a cada mensagem nova, você adiciona
à lista e manda o histórico inteiro. O modelo sempre recebe
o contexto completo.

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta do capítulo:

```powershell
New-Item -ItemType Directory -Force -Path cap02
```

No VS Code, crie `cap02.ipynb` dentro de `cap02`
e selecione o kernel `Python (agente)`.

---

## As células do notebook

### Célula 1 — Imports e cliente

O `client` criado aqui fica disponível para todas as
células seguintes do notebook — não precisa recriar a cada célula.

```python
from google import genai
from google.genai import types

client = genai.Client()
```

---

### Célula 2 — Inspecionando Content e Part

O Gemini reconhece três valores de `role`:
`"user"` para mensagens do usuário, `"model"` para respostas
do Gemini, e `"tool"` para resultados de ferramentas —
este aparece no Capítulo 3.

O campo `parts` é uma lista porque uma mensagem pode ter
mais de um tipo de conteúdo ao mesmo tempo: texto, imagem,
resultado de ferramenta. Por enquanto, sempre um único `Part` com texto.

```python
msg_usuario = types.Content(
    role="user",
    parts=[types.Part(text="Olá, meu nome é João")]
)

msg_modelo = types.Content(
    role="model",
    parts=[types.Part(text="Olá João, como posso ajudar?")]
)

print(f"role: {msg_usuario.role}")
print(f"texto: {msg_usuario.parts[0].text}")
print()
print(f"role: {msg_modelo.role}")
print(f"texto: {msg_modelo.parts[0].text}")
```

---

### Célula 3 — O loop de conversa com memória

A pergunta central deste capítulo: quem tem a memória?
**Você tem.** O modelo não guarda nada.

O `historico` é uma lista Python que vive no seu processo.
A cada mensagem nova você adiciona à lista.
A cada chamada à API você manda a lista inteira.
O modelo recebe tudo como se fosse uma única conversa longa.

**Experimento obrigatório — faça esta sequência:**
```
Você: Meu nome é João e trabalho com Python
Você: Qual linguagem eu trabalho?
Você: E qual é o meu nome?
```

O modelo deve lembrar de tudo — porque você mandou o histórico inteiro.

```python
historico = []

print("Chat com memória (digite 'sair' para encerrar)")
print("-" * 50)

while True:
    entrada = input("Você: ").strip()

    if entrada.lower() == "sair":
        break

    historico.append(
        types.Content(role="user", parts=[types.Part(text=entrada)])
    )

    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=historico,
    )

    texto_resposta = response.text

    historico.append(
        types.Content(role="model", parts=[types.Part(text=texto_resposta)])
    )

    print(f"Gemini: {texto_resposta}")
    print(f"[histórico: {len(historico)} mensagens]\n")
```

---

### Célula 4 — Forçando a falha

Aqui a diferença entre `contents=historico` e `contents=entrada`
fica evidente. Faça a mesma sequência de perguntas e observe
que o modelo não lembra de nada quando o histórico não é enviado.

```python
print("Chat SEM memória — para comparar")
print("-" * 50)

while True:
    entrada = input("Você: ").strip()
    if entrada.lower() == "sair":
        break

    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=entrada,
    )

    print(f"Gemini: {response.text}\n")
```

---

### Célula 5 — O custo crescente do histórico

Cada mensagem adicionada ao histórico aumenta o número de tokens
enviados na próxima chamada. Uma conversa de 10 turnos pode ter
5.000 a 10.000 tokens só de histórico — antes de contar a nova
pergunta. Para desenvolvimento com a camada gratuita isso não é
problema, mas é algo a ter em mente: o histórico cresce
e tokens consomem cota.

```python
historico = []

perguntas = [
    "Meu nome é Maria",
    "Trabalho com análise de dados",
    "Qual é o meu nome?",
    "E com o que eu trabalho?",
]

for pergunta in perguntas:
    historico.append(
        types.Content(role="user", parts=[types.Part(text=pergunta)])
    )

    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=historico,
    )

    historico.append(
        types.Content(role="model", parts=[types.Part(text=response.text)])
    )

    tokens = response.usage_metadata.total_token_count
    print(f"Pergunta: {pergunta}")
    print(f"Resposta: {response.text.strip()}")
    print(f"Tokens nesta chamada: {tokens} | Mensagens no histórico: {len(historico)}")
    print()
```

---

### Célula 6 — System prompt: personalidade do agente

O `system_instruction` define a personalidade permanente do agente.
Ele não entra no `historico` — é enviado separadamente a cada chamada.
Mudar o `system_instruction` muda o comportamento sem alterar o loop.

```python
historico = []

while True:
    entrada = input("Você: ").strip()
    if entrada.lower() == "sair":
        break

    historico.append(
        types.Content(role="user", parts=[types.Part(text=entrada)])
    )

    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=historico,
        config=types.GenerateContentConfig(
            system_instruction=(
                "Você é um assistente técnico especializado em Python. "
                "Responda sempre em português e de forma direta e objetiva. "
                "Quando não souber algo, diga claramente."
            )
        )
    )

    texto_resposta = response.text
    historico.append(
        types.Content(role="model", parts=[types.Part(text=texto_resposta)])
    )

    print(f"Gemini: {texto_resposta}\n")
```

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| `historico = []` | lista que acumula a conversa | o único lugar onde a memória existe |
| `types.Content` | estrutura de uma mensagem | formato que o Gemini entende |
| `role="user"` | identifica quem falou | o modelo usa para entender a conversa |
| `role="model"` | identifica a resposta do Gemini | necessário para o histórico funcionar |
| `types.Part` | unidade de conteúdo dentro de uma mensagem | permite múltiplos tipos |
| `contents=historico` | manda o histórico inteiro | por que a memória funciona |
| `system_instruction` | personalidade permanente do agente | enviado separado, não entra no histórico |
| `while True` | loop infinito de conversa | estrutura do chat interativo |

---

## A próxima dor

O chat com memória funciona. Mas tente:

```
Você: Liste os arquivos da minha pasta atual
Gemini: Desculpe, não tenho acesso ao seu sistema de arquivos...
```

O modelo só faz texto. Não acessa arquivos, não roda comandos.
Quando você pede algo que exige ação real, ele recusa —
ou pior, inventa uma resposta.

Essa limitação é o problema central que o Capítulo 3 resolve.

---

## Arquivo final do capítulo

Quando terminar o notebook, crie o arquivo `cap02\chat.py`:

```python
from google import genai
from google.genai import types

client = genai.Client()

historico = []

print("Chat com memória (digite 'sair' para encerrar)")
print("-" * 50)

while True:
    entrada = input("Você: ").strip()

    if entrada.lower() == "sair":
        break

    historico.append(
        types.Content(role="user", parts=[types.Part(text=entrada)])
    )

    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=historico,
    )

    texto_resposta = response.text

    historico.append(
        types.Content(role="model", parts=[types.Part(text=texto_resposta)])
    )

    print(f"Gemini: {texto_resposta}")
    print(f"[histórico: {len(historico)} mensagens]\n")
```


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap01-chamada-simples">← Cap 1 — Primeira chamada</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap03-primeira-tool">Cap 3 — Primeira tool →</a></div>
</div>
