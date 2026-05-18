---
title: "Introdução — O que vamos construir"
layout: default
---

# Introdução — O que vamos construir

Antes de escrever qualquer linha de código,
você vai ver o destino.

Não para decorar os detalhes —
mas para ter uma imagem mental do que cada capítulo
está construindo em direção a.

---

## O destino

No final deste livro você vai ter um sistema com duas partes:
um servidor Python e um chat que roda no browser.

O chat vai parecer um terminal. Fundo escuro, fonte mono, prompt `$`.
Você digita uma tarefa e o agente começa a trabalhar —
e você vê cada passo acontecendo em tempo real.

```
$ analise os arquivos Python nesta pasta e me diga
  qual tem mais linhas

→ list_dir(path='.')
← loop.py  server.py  tools.py  verificar.py

→ read_file(path='loop.py')
← 180 linhas encontradas...

→ read_file(path='server.py')
← 94 linhas encontradas...

  [continua para cada arquivo]

O arquivo com mais linhas é loop.py, com 180 linhas.
Os demais: server.py (94), tools.py (67), verificar.py (42).

✓ concluído
```

Cada linha `→` é uma ferramenta sendo chamada.
Cada linha `←` é o resultado chegando de volta.
O texto final é o agente sintetizando tudo.

Isso não é mágica. Cada parte tem um nome e uma implementação
que você vai construir do zero.

---

## O que é um agente

"Agente" é uma palavra que aparece muito em discussões sobre
IA (Inteligência Artificial — sistemas computacionais que realizam
tarefas que normalmente exigiriam inteligência humana).
Vale definir com precisão o que significa aqui.

Um agente é um programa que usa um modelo de linguagem
para decidir o que fazer, e tem acesso a ferramentas
para executar essas decisões.

A distinção importante é entre um modelo que só responde
e um modelo que age.

Quando você acessa o ChatGPT ou o Gemini no browser
e faz uma pergunta, o modelo responde com texto.
Se você perguntar "quantos arquivos tem na minha pasta atual?",
o modelo vai inventar uma resposta ou dizer que não tem acesso.

Um agente tem ferramentas reais. Ele pode chamar `list_dir`
e obter a resposta verdadeira. Pode ler um arquivo, rodar um comando,
escrever em disco.

A diferença não é o modelo — é o que está em volta dele.

---

## O padrão ReAct

O sistema que vamos construir segue um padrão chamado ReAct.
O nome vem de Reason + Act — razão e ação.

O ciclo é simples:

```
1. O modelo recebe uma pergunta
2. O modelo raciocina sobre o que precisa fazer
3. O modelo pede uma ferramenta (tool call)
4. Você executa a ferramenta
5. O resultado volta para o modelo
6. O modelo decide se precisa de mais ferramentas
7. Se não: responde. Se sim: volta para 3.
```

Esse ciclo — razão, ação, observação, repetição — é o coração
de quase todo sistema agentico existente.
O Gemini CLI (Command-Line Interface — interface de linha de comando,
um programa que você usa digitando comandos no terminal) do Google usa este padrão.
O Claude Code da Anthropic usa este padrão.

A implementação que vamos construir tem dezenas de linhas,
não dezenas de milhares. Você vai entender cada uma.

---

## Como o livro está organizado

O livro cresce em espiral. Cada parte adiciona uma camada
ao que a parte anterior construiu.

**Parte I — Fundação** (capítulos 1 a 3)

O ponto de partida honesto: uma chamada de API
(Application Programming Interface — interface que permite
que dois programas se comuniquem pela internet) simples.
Você vai descobrir que o modelo não tem memória,
que não consegue agir, que cada chamada é independente.
Essas três limitações motivam tudo que vem depois.

**Parte II — O Loop ReAct** (capítulos 4 a 8)

A construção do loop central, degrau por degrau.
Cada capítulo resolve uma dor do anterior.
Ao final da Parte II você tem um agente funcional no terminal —
capaz de usar ferramentas, raciocinar em múltiplos turnos,
e operar de forma segura.

**Parte III — O servidor** (capítulos 9 e 10)

O loop sai do terminal e vai para o browser.
Você aprende FastAPI (framework — estrutura de desenvolvimento —
Python para criar servidores web), SSE (Server-Sent Events,
mecanismo que permite ao servidor enviar dados ao browser
continuamente, sem que o browser precise ficar pedindo),
e como o frontend recebe eventos em tempo real.
O chat começa a existir.

**Parte IV — Infraestrutura** (capítulos 11 a 13)

Sessão (o agente lembra entre conversas),
hooks (pontos de extensão no loop),
e sub-agentes (o orquestrador delega para especialistas).
Aqui aparecem os primeiros conceitos de arquitetura —
não por dogma, mas porque o sistema ficou complexo o suficiente
para precisar deles.

**Parte V — Abertura** (capítulo 14)

MCP (Model Context Protocol — Protocolo de Contexto do Modelo,
padrão aberto para conectar ferramentas externas a agentes)
e extensibilidade.
O sistema que você construiu pode ser estendido
por qualquer ferramenta externa que fale o protocolo padrão.

---

## O que você vai saber ao final

Não só "como usar" — mas "por que funciona assim".

Você vai entender:

- Por que o modelo precisa receber o histórico inteiro a cada chamada
- Como `function_call` funciona: o modelo pede, você executa
- Por que o loop é um `while True` com uma condição de parada específica
- O que `async/await` resolve e quando importa
- O que um `AsyncGenerator` é e por que é melhor que uma lista para streaming
- Como um decorator funciona por dentro
- O que SSE é e como difere de WebSocket
  (protocolo de comunicação bidirecional em tempo real entre browser e servidor)
- Por que isolamento de ferramentas importa em sistemas multi-agente

E vai ter construído algo real — um sistema que você pode
modificar, estender, e usar como base para projetos próprios.

---

## Uma nota sobre as ferramentas

Este livro usa Python no backend e HTML puro no frontend.

Python porque é a língua franca do ecossistema de IA —
os SDKs (Software Development Kits — conjuntos de ferramentas
e bibliotecas que facilitam o desenvolvimento para uma plataforma)
dos principais modelos têm suporte Python de primeira classe.

HTML puro porque frameworks escondem o que está acontecendo.
Nas primeiras fases, você vai ver `EventSource`, `fetch`, e
manipulação de DOM (Document Object Model — a representação
em memória dos elementos de uma página HTML, que o JavaScript
pode ler e modificar) diretamente, sem camadas de abstração.

React (biblioteca JavaScript para construir interfaces interativas)
aparece na Parte III, quando o estado do frontend
ficar complexo o suficiente para justificar. Não antes.

Para gerenciar Python, usamos `uv` — mais rápido e mais simples
que `pip` + `virtualenv`. Para pacotes JavaScript (quando chegar a hora),
usamos `pnpm` — mais eficiente que `npm` porque usa links simbólicos
ao invés de copiar arquivos para cada projeto.

O ambiente é Windows com PowerShell.
O Capítulo 0 configura tudo isso passo a passo.

---

## Pronto para começar

O Capítulo 1 tem 8 linhas de código Python.

Você vai rodar, ver o modelo responder,
e depois tentar uma segunda pergunta.

O que acontece nessa segunda pergunta
é o ponto de partida de tudo.
