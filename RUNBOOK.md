# Runbook Operacional — Sistema SEO Blog

> Procedimentos de operação. Atualizar conforme novos casos aparecem.

---

## Setup local (dev)

```bash
# 1. Subir Postgres + Redis
cd seo-blog-backend
docker compose up -d

# 2. Variáveis de ambiente
cp .env.example .env   # ajustar OPEN_ROUTER_KEY e Cloudflare R2 quando tiver

# 3. Backend
npm install
npx prisma migrate dev
npm run db:seed        # cria admin + Health Voice + content types
npm run start:dev      # localhost:3333

# 4. Frontend admin
cd ../seo-blog-frontend
npm install
npm run dev            # localhost:3000

# 5. LP (consumidor)
cd ../health-voice/health-voice-institutional-v2
cp .env.local.example .env.local
npm install
npm run dev            # localhost:3001 (porta 3000 ocupada)
```

**Credencial seed:** `admin@seoblog.local` / `123456` — **trocar no primeiro login**.

---

## Como adicionar um site novo

1. Logar no painel admin (`/sites`).
2. Clicar **Novo site**. Preencher:
   - **Slug** kebab-case (ex.: `juridia`). Usado nas rotas públicas `/public/:siteSlug/...`.
   - **Nome**, **domínio** (ex.: `juridia.com.br`).
   - **Autor** (aparece como autor dos posts no front).
   - **Tom de voz** — orienta a IA.
   - **Locales suportados** (csv: `pt-BR,en,es`).
   - **Revalidate URL** = `https://<domínio>/api/revalidate` (criar essa rota na LP — ver "Plugar LP nova").
   - **Revalidate Secret** = string aleatória forte (`openssl rand -hex 32`).
3. Criar **Tipos de conteúdo** em `/content-types` (ex.: `blog` → `/blog`, `noticia` → `/noticias`).
4. (Opcional) Configurar prompts custom em `/prompts` — se não, defaults da casa em `expansion.service.ts` são usados.
5. Concluir: começar a importar ideias em `/ideias`.

---

## Como plugar uma LP nova

Copiar os 5 arquivos da branch `feat/seo-blog-integration` da `health-voice-institutional-v2` para a nova LP:

```
src/lib/seo-blog.ts           # cliente HTTP tipado
src/lib/markdown.ts           # renderer
src/app/blog/page.tsx         # listagem
src/app/blog/[slug]/page.tsx  # detalhe
src/app/api/revalidate/route.ts
src/app/rss.xml/route.ts
```

E **mergear** em `src/app/sitemap.ts` o trecho que busca posts do CMS via `listBlogPosts()`.

`.env.local` da LP:
```
SEO_BLOG_API_URL=https://api.seoblog.<empresa>.com   # ou http://localhost:3333 em dev
SEO_BLOG_SITE_SLUG=<slug-do-site-no-cms>
SEO_BLOG_REVALIDATE_SECRET=<mesmo do painel>
NEXT_PUBLIC_SITE_HOST=https://<dominio-da-lp>
```

Depois cadastrar o site no painel (passo anterior) com `revalidateUrl` apontando para `https://<dominio-da-lp>/api/revalidate`.

---

## Como gerar e publicar conteúdo (fluxo do operador)

1. **Importar ideias** em `/ideias` → "Importar em lote" → escolher aba (Paste / CSV / JSON / Markdown) → pré-visualizar → importar.
2. **Expandir** cada ideia clicando ⚡ — gera ~7 chamadas de texto + 1 imagem (~US$ 0.10 por post). Aparece confirmação com custo estimado.
3. **Revisar** em `/revisao` (fila de status EXPANDED). Botões 1-click Aprovar/Rejeitar.
4. **Agendar em lote** em `/calendario` → "Agendar em lote" → seleciona N approved + cadência → pré-visualiza datas → confirma.
5. Worker BullMQ publica automaticamente no horário marcado:
   - Marca Content como PUBLISHED.
   - Chama `revalidateUrl` da LP → Next.js invalida ISR.
   - Ping IndexNow (Bing/Yandex) se `indexnowKey` configurado.

---

## Quando uma publicação falha

1. `/publicacoes` mostra jobs FAILED com erro inline.
2. Clicar em **Retry** reenfileira (3 tentativas com backoff exponencial 5s).
3. Erros comuns:
   - **OpenRouter 401** → chave inválida, conferir `OPEN_ROUTER_KEY` no `.env`.
   - **OpenRouter 429** → rate limit do provedor; deixar BullMQ retry.
   - **R2 upload fail** → checar credenciais Cloudflare ou fallback `STORAGE_DRIVER=local`.
   - **Revalidate webhook 401** → secret divergente entre painel e `.env.local` da LP.
   - **Revalidate webhook timeout** → LP fora do ar; publicação no CMS já aconteceu, só o cache da LP fica stale até `revalidate=3600` expirar.

---

## SEO: configurar Search Console

O Google **descontinuou** o ping automático de sitemap em 2023. Para cada novo site:

1. Cadastrar propriedade em https://search.google.com/search-console (preferir Domain property).
2. Submeter sitemap manualmente: `https://<domínio>/sitemap.xml` (Next.js merge-a com URLs do blog automaticamente).
3. Monitorar "Coverage" para ver índice progredindo.

Bing/Yandex pegam via IndexNow automaticamente quando `indexnowKey` é configurado em `/sites`. Gerar a key (UUID v4) e colocar o arquivo `<key>.txt` na raiz pública (root da LP).

---

## Backup do banco

Em dev (Docker local):
```bash
docker exec seoblog-postgres pg_dump -U seoblog seoblog > backup-$(date +%Y%m%d).sql
```

Em produção (VPS): cron job diário com upload pra R2/S3. Configurar quando a VPS subir.

---

## Custos: limites a vigiar

| Métrica | Limite alvo | Onde ver |
|---|---|---|
| Custo IA / dia / site | < US$ 5 com prompts atuais | `/dashboard` ou `/custos` |
| Falhas de publicação | < 5% últimos 30 dias | `/dashboard` (successRate) |
| Posts pendentes em revisão | < 50 simultâneos | `/revisao` |

Alertas automáticos (email/Slack) ficaram para v1.1 — por ora **conferir manualmente** no dashboard.

---

## Adiamentos conhecidos para v1.1

- Alertas email/Slack em falhas de publicação e gasto/dia alto.
- Drag-drop no calendário para reagendar.
- Editor markdown WYSIWYG (hoje é textarea monoespaçado).
- SDK como pacote npm `@seo-blog/sdk` (hoje copiado por LP).
- Aplicação inline de links internos no body_md (hoje só bloco no fim).
- Domínio único `api.seoblog.<empresa>.com` (vem junto com a VPS).
- Embedding-based internal linking (hoje é por tag/categoria).
- ETag em endpoints públicos.

Tudo registrado em `PLANO.md` e `PONTOS-ATENCAO.md`.
