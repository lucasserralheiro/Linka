# Fase 1 — Ambiente Local do Twenty CRM (PRODAM MVP) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rodar o Twenty CRM inteiro localmente (Postgres + Redis via Docker Compose, servidor, worker e frontend nativos) e validar um fluxo de ponta a ponta com dados fictícios, antes de qualquer investimento na Fase 2 (hospedagem gratuita na nuvem).

**Architecture:** Postgres e Redis sobem em containers Docker via `packages/twenty-docker/docker-compose.dev.yml` (infraestrutura apenas). O servidor NestJS, o worker (BullMQ) e o frontend Vite rodam nativamente via Nx, conectando-se a esses containers através de variáveis de ambiente em `packages/twenty-server/.env` e `packages/twenty-front/.env`. Esse é o setup de desenvolvimento padrão já documentado no repositório — este plano não cria infraestrutura nova, apenas guia a execução e a validação dela para o objetivo do MVP da PRODAM.

**Tech Stack:** Docker Compose, PostgreSQL 16, Redis 7, NestJS (twenty-server), Vite/React (twenty-front), Nx.

## Global Constraints

- Uso interno da PRODAM — sem exposição pública nesta fase.
- Usar apenas dados fictícios ao validar o fluxo (nenhum dado real de cidadão/servidor), conforme a spec de design.
- Configuração via variáveis de ambiente, sem hardcode de provedor — os nomes das variáveis usadas aqui (`PG_DATABASE_URL`, `REDIS_URL`, `STORAGE_TYPE`, `FRONTEND_URL`, `REACT_APP_SERVER_BASE_URL`) são os mesmos que serão reapontados para provedores cloud na Fase 2, sem mudança de código.
- Este diretório **não é um repositório git** (`git status` retorna "not a git repository"). Os passos abaixo não incluem `git commit`; se em algum momento o usuário inicializar um repositório aqui, os arquivos de ambiente (`.env`) **não devem** ser versionados — já estão listados em `.gitignore`.

---

### Task 1: Subir Postgres e Redis via Docker Compose e preparar os arquivos `.env`

**Files:**
- Modify (gerado automaticamente, não editar à mão): `packages/twenty-server/.env`
- Modify (gerado automaticamente, não editar à mão): `packages/twenty-front/.env`
- Referência (não modificar): `packages/twenty-docker/docker-compose.dev.yml`
- Referência (não modificar): `packages/twenty-utils/setup-dev-env.sh`

**Interfaces:**
- Produces: `packages/twenty-server/.env` com `PG_DATABASE_URL=postgres://postgres:postgres@localhost:5432/default` e `REDIS_URL=redis://localhost:6379` (consumido pelo servidor e pelo worker na Task 2).
- Produces: `packages/twenty-front/.env` com `REACT_APP_SERVER_BASE_URL=http://localhost:3000` (consumido pelo frontend na Task 2).
- Produces: containers Docker `twenty-dev-db-1` e `twenty-dev-redis-1` escutando em `localhost:5432` e `localhost:6379` (consumidos pela Task 2).

- [ ] **Step 1: Rodar o script oficial de setup em modo Docker**

Run: `bash packages/twenty-utils/setup-dev-env.sh --docker`

Expected: o script termina sem erro, imprimindo linhas `done:` para cada etapa (subida do Postgres/Redis via Docker, criação dos bancos `default` e `test`, cópia dos `.env`, inicialização do schema).

- [ ] **Step 2: Verificar que os containers estão de pé e saudáveis**

Run: `docker compose -f packages/twenty-docker/docker-compose.dev.yml ps`

Expected: duas linhas de serviço (`db`, `redis`), ambas com status `running (healthy)`.

- [ ] **Step 3: Verificar que os `.env` foram criados com os valores locais corretos**

Run: `grep -E "PG_DATABASE_URL|REDIS_URL" packages/twenty-server/.env`

Expected:
```
PG_DATABASE_URL=postgres://postgres:postgres@localhost:5432/default
REDIS_URL=redis://localhost:6379
```

Run: `grep "REACT_APP_SERVER_BASE_URL" packages/twenty-front/.env`

Expected:
```
REACT_APP_SERVER_BASE_URL=http://localhost:3000
```

- [ ] **Step 4: Verificar conectividade direta com o banco e o Redis**

Run: `docker compose -f packages/twenty-docker/docker-compose.dev.yml exec -T db pg_isready -U postgres -d default`

Expected: `/var/run/postgresql:5432 - accepting connections`

Run: `docker compose -f packages/twenty-docker/docker-compose.dev.yml exec -T redis redis-cli ping`

Expected: `PONG`

---

### Task 2: Subir servidor, worker e frontend, e verificar que os três respondem

**Files:**
- Nenhum arquivo novo — apenas execução dos targets Nx já existentes.

**Interfaces:**
- Consumes: `packages/twenty-server/.env` e `packages/twenty-front/.env` da Task 1.
- Produces: servidor GraphQL/REST respondendo em `http://localhost:3000`, consumido pelo frontend e pela validação da Task 3.
- Produces: frontend servido em `http://localhost:3001`, consumido pela validação manual da Task 3.

