---
title: "Capítulo 10 — O chat no browser"
layout: default
---

# Capítulo 10 — O chat no browser

---

## A dor

O servidor do Capítulo 9 está rodando. Você pode mandar
requisições via `curl` ou `requests` e ver os eventos chegando.
Mas nenhum usuário normal vai abrir o terminal para conversar
com seu agente.

Falta a interface: uma página HTML simples que conecta no
`/stream`, lê os eventos conforme chegam, e renderiza cada
tipo de forma visual diferente. Texto branco fluindo, tool calls
em âmbar, resultados em verde — uma estética de terminal moderno.

E o requisito mais importante: cada evento deve aparecer no momento
em que acontece. Sem espera, sem refresh, sem botões.

---

## EventSource: SSE no browser

No backend, mandamos `data: {...}\n\n` pela conexão SSE.
No browser, a API (Application Programming Interface) que consome
esse stream chama-se `EventSource`. Ela é nativa do JavaScript,
sem biblioteca alguma.

Uso básico:

```javascript
const fonte = new EventSource("/stream");

fonte.onmessage = (event) => {
    const dados = JSON.parse(event.data);
    console.log(dados);
};
```

Cada mensagem `data: ...\n\n` do servidor dispara `onmessage`.
O `event.data` contém o conteúdo após `data: `.

### O problema: EventSource só faz GET

`EventSource` só suporta requisições GET. Nosso endpoint
no Capítulo 9 é POST (porque envia o prompt no body).
Soluções possíveis:

1. Mudar o endpoint para GET com query string
2. Usar `fetch()` com leitura manual do stream

Vamos pela segunda — é mais flexível, embora ligeiramente
mais código. A API `fetch()` aceita POST, e podemos ler a resposta
em chunks (pedaços) usando `ReadableStream`.

---

## Pattern matching em event.type

Cada Event tem um tipo (`content`, `tool_request`, `tool_result`,
`finished`, `error`). No frontend, cada tipo é renderizado
diferente:

```javascript
switch (data.type) {
    case "content":      // texto fluindo, cor padrão
    case "tool_request": // âmbar, com →
    case "tool_result":  // verde, com ←
    case "finished":     // verde claro, ✓
    case "error":        // vermelho, ✗
}
```

Esse switch é o coração do renderizador. Cada caso transforma
um Event abstrato em pixels concretos.

---

## Estética de terminal

A escolha estética combina com o conteúdo do livro:
fundo escuro, fonte monoespaçada, prompt `$`, sem ornamentos.
Familiar para quem trabalha com código.

| Elemento | Estilo |
|---|---|
| Fundo | preto suave (`#0d1117`) |
| Texto | branco/cinza claro |
| Tool requests | âmbar (`#ffb86c`) |
| Tool results | verde suave (`#50fa7b`) |
| Erros | vermelho (`#ff5555`) |
| Fonte | `'Courier New', monospace` |

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Crie a pasta:

```powershell
New-Item -ItemType Directory -Force -Path cap10
```

Você não vai precisar de notebook neste capítulo —
apenas HTML, CSS e JavaScript em um único arquivo.
Os experimentos são interagir com a página no browser.

---

## O arquivo único: index.html

Crie `cap10\index.html` com o conteúdo abaixo, dividido em
quatro partes explicadas na sequência: estrutura, estilo, lógica e união.

### Parte 1 — Estrutura HTML

A estrutura é mínima: um cabeçalho, uma área de mensagens,
uma área de input. A semântica é toda terminal — `<pre>` para
preservar formatação, `<input>` para o prompt.

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Agente</title>
    <style>/* CSS na Parte 2 */</style>
</head>
<body>
    <div class="terminal">
        <div class="header">
            <span class="dot red"></span>
            <span class="dot yellow"></span>
            <span class="dot green"></span>
            <span class="title">agente</span>
        </div>

        <div class="messages" id="messages"></div>

        <div class="input-line">
            <span class="prompt">$</span>
            <input
                type="text"
                id="prompt-input"
                placeholder="Digite sua pergunta e pressione Enter..."
                autocomplete="off"
                autofocus
            >
        </div>
    </div>

    <script>/* JS na Parte 3 */</script>
