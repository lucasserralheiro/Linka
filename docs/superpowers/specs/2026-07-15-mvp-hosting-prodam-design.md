# MVP do Twenty CRM para a PRODAM — hospedagem gratuita

## Contexto

A PRODAM (empresa pública de TI) baixou o Twenty CRM (open source, AGPLv3 com
partes Enterprise sob licença comercial separada) e quer validar um MVP de
uso **interno** antes de qualquer decisão de infraestrutura definitiva.
Requisitos que guiaram este design:

- Uso interno da PRODAM (não é um produto oferecido a terceiros/prefeituras/
  cidadãos nesta fase) — isso mantém as obrigações da AGPL simples: não há
  necessidade de publicar código-fonte modificado para o público, porque não
  há distribuição/oferta de serviço a terceiros.
- Custo zero ou o mais próximo disso.
- O MVP hospedado precisa parecer "sempre disponível" (sem o delay de
  cold start perceptível na maior parte do tempo).
- Preparar primeiro um ambiente local (docker-compose) para validar a
  aplicação antes de investir tempo subindo para a nuvem.
- Configuração por variáveis de ambiente, sem prender o projeto a um único
  provedor de banco/storage (poder trocar Supabase ↔ Neon, R2 ↔ outro S3
  compatível, sem mudar código).

## Licenciamento (achado relevante)

O repositório é majoritariamente **AGPLv3**. Arquivos marcados com o
comentário `/* @license Enterprise */` no topo (SSO, domínios customizados,
papéis/permissões granulares por registro, billing) são de licença comercial
separada e ficam bloqueados sem uma `ENTERPRISE_KEY` paga — não afetam o
núcleo (objetos, pipelines, workflows, automações, API GraphQL, usuários
ilimitados), que é livre. `IS_BILLING_ENABLED` vem desligado por padrão em
self-hosting.

## Fases de execução

1. **Fase 1 — Local**: rodar a aplicação inteira via
   `packages/twenty-docker/docker-compose.dev.yml` (Postgres + Redis locais,
   `STORAGE_TYPE=local`). Objetivo: validar que a aplicação funciona, entender
   a configuração via env vars, popular dados de teste (fictícios), sem
   nenhum custo ou dependência de provedor externo.
2. **Fase 2 — MVP hospedado gratuito**: só depois da Fase 1 validada,
   replicar a mesma stack trocando apenas os valores das env vars para os
   provedores cloud gratuitos descritos abaixo. Nenhuma mudança de código
   entre as fases — só configuração.

## Arquitetura (Fase 2 — hospedado)

```
Usuário → Vercel (twenty-front, build estático do Vite)
              │  HTTPS/GraphQL via REACT_APP_SERVER_BASE_URL
              ▼
        Render Web Service (imagem oficial twentycrm/twenty:latest)
              │  roda servidor + worker no MESMO container
              │  (ver "Achado: Render free não tem Background Worker")
              ├──▶ Supabase (Postgres free)
              ├──▶ Upstash (Redis free — filas BullMQ e cache)
              └──▶ Cloudflare R2 (arquivos/anexos, free — S3-compatible)

UptimeRobot (free) → pinga um endpoint que consulta o banco a cada ~5 min,
                      evitando a hibernação do Render (15 min) e a pausa do
                      Supabase (7 dias de inatividade real de query)
```

### Achado: Render free não tem "Background Worker"

O plano gratuito do Render só oferece Web Service, Static Site e Cron Job —
não existe tipo de serviço "Background Worker" no free tier em nenhuma
condição. Como o Twenty precisa de um processo worker (BullMQ) separado do
servidor para processar e-mails, automações e importações, a solução é
sobrescrever o comando Docker do único Web Service gratuito para rodar
servidor e worker no mesmo container (worker em background, servidor em
foreground escutando a porta que o Render espera). O comando exato de
inicialização é detalhe de implementação, não deste documento de design.

### Por que cada provedor

| Peça | Provedor | Por quê |
|---|---|---|
| Frontend | Vercel Hobby | Escolha explícita da PRODAM, com risco de ToS aceito (ver Riscos) |
| API + worker | Render free (1 Web Service) | Único jeito de rodar worker sem custo |
| Banco | Supabase Postgres free | Não tem teto de horas de computação — compatível com manter "sempre ativo" via ping. Neon foi avaliado e tem cold start mais rápido, mas seu free tier tem teto de 100h de computação/mês, que um ping constante estouraria em poucos dias — por isso não é a melhor escolha para "sempre disponível", embora continue disponível como alternativa fácil de trocar (mesma `PG_DATABASE_URL`) |
| Fila/cache | Upstash Redis free | 256MB, 500k comandos/mês, conectar via `rediss://` (TLS) |
| Arquivos/anexos | Cloudflare R2 free (10GB) | Necessário: Render free não tem disco persistente — sem isso, anexos somem a cada redeploy/restart |
| Manter acordado | UptimeRobot free | Ping a cada ~5 min num endpoint que realmente consulta o Postgres (não só um health check estático) |

## Configuração — mesmas chaves, valores diferentes por ambiente

O Twenty já é configurável 100% via variáveis de ambiente. A regra é: nunca
hardcodar provedor no código, só trocar valores de `.env` (não versionado)
por ambiente.

| Variável | Fase 1 (local) | Fase 2 (hospedado) |
|---|---|---|
| `PG_DATABASE_URL` | Postgres em container (`docker-compose.dev.yml`) | Supabase (ou Neon, trocável) |
| `REDIS_URL` | Redis em container | Upstash (`rediss://...`) |
| `STORAGE_TYPE` + `STORAGE_S3_*` | `local` | `s3` → Cloudflare R2 |
| `SERVER_URL` / `FRONTEND_URL` | `localhost` | URLs reais do Render/Vercel |
| `REACT_APP_SERVER_BASE_URL` (build-time do front) | `localhost:3000` | URL pública do Render |

## Riscos aceitos e observações de compliance

- **Vercel Hobby**: os termos proíbem uso comercial/institucional; risco
  aceito conscientemente pela PRODAM para esta fase de teste.
- **Sem backup automático**: Supabase/Neon free não fazem backup — ambiente
  de teste, não de produção.
- **512MB de RAM no Render free**, com servidor+worker no mesmo processo —
  se ocorrer OOM/crash sob carga, é sinal de migrar para plano pago ou
  infraestrutura própria da PRODAM.
- **Dados**: por ser empresa pública, recomenda-se **não usar dados reais de
  cidadãos/servidores** neste ambiente com provedores gratuitos
  internacionais — usar dados fictícios até uma avaliação formal de
  segurança/LGPD antes de qualquer uso com dados reais.
- **Recursos Enterprise bloqueados** (SSO, papéis granulares, domínio
  customizado): não chegam a ser necessários para um MVP interno com poucos
  usuários testando.

## Plano de validação (smoke test pós-deploy da Fase 2)

1. Frontend carrega no domínio Vercel e completa o signup do primeiro
   workspace.
2. Login funciona (fluxo de e-mail) e o GraphQL responde sem erro de CORS
   entre Vercel e Render.
3. Criar um registro (ex: empresa/pessoa) e confirmar que persiste no
   Supabase.
4. Rodar uma automação/workflow simples e confirmar que o worker (rodando
   junto no mesmo container) processa a fila no Upstash.
5. Fazer upload de um anexo, forçar um redeploy no Render, e confirmar que
   o anexo continua acessível (prova que o R2 está funcionando, não o disco
   efêmero).
6. Deixar 20+ minutos parado, acessar de novo e confirmar que o UptimeRobot
   evitou a hibernação.