- [ ] **Step 1: Iniciar servidor, worker e frontend juntos**

Run (em um terminal dedicado, mantenha rodando): `yarn start`

Expected: logs mostrando `twenty-server` subindo primeiro, seguido de `twenty-front` (Vite) e, depois que a porta 3000 responde, o processo `worker` iniciando sem erros de conexão com Postgres/Redis. Sem stack traces de crash.

- [ ] **Step 2: Verificar que a API do servidor responde**

Run (em outro terminal, com `yarn start` ainda rodando): `curl -sf http://localhost:3000/healthz`

Expected: resposta HTTP 200 (o comando não deve retornar erro de conexão recusada).

- [ ] **Step 3: Verificar que o frontend está servindo HTML**

Run: `curl -sf -o /dev/null -w "%{http_code}\n" http://localhost:3001`

Expected: `200`

- [ ] **Step 4: Verificar nos logs que o worker conectou ao Redis sem erro**

Run: revisar o output do terminal do `yarn start` procurando a inicialização do processo worker (`queue-worker`).

Expected: nenhuma linha contendo `ECONNREFUSED`, `Redis connection error` ou stack trace relacionada a `ioredis`/`bullmq`.

---

### Task 3: Validar fluxo de ponta a ponta com dados fictícios

**Files:**
- Nenhum arquivo — validação manual via navegador, com `yarn start` da Task 2 rodando.

**Interfaces:**
- Consumes: `http://localhost:3001` (frontend) e `http://localhost:3000` (API) da Task 2.

- [ ] **Step 1: Criar o primeiro workspace e usuário admin**

Abrir `http://localhost:3001` no navegador, clicar em "Sign up" (ou "Continue with Email" se aparecer credencial pré-preenchida de ambiente de teste) e criar o primeiro workspace com um e-mail e nome **fictícios** (ex: `teste.prodam@example.com`, sem dados reais de servidor/cidadão).

Expected: após o cadastro, o usuário cai no workspace recém-criado, na tela inicial de objetos (Companies/People).

- [ ] **Step 2: Criar um registro de teste**

Na tela de "People" (ou "Companies"), clicar em "New" e criar um registro fictício (ex: nome "Fulano de Tal Teste").

Expected: o registro aparece na listagem imediatamente, sem erro no console do navegador.

- [ ] **Step 3: Confirmar persistência no Postgres local**

Run: `docker compose -f packages/twenty-docker/docker-compose.dev.yml exec -T db psql -U postgres -d default -c "SELECT schema_name FROM information_schema.schemata WHERE schema_name LIKE 'workspace_%';"`

Expected: pelo menos um schema `workspace_<uuid>` listado — confirma que o workspace criado no Step 1 gerou seu próprio schema no Postgres (arquitetura multi-tenant documentada no `CLAUDE.md` do projeto).

- [ ] **Step 4: Rodar uma automação simples e confirmar que o worker processa a fila**

Em Settings → Workflows (ou equivalente na versão instalada), criar um workflow simples disparado por "Record is created" no objeto usado no Step 2, com uma ação trivial (ex: atualizar um campo de texto). Criar um novo registro fictício para disparar o workflow.

Expected: o campo é atualizado automaticamente em poucos segundos — confirma que o worker (Task 2) está consumindo a fila do Redis local corretamente.

---

### Task 4: Documentar a configuração local validada e o que muda na Fase 2

**Files:**
- Modify: `docs/superpowers/specs/2026-07-15-mvp-hosting-prodam-design.md`

**Interfaces:**
- Consumes: resultado das Tasks 1-3 (confirmação de que a Fase 1 funciona).
- Produces: seção "Fase 1 validada em" na spec, servindo de base para a próxima etapa (plano da Fase 2, a ser escrito depois desta validação).

- [ ] **Step 1: Adicionar ao final da spec uma nota de validação**

Editar `docs/superpowers/specs/2026-07-15-mvp-hosting-prodam-design.md`, adicionando ao final do documento:

```markdown
## Fase 1 — validada em <DATA>

Ambiente local rodando via `packages/twenty-utils/setup-dev-env.sh --docker` +
`yarn start`. Fluxo de ponta a ponta (signup, criação de registro, workflow
processado pelo worker) confirmado com dados fictícios. Variáveis usadas:
`PG_DATABASE_URL`, `REDIS_URL`, `REACT_APP_SERVER_BASE_URL`, `FRONTEND_URL`.

Próximo passo: planejar a Fase 2 (hospedagem gratuita — Vercel, Render,
Supabase, Upstash, Cloudflare R2) trocando apenas os valores dessas mesmas
variáveis, sem alterar código.
```

(Substituir `<DATA>` pela data em que a Task 3 foi concluída com sucesso.)

- [ ] **Step 2: Confirmar que a nota foi salva corretamente**

Run: `tail -n 15 "docs/superpowers/specs/2026-07-15-mvp-hosting-prodam-design.md"`

Expected: a seção "Fase 1 — validada em" aparece no final do arquivo.
