---
title: "Sumário"
layout: default
---

# Construindo um Orquestrador Agentico
## Do loop mínimo ao chat inteligente

*Uma jornada em código: cada conceito nasce de uma dor real*

---

> **Perfil do leitor**
> Você sabe Python básico — escreve funções, usa listas, entende um `for`.
> Nunca trabalhou com APIs de modelos de linguagem, código assíncrono,
> servidores web, ou frontend.
> Trabalha em Windows com PowerShell.
> Quer construir algo real, entendendo cada linha.

---

## Sumário

### Prefácio — Por que este livro existe

A filosofia por trás da abordagem: conceito depois da dor,
nunca antes. Por que a dificuldade é parte do processo,
não um obstáculo a contornar.

---

### Introdução — O que vamos construir

O destino antes do mapa. Você vai ver o sistema completo
funcionando antes de escrever uma linha de código.

- O que é um orquestrador agentico
- A diferença entre "chamar uma API" e "ter um agente"
- O padrão ReAct (Reason + Act): razão, ação, observação
- Como este livro está organizado: a espiral de dores e soluções
- O que você vai saber ao final

---

### Capítulo 0 — Ambiente e ferramentas

Tudo que você precisa antes de começar.
Sem surpresas no meio do caminho.

#### 0.1 — O sistema
- Windows com PowerShell
- Terminal integrado do VS Code ou Windows Terminal

#### 0.2 — Python com uv
- O que é `uv` e por que substituir pip + virtualenv
- Instalação no Windows via PowerShell
- `uv init`, `uv add`, `uv run` — os três comandos essenciais
- Verificando a instalação

#### 0.3 — A API key do Gemini
- Onde criar a chave de acesso
- Limites da camada gratuita: 1.500 req/dia, 15 req/min
- Configurar como variável de ambiente no PowerShell
- Verificando que está funcionando

#### 0.4 — O frontend: HTML puro primeiro, React depois
- Por que começamos com HTML puro (sem ferramentas de build)
- Quando e por que React entra (Parte III)
- Node.js e pnpm para quando chegar a hora

#### 0.5 — Estrutura do projeto
- Como as pastas vão crescer ao longo do livro
- O arquivo `AGENT.md` — contexto persistente do agente

#### 0.6 — Verificação final
- Script `verificar.py` que testa tudo de uma vez
- O que cada erro significa e como resolver

---

## Parte I — Fundação: a arte de fazer o mínimo funcionar

*Capítulos 1 a 3. Três problemas, três conceitos, zero frameworks.*

---

### Capítulo 1 — A chamada mais simples possível

**Conceito central: o que é uma API e por que o modelo não lembra de nada.**

Oito linhas de Python. O modelo responde.
Depois você tenta uma segunda pergunta e descobre a limitação central.

- O que acontece nos bastidores de uma chamada de API
- `genai.Client()`, `generate_content()`, `response.text`
- Stateless: por que cada chamada começa do zero
- **A dor que leva ao próximo capítulo:** sem histórico, não há memória

---

### Capítulo 2 — A conversa com memória

**Conceito central: estado. Quem guarda a memória?**

O modelo não tem estado — você tem.
O histórico é uma lista que você manda inteira a cada chamada.

- `role: "user"` vs `role: "model"` — o contrato de turnos
- `types.Content` e `types.Part` — a estrutura que o Gemini entende
- Loop de conversa com `input()` no terminal
- Experimento: remover o histórico e ver o modelo esquecer tudo
- **A dor que leva ao próximo capítulo:** o modelo não consegue agir

---

### Capítulo 3 — O modelo não consegue agir

**Conceito central: a separação entre planejador e executor.**

O modelo só faz texto. Ele não abre arquivos, não roda comandos.
A solução é declarar uma ferramenta que você executa no lugar dele.

- Por que modelos de linguagem são limitados ao texto
- O que é uma `FunctionDeclaration` e por que a `description` importa
- O modelo pede, você executa: `function_call` na resposta
- Execução manual: ver o pedido, chamar a função, devolver o resultado
- **A dor que leva ao próximo capítulo:** para depois de uma tool

---

## Parte II — O Loop ReAct: razão, ação, observação

