# Deploy gratuito do Twenty CRM (protótipo) — Vercel + Render + Neon + Upstash

Guia para colocar o Twenty no ar de graça para testes internos da empresa. Arquitetura:

- **Vercel** — hospeda o frontend (React/Vite)
- **Render** (free) — hospeda o backend (API GraphQL)
- **Neon** — Postgres gratuito
- **Upstash** — Redis gratuito

**Limitações que você já topou em aceitar (protótipo, não produção):**
- A API do Render "dorme" após 15 min sem uso — primeiro acesso depois de um tempo parado demora 30-60s.
- Não há worker gratuito no Render, então **e-mails automáticos e workflows agendados não disparam sozinhos**. CRUD, telas, login e uso manual funcionam normalmente.
- Arquivos anexados/logo (upload local) não persistem de forma confiável — o disco do Render free é efêmero. Para um protótipo isso raramente importa.
- Vercel Hobby é formalmente para uso pessoal/não-comercial; para um teste interno curto o risco de problema é baixo, mas fica registrado aqui.

---

## Pré-requisito: subir o código pro GitHub

Render e Vercel fazem deploy a partir de um repositório Git. Sua pasta local ainda não é um repositório Git. No terminal, na raiz do projeto:

```bash
git init
git add .
git commit -m "Initial commit"
```

Crie um repositório **privado** no GitHub (github.com/new) e conecte:

```bash
git remote add origin https://github.com/SEU_USUARIO/twenty-crm.git
git branch -M main
git push -u origin main
```

---

## Passo 1 — Banco de dados (Neon)

1. Crie conta em [neon.com](https://neon.com) → **Create a project**.
2. No painel do projeto, na seção **Connection string**, copie a conexão **direta** (não a "pooled"/`-pooler`) — o Twenty usa TypeORM com pool próprio, então precisa da conexão direta. O formato é:
   ```
   postgresql://usuario:senha@ep-xxxx.região.aws.neon.tech/dbname?sslmode=require
   ```
3. Guarde essa URL — é o seu `PG_DATABASE_URL`.

## Passo 2 — Redis (Upstash)

1. Crie conta em [upstash.com](https://upstash.com) → **Create Database** → Redis → escolha uma região.
2. Na aba **Details** do banco, copie a connection string no formato `rediss://default:SENHA@HOST:PORTA` (é a string TCP compatível com ioredis, não a REST URL/token que aparece ao lado).
3. Guarde essa URL — é o seu `REDIS_URL`.

## Passo 3 — Inicializar o schema do banco (do seu próprio PC)

Seu computador já tem tudo instalado (Node, Yarn, o build do `twenty-shared`). Vamos usar ele só para rodar a migration inicial contra o banco do Neon — depois disso o Render só precisa iniciar o servidor.

No terminal, na raiz do projeto:

```bash
PG_DATABASE_URL="postgresql://usuario:senha@ep-xxxx.região.aws.neon.tech/dbname?sslmode=require" \
REDIS_URL="rediss://default:SENHA@HOST:PORTA" \
APP_SECRET="$(openssl rand -base64 32)" \
NODE_ENV=production \
npx nx run twenty-server:database:init
```

Guarde o valor de `APP_SECRET` que foi gerado (rode `echo $APP_SECRET` antes de fechar o terminal, ou gere um separadamente com `openssl rand -base64 32` e reuse o mesmo nos três comandos) — você vai precisar dele no Render.

Isso cria as tabelas e schemas no banco Neon remoto. Se der certo, o comando termina sem erro.

## Passo 4 — Backend no Render

1. Crie conta em [render.com](https://render.com) → **New** → **Web Service** → conecte o repositório do GitHub.
2. Configure:
   - **Root Directory**: deixe em branco (raiz do repo)
   - **Runtime**: Node
   - **Build Command**:
     ```
     corepack enable && yarn install --immutable && npx nx build twenty-shared && npx nx run twenty-server:build
     ```
   - **Start Command**:
     ```
     node packages/twenty-server/dist/main
     ```
   - **Instance Type**: Free
3. Em **Environment Variables**, adicione:
   | Nome | Valor |
   |---|---|
   | `NODE_ENV` | `production` |
   | `NODE_VERSION` | `24.16.0` |
   | `PG_DATABASE_URL` | (a URL direta do Neon) |
   | `REDIS_URL` | (a URL do Upstash) |
   | `APP_SECRET` | (o mesmo valor usado no Passo 3) |
   | `SERVER_URL` | `https://SEU-APP.onrender.com` (você só sabe a URL final depois do primeiro deploy — pode editar depois) |
   | `FRONTEND_URL` | `https://SEU-APP.vercel.app` (idem — edite depois de criar o projeto na Vercel) |
   | `DISABLE_DB_MIGRATIONS` | `true` (já rodou no Passo 3) |
4. Clique em **Create Web Service**. O primeiro build demora — é um monorepo grande.
5. Quando terminar, copie a URL pública que o Render gerou (algo como `https://twenty-crm-xxxx.onrender.com`) e volte para atualizar a variável `SERVER_URL` com essa URL exata, salvando (isso redesploya automaticamente).

## Passo 5 — Frontend na Vercel

1. Crie conta em [vercel.com](https://vercel.com) → **Add New** → **Project** → importe o mesmo repositório.
2. Na tela de configuração:
   - **Framework Preset**: Vite
   - **Root Directory**: deixe em branco (raiz do repo)
   - Clique em **Override** em Build Command e Output Directory:
     - **Install Command**: `echo "skip"` (o install roda dentro do build command abaixo)
     - **Build Command**:
       ```
       corepack enable && yarn install --immutable && npx nx build twenty-shared && npx nx run twenty-front:build
       ```
     - **Output Directory**: `packages/twenty-front/build`
3. Em **Environment Variables**, adicione:
   | Nome | Valor |
   |---|---|
   | `REACT_APP_SERVER_BASE_URL` | `https://SEU-APP.onrender.com` (a URL do Render do Passo 4) |
   | `VITE_BUILD_SOURCEMAP` | `false` |
4. Clique em **Deploy**.
5. Depois do deploy, crie um arquivo `vercel.json` na raiz do repositório (fora de qualquer pasta de package) com este conteúdo, para as rotas do React Router não darem 404 ao recarregar a página:
   ```json
   {
     "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
   }
   ```
   Commite e dê push — a Vercel redesploya sozinha.
6. Copie a URL final da Vercel (`https://SEU-APP.vercel.app`) e volte no Render para atualizar `FRONTEND_URL` com essa URL exata (necessário para o CORS liberar as requisições do frontend).

## Passo 6 — Testar

1. Abra a URL da Vercel no navegador.
2. Se a API estava "dormindo", a primeira tentativa de login pode demorar até 1 minuto — é o Render acordando o serviço. Tente de novo se der timeout na primeira vez.
3. Faça login/crie workspace e confira se companies/people/etc. carregam normalmente.

---

## Se depois quiser resolver as limitações

- **E-mails e workflows automáticos**: só funcionam com um worker rodando 24/7. No Render isso custa a partir de $7/mês (plano pago para Background Worker). A alternativa 100% gratuita é migrar tudo para uma VPS Oracle Cloud Always Free rodando o `docker-compose` completo (server + worker + Postgres + Redis juntos, sem dormir). Posso montar esse guia depois se topar.
- **Cold start de 30-60s**: resolve com plano pago do Render (a partir de $7/mês, sem sleep) ou migrando pra Oracle Cloud.
- **Arquivos que não persistem**: configure `STORAGE_TYPE=s3` com um bucket gratuito do Cloudflare R2 (10GB grátis) em vez do storage local.
