# Como publicar no GitHub Pages

Instruções para subir o livro no GitHub Pages com Jekyll.

---

## Pré-requisitos

- Conta no GitHub (github.com)
- Git instalado no Windows
- PowerShell

---

## Passo a passo

### 1. Inicialize o repositório Git

Abra o PowerShell **dentro da pasta `livro\`**:

```powershell
cd C:\projetos\agente\livro
git init
git add .
git commit -m "primeiro commit — estrutura do livro"
```

### 2. Crie o repositório no GitHub

1. Acesse [github.com](https://github.com)
2. Clique em **New repository** (botão verde no canto superior direito)
3. Nome sugerido: `orquestrador-agente-livro`
4. Marque como **Public** (obrigatório para GitHub Pages gratuito)
5. **Não** marque "Add a README file"
6. Clique **Create repository**

### 3. Conecte e envie

```powershell
git remote add origin https://github.com/SEU_USUARIO/orquestrador-agente-livro.git
git branch -M main
git push -u origin main
```

Substitua `SEU_USUARIO` pelo seu nome de usuário do GitHub.

### 4. Ative o GitHub Pages

1. No repositório, clique em **Settings**
2. No menu lateral, clique em **Pages**
3. Em **Source**, selecione **Deploy from a branch**
4. Em **Branch**, selecione **main** e **(root)**
5. Clique **Save**

Aguarde 1 a 2 minutos. O site estará disponível em:

```
https://SEU_USUARIO.github.io/orquestrador-agente-livro/
```

---

## Ajuste obrigatório no _config.yml

Antes do primeiro push, edite `_config.yml` e descomente as linhas:

```yaml
baseurl: "/orquestrador-agente-livro"
url: "https://SEU_USUARIO.github.io"
```

Substitua `SEU_USUARIO` pelo seu nome de usuário do GitHub.

---

## Adicionar um novo capítulo

1. Crie o arquivo `.md` na pasta `livro\`
2. Certifique-se de que começa com o front matter:
   ```markdown
   ---
   title: "Capítulo X — Título"
   layout: default
   ---
   ```
3. Adicione o link em `index.md`
4. Faça commit e push:
   ```powershell
   git add .
   git commit -m "adiciona capítulo X"
   git push
   ```

O GitHub Pages atualiza automaticamente em 1-2 minutos.

---

## Temas disponíveis no _config.yml

```yaml
theme: jekyll-theme-hacker     # fundo escuro — atual
theme: jekyll-theme-cayman     # azul, limpo
theme: jekyll-theme-minimal    # minimalista, branco
theme: jekyll-theme-slate      # cinza escuro
theme: minima                  # padrão Jekyll
```

Para trocar: edite `_config.yml`, altere o `theme`, faça commit e push.
