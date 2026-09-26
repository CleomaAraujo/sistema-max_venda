# RepresentaPro CRM

SaaS multitenant de CRM + Força de Vendas + Gestão Comercial para escritórios
de representação comercial.

## Estrutura

```
representapro-saas/
├── frontend/    React + TypeScript + Vite + React Router
├── backend/     Node.js + TypeScript + Express + PostgreSQL
└── database/    migrations e seeds
```

## Como rodar (desenvolvimento)

Backend:
```
cd backend
cp .env.example .env    # preencha DATABASE_URL com seu Postgres local
npm install
npm run dev              # http://localhost:3333/health
```

Frontend:
```
cd frontend
cp .env.example .env
npm install
npm run dev               # http://localhost:5173
```

## Banco de dados (Passo 2)

Com o `DATABASE_URL` já preenchido no `backend/.env`:
```
cd backend
npm run migrate:up   # cria tenants, roles, permissions, role_permissions, users
npm run seed          # cria os 5 perfis (roles), o catálogo de permissões,
                       # o tenant "LBM Representação Comercial" e o usuário
                       # administrador (a senha provisória aparece no console)
```
Para desfazer a última migration: `npm run migrate:down`.

## Row Level Security (Passo 3)

A tabela `users` (e toda tabela comercial que vier a partir da Fase 2) usa
`FORCE ROW LEVEL SECURITY`: mesmo o usuário dono das tabelas — que normalmente
é o único usuário de banco disponível num PaaS gerenciado — fica sujeito à
policy. Duas formas de abrir uma conexão no backend:

- `withTenant(tenantId, fn)` — uso normal, em toda rota autenticada: seta
  `app.tenant_id` na sessão, e as policies só deixam passar linhas daquele tenant.
- `withServiceContext(fn)` — bypass controlado, só para fluxos internos e
  confiáveis: login por e-mail (antes de saber o tenant), seeds/scripts de
  manutenção, e futuramente o painel administrativo da plataforma (Fase 6).
  **Nunca** usar isso num caminho que aceite tenant_id vindo do usuário final.

Sem setar nenhum dos dois, a query simplesmente não vê nenhuma linha — o
padrão é fail-closed, não fail-open.

## Autenticação (Passo 4)

- **Access token** (JWT curto, 15 min): fica só em memória no frontend
  (nunca `localStorage`), enviado como `Authorization: Bearer ...` em cada
  request. Reduz a superfície de ataque de XSS — mesmo que um script
  malicioso rode na página, não tem um `localStorage` pra ler.
- **Refresh token** (JWT longo, 30 dias): vai num cookie `httpOnly`,
  `sameSite=strict`, restrito ao path `/auth` — o JavaScript da página não
  consegue lê-lo nem enviá-lo para outro domínio.
- **Rotação**: cada `/auth/refresh` invalida o refresh token usado e emite
  um par novo (access + refresh). Isso limita o estrago se um refresh token
  vazar — ele só serve uma vez.
- **Revogação**: cada refresh token vira uma linha em `refresh_tokens`
  (hash SHA-256, nunca o token em texto puro); `/auth/logout` marca a linha
  como revogada.
- **Descoberta do tenant pelo e-mail**: como a tela de login não pergunta
  "empresa", o backend busca o usuário pelo e-mail via `withServiceContext`
  (bypass controlado) — depois disso, tudo usa `withTenant` normalmente.

Testando por `curl`:
```
curl -c cookies.txt -X POST http://localhost:3333/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"cleomar@lbm.com.br","password":"<senha do seed>"}'

curl -b cookies.txt -X POST http://localhost:3333/auth/refresh
curl -b cookies.txt -X POST http://localhost:3333/auth/logout
```

## Deploy no Fly.io

Backend e frontend são dois apps Fly separados (cada um com seu próprio
`Dockerfile` + `fly.toml`), e o Postgres é um terceiro app gerenciado pela
própria Fly (Fly Postgres).

