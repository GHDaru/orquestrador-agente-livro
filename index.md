---
title: Construindo um Orquestrador Agentico
layout: default
---

# Construindo um Orquestrador Agentico
## Do loop mínimo ao chat inteligente

*Uma jornada em código: cada conceito nasce de uma dor real.*

---

## Sumário

### Abertura

- [Prefácio](prefacio) — por que aprender sentindo a dificuldade
- [Introdução](introducao) — o que vamos construir

### Parte 0 — Antes de começar

- [Capítulo 0 — Ambiente e ferramentas](cap00-ambiente) — Python, uv, Gemini, Jupyter, VS Code

### Parte I — Fundação: a arte de fazer o mínimo funcionar

- [Capítulo 1 — A chamada mais simples possível](cap01-chamada-simples)
- [Capítulo 2 — A conversa com memória](cap02-conversa-memoria)
- [Capítulo 3 — O modelo não consegue agir](cap03-primeira-tool)

### Parte II — O Loop ReAct: razão, ação, observação

- [Capítulo 4 — O loop automático](cap04-loop-automatico)
- [Capítulo 5 — Nomeando as coisas (Enum, dataclass, types)](cap05-tipos-enum)
- [Capítulo 6 — Tempo real: async/await](cap06-async-await)
- [Capítulo 7 — Produzindo eventos em tempo real](cap07-generator)
- [Capítulo 8 — O decorator @tool](cap08-decorator-tool)

### Parte III — O servidor: tornando o loop acessível

- [Capítulo 9 — FastAPI e SSE](cap09-fastapi-sse)
- [Capítulo 10 — O chat no browser](cap10-chat-browser)

### Parte IV — Infraestrutura: o agente cresce

- [Capítulo 11 — Sessão: o agente que lembra](cap11-sessao)
- [Capítulo 12 — Hooks: pontos de extensão](cap12-hooks)
- [Capítulo 13 — Sub-agentes: o orquestrador delega](cap13-subagentes)

### Parte V — Abertura: o sistema completo

- [Capítulo 14 — MCP, Skills e extensibilidade](cap14-mcp-skills)

### Apêndices

- [Apêndice A — Referência rápida de PowerShell](apendiceA-powershell)
- [Apêndice B — DDD Hexagonal em 2 páginas](apendiceB-ddd)
- [Apêndice C — Glossário da linguagem ubíqua](apendiceC-glossario)
- [Apêndice D — Código completo por fase](apendiceD-codigo)

---

> **Como usar este livro**
>
> Cada capítulo descreve o que colocar em cada célula do Jupyter Notebook.
> Os experimentos não são opcionais — a compreensão acontece na prática.
> Ao final de cada capítulo, crie o arquivo `.py` limpo correspondente.
