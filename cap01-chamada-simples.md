---
title: "Capítulo 1 — A chamada mais simples possível"
layout: default
---

# Capítulo 1 — A chamada mais simples possível

---

## A dor

Antes de qualquer código, um experimento mental.

Imagine que você liga para alguém e faz uma pergunta.
A pessoa responde. Você desliga.

Na próxima ligação, você pergunta: "E sobre o que eu disse antes?"
A pessoa não faz ideia do que você está falando.
Cada ligação começa do zero — sem memória das anteriores.

É exatamente assim que modelos de linguagem funcionam por padrão.

---

## Preparando o ambiente

> **Atenção:** abra o PowerShell e certifique-se de estar
> na pasta raiz do projeto antes de continuar:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta do capítulo:

```powershell
New-Item -ItemType Directory -Force -Path cap01
```

No VS Code, abra a pasta `cap01`, pressione `Ctrl+Shift+P`,
digite `Create: New Jupyter Notebook` e salve como `cap01.ipynb`.

Selecione o kernel `Python (agente)` no canto superior direito.

---

## As células do notebook

### Célula 1 — A chamada mínima

O SDK (Software Development Kit — conjunto de ferramentas de
desenvolvimento) do Gemini lê `GEMINI_API_KEY` automaticamente
do ambiente. O objeto `Client` gerencia a conexão.
`generate_content()` faz a chamada HTTP
(HyperText Transfer Protocol — protocolo de comunicação da internet)
e retorna um objeto com a resposta.

```python
from google import genai

client = genai.Client()

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Qual é a capital do Brasil?"
)

print(response.text)
```

Execute com `Ctrl+Enter` e observe a resposta aparecer abaixo da célula.

---

### Célula 2 — Inspecionando o objeto response

A resposta do modelo não é só texto.
`candidates` contém os candidatos de resposta (normalmente apenas um).
`usage_metadata` mostra quantos tokens foram consumidos.

Tokens são a unidade de processamento do modelo —
pedaços de texto de aproximadamente 3 a 5 caracteres cada.
Os limites gratuitos da API são medidos em tokens por minuto
e requisições por dia.

```python
print(type(response))
print()
print(response.candidates)
print()
print(response.usage_metadata)
```

---

### Célula 3 — Tabela de modelos textuais disponíveis