*Capítulos 4 a 8. O coração do sistema, construído degrau por degrau.*

---

### Capítulo 4 — O loop automático

**Conceito central: o `while True` como estrutura do ReAct.**

O modelo pode precisar de várias tools em sequência.
O código do Capítulo 3 parava depois de uma.
A solução é um ciclo que só para quando o modelo não pede mais nada.

- O padrão ReAct em código: reason → act → observe → repeat
- A condição de parada: nenhuma tool call = encerrou
- Acumulando resultados antes de continuar
- Observando o histórico crescer a cada turno
- **A dor que leva ao próximo capítulo:** strings soltas, sem estrutura

---

### Capítulo 5 — Nomeando as coisas

**Conceito central: tipos como linguagem. Enum, @dataclass, type hints.**

O loop funciona mas é frágil.
Um erro de digitação em `"tool_requst"` passa despercebido.
Tipos existem para transformar erros invisíveis em erros visíveis.

- A dor das strings soltas: erros silenciosos
- `Enum`: um conjunto fechado de valores válidos
- `@dataclass`: estrutura com campos definidos
- O bug do mutable default e `field(default_factory=dict)`
- Type hints: documentação que o Python verifica
- **A dor que leva ao próximo capítulo:** o loop bloqueia enquanto espera

---

### Capítulo 6 — Tempo real: async/await

**Conceito central: concorrência. O que acontece enquanto você espera.**

Cada chamada ao Gemini demora segundos.
Nesse tempo, o Python fica completamente parado.
`async/await` permite que o programa continue trabalhando enquanto espera.

- O modelo mental: garçom vs cozinheiro
- O que "bloquear" significa na prática
- `async def`: uma função que pode pausar sem parar o processo
- `await`: "espere isto, mas libere o Python para outras tarefas"
- `asyncio.to_thread()`: rodar código bloqueante sem travar tudo
- `asyncio.run()`: o ponto de entrada de qualquer programa async
- **A dor que leva ao próximo capítulo:** o frontend ainda espera tudo terminar

---

### Capítulo 7 — Produzindo eventos em tempo real

**Conceito central: `AsyncGenerator` e `yield`.**

Com `async def` mas retornando uma lista, o frontend
ainda espera o loop terminar para receber qualquer coisa.
`yield` muda isso: cada evento é entregue no momento em que acontece.

- O modelo mental: torneira vs balde
- A diferença entre `return` (entrega tudo no final) e `yield` (entrega aos poucos)
- Geradores síncronos primeiro: entendendo `yield` sem async
- `AsyncGenerator[Event, None]`: o tipo que combina async com yield
- `async for`: como consumir um AsyncGenerator
- **A dor que leva ao próximo capítulo:** registrar tools em três lugares

---

### Capítulo 8 — O decorator `@tool`

**Conceito central: funções que configuram funções.**

Adicionar uma tool exige mexer em três lugares: a função,
o dicionário, o schema. Com um decorator, é um bloco só.

- A dor de registrar em três lugares
- O que é um decorator: função que recebe função e retorna função
- Passo a passo: construindo o decorator `@tool` do zero
- `MAX_TURNS`: o invariante de segurança do loop
- **O loop está completo. A Parte III começa.**

---

## Parte III — O servidor: tornando o loop acessível

*Capítulos 9 e 10. O loop sai do terminal e vai para o browser.*

---

### Capítulo 9 — FastAPI e SSE: o loop vira servidor

**Conceito central: HTTP (HyperText Transfer Protocol), streaming, e SSE.**

- O que é FastAPI e por que não Flask
- `POST /run`: receber prompt, rodar loop, retornar JSON
- O que é SSE (Server-Sent Events — Eventos Enviados pelo Servidor)
- `StreamingResponse`: cada `Event` vira uma linha `data: {json}\n\n`
- CORS (Cross-Origin Resource Sharing — Compartilhamento entre Origens)
- Testando sem frontend com `curl`

---

### Capítulo 10 — O chat no browser

**Conceito central: `EventSource`, pattern-matching em eventos, UX de terminal.**