Resumo dos comandos (veja a ordem completa no card do chat):
```
# 1) Postgres
fly postgres create --name representapro-db --region gru

# 2) Backend
cd backend
fly launch --no-deploy    # confirma/edita o fly.toml já existente
fly postgres attach representapro-db --app representapro-backend
fly secrets set JWT_ACCESS_SECRET=... JWT_REFRESH_SECRET=... \
  CORS_ORIGIN=https://representapro-frontend.fly.dev NODE_ENV=production \
  --app representapro-backend
fly deploy --app representapro-backend

# 3) Rodar migrations + seed contra o Postgres da Fly, a partir da sua máquina
fly proxy 5433:5432 --app representapro-db &
DATABASE_URL="postgresql://...(fly postgres connect te dá essa string)...@localhost:5433/..." \
  npm run migrate:up
DATABASE_URL="..." npm run seed

# 4) Frontend (edite VITE_API_URL no fly.toml antes, com a URL real do passo 2)
cd ../frontend
fly launch --no-deploy
fly deploy --app representapro-frontend
```

`fly postgres attach` já cria a secret `DATABASE_URL` no app do backend
automaticamente — não precisa configurar isso à mão. As migrations/seed
rodam da sua máquina (não dentro do container), usando `fly proxy` para
abrir um túnel até o Postgres da Fly — assim a imagem de produção do
backend continua enxuta, sem `node-pg-migrate`/`tsx` dentro dela.

## Deploy no Render

O `render.yaml` na raiz é um Blueprint do Render: define os dois serviços
(backend Node e frontend estático) e o Postgres gerenciado de uma vez só,
sem precisar de Docker — o Render builda direto com `npm install && npm run build`.

1. No dashboard do Render: **New > Blueprint**, aponte para este repositório
   (precisa estar num Git remoto — GitHub/GitLab) e clique **Apply**.
2. O Render cria os 3 recursos (`representapro-db`, `representapro-backend`,
   `representapro-frontend`) e já injeta `DATABASE_URL` no backend
   automaticamente, além de gerar `JWT_ACCESS_SECRET`/`JWT_REFRESH_SECRET`
   sozinho (`generateValue: true` no blueprint).
3. Depois do primeiro deploy, pegue as URLs reais que o Render gerou
   (algo como `https://representapro-backend-xxxx.onrender.com`) e ajuste
   `CORS_ORIGIN` (no backend) e `VITE_API_URL` (no frontend) no `render.yaml`
   — ou direto no dashboard, em Environment — e faça redeploy dos dois.
4. **Migrations + seed**: rode da sua máquina, apontando pro Postgres do
   Render. No dashboard do banco `representapro-db`, copie a **External
   Database URL**, cole em `backend/.env` (`DATABASE_URL=...`) e rode:
   ```
   cd backend
   npm run migrate:up
   npm run seed
   ```
   Diferente do Fly, o Postgres do Render já tem uma URL externa acessível
   direto da internet — não precisa de proxy/túnel para isso.

## Multitenancy

Isolamento por linha (`tenant_id` em cada tabela comercial) reforçado por
Row Level Security no PostgreSQL — cada request injeta o `tenant_id` do
usuário autenticado na sessão do banco antes de qualquer query.

## Fases de construção

1. **Fundação** — estrutura ✅, PostgreSQL + migrations ✅, RLS ✅, autenticação ✅, usuários ✅, permissões ✅ *(falta só o cadastro de novas empresas/onboarding)*
2. **Comercial** — representadas, clientes, vendedores, produtos, categorias, tabela de preços
3. **Vendas** — pedidos, itens, negociação, visitas, agenda, metas
4. **Financeiro comercial** — comissões, faturamento, recebimentos
5. **Gestão** — dashboard, relatórios, indicadores, exportações
6. **SaaS** — planos, assinaturas, limites, painel administrativo, logs, auditoria
7. **Produção** — segurança, backup, deploy, domínio, PWA/mobile, integrações/API
