---
title: "Apêndice B — DDD Hexagonal em 2 páginas"
layout: default
---

# Apêndice B — DDD Hexagonal em 2 páginas

Os conceitos de DDD (Domain-Driven Design) e Arquitetura Hexagonal
que apareceram no livro, em resumo. Sem dogma — apenas o que importa
para entender as decisões arquiteturais que tomamos.

---

## A ideia central

Software complexo precisa separar:

- **O que** o sistema faz (regras de negócio, conceitos essenciais)
- **Como** o sistema faz (banco, rede, framework, interface)

A maioria dos sistemas mistura tudo. Resultado: trocar
o banco de dados exige reescrever metade do código.
Adicionar uma interface nova vira épico.

DDD propõe uma arquitetura em camadas onde o domínio
(o "o quê") fica protegido no centro, e a infraestrutura
(o "como") fica nas bordas.

---

## A metáfora hexagonal

Imagine seu domínio como um hexágono no centro.
Cada lado é uma **porta** (interface, contrato).
Plugada em cada porta há um **adapter** (implementação concreta).

```
                ┌─────────────┐
                │   Frontend  │ ← Driving Adapter
                │   (React)   │   (quem chama o domínio)
                └──────┬──────┘
                       │
                  ┌────▼────┐
                  │ Driving │ ← Port
                  │  Port   │   (interface "entrante")
                  └────┬────┘
                       │
        ┌──────────────▼──────────────┐
        │                             │
        │         DOMÍNIO             │
        │   (regras de negócio)       │
        │                             │
        └─────┬──────────────┬────────┘
              │              │
        ┌─────▼────┐    ┌────▼─────┐
        │ Driven   │    │ Driven   │ ← Ports
        │  Port    │    │  Port    │   (interfaces "saintes")
        └─────┬────┘    └────┬─────┘
              │              │
       ┌──────▼──────┐  ┌────▼──────┐
       │ FileStore   │  │  Gemini   │ ← Driven Adapters
       │  (disco)    │  │   (API)   │   (o domínio chama)
       └─────────────┘  └───────────┘
```

### Driving vs Driven

- **Driving** (que dirige): vem de fora e chama o domínio.
  Exemplo: o frontend que aciona o loop ReAct.

- **Driven** (que é dirigido): é chamado pelo domínio para
  serviços externos. Exemplo: o `FileSessionStore` que o domínio
  usa para persistir.

A regra de ouro: o domínio **não conhece** os adapters.
Ele só conhece as ports (interfaces). Os adapters dependem
do domínio, nunca o contrário.

---

## Os conceitos táticos

DDD tem várias ferramentas para modelar o domínio.
As que apareceram no livro:

### Entidade

Objeto com **identidade única** que persiste no tempo.
Mesmo que todos os campos mudem, é "a mesma coisa".

**No livro:** `Session` é uma entidade — tem `id` único.
A sessão `demo-001` é a mesma seja lá quantas mensagens
você adicionar.

### Value Object

Objeto definido **apenas por seus valores**, sem identidade.
Dois value objects com os mesmos valores são equivalentes.

**No livro:** `LocalAgentDefinition` é um value object —
duas instâncias com mesmo nome, prompt e tools são iguais.

### Domain Service

Lógica de negócio que **não pertence** a uma entidade específica,
mas opera sobre o domínio.

**No livro:** `LoopDetectionService` é um domain service —
detectar loop não é responsabilidade de uma sessão ou tool,
mas regra do domínio.

### Application Service

Coordena ações sobre o domínio, geralmente representando
um caso de uso. Fica entre o adapter externo e o domínio.

**No livro:** os handlers de hooks são application services —
orquestram comportamentos sem alterar o núcleo.

### Bounded Context

Fronteira dentro da qual termos têm significado próprio
e estado é isolado. Diferentes contextos podem ter
"a mesma palavra" com significados diferentes.

**No livro:** cada sub-agente é um bounded context —
seu próprio histórico, suas próprias tools, seu próprio
sentido de "tarefa concluída".

### Anti-Corruption Layer

Camada que traduz entre o vocabulário do seu domínio
e o vocabulário de sistemas externos.

**No livro:** `McpClientAdapter` é uma ACL —
internamente fala MCP, externamente expõe `BaseTool`.

---

## Como isso apareceu no livro

| Capítulo | Conceito DDD | Onde |
|---|---|---|
| 5 | Value Object | `EventType`, `ToolCall` |
| 8 | Linguagem ubíqua | termos: Event, Turn, Tool |
| 11 | Entidade, Port, Adapter | `Session`, `SessionPort`, `FileSessionStore` |
| 11 | Domain Service | `LoopDetectionService` |
| 12 | Application Service | handlers de hooks |
| 13 | Bounded Context, Value Object | sub-agentes, `LocalAgentDefinition` |
| 14 | Anti-Corruption Layer | `McpClientAdapter` |

---

## O que DDD não é

- Não é um framework. É um conjunto de práticas.
- Não exige arquitetura hexagonal. Mas combina bem.
- Não é para todo sistema. Sistemas simples não precisam.
- Não é sobre código. É sobre **modelagem**.

A regra prática: se o sistema é simples e pequeno, DDD é overkill.
Se o sistema é grande e complexo, DDD é o que evita que vire
um pesadelo daqui a dois anos.

O agente que construímos no livro é pequeno. Mas mostramos
os conceitos para você reconhecer quando aplicá-los em
projetos maiores.

---

## Para aprofundar

- **Eric Evans** — *Domain-Driven Design* (o livro azul, 2003)
- **Vaughn Vernon** — *Implementing Domain-Driven Design* (o livro vermelho, 2013)
- **Alistair Cockburn** — *Hexagonal Architecture* (artigo original, 2005)
- **Vladimir Khorikov** — vários cursos modernos online

Mas o melhor caminho é construir e refatorar. DDD se ensina
no código, não em livros.


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="apendiceA-powershell">← Apêndice A</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="apendiceC-glossario">Apêndice C →</a></div>
</div>
