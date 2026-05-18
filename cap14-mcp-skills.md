---
title: "Capítulo 14 — MCP, Skills e extensibilidade"
layout: default
---

# Capítulo 14 — MCP, Skills e extensibilidade

---

## A dor

O sistema do Capítulo 13 funciona. Mas todas as ferramentas
estão **dentro** do seu código. Se um colega quiser adicionar
uma ferramenta de busca no Slack, ele precisa:

1. Clonar seu repositório
2. Aprender suas convenções (decorator `@tool`, etc.)
3. Modificar seu código
4. Manter um fork ou negociar pull requests

Pior: cada agente diferente reinventa a roda. O agente do colega
A tem `slack_search`. O do colega B tem `jira_create_ticket`.
Mas eles não compartilham nada — cada um é uma ilha.

A solução é um **protocolo aberto**: uma especificação que define
como um agente conversa com ferramentas externas, independente
de linguagem, framework ou empresa. Esse protocolo existe e se
chama MCP (Model Context Protocol — Protocolo de Contexto do Modelo),
criado pela Anthropic e adotado amplamente em 2025.

E há outro problema: o agente pode precisar de **conhecimento
especializado** que ocupa muito espaço para estar sempre presente.
Como "todas as convenções do projeto X" ou "regras de negócio
do produto Y". A solução são **Skills**: pacotes de conhecimento
que o agente carrega on-demand quando a tarefa pede.

---

## O que é MCP

MCP é um protocolo cliente-servidor. Há dois lados:

**Servidor MCP** — expõe ferramentas. É um processo que roda
independente, escutando em um canal (stdin/stdout ou HTTP).
Quando pedem, lista as ferramentas disponíveis. Quando chamam,
executa e devolve o resultado.

**Cliente MCP** — consome ferramentas. É seu agente, que conecta
em um ou mais servidores MCP, descobre as ferramentas e as
oferece ao modelo como se fossem suas.

```
Seu Agente (cliente MCP)
    │
    ├── conecta em → Servidor MCP do Slack
    │                  (expõe: slack_search, slack_post)
    │
    ├── conecta em → Servidor MCP do GitHub
    │                  (expõe: gh_pr_create, gh_issue_list)
    │
    └── conecta em → Servidor MCP do filesystem
                       (expõe: list_dir, read_file, write_file)
```

O modelo vê todas essas ferramentas como uma lista única.
Não sabe nem importa onde estão implementadas.

### Por que isto é DDD — Anti-Corruption Layer

Em DDD, quando você integra com um sistema externo, é boa prática
criar uma **Anti-Corruption Layer** (Camada Anti-Corrupção):
um adapter que traduz entre o vocabulário do sistema externo
e o vocabulário do seu domínio.

O `McpClientAdapter` é exatamente isso. Por dentro, ele fala MCP
(protocolo externo). Para o resto do código, ele expõe ferramentas
no formato `BaseTool` que o domínio entende. Você troca de servidor
MCP, ou MCP por outro protocolo, sem o domínio nunca saber.

---

## O que é uma Skill

Skill é um pacote de conhecimento estruturado: um arquivo
`SKILL.md` com instruções específicas, mais possíveis arquivos
de suporte (templates, scripts, dados).

```
skills/
├── escrever-email-cliente/
│   ├── SKILL.md           ← instruções
│   └── tom_da_empresa.md  ← referência
└── analisar-codigo-py/
    ├── SKILL.md
    └── padroes_projeto.md
```

A primeira linha do `SKILL.md` tem uma descrição curta —
quando ativar essa skill. O agente lê **só essa descrição**
ao iniciar. Quando julga que a skill é relevante, chama
`activate_skill("nome")`, que carrega o conteúdo completo
e injeta no system_instruction do próximo turno.

Vantagem: o agente "sabe" que existem 50 skills disponíveis,
mas só carrega no contexto as 2 ou 3 que importam para
a tarefa atual. Economia massiva de tokens.

---

## Os componentes deste capítulo

| Componente | O que faz |
|---|---|
| `McpClientAdapter` | conecta a um servidor MCP, lista e chama ferramentas |
| `McpToolProxy` | embrulha uma ferramenta MCP como `BaseTool` local |
| `SkillRegistry` | escaneia o diretório `skills/`, indexa descrições |
| `activate_skill` | tool que carrega o conteúdo de uma skill |

---

## Preparando o ambiente

> **Atenção:** certifique-se de estar na pasta raiz do projeto:
>
> ```powershell
> cd C:\projetos\agente
> ```

Adicione a biblioteca MCP:

```powershell
uv add mcp
```

Crie a pasta:

```powershell
New-Item -ItemType Directory -Force -Path cap14
New-Item -ItemType Directory -Force -Path cap14\skills
```

