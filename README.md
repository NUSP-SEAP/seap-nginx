# Senado NUSP — Sistema de Operação de Áudio

Aplicação web responsável pelos formulários de **checklist diário das salas**, **registro de operação de áudio (ROA)**, **registro de anormalidades (RAOA)** e pelo **dashboard administrativo** utilizado pela equipe de áudio do Senado.

- Para visão funcional completa (fluxos, regras de negócio, telas e perfis de usuário), consulte `documentacao_sistema_unificada.md`.
- Para detalhes de banco de dados (schemas `pessoa`, `cadastro`, `forms` e `operacao`), utilize o script `n8n_data.sql` ou uma ferramenta como DBeaver.
- Para uma visão mais aprofundada da infraestrutura (VPS, Nginx, Django, Postgres e n8n legado), veja `senado_nusp_django/detalhes_projeto_21_11_2025.md`.

---

## Arquitetura e stack

Este sistema roda em uma **VPS Linux** com **Nginx** servindo o front-end estático e fazendo proxy para o backend em **Django**, que por sua vez persiste os dados em um banco **PostgreSQL** compartilhado com o ambiente antigo do **n8n** (banco `n8n_data`).

O backend expõe endpoints REST sob `/webhook/...` para:

- Autenticação e sessão (login, whoami, logout).
- Lookups de operadores, salas, comissões e registros de operação.
- Registro de checklist diário.
- Registro de operação de áudio (sessão + entradas de operadores).
- Registro de anormalidades (RAOA).
- Dashboard administrativo (tabelas dinâmicas de operadores, checklists, operações e anormalidades).
- Cadastro de operadores e de administradores (apenas para superusuário autorizado).

A organização geral é:

- **Front-end (`app/`)**
  - HTML, CSS e JavaScript estático.
  - Páginas principais:
    - `app/index.html` (login).
    - `app/home.html` (home do operador).
    - `app/admin/*.html` (dashboard, cadastros e formulários de detalhe).
    - `app/forms/...` (checklist, operação de áudio e anormalidade).
  - Arquivos JS organizados por contexto em `app/js/app/...` e utilitários em `app/js/`.

- **Backend Django (`senado_nusp_django/`)**
  - Projeto `senado_nusp` com app `api`.
  - Camadas:
    - `api/db/` – acesso direto ao Postgres (queries e inserts).
    - `api/services/` – regras de negócio (sessão de operação, RAOA, dashboard etc.).
    - `api/views/` – views Django (endpoints `/webhook/...`).
  - Servido por **Gunicorn** por trás de **Nginx** (service `senado-api.service`).

- **Banco de dados (PostgreSQL)**
  - Banco: `n8n_data`.
  - Principais schemas:
    - `pessoa` – operadores, administradores, sessões de autenticação.
    - `cadastro` – salas, comissões.
    - `forms` – checklists diários.
    - `operacao` – registros de operação de áudio (sessões, entradas) e anormalidades (RAOA).

- **n8n (legado)**
  - Ainda disponível via Docker, mas a maior parte dos fluxos relevantes já foi portada para o Django.
  - Algumas rotas `/webhook/*` não migradas podem continuar apontando para o n8n via Nginx (ver `sites-available/app-root`).

### Detalhes da stack e integrações

- Implementação do backend em **Python 3**.
- Framework backend: **Django 5.x**.
- Servidor WSGI: **Gunicorn**, gerenciado por **systemd** (`senado-api.service`).
- Servidor HTTP e proxy: **Nginx**.
- Banco de dados: **PostgreSQL** (`n8n_data`, usuário `n8n_user`).
- Autenticação:
  - JWT (HS256) com tempo de expiração configurável.
  - Senhas armazenadas com **bcrypt**.
- CORS configurado para o domínio público do sistema (`https://senado-nusp.cloud`).
- Dependências Python do backend listadas em:
  - `senado_nusp_django/requirements.txt`.

---

## Execução do código

### Variáveis de ambiente

Para executar o backend Django é necessário configurar o arquivo `.env` na pasta `senado_nusp_django/` (ou definir variáveis equivalentes no ambiente).

Principais variáveis:

- **Django / segurança**
  - `DEBUG` – modo debug (use `False` em produção).
  - `SECRET_KEY` – chave secreta do Django.

- **Banco de dados**
  - `DATABASE_URL` – URL completa de conexão PostgreSQL  
    (ex.: `postgres://<usuario>:<senha>@127.0.0.1:5432/n8n_data`).

- **CORS / hosts**
  - `CORS_ALLOWED_ORIGINS` – origens autorizadas (ex.: `https://senado-nusp.cloud`).
  - `ALLOWED_HOSTS` – hosts permitidos pelo Django (domínio público + `localhost`).

- **JWT / autenticação**
  - `AUTH_JWT_SECRET` – segredo para assinar tokens JWT.
  - `AUTH_JWT_TTL_SEC` – tempo de vida do token em segundos (ex.: `3600`).

- **Sessão**
  - `SESSION_TOUCH_MAX_AGE_SECONDS` – janela de renovação de sessão (equivalente ao comportamento atual do n8n).