- `EventSource`: a API (interface) do browser para consumir SSE
- Pattern-match em `event.type`: cada tipo renderiza diferente
- HTML puro: `<textarea>`, `<pre>`, `fetch`
- A estética do terminal: fonte mono, fundo escuro, prompt `$`
- Slash commands: `/clear`, `/help`

---

## Parte IV — Infraestrutura: o agente cresce

*Capítulos 11 a 13. DDD (Domain-Driven Design) entra naturalmente.*

---

### Capítulo 11 — Sessão: o agente que lembra

**Conceito central: estado persistente. Port e Adapter.**

- O problema: cada restart apaga tudo
- `Session` como entidade: id, histórico, timestamps
- `SessionPort`: a interface que separa "o que" do "como"
- `FileSessionStore`: salva JSON em disco
- O arquivo `AGENT.md`: contexto persistente do projeto
- `LoopDetectionService`: detectar quando o agente entrou em loop

---

### Capítulo 12 — Hooks: pontos de extensão no loop

**Conceito central: lifecycle events. O decorator como contrato de extensão.**

- `HookEvent`: enum dos pontos de ciclo de vida
- `HookRegistry`: lista de handlers por evento
- O decorator `@hooks.on(HookEvent.BEFORE_PROMPT)`
- `MessageBus`: pub/sub para aprovação assíncrona de tools
- Botões Aprovar/Rejeitar inline no chat

---

### Capítulo 13 — Sub-agentes: o orquestrador delega

**Conceito central: isolamento. Cada agente tem seu próprio ToolRegistry.**

- `LocalAgentDefinition`: o blueprint de um agente
- `LocalAgentExecutor`: instancia loop com ToolRegistry isolado
- A tool `complete_task`: como um sub-agente sinaliza conclusão
- `AgentTerminateMode`: GOAL, MAX_TURNS, TIMEOUT
- Capítulos aninhados no chat mostrando sub-agentes trabalhando

---

## Parte V — Abertura: o sistema completo

*Capítulo 14.*

---

### Capítulo 14 — MCP, Skills e extensibilidade

**Conceito central: o sistema aberto.**

- O que é MCP (Model Context Protocol — Protocolo de Contexto do Modelo)
- `McpClientAdapter`: conecta servidores externos
- `SkillRegistry`: conhecimento especializado on-demand
- A tool `activate_skill`: carrega conhecimento quando necessário

---

### Epílogo — A jornada e o que fica

A trajetória dos conceitos: de strings soltas a tipos,
de síncrono a async, de script a sistema.
O que vem a seguir.

---

## Apêndices

### Apêndice A — Referência rápida de comandos PowerShell
`uv`, `pnpm`, `uvicorn`, `curl` no Windows

### Apêndice B — O que é DDD Hexagonal (em 2 páginas)
Domain, Application, Infrastructure, Ports e Adapters.

### Apêndice C — Glossário da linguagem ubíqua
Todos os termos introduzidos em cada capítulo, em ordem de aparição.

### Apêndice D — Código completo por fase
O arquivo final de cada fase, sem comentários pedagógicos.

---

## Guia de leitura — como cada capítulo funciona

```
┌─ A dor ──────────────────────────────────────────────────────┐
│  O problema concreto que você vai sentir antes da solução.   │
│  Código com a limitação exposta.                             │
└──────────────────────────────────────────────────────────────┘
┌─ O experimento ──────────────────────────────────────────────┐
│  Rode este código. Observe o comportamento.                  │
│  Os experimentos não são opcionais.                          │
└──────────────────────────────────────────────────────────────┘
┌─ O conceito ─────────────────────────────────────────────────┐
│  Agora que você sentiu: o nome, a teoria, o porquê.         │
└──────────────────────────────────────────────────────────────┘
┌─ A solução ──────────────────────────────────────────────────┐
│  O código que resolve a dor. Comentado linha a linha.        │
└──────────────────────────────────────────────────────────────┘
┌─ O que entra neste capítulo ─────────────────────────────────┐
│  Tabela: conceito → por que aparece aqui                     │
└──────────────────────────────────────────────────────────────┘
┌─ A próxima dor ──────────────────────────────────────────────┐
│  O que o próximo capítulo vai resolver.                      │
└──────────────────────────────────────────────────────────────┘
```

*Versão 0.2 — Windows + PowerShell*
