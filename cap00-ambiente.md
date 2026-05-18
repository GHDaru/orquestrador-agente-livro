---
title: "Capítulo 0 — Ambiente e ferramentas"
layout: default
---

# Capítulo 0 — Ambiente e ferramentas

> *Antes de escrever a primeira linha de código do agente,
> vamos garantir que o chão está firme.*

Este capítulo não tem conceitos novos de agentes ou IA.
Tem configuração. É tentador pular — não pule.
Um ambiente mal configurado transforma erros de 30 segundos
em horas de investigação.

Ao final deste capítulo você vai ter:

- Python moderno gerenciado pelo `uv`
- Uma API key do Gemini funcionando
- Jupyter Notebook configurado no VS Code
- Um script de verificação que confirma tudo
- A estrutura de pastas que vai crescer ao longo do livro

---

## 0.1 — O sistema

Este livro foi escrito para Windows com PowerShell
(o terminal moderno do Windows, mais poderoso que o Prompt de Comando).

Você vai precisar de:

- Windows 10 ou 11
- PowerShell 5.1+ (já vem instalado no Windows 10/11)
  ou Windows Terminal (recomendado — baixe na Microsoft Store)
- VS Code (Visual Studio Code) — editor de código gratuito da Microsoft
- Conexão com a internet para as chamadas de API

### Abrindo o PowerShell

Pressione `Win + X` e escolha **Windows PowerShell**
ou **Terminal** (se tiver o Windows Terminal instalado).

Verifique a versão:

```powershell
$PSVersionTable.PSVersion
```

Se mostrar 5.1 ou superior, está ótimo.

### Política de execução de scripts

O Windows por padrão bloqueia a execução de scripts.
Para o `uv` e outras ferramentas funcionarem, rode:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

Confirme com `S` quando perguntado.

---

## 0.2 — Python com uv

### O problema com pip e virtualenv

Se você já usou Python antes, provavelmente conhece este ritual:

```powershell
python -m venv .venv
.venv\Scripts\activate
pip install requests
# ... aguarda ...
# espera, qual python está ativo mesmo?
```

`pip` é lento. `virtualenv` precisa ser ativado manualmente
a cada sessão. O arquivo `requirements.txt` não trava versões
de dependências indiretas, o que pode causar comportamentos
diferentes em máquinas diferentes.

### O que é uv

`uv` é um gerenciador de projetos Python escrito em Rust
(linguagem de programação conhecida por velocidade e segurança).
Faz o que `pip`, `virtualenv`, `pip-tools` e `pyenv` fazem —
em um único comando, muito mais rápido.

Instalar um pacote com `pip` pode demorar 30 segundos.
Com `uv`, demora 2. O ambiente virtual é criado e
gerenciado automaticamente. A versão do Python fica
travada no projeto.

### Instalando uv no Windows

No PowerShell:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Feche o PowerShell e abra novamente. Depois verifique:

```powershell
uv --version
# uv 0.5.x (ou mais recente)
```

### Os três comandos que você vai usar

```powershell
# Cria um projeto novo com pyproject.toml
uv init nome-do-projeto

# Adiciona uma dependência (e já instala)
uv add google-genai

# Roda um arquivo Python no ambiente do projeto
uv run python meu_script.py
```

### Criando o projeto do livro

```powershell
New-Item -ItemType Directory -Force -Path C:\projetos
cd C:\projetos

mkdir agente
cd agente
uv init .
Remove-Item hello.py
```

### Adicionando as dependências

```powershell
uv add google-genai fastapi uvicorn jupyter ipykernel
```

O `jupyter` e `ipykernel` são necessários para os notebooks
que vamos usar a partir do Capítulo 1.

O `pyproject.toml` resultante:

```toml
[project]
name = "agente"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "google-genai>=1.0.0",
    "fastapi>=0.115.0",
    "uvicorn>=0.30.0",
    "jupyter>=1.0.0",
    "ipykernel>=6.0.0",
]
```

---

## 0.3 — A API key do Gemini

### O que é uma API key

Uma API key (chave de API — uma senha que identifica quem está
fazendo requisições a um serviço) é o que o Google usa
para saber que é você chamando o Gemini.

### Criando a key