</body>
</html>
```

### Parte 2 — Estilo CSS

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    background: #0d1117;
    color: #c9d1d9;
    font-family: 'Courier New', Consolas, monospace;
    font-size: 14px;
    line-height: 1.6;
    min-height: 100vh;
    display: flex;
    justify-content: center;
    padding: 20px;
}

.terminal {
    width: 100%;
    max-width: 900px;
    background: #161b22;
    border-radius: 8px;
    border: 1px solid #30363d;
    display: flex;
    flex-direction: column;
    height: calc(100vh - 40px);
}

.header {
    padding: 12px 16px;
    border-bottom: 1px solid #30363d;
    display: flex;
    align-items: center;
    gap: 8px;
}

.dot {
    width: 12px;
    height: 12px;
    border-radius: 50%;
}
.red    { background: #ff5f56; }
.yellow { background: #ffbd2e; }
.green  { background: #27c93f; }

.title {
    margin-left: 12px;
    color: #8b949e;
    font-size: 13px;
}

.messages {
    flex: 1;
    overflow-y: auto;
    padding: 16px;
    white-space: pre-wrap;
    word-wrap: break-word;
}

.input-line {
    border-top: 1px solid #30363d;
    padding: 12px 16px;
    display: flex;
    align-items: center;
    gap: 8px;
}

.prompt {
    color: #58a6ff;
}

input {
    flex: 1;
    background: transparent;
    border: none;
    color: #c9d1d9;
    font-family: inherit;
    font-size: inherit;
    outline: none;
}

.user-message    { color: #58a6ff; margin-top: 12px; }
.content         { color: #c9d1d9; }
.tool-request    { color: #ffb86c; }
.tool-result     { color: #50fa7b; font-size: 13px; opacity: 0.85; }
.finished        { color: #50fa7b; opacity: 0.6; margin-top: 8px; }
.error           { color: #ff5555; }
```

### Parte 3 — JavaScript

```javascript
const messagesEl = document.getElementById("messages");
const inputEl    = document.getElementById("prompt-input");

function addLine(texto, classe) {
    const linha = document.createElement("div");
    linha.className = classe;
    linha.textContent = texto;
    messagesEl.appendChild(linha);
    messagesEl.scrollTop = messagesEl.scrollHeight;
    return linha;
}

function appendToLine(linha, texto) {
    linha.textContent += texto;
    messagesEl.scrollTop = messagesEl.scrollHeight;
}

async function enviarPrompt(prompt) {
    addLine(`$ ${prompt}`, "user-message");
    inputEl.disabled = true;

    try {
        const resp = await fetch("http://localhost:8000/stream", {
            method:  "POST",
            headers: { "Content-Type": "application/json" },
            body:    JSON.stringify({ prompt }),
        });

        const reader  = resp.body.getReader();
        const decoder = new TextDecoder();
        let buffer    = "";
        let linhaTexto = null;

        while (true) {
            const { done, value } = await reader.read();
            if (done) break;

            buffer += decoder.decode(value, { stream: true });
            const linhas = buffer.split("\n\n");
            buffer = linhas.pop();

            for (const linha of linhas) {
                if (!linha.startsWith("data: ")) continue;

                const data = JSON.parse(linha.slice(6));
                renderizar(data);
            }
        }

        function renderizar(data) {
            switch (data.type) {
                case "content":
                    if (!linhaTexto || linhaTexto.dataset.type !== "content") {
                        linhaTexto = addLine("", "content");
                        linhaTexto.dataset.type = "content";
                    }
                    appendToLine(linhaTexto, data.payload.text);
                    break;

                case "tool_request":
                    linhaTexto = null;
                    const args = JSON.stringify(data.payload.args);
                    addLine(`→ ${data.payload.name}(${args})`, "tool-request");
                    break;

                case "tool_result":
                    const out = data.payload.output;
                    const preview = out.length > 200
                        ? out.slice(0, 200) + "..."
                        : out;
                    addLine(`← ${preview}`, "tool-result");
                    break;

                case "finished":
                    addLine("✓ concluído", "finished");
                    break;

                case "error":
                    addLine(`✗ ${data.payload.message}`, "error");
                    break;
            }
        }
    } catch (err) {
        addLine(`✗ Erro de conexão: ${err.message}`, "error");
    } finally {
        inputEl.disabled = false;
        inputEl.value = "";
        inputEl.focus();
    }
}

inputEl.addEventListener("keydown", (e) => {
    if (e.key === "Enter" && inputEl.value.trim()) {
        enviarPrompt(inputEl.value.trim());
    }
});

addLine("Agente pronto. Digite uma pergunta abaixo.", "finished");
```