No VS Code, crie `cap14.ipynb` dentro de `cap14`.

---

## As células do notebook

### Célula 1 — BaseTool (a abstração comum)

Para mostrar como tools internas e MCP coexistem, definimos
uma interface comum `BaseTool`:

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass


class BaseTool(ABC):
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    def description(self) -> str: ...

    @abstractmethod
    async def execute(self, args: dict) -> str: ...


class LocalTool(BaseTool):
    def __init__(self, name: str, description: str, fn):
        self._name = name
        self._desc = description
        self._fn   = fn

    def name(self):        return self._name
    def description(self): return self._desc

    async def execute(self, args: dict) -> str:
        result = self._fn(**args)
        return result if isinstance(result, str) else str(result)


print("BaseTool definida.")
```

---

### Célula 2 — McpToolProxy

Um Proxy é um objeto que se faz passar por outro, mas internamente
delega para algum outro lugar. O `McpToolProxy` se parece com
qualquer tool local, mas internamente fala MCP:

```python
class McpToolProxy(BaseTool):
    def __init__(self, mcp_client, mcp_tool_info):
        self.client = mcp_client
        self.info   = mcp_tool_info

    def name(self):        return self.info["name"]
    def description(self): return self.info["description"]

    async def execute(self, args: dict) -> str:
        result = await self.client.call_tool(self.name(), args)
        return str(result)


print("McpToolProxy definida.")
```

O loop não distingue — chama `await tool.execute(args)` em qualquer
`BaseTool` e funciona. Esse é o ponto da abstração.

---

### Célula 3 — McpClientAdapter (versão simplificada)

A versão real usaria a biblioteca `mcp` para conectar via stdio
ou HTTP. Para este notebook, fazemos uma versão simplificada
que simula o protocolo. Em produção, você usaria a biblioteca
oficial.

```python
class McpClientAdapter:
    """Cliente MCP simplificado para fins didáticos.

    Em produção, use a biblioteca oficial 'mcp' que cuida
    de conexão (stdio/HTTP), handshake, e lifecycle.
    """

    def __init__(self, server_name: str, simulated_tools: list):
        self.server_name      = server_name
        self.simulated_tools  = simulated_tools

    async def list_tools(self) -> list[dict]:
        """Em produção, faz uma requisição ao servidor MCP."""
        return self.simulated_tools

    async def call_tool(self, name: str, args: dict) -> str:
        """Em produção, faz uma requisição ao servidor MCP."""
        if name == "get_weather":
            return f"Tempo em {args.get('city', '?')}: ensolarado, 25°C (simulado)"
        return f"[mcp:{self.server_name}] resultado para {name}({args})"


weather_server = McpClientAdapter(
    server_name="weather",
    simulated_tools=[
        {
            "name":        "get_weather",
            "description": "Obtém a previsão do tempo de uma cidade.",
            "parameters":  {"city": "Nome da cidade"}
        }
    ]
)

tools_mcp = await weather_server.list_tools()
print(f"Tools disponíveis no servidor: {[t['name'] for t in tools_mcp]}")
```

---

### Célula 4 — Adicionando tools MCP ao agente

```python
mcp_proxies = []

for info in await weather_server.list_tools():
    proxy = McpToolProxy(weather_server, info)
    mcp_proxies.append(proxy)
    print(f"Registrada: {proxy.name()} - {proxy.description()}")


resultado = await mcp_proxies[0].execute({"city": "Florianópolis"})
print(f"\nResultado: {resultado}")
```

A tool MCP é chamada do jeito normal. O proxy esconde
toda a complexidade do protocolo.

---

### Célula 5 — Skills: estrutura no disco

Crie alguns arquivos de skill para teste. No PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path cap14\skills\formatar-saida
```

Crie `cap14\skills\formatar-saida\SKILL.md`:

```markdown
# Formatar saída em tabela

Use esta skill quando o usuário pedir resultados em formato
de tabela ou lista estruturada.

## Convenções

- Sempre use Markdown
- Tabelas com cabeçalho separado por traços
- Listas com hífen
- Código em blocos triplo-crase

## Exemplo

| Coluna | Descrição |
|---|---|
| nome   | Texto curto |
| valor  | Número |
```

E `cap14\skills\python-tipado\SKILL.md`:

```markdown
# Python tipado profissional

Use esta skill quando o usuário pedir código Python.

## Regras

- Sempre type hints em parâmetros e retorno
- Use `from __future__ import annotations` para hints recursivos
- Prefira `Pathlib` a `os.path`
- Docstrings curtas no padrão Google
```

---

### Célula 6 — SkillRegistry