1. Acesse [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
2. Faça login com sua conta Google
3. Clique em **Create API key**
4. Copie a chave gerada — ela começa com `AIza`

### Limites da camada gratuita

| Modelo | Req/dia (free) | Req/min (free) |
|---|---|---|
| `gemini-2.5-flash` | 1.500 | 15 |
| `gemini-2.5-flash-lite` | 1.500 | 30 |
| `gemini-2.5-pro` | 50 | 5 |

### Configurando no PowerShell

Para a sessão atual:

```powershell
$env:GEMINI_API_KEY = "AIza..."
```

Para persistir entre sessões:

```powershell
# Cria o arquivo de perfil se não existir
New-Item -Path $PROFILE -ItemType File -Force
notepad $PROFILE
```

No Bloco de Notas, adicione e salve:

```powershell
$env:GEMINI_API_KEY = "AIza..."
```

Verifique:

```powershell
echo $env:GEMINI_API_KEY
```

> **Nunca coloque a API key diretamente no código.**
> O SDK do Gemini lê `GEMINI_API_KEY` automaticamente do ambiente.

---

## 0.4 — VS Code e Jupyter Notebook

### Instalando o VS Code

Baixe em [code.visualstudio.com](https://code.visualstudio.com)
e execute o instalador. Marque a opção
"Add to PATH" durante a instalação.

Após instalar, abra o VS Code e instale as extensões necessárias:

### Extensões necessárias

No VS Code, abra o painel de extensões com `Ctrl+Shift+X`
e instale:

| Extensão | Publisher | Para que serve |
|---|---|---|
| Python | Microsoft | suporte completo a Python |
| Jupyter | Microsoft | rodar notebooks `.ipynb` no VS Code |
| Pylance | Microsoft | autocomplete e verificação de tipos |

### O que é um Jupyter Notebook

Jupyter Notebook é um formato de documento (`.ipynb`) que mistura
células de texto explicativo (Markdown) com células de código Python
executável — cada célula pode ser rodada individualmente,
e o resultado aparece logo abaixo dela.

Isso é ideal para aprender: você executa um trecho, vê o resultado,
lê a explicação, e executa o próximo — sem precisar rodar
o arquivo inteiro de uma vez.

### Configurando o kernel Python no VS Code

O kernel é o processo Python que executa o código das células.
Precisamos apontar para o ambiente do `uv` do projeto.

**Passo 1 — Registre o ambiente do projeto como kernel:**

```powershell
cd C:\projetos\agente
uv run python -m ipykernel install --user --name agente --display-name "Python (agente)"
```

**Passo 2 — Abra o VS Code na pasta do projeto:**

```powershell
code .
```

**Passo 3 — Crie um notebook de teste:**

No VS Code, pressione `Ctrl+Shift+P`, digite
`Create: New Jupyter Notebook` e pressione Enter.

**Passo 4 — Selecione o kernel:**

No canto superior direito do notebook, clique em
**Select Kernel** → **Python Environments** →
escolha `Python (agente)`.

**Passo 5 — Teste:**

Na primeira célula do notebook, escreva e execute com `Ctrl+Enter`:

```python
import sys
print(sys.executable)
# Deve mostrar um caminho dentro de C:\projetos\agente\.venv
```

Se mostrar o caminho correto, o kernel está configurado.

### Estrutura de cada capítulo

A partir do Capítulo 1, cada capítulo tem dois arquivos:

```
cap01\
  ├── cap01.ipynb    ← notebook: experimentos passo a passo
  └── call.py        ← código final limpo
```

**O notebook** é onde você vai experimentar.
Cada seção do capítulo descreve o que colocar em cada célula —
você copia o código para a célula correspondente e executa.

**O arquivo `.py`** é o código final do capítulo,
limpo e sem comentários pedagógicos.
Você só vai criá-lo quando terminar o notebook.

### Como criar o notebook de cada capítulo

Para cada capítulo novo:

```powershell
New-Item -ItemType Directory -Force -Path cap01
```

No VS Code, com a pasta `cap01` selecionada no Explorer,
pressione `Ctrl+Shift+P` → `Create: New Jupyter Notebook`.
Salve como `cap01.ipynb` dentro da pasta `cap01`.

Selecione o kernel `Python (agente)` no canto superior direito.

---

## 0.5 — O frontend: HTML puro primeiro, React depois

Nas primeiras fases do livro, o frontend é um único arquivo `.html`.
Sem build step, sem `node_modules`, sem ferramentas extras.
Você salva o arquivo, abre no browser, e funciona.

### Quando React entra

Na Parte III, quando o frontend tiver estado complexo
(streaming de eventos, aprovação de ferramentas, histórico),
o HTML puro vai mostrar suas limitações. Aí React entra —
não como requisito do livro, mas como solução para uma dor real.

### Node.js e pnpm (para a Parte III)

Baixe Node.js LTS (Long-Term Support — versão com suporte
de longo prazo) em [nodejs.org](https://nodejs.org).

Após instalar:

```powershell
node --version   # v20.x.x ou mais recente
npm install -g pnpm
pnpm --version   # 9.x.x
```

> pnpm (performant npm) usa links simbólicos ao invés de copiar
> arquivos — instala mais rápido e ocupa menos espaço em disco.

---

## 0.6 — Estrutura do projeto

```
agente\
├── pyproject.toml         ← dependências Python (uv)
├── AGENT.md               ← contexto persistente do agente
│
├── cap01\
│   ├── cap01.ipynb        ← notebook de experimentos
│   └── call.py            ← código final
├── cap02\
│   ├── cap02.ipynb
│   └── chat.py
├── cap03\
│   ├── cap03.ipynb
│   └── tool_manual.py
│   ... (continua)
│
└── servidor\              ← Parte III em diante
    ├── server.py
    └── frontend\
        └── index.html
```

Crie agora:

```powershell
New-Item -ItemType File -Force -Path AGENT.md
New-Item -ItemType Directory -Force -Path cap01
```

---

## 0.7 — Verificação final

Crie o arquivo `verificar.py`:

```python
# verificar.py
import sys
import os

print("Verificando ambiente...\n")

versao = sys.version_info
assert versao >= (3, 11), (
    f"Python 3.11+ necessário. Você tem {versao.major}.{versao.minor}."
)
print(f"✓ Python {versao.major}.{versao.minor}.{versao.micro}")

key = os.environ.get("GEMINI_API_KEY", "")
assert key.startswith("AIza"), (
    "GEMINI_API_KEY não configurada.\n"
    "No PowerShell: $env:GEMINI_API_KEY = 'AIza...'"
)
print(f"✓ GEMINI_API_KEY configurada ({key[:8]}...)")

try:
    from google import genai
    print("✓ google-genai instalado")
except ImportError:
    print("✗ google-genai não instalado. Rode: uv add google-genai")
    sys.exit(1)

try:
    import jupyter
    print("✓ jupyter instalado")
except ImportError:
    print("✗ jupyter não instalado. Rode: uv add jupyter ipykernel")

try:
    import fastapi
    print("✓ fastapi instalado")
except ImportError:
    print("✗ fastapi não instalado. Rode: uv add fastapi")

try:
    import uvicorn
    print("✓ uvicorn instalado")
except ImportError:
    print("✗ uvicorn não instalado. Rode: uv add uvicorn")

print("\nTestando chamada à API do Gemini...")
try:
    client = genai.Client()
    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents="Responda apenas com a palavra: funcionando"
    )
    assert "funcionando" in response.text.lower()
    print(f"✓ API respondeu: '{response.text.strip()}'")
except Exception as e:
    print(f"✗ Erro na chamada à API: {e}")
    sys.exit(1)

print()
print("─" * 40)
print("Ambiente configurado. Pode começar.")
print("─" * 40)
```

Rode:

```powershell
uv run python verificar.py
```

### Problemas comuns

**`uv : O termo 'uv' não é reconhecido`**
```powershell
$env:PATH += ";$env:USERPROFILE\.local\bin"
```

**`GEMINI_API_KEY não configurada`**
```powershell
$env:GEMINI_API_KEY = "AIza..."
```

**`ModuleNotFoundError: No module named 'google'`**
```powershell
uv add google-genai
```

**Kernel não aparece no VS Code**
```powershell
uv run python -m ipykernel install --user --name agente --display-name "Python (agente)"
```
Reinicie o VS Code após instalar.

---

## Resumo do capítulo

| Ferramenta | Versão mínima | Para que serve |
|---|---|---|
| Windows | 10 ou 11 | sistema operacional |
| PowerShell | 5.1+ | terminal |
| Python | 3.11+ | linguagem do backend |
| uv | 0.5+ | gerenciar Python e dependências |
| VS Code | qualquer | editor + ambiente Jupyter |
| Extensão Python | — | suporte Python no VS Code |
| Extensão Jupyter | — | rodar notebooks no VS Code |
| google-genai | 1.0+ | SDK do Gemini |
| jupyter + ipykernel | — | notebooks interativos |
| fastapi + uvicorn | — | servidor HTTP (Parte III) |
| Node.js + pnpm | 20+ / 9+ | frontend React (Parte III) |

## O que vem a seguir

No **Capítulo 1** você vai criar o primeiro notebook
e fazer o Gemini responder em 8 linhas de Python —
célula por célula, vendo o resultado de cada uma.
