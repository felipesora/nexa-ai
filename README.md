# Nexa AI

Aplicação de gestão de tarefas organizada em workspaces, com microsserviços Java (Spring Boot) e front em React.

| Pasta | O que é | Porta |
|---|---|---|
| `nexa-ai-gateway` | Spring Cloud Gateway: valida o JWT e roteia para os serviços | 8080 |
| `nexa-ai-auth-service` | Usuários, perfis, login e emissão de JWT | 8081 |
| `nexa-ai-task-service` | Workspaces, tarefas, subtarefas e tags | 8082 |
| `nexa-ai-ai-service` | Serviço de IA (em construção) | — |
| `nexa-ai-web` | Front em React + Vite + TypeScript | 5173 (dev) / 80 (prod) |
| `database/init` | Scripts SQL executados na primeira criação do banco | — |

Cada pasta `nexa-ai-*` é um **git submodule** com repositório próprio.

## Clonar

```bash
git clone --recurse-submodules https://github.com/felipesora/nexa-ai.git
```

Se já clonou sem os submodules:

```bash
git submodule update --init --recursive
```

---

## Ambiente de desenvolvimento

Compila o código local de cada serviço. É o ambiente usado no dia a dia.

### 1. Criar o `.env.develop` na raiz

```env
# MySQL
MYSQL_ROOT_PASSWORD=<senha-root>
MYSQL_DATABASE=nexa_develop
MYSQL_USER=<usuario>
MYSQL_PASSWORD=<senha>
MYSQL_PORT=3306

# Conexão dos serviços
DB_HOST=mysql
DB_PORT=3306

# JWT
JWT_EXPIRATION=2592000000
JWT_KEY=<chave-hex>

# Portas dos serviços
GATEWAY_PORT=8080
AUTH_SERVICE_PORT=8081
TASK_SERVICE_PORT=8082

# URL serviços
USUARIO_SERVICE_URL=http://auth-service:8081
TASK_SERVICE_URL=http://task-service:8082
FRONTEND_URL=http://localhost:5173

# Frontend
VITE_API_URL=http://localhost:8080
```

> Para gerar uma `JWT_KEY`: `openssl rand -hex 32`

### 2. Subir o backend

```bash
docker compose --env-file .env.develop up -d --build
```

O `--build` recompila as imagens com o código atual. Sem ele, o Docker reutiliza as imagens antigas.

### 3. Rodar o front

```bash
cd nexa-ai-web
npm install
npm run dev
```

O front lê o `.env.develop` da raiz e fica em http://localhost:5173.

### Endereços

| O quê | URL |
|---|---|
| Front | http://localhost:5173 |
| Gateway | http://localhost:8080 |
| Swagger auth | http://localhost:8081/swagger-ui.html |
| Swagger task | http://localhost:8082/swagger-ui.html |
| MySQL | localhost:3306 |

### Comandos úteis

```bash
# ver status e logs
docker compose --env-file .env.develop ps
docker compose --env-file .env.develop logs -f auth-service

# parar (mantém o banco)
docker compose --env-file .env.develop down

# parar e APAGAR o banco
docker compose --env-file .env.develop down -v
```

---

## Ambiente de produção

Usa as imagens publicadas no Docker Hub, sem compilar nada local. Só o front (nginx) fica exposto: ele serve a aplicação e repassa as chamadas `/v1/` para o gateway.

```
Navegador ──► web (nginx :80) ──/v1/──► gateway ──► auth-service / task-service ──► MySQL
```

### Como as imagens são publicadas

A cada merge na `main` de auth-service, task-service, gateway e web, o GitHub Actions roda os testes e, se eles passarem, publica a imagem no Docker Hub com 3 tags:

- `latest`
- `1.<número de commits>` (ex.: `1.42`)
- hash curto do commit (ex.: `a1b2c3d`)

Cada repositório precisa de:
- **Variable** `DOCKERHUB_USERNAME`
- **Secret** `DOCKERHUB_TOKEN` (access token do Docker Hub com permissão Read & Write)

### 1. Criar o `.env.prod` na raiz

```env
# MySQL
MYSQL_ROOT_PASSWORD=<senha-root-forte>
MYSQL_DATABASE=nexa_prod
MYSQL_USER=<usuario>
MYSQL_PASSWORD=<senha-forte>

# Conexão dos serviços
DB_HOST=mysql
DB_PORT=3306

# JWT
JWT_EXPIRATION=86400000
JWT_KEY=<chave-hex-diferente-da-de-dev>

# URL serviços
USUARIO_SERVICE_URL=http://auth-service:8081
TASK_SERVICE_URL=http://task-service:8082
FRONTEND_URL=http://localhost

# Docker Hub / imagens
DOCKERHUB_USERNAME=<usuario-dockerhub>
WEB_PORT=80
# AUTH_TAG=1.41     # opcional: fixa uma versão em vez de latest
# TASK_TAG=
# GATEWAY_TAG=
# WEB_TAG=
```

### 2. Subir

Dev e prod usam os mesmos nomes de container, então **pare o dev antes**:

```bash
docker compose --env-file .env.develop down
```

Depois:

```bash
docker compose -p nexa-ai-prod --env-file .env.prod -f docker-compose.prod.yml pull
docker compose -p nexa-ai-prod --env-file .env.prod -f docker-compose.prod.yml up -d
```

- `pull` baixa a versão mais recente das imagens.
- `-p nexa-ai-prod` separa o projeto, e portanto o volume do banco, do ambiente de dev.

Acesse em http://localhost.

### Atualizar para a versão mais nova

Rode o `pull` e o `up -d` de novo. Só os containers com imagem nova são recriados.

### Voltar para uma versão anterior

Defina a tag no `.env.prod` (ex.: `AUTH_TAG=1.41`) e rode o `up -d` de novo.

### Comandos úteis

```bash
docker compose -p nexa-ai-prod --env-file .env.prod -f docker-compose.prod.yml ps
docker compose -p nexa-ai-prod --env-file .env.prod -f docker-compose.prod.yml logs -f web gateway-service
docker compose -p nexa-ai-prod --env-file .env.prod -f docker-compose.prod.yml down
```

> Em produção, o Swagger e as portas dos serviços não ficam acessíveis de fora. Para consultar a documentação da API, use o ambiente de dev.