```python
from pathlib import Path


@dataclass
class Skill:
    name:         str
    description:  str   # primeira linha de SKILL.md
    full_content: str   # conteúdo completo
    path:         Path


class SkillRegistry:
    def __init__(self, base_dir: Path):
        self.base_dir = base_dir
        self.skills: dict[str, Skill] = {}
        self._scan()

    def _scan(self):
        if not self.base_dir.exists():
            return

        for skill_dir in self.base_dir.iterdir():
            if not skill_dir.is_dir():
                continue

            skill_md = skill_dir / "SKILL.md"
            if not skill_md.exists():
                continue

            content = skill_md.read_text(encoding="utf-8")
            lines   = content.strip().split("\n")

            description = "Skill sem descrição."
            for line in lines[1:]:
                line = line.strip()
                if line and not line.startswith("#"):
                    description = line
                    break

            self.skills[skill_dir.name] = Skill(
                name=skill_dir.name,
                description=description,
                full_content=content,
                path=skill_dir
            )

    def list_summary(self) -> str:
        if not self.skills:
            return "Nenhuma skill disponível."
        return "\n".join(
            f"- {s.name}: {s.description}"
            for s in self.skills.values()
        )

    def get_content(self, name: str) -> str | None:
        s = self.skills.get(name)
        return s.full_content if s else None


skills_dir = Path("cap14") / "skills"
registry   = SkillRegistry(skills_dir)

print("Skills encontradas:")
print(registry.list_summary())
```

---

### Célula 7 — A tool activate_skill

A tool `activate_skill` é especial: quando chamada, ela carrega
o conteúdo da skill no contexto. Não é executar — é injetar
conhecimento.

```python
activated_skills: dict[str, str] = {}


def activate_skill_fn(skill_name: str) -> str:
    content = registry.get_content(skill_name)
    if content is None:
        return f"Skill '{skill_name}' não encontrada."

    activated_skills[skill_name] = content
    return f"Skill '{skill_name}' ativada. Suas regras estão agora no contexto."


skill_info = activate_skill_fn("formatar-saida")
print(skill_info)
print()
print(f"Skills ativas: {list(activated_skills.keys())}")
```

---

### Célula 8 — Loop com tools internas + MCP + skills

```python
import os
from enum import Enum
from typing import AsyncGenerator
from google import genai
from google.genai import types

client = genai.Client()


class EventType(str, Enum):
    CONTENT      = "content"
    TOOL_REQUEST = "tool_request"
    TOOL_RESULT  = "tool_result"
    FINISHED     = "finished"
    SKILL        = "skill"


@dataclass
class Event:
    type:    EventType
    payload: dict = field(default_factory=dict)


def list_dir(path: str) -> str:
    try:
        return "\n".join(sorted(os.listdir(path)))
    except Exception as e:
        return f"Erro: {e}"


all_tools: dict[str, BaseTool] = {
    "list_dir": LocalTool(
        "list_dir",
        "Lista arquivos em um diretório.",
        list_dir,
    ),
    "get_weather": mcp_proxies[0],
    "activate_skill": LocalTool(
        "activate_skill",
        f"Ativa uma skill (carrega conhecimento). Disponíveis:\n{registry.list_summary()}",
        activate_skill_fn,
    ),
}


def build_schema_from_tools(tools: dict[str, BaseTool]):
    return [
        types.FunctionDeclaration(
            name=t.name(),
            description=t.description(),
            parameters=types.Schema(
                type=types.Type.OBJECT,
                properties={"_arg": types.Schema(type=types.Type.STRING, description="Argumento")},
                required=[]
            )
        )
        for t in tools.values()
    ]


async def rodar_completo(prompt: str) -> AsyncGenerator[Event, None]:
    historico = [types.Content(role="user", parts=[types.Part(text=prompt)])]

    for turno in range(10):
        skills_context = "\n\n".join(activated_skills.values())
        system = (
            "Você é um assistente útil. Responda em português.\n\n"
            + (f"Skills ativadas:\n{skills_context}" if skills_context else "")
        )

        schemas = [
            types.FunctionDeclaration(
                name=t.name(),
                description=t.description(),
                parameters=types.Schema(
                    type=types.Type.OBJECT,
                    properties={
                        "city":        types.Schema(type=types.Type.STRING, description="Cidade"),
                        "path":        types.Schema(type=types.Type.STRING, description="Caminho"),
                        "skill_name":  types.Schema(type=types.Type.STRING, description="Nome da skill"),
                    },
                    required=[]
                )
            )
            for t in all_tools.values()
        ]

        response = await client.aio.models.generate_content(
            model="gemini-2.5-flash",
            contents=historico,
            config=types.GenerateContentConfig(
                system_instruction=system,
                tools=[types.Tool(function_declarations=schemas)],
            )
        )

        parts = response.candidates[0].content.parts
        historico.append(response.candidates[0].content)

        tool_calls = [p.function_call for p in parts if p.function_call]

        for p in parts:
            if p.text:
                yield Event(type=EventType.CONTENT, payload={"text": p.text})

        if not tool_calls:
            yield Event(type=EventType.FINISHED)
            return

        tool_result_parts = []
        for fc in tool_calls:
            name = fc.name
            args = dict(fc.args) if fc.args else {}

            args = {k: v for k, v in args.items() if v}

            if name == "activate_skill":
                yield Event(type=EventType.SKILL,
                            payload={"name": args.get("skill_name", "?")})
            else:
                yield Event(type=EventType.TOOL_REQUEST, payload={"name": name, "args": args})

            tool = all_tools.get(name)
            if tool is None:
                resultado = f"Tool '{name}' não existe"
            else:
                resultado = await tool.execute(args)

            yield Event(type=EventType.TOOL_RESULT, payload={"name": name, "output": resultado})

            tool_result_parts.append(
                types.Part(function_response=types.FunctionResponse(
                    name=name, response={"result": resultado}
                ))
            )

        historico.append(types.Content(role="tool", parts=tool_result_parts))


async for ev in rodar_completo(
    "Antes de qualquer coisa, ative a skill 'formatar-saida'. "
    "Depois pegue o tempo em São Paulo e me responda em tabela."
):
    if ev.type == EventType.CONTENT:
        print(f"Gemini: {ev.payload['text']}")
    elif ev.type == EventType.SKILL:
        print(f"📚 skill ativada: {ev.payload['name']}")
    elif ev.type == EventType.TOOL_REQUEST:
        print(f"→ {ev.payload['name']}")
    elif ev.type == EventType.TOOL_RESULT:
        print(f"← {ev.payload['output'][:80]}")
    elif ev.type == EventType.FINISHED:
        print("✓ concluído")
```

O agente usou:
- Uma tool MCP (`get_weather`) que vem de fora
- Uma skill (`formatar-saida`) que injetou regras de saída
- Tudo através da mesma interface `BaseTool`

---

## O que entra neste capítulo

| Conceito | O que é | Por que aparece aqui |
|---|---|---|
| MCP | Model Context Protocol — padrão aberto | extensibilidade entre projetos |
| servidor MCP | processo que expõe ferramentas | qualquer linguagem, qualquer lugar |
| cliente MCP | seu agente, conectado a servidores | descobre e usa tools externas |
| `BaseTool` | interface comum local + MCP | abstração unificada |
| `McpToolProxy` | wrap de tool MCP em BaseTool | esconde diferenças do protocolo |
| Anti-Corruption Layer (DDD) | adapter entre domínio e externo | nome desse padrão arquitetural |
| Skill | pacote de conhecimento on-demand | conteúdo carregado sob demanda |
| `SkillRegistry` | indexa skills disponíveis | descoberta dinâmica |
| `activate_skill` | tool que injeta skill no contexto | seleção on-demand pelo modelo |
| ativação on-demand | só carrega o que importa | economia de tokens |

---

## Epílogo: a jornada

Você começou com 8 linhas de código que faziam o Gemini responder.
Terminou com um sistema completo: orquestrador, sub-agentes,
sessão persistente, hooks, MCP, skills.

Em ordem, sentiu cada dor:

1. Não havia memória → criou o histórico
2. Não havia ação → declarou ferramentas
3. O loop parava → automatizou com `while True`
4. Strings soltas viraram bug → introduziu tipos
5. O Python travava → adotou async/await
6. Eventos só chegavam no final → mudou para AsyncGenerator
7. Manutenção espalhada → criou o decorator @tool
8. Faltava interface → FastAPI + SSE + HTML
9. Sem persistência → Port/Adapter para sessão
10. Sem extensibilidade interna → hooks
11. Tarefas grandes inviáveis → sub-agentes
12. Sem extensibilidade externa → MCP e skills

Cada conceito apareceu porque uma dor real o motivou.
Esse é o jeito mais durável de aprender: você não decora —
você reconstrói.

O sistema que você construiu não está completo.
Falta polimento, robustez, telemetria de produção, segurança.
Mas a arquitetura está estabelecida.
A partir daqui você sabe **por que** cada peça existe —
e isso é o que permite estendê-la com confiança.

---

## Os apêndices

A seguir você encontra:

- **Apêndice A** — Referência rápida de PowerShell
- **Apêndice B** — DDD Hexagonal em 2 páginas
- **Apêndice C** — Glossário da linguagem ubíqua
- **Apêndice D** — Código completo por fase


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap13-subagentes">← Cap 13 — Sub-agentes</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="apendiceA-powershell">Apêndice A →</a></div>
</div>