### Parte 4 — Juntando tudo

Junte as três partes num único arquivo `cap10\index.html`:
o CSS dentro de `<style>`, o JS dentro de `<script>`,
e tudo dentro da estrutura HTML.

---

## Como rodar

Você precisa de **duas coisas rodando ao mesmo tempo**:

### Terminal 1 — o backend

```powershell
cd cap10
Copy-Item ..\cap09\loop.py loop.py
Copy-Item ..\cap09\server.py server.py
uv run uvicorn server:app --reload --port 8000
```

### Terminal 2 — servir o HTML

Por causa de CORS, o HTML precisa ser servido por um servidor
HTTP, não aberto direto como arquivo. Use o servidor builtin
do Python:

```powershell
cd cap10
python -m http.server 5500
```

Abra no browser:

```
http://localhost:5500/index.html
```

---

## Slash commands (extra opcional)

Comandos especiais como `/clear`, `/help` são interceptados
no JavaScript **antes** de enviar ao backend. Não é processado
pelo Gemini — é só UI.

Adicione antes do `fetch(...)`:

```javascript
if (prompt.startsWith("/")) {
    const cmd = prompt.slice(1).toLowerCase().trim();

    if (cmd === "clear") {
        messagesEl.innerHTML = "";
        inputEl.value = "";
        return;
    }

    if (cmd === "help") {
        addLine("Comandos: /clear, /help", "finished");
        inputEl.value = "";
        return;
    }

    addLine(`Comando desconhecido: /${cmd}`, "error");
    inputEl.value = "";
    return;
}
```

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| `fetch()` | API JS para requisições HTTP | substitui XHR antigo |
| `ReadableStream` | leitura de resposta em chunks | consome SSE em POST |
| `TextDecoder` | converte bytes em string | necessário para decodificar chunks |
| pattern matching | switch em `event.type` | renderiza cada tipo de evento |
| DOM (Document Object Model) | árvore de elementos da página | onde JS adiciona/modifica conteúdo |
| `appendChild` | adiciona elemento na página | escreve cada linha do terminal |
| slash commands | comandos `/` interceptados no client | features de UX sem rodar no backend |

---

## A Parte III termina aqui

Você tem agora:

- Loop ReAct completo com type safety, async, streaming, segurança
- Servidor FastAPI expondo via HTTP + SSE
- Chat no browser que mostra eventos em tempo real

É um agente funcional. Mas tem limitações sérias:

- Cada conversa começa do zero (sem persistência)
- Não há como interceptar comportamento (logging, aprovação)
- Não há como decompor tarefas grandes (sub-agentes)

Essas três dores são o tema da Parte IV.

---

## Sem arquivo separado

Diferente dos outros capítulos, este não tem `.py` final:
o entregável é o `index.html` com o conteúdo das quatro partes
juntas. O backend continua sendo o `loop.py` + `server.py`
do Capítulo 9.


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap09-fastapi-sse">← Cap 9 — FastAPI</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="cap11-sessao">Cap 11 — Sessão →</a></div>
</div>