O parâmetro `model` define qual modelo do Gemini processa a requisição.
Cada modelo tem velocidade, capacidade e custo diferentes.
A tabela abaixo lista os modelos textuais disponíveis em maio de 2026
(fonte: [ai.google.dev/gemini-api/docs/pricing](https://ai.google.dev/gemini-api/docs/pricing)):

| Modelo | Camada gratuita | Req/dia | Req/min | Uso recomendado |
|---|---|---|---|---|
| `gemini-2.5-flash` | ✓ | 1.500 | 15 | **Recomendado** — estável, rápido, gratuito |
| `gemini-2.5-flash-lite` | ✓ | 1.500 | 30 | Tarefas simples, máxima velocidade |
| `gemini-2.5-pro` | ✓ (limitado) | 50 | 5 | Raciocínio complexo, quota muito baixa |
| `gemini-3-flash-preview` | ✓ | 1.500 | 15 | Preview — pode mudar sem aviso |
| `gemini-3.1-pro-preview` | ✗ | — | — | Produção paga, última geração |

Usaremos `gemini-2.5-flash` em todo o livro — estável,
gratuito e capaz o suficiente para tudo que construiremos.

```python
modelos = [
    ("gemini-2.5-flash",       "✓",            "1.500", "15", "Recomendado"),
    ("gemini-2.5-flash-lite",  "✓",            "1.500", "30", "Tarefas simples"),
    ("gemini-2.5-pro",         "✓ (limitado)", "50",    "5",  "Raciocínio complexo"),
    ("gemini-3-flash-preview", "✓",            "1.500", "15", "Preview"),
    ("gemini-3.1-pro-preview", "✗",            "—",     "—",  "Produção paga"),
]

print(f"{'Modelo':<28} {'Free':^14} {'Req/dia':^8} {'Req/min':^8} Uso")
print("-" * 80)
for m in modelos:
    print(f"{m[0]:<28} {m[1]:^14} {m[2]:^8} {m[3]:^8} {m[4]}")
```

---

### Célula 4 — A dor: cada chamada é independente

O servidor do Gemini não mantém estado entre chamadas.
Cada `generate_content()` é uma requisição HTTP independente —
depois que a resposta volta, o servidor esquece que você existiu.
Isso é chamado de stateless: sem estado entre requisições.

```python
r1 = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Qual é a capital do Brasil?"
)
print("Resposta 1:", r1.text)

r2 = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="E qual é a população dessa cidade?"
)
print("Resposta 2:", r2.text)

r3 = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="O que eu perguntei antes?"
)
print("Resposta 3:", r3.text)
```

Observe a terceira resposta. O modelo vai dizer algo como:

```
Como uma inteligência artificial, eu não tenho memória de
conversas anteriores da mesma forma que um ser humano.
Cada interação é processada de forma independente...
```

O modelo não nega categoricamente — tenta ser útil e sugere que
talvez saiba. Mas não sabe. Cada chamada começa do zero.

---

### Célula 5 — Memória simulada na mão (preview do Capítulo 2)

O Capítulo 2 vai resolver a ausência de memória automaticamente.
Mas já podemos ver a mecânica: basta montar o histórico
e mandá-lo inteiro numa única chamada.

`types.Content` é a estrutura de uma mensagem no Gemini.
`role` identifica quem produziu a mensagem: `"user"` para você,
`"model"` para o Gemini.
`types.Part` é a unidade de conteúdo dentro de uma mensagem —
por enquanto sempre texto, mas pode ser imagem ou resultado de ferramenta.

```python
from google.genai import types

historico = [
    types.Content(
        role="user",
        parts=[types.Part(text="Qual é a capital do Brasil?")]
    ),
    types.Content(
        role="model",
        parts=[types.Part(text="A capital do Brasil é Brasília.")]
    ),
    types.Content(
        role="user",
        parts=[types.Part(text="O que eu perguntei antes?")]
    ),
]

r = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=historico,
)

print(r.text)
```

O modelo responde corretamente — não porque tem memória,
mas porque recebeu a conversa inteira.
A "memória" não veio do modelo. Veio da lista `historico`.

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| API | interface para programas comunicarem-se pela internet | como seu código fala com o Gemini |
| HTTP | protocolo de comunicação da internet | o canal pelo qual a API funciona |
| stateless | sem estado entre requisições | explica por que não há memória |
| tokens | unidade de processamento do modelo | base dos limites e custos da API |
| `genai.Client()` | objeto que gerencia a conexão | ponto de entrada para qualquer chamada |
| `generate_content()` | função que faz a chamada ao modelo | a operação central do sistema |
| `model=` | qual modelo usar | define velocidade, capacidade e custo |
| `contents` | o que você manda para o modelo | por onde entra o texto |
| `response.text` | o texto da resposta | por onde sai o texto |
| `types.Content` | estrutura de uma mensagem | formato que o Gemini entende |
| `types.Part` | unidade de conteúdo dentro de uma mensagem | permite múltiplos tipos |
| `role` | quem produziu a mensagem | user, model — o Gemini usa para navegar a conversa |

---

## A próxima dor

O Capítulo 2 resolve a ausência de memória de forma permanente —
a lista de histórico cresce automaticamente a cada turno.

Mas vai surgir uma nova limitação: o modelo só faz texto.
Ele não consegue agir no mundo fora da conversa.
Essa dor aparece no Capítulo 3.

---

## Arquivo final do capítulo

Quando terminar o notebook, crie o arquivo `cap01\call.py`:

```python
from google import genai

client = genai.Client()

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Qual é a capital do Brasil?"
)

print(response.text)

response2 = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="O que eu perguntei antes?"
)

print(response2.text)
```


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap00-ambiente">← Cap 0 — Ambiente</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap02-conversa-memoria">Cap 2 — Memória →</a></div>
</div>