- **Arquivos públicos**
  - `FILES_DIR` – diretório de uploads no servidor (ex.: `/srv/senado/media`).
  - `FILES_URL_PREFIX` – prefixo público para os arquivos (ex.: `/files/`).

### Rodando o backend localmente

1. Criar e ativar o virtualenv:

   ```bash
   cd senado_nusp_django
   python3 -m venv .venv
   source .venv/bin/activate  # Linux/macOS
   # No Windows: .venv\Scripts\activate
   ```

2. Instalar dependências:

   ```bash
   pip install -r requirements.txt
   ```

3. Configurar o `.env` com as variáveis descritas acima.

4. Aplicar migrações:

   ```bash
   python manage.py migrate
   ```

5. Subir o servidor de desenvolvimento:

   ```bash
   python manage.py runserver 0.0.0.0:8000
   ```

   O backend ficará acessível em `http://localhost:8000`.  
   Em ambiente de desenvolvimento você pode chamar diretamente os endpoints `/webhook/...`.

### Front-end estático

O front-end vive na pasta `app/` e em produção é servido por Nginx a partir de `/var/www/app`.

Para testar localmente você pode:

- Abrir `app/index.html` diretamente no navegador, ou
- Servir o diretório `app/` com um servidor simples:

  ```bash
  cd app
  python -m http.server 8080
  ```

E então acessar `http://localhost:8080`.

> Em produção, o Nginx está configurado com:
> - `root /var/www/app;`
> - `location = /` → `index.html`
> - `location = /home` → `home.html`
> - `location /forms/` → formulários de checklist/operacao/anormalidade
> - `location ^~ /files/` → diretório público de uploads (`FILES_DIR`)

### Ambiente de produção (via systemd + Nginx)

Na VPS o backend Django é executado por um serviço systemd (`senado-api.service`) que sobe o Gunicorn na porta 8000. O Nginx faz o proxy dessas requisições.

Alguns comandos úteis (executar via SSH na VPS):

```bash
# Ver status do backend
sudo systemctl status senado-api

# Reiniciar backend depois de um deploy
sudo systemctl restart senado-api

# Ver status do Nginx
sudo systemctl status nginx
```

---

## Alterações, teste e validação

### Fluxo de alterações

- Toda alteração no código deve seguir o fluxo de branches e revisão definido pelo time (ex.: Git Flow ou similar).
- Recomenda-se:
  - Criar branches de feature a partir da `main`/`master`.
  - Abrir Merge Requests/PRs com revisão antes de fazer deploy em produção.

### Testes automatizados

Os testes automatizados do backend podem ser implementados usando o próprio framework de testes do Django ou **pytest**.

- Para rodar testes Django padrão:

  ```bash
  cd senado_nusp_django
  python manage.py test
  ```

- Caso o projeto passe a usar **pytest**, a recomendação é padronizar um comando único (ex.: `pytest`) e documentar em um `Makefile` ou script de automação.

### Validações de código

Ferramentas de estilo/qualidade (por exemplo `black`, `isort`, `flake8`, `mypy`) podem ser adicionadas ao projeto.  
Quando forem configuradas, recomenda-se padronizar comandos como:

```bash
# Exemplo (ajustar conforme ferramentas adotadas)
black .
isort .
flake8
```

### Testes de integração e aceitação

- Para validar os fluxos ponta-a-ponta (login, checklist, ROA, RAOA, dashboard), use:
  - Ambiente de homologação (quando existir), ou
  - Ambiente local + base de testes.
- A documentação funcional (`documentacao_sistema_unificada.md`) pode ser usada como base para cenários de teste.

---

## Atualização e monitoramento

### Deploy / atualização de versão

Um fluxo típico de deploy na VPS seria:

```bash
# 1. Conectar na VPS
ssh usuario@servidor

# 2. Ir até o diretório do projeto
cd /srv/senado/backend/senado_nusp_django  # ajustar se necessário

# 3. Atualizar código
git pull origin main  # ou a branch padrão do projeto

# 4. Atualizar dependências, se houver mudanças em requirements.txt
source /srv/senado/backend/venv/bin/activate
pip install -r requirements.txt

# 5. Aplicar migrações
python manage.py migrate

# 6. Reiniciar o serviço do backend
sudo systemctl restart senado-api
```

> Ajuste os caminhos e nomes de branch conforme a realidade do repositório.

### Logs e monitoramento

- Logs do backend (Gunicorn + Django) podem ser consultados via `journalctl`:

  ```bash
  # Últimas linhas do serviço Django
  sudo journalctl -u senado-api -n 200 -f
  ```

- Logs do Nginx:

  ```bash
  # Erros recentes do Nginx
  sudo journalctl -u nginx -n 200 -f
  ```

- Logs de acesso e erro em arquivo (caso configurados em `/var/log/nginx` ou `/srv/senado/logs`) também podem ser inspecionados com `tail -f`.

- Para monitoramento mais avançado (métricas, alertas), recomenda-se integrar o serviço a ferramentas de observabilidade (Prometheus, Grafana, etc.), conforme a estratégia de DevOps adotada.

---