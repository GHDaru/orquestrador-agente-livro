---
title: "Apêndice A — Referência rápida de PowerShell"
layout: default
---

# Apêndice A — Referência rápida de PowerShell

Comandos PowerShell usados ao longo do livro,
organizados por contexto.

---

## Navegação e arquivos

```powershell
# Onde estou?
Get-Location           # equivalente a pwd no Linux

# Mudar de pasta
cd C:\projetos\agente

# Voltar um nível
cd ..

# Listar arquivos
Get-ChildItem          # ou apenas: ls, dir

# Criar pasta
New-Item -ItemType Directory -Force -Path nome

# Criar arquivo vazio
New-Item -ItemType File -Force -Path arquivo.txt

# Copiar arquivo
Copy-Item origem.py destino.py

# Remover arquivo
Remove-Item arquivo.txt

# Remover pasta com tudo dentro
Remove-Item -Recurse -Force nome
```

---

## Variáveis de ambiente

```powershell
# Listar variável
echo $env:GEMINI_API_KEY

# Definir para a sessão atual
$env:GEMINI_API_KEY = "AIza..."

# Persistir (edite seu perfil)
notepad $PROFILE

# Recriar perfil se não existir
New-Item -Path $PROFILE -ItemType File -Force
```

---

## uv — Python

```powershell
# Verificar versão
uv --version

# Instalar uv
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Criar projeto
uv init nome-projeto

# Adicionar dependência
uv add google-genai
uv add fastapi uvicorn

# Adicionar várias
uv add google-genai fastapi uvicorn jupyter ipykernel

# Rodar script no ambiente do projeto
uv run python script.py

# Instalar uma versão específica de Python
uv python install 3.12

# Atualizar dependências
uv sync
```

---

## Jupyter e VS Code

```powershell
# Registrar o kernel do projeto
uv run python -m ipykernel install --user --name agente --display-name "Python (agente)"

# Abrir VS Code na pasta atual
code .
```

---

## Servidor FastAPI

```powershell
# Rodar uvicorn com reload
uv run uvicorn server:app --reload --port 8000

# Sem reload (produção)
uv run uvicorn server:app --port 8000 --host 0.0.0.0

# Servir frontend estático
python -m http.server 5500
```

---

## curl no Windows

PowerShell tem um `curl` próprio que é apenas um atalho
para `Invoke-WebRequest`. Para usar o curl real, instale-o
ou use o do Git Bash.

```powershell
# GET simples
Invoke-WebRequest http://localhost:8000/health

# POST com JSON
Invoke-RestMethod -Uri http://localhost:8000/run `
  -Method Post `
  -ContentType "application/json" `
  -Body '{"prompt": "olá"}'

# Streaming (SSE) — melhor usar curl real:
curl -N -X POST http://localhost:8000/stream `
  -H "Content-Type: application/json" `
  -d '{\"prompt\": \"liste arquivos\"}'
```

O acento grave (`) é o caractere de continuação de linha
do PowerShell — equivalente ao `\` do bash.

---

## Git

```powershell
# Inicializar repositório
git init

# Adicionar tudo
git add .

# Commit
git commit -m "mensagem"

# Conectar ao GitHub
git remote add origin https://github.com/USUARIO/REPO.git
git branch -M main
git push -u origin main

# Verificar status
git status

# Ver diferenças
git diff
```

---

## Permissões de execução

Se o PowerShell bloquear scripts:

```powershell
# Permitir scripts assinados, apenas para o usuário atual
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Ver política atual
Get-ExecutionPolicy
```


---

<div class="nav-rodape" style="display: flex; justify-content: space-between; padding: 20px 0; margin-top: 40px; border-top: 1px solid #444;">
  <div><a href="cap14-mcp-skills">← Cap 14 — MCP</a></div>
  <div><a href="index">Sumário</a></div>
  <div><a href="apendiceB-ddd">Apêndice B →</a></div>
</div>
