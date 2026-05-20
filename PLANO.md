# Plano Geral — Sistema de Publicação SEO Multi-Site

> **Documento vivo.** Atualizar a cada decisão técnica, ajuste de escopo ou conclusão de etapa. Marcar fases com `[x]` ao concluir.
> **Última atualização:** 2026-05-19

---

## Iniciativas paralelas

- **[Hub de Tracking de LPs](./tracking/README.md)** — expansão do `seo-blog` para também ser o hub central de tracking comportamental, atribuição e analytics de todas as LPs do grupo. Plano em fases, playbook replicável e docs vivos próprios em [`docs/tracking/`](./tracking/). Iniciada em 2026-05-19.

---

## 1. Visão de Produto

Plataforma centralizada (multi-tenant) para:

1. Importar em lote ideias de pauta (CSV/JSON/bulk paste) geradas externamente por operadores.
2. Expandir cada ideia em conteúdo completo via IA (texto + imagem) sob demanda.
3. Revisar/editar/aprovar manualmente.
4. Agendar publicações no tempo.
5. Publicar automaticamente nas LPs (Health Voice, JuridIA, etc.) com **SEO completo** (meta tags, JSON-LD, sitemap, RSS, links internos).
6. Servir o conteúdo às LPs Next.js via API pública + revalidate on-demand (ISR).

---

## 2. Stack Definida

| Camada | Tecnologia | Justificativa |
|---|---|---|
| Backend admin/CMS | **NestJS** | Padrão da casa, modular, DI nativa |
| ORM | **Prisma** | Padrão da casa, type-safe, migrations |
| DB | **Postgres self-hosted** (Docker dev → VPS prod) | Sem Supabase; mesmo padrão do health-voice-api |
| Storage de imagem | **Cloudflare R2** (via `@aws-sdk/client-s3`) | Padrão já em uso no `health-voice-api/src/shared/storage/r2-storage.ts` |
| Fila/jobs | **BullMQ + Redis** | Agendamento delayed, retries, observável |
| Frontend admin | **Next.js (App Router)** | Padrão da casa, SSR, ISR |
| UI admin | **shadcn/ui + Tailwind** | Acelera dev de painel CRUD |
| Auth admin | **JWT custom no NestJS** (padrão health-voice-api) | `@nestjs/jwt` + `bcryptjs`, `AuthGuard` global, decorators `@IsPublic` / `@RequiresSecurityToken`, payload `{sub,email,name,role}` |
| LLM gateway | **OpenRouter** | Padrão da casa, multi-modelo, billing único |
| Modelo de texto (default) | A definir — testar Claude Sonnet 4.6 vs GPT-5 vs Gemini 2.5 Pro | Configurável por site |
| Modelo de imagem (default) | **`google/gemini-2.5-flash-image`** (~$0.04/imagem) | FLUX.2 Klein 4B não aparece no listing direto da OpenRouter no momento — ver R10. Configurável por env `DEFAULT_IMAGE_MODEL`. |
| LPs consumidoras | **Next.js existentes** (ex.: `health-voice-institutional-v2`) | Já têm SSR/SSG, plug-and-play via API |

---

## 3. Arquitetura de Alto Nível

```
┌────────────────────────────┐
│  Operador (Admin Web)      │
│  Next.js (seo-blog-frontend)│
└──────────────┬─────────────┘
               │ REST/JSON (Bearer)
               ▼
┌────────────────────────────┐    ┌──────────────────┐
│  NestJS (seo-blog-backend) │───▶│ OpenRouter (LLM  │
│  - Auth                    │    │ + Image API)     │
│  - Sites/Tenants           │    └──────────────────┘
│  - Ideas/Contents          │
│  - Prompts                 │    ┌──────────────────┐
│  - Schedule (BullMQ)       │───▶│ Supabase Storage │
│  - SEO endpoints públicos  │    │ (imagens)        │
│  - Webhook revalidate      │    └──────────────────┘
└──────┬─────────────────────┘
       │ Postgres (Supabase)
       ▼
┌────────────────────────────┐
│  LP Next.js (cada site)    │
│  ex.: healthvoice.com.br   │
│  - /blog [ISR]             │
│  - /blog/[slug] [ISR]      │
│  - /sitemap.xml (proxy)    │
│  - /rss.xml (proxy)        │
│  - generateMetadata + JSON-LD │
└────────────────────────────┘
```

---

## 4. Modelagem de Dados (rascunho v1)

| Tabela | Campos principais |
|---|---|
| `sites` | id, slug, name, domain, default_locale (`pt-BR`), supported_locales[] (`pt-BR`,`en`,`es`), tone_of_voice, author_name (usado como autor visível dos posts), og_defaults, indexnow_key, revalidate_secret, revalidate_url |
| `content_types` | id, site_id, slug (`blog`, `noticia`, `case`), schema_extra (jsonb), route_prefix |
| `prompt_templates` | id, site_id, content_type_id, field (`body`,`title`,`meta`,`image_prompt`,`translation`...), locale, model, prompt, temperature |
| `ideas` | id, site_id, content_type_id, title_seed, briefing, keywords[], status (`pending`,`expanded`,`discarded`) |
| `contents` | id, idea_id, site_id, content_type_id, locale, title, slug, body_md, body_html, excerpt, meta_description, og_image_id, json_ld (jsonb), status (`draft`,`expanded`,`approved`,`scheduled`,`published`,`unpublished`,`archived`), version, published_at, scheduled_for, source_content_id (para traduções) |
| `content_versions` | id, content_id, snapshot (jsonb), created_by, created_at |
| `tags` / `categories` | id, site_id, slug, name |
| `content_tags` / `content_categories` | join |
| `media_assets` | id, site_id, url, alt, width, height, source (`ai`,`upload`), generation_prompt, model, cost_cents |
| `publish_jobs` | id, content_id, scheduled_for, status, attempts, last_error |
| `ai_usage_log` | id, site_id, content_id, model, input_tokens, output_tokens, cost_cents, kind (`text`,`image`), created_at |
| `internal_links_index` | content_id, related_content_id, score |
| `admin_users` | id, email, password (bcrypt hash), name, role (`ADMIN`,`EDITOR`,`REVISOR`), site_access[] (ids), active, last_login_at — segue convenção do health-voice-api (uppercase em role, campo `password` não `password_hash`) |
| `revoked_tokens` | id, jti, expires_at (logout/blacklist) |

---

## 5. Plano por Fases (fragmentado, componentizado)

### Convenções
- Cada fase entrega valor demonstrável.
- Cada item **back/front** está separado para permitir paralelismo.
- Componentes têm nomes propostos — devem ser refletidos no código.
- Critério de "pronto" sempre exige teste manual ponta-a-ponta + commit no changelog.

---

### FASE 0 — Setup & Infra `[x]`  ✅ concluída 2026-05-18

**Backend (`seo-blog-backend`)**
- [x] `0.B1` Scaffold NestJS espelhando estrutura do `health-voice-api`: `src/modules/<dominio>/` + `src/shared/<infra>/`
- [x] `0.B2` Prisma init + `shared/database/prisma/prisma.service.ts` + `database.module.ts` (mesmo padrão)
- [x] `0.B3` `shared/env/` com `EnvModule` + `EnvService` tipado (não usar `ConfigService` direto)
- [x] `0.B4` Health-check endpoint `/health` (decorado com `@IsPublic()`)
- [ ] `0.B5` Logger estruturado (pino) + interceptor de log de request — **adiado para Fase 1** (Nest Logger basta na Fase 0)
- [x] `0.B6` Docker compose local (**Postgres + Redis**) — setup de execução até VPS estar pronta. Sem MinIO (storage é R2 remoto)
- [x] `0.B7` BullMQ module configurado (sem jobs ainda)
- [x] `0.B8` `shared/storage/` — `R2StorageService` + `LocalDiskStorageService` (fallback de dev) selecionados via `STORAGE_DRIVER`
- [x] `0.B9` Decorators globais: `@IsPublic()`, `@RequiresSecurityToken()`, `@IsAdmin()`, `@CurrentUser()`, `@CurrentUserId()`

**Frontend (`seo-blog-frontend`)**
- [x] `0.F1` Scaffold Next.js 15 (App Router, TS, Tailwind)
- [ ] `0.F2` Instalar shadcn/ui + base de componentes (`Button`, `Input`, `Card`, `Dialog`, `Table`, `Form`) — **adiado para Fase 1** (tokens CSS já preparados em `globals.css`)
- [ ] `0.F3` Layout base (`AppShell` com sidebar + topbar + tenant switcher) — **adiado para Fase 1**
- [x] `0.F4` Cliente HTTP (`lib/api.ts`) com fetch wrapper + auth header
- [ ] `0.F5` Tema escuro/claro — tokens prontos, toggle na Fase 1
- [x] `0.F6` Env schema (`zod`) em `lib/env.ts`

**Saída esperada:** dois apps rodando localmente, health-check OK, login mock. ✅ **Verificado 2026-05-18:**
- `curl http://localhost:3333/health` → `{"status":"ok","services":{"db":"up"}}`
- `GET /` `/login` `/dashboard` → 200
- Postgres + Redis containers healthy

---

### FASE 1 — Multi-Tenant & Schema Base `[x]`  ✅ concluída 2026-05-18

**Backend**
- [x] `1.B1` Migration: `sites`, `content_types`, `admin_users` (espelhando `AdminUser` do health-voice-api), `revoked_tokens`
- [x] `1.B2` Módulo `SitesModule` (CRUD)
- [x] `1.B3` Decorators `@CurrentSiteId` / `@CurrentSiteIdOptional` (X-Site-Id header). Validação de acesso ao site fica para Fase 2 quando entidades scoped chegarem.
- [x] `1.B4` Seed inicial: site "Health Voice" + content_types `blog`/`noticia` + admin
- [x] `1.B5` `shared/auth/` — `JwtModule.registerAsync` global, `AuthService`, `AuthGuard` global (APP_GUARD), `AdminGuard`
- [x] `1.B6` `AuthController`: `POST /auth/login`, `GET /auth/me`, `POST /auth/change-password`
- [x] `1.B7` `AdminUsersController` — CRUD + reset-password, `@IsAdmin()` class-wide

**Frontend**
- [x] `1.F1` Tela login real (POST /auth/login + redirect)
- [x] `1.F2` `<SiteSwitcher />` no topbar
- [x] `1.F3` Página `/sites` (CRUD com Dialog, incluindo `authorName`, `supportedLocales`, `revalidateUrl`, `revalidateSecret`)
- [x] `1.F4` Página `/content-types` (CRUD scoped pelo currentSite)
- [x] `1.F5` Página `/users` (CRUD, restrita a `ADMIN`)
- [x] `1.F6` Providers: `AuthProvider`, `CurrentSiteProvider`, `QueryProvider`. CurrentLocaleProvider postergado para Fase 7.1.

**Adicional entregue:** UI primitives próprios (Button, Input, Label, Card, Table, Badge, Dialog, Select) com tokens Tailwind shadcn-compatible. CLI do shadcn fica para quando precisar de componentes Radix (combobox, popover, etc.).

**Saída esperada:** logar, criar/editar sites e content types pelo painel. ✅ **Verificado 2026-05-18:**
- POST `/auth/login` retorna token + user
- GET `/auth/me` retorna user
- GET `/sites` lista Health Voice (1 site, 2 content types)
- GET `/admin-users` lista admin seed
- Unauthenticated → 401
- Frontend `/login` faz login real, `/dashboard` `/sites` `/content-types` `/users` carregam atrás de auth gate
- `tsc --noEmit` no front: 0 erros

---

### FASE 2 — Ideias (Importação em Lote) `[x]`  ✅ concluída 2026-05-18

**Backend**
- [x] `2.B1` Migration: `ideas`
- [x] `2.B2` `IdeasModule` (CRUD + bulk-delete + bulk-status + bulk-move-type)
- [x] `2.B3` Parsers movidos para o **frontend** (`lib/parsers.ts`): CSV, JSON, markdown-list, bulk-paste. Frontend faz parse + preview e envia array tipado para o backend.
- [x] `2.B4` `POST /ideas/bulk` com `BulkCreateIdeasDto` (limit 1000)
- [x] `2.B5` Validação via class-validator + `assertSiteContentTypeMatch` no service

**Frontend**
- [x] `2.F1` `/ideias` com tabela paginada + filtros (status, content_type, search), seleção múltipla
- [x] `2.F2` `<BulkImportDialog />` com 4 abas (Paste/CSV/JSON/Markdown)
- [x] `2.F3` `ImportPreview` integrado no dialog (tabela editável por linha antes do submit)
- [x] `2.F4` `<IdeaFormDialog />` (criar/editar)
- [x] `2.F5` Bulk actions: apagar, mudar status (mover de tipo pode entrar via `/ideas/bulk-move-type` quando precisar)

**Saída esperada:** importar 365 linhas, vê-las listadas, editar/deletar. ✅ **Verificado 2026-05-18:**
- `POST /ideas/bulk` (3 ideias) → `{count:3}`
- `GET /ideas?siteId=…` → total/items corretos com `contentType` aninhado
- `POST /ideas/bulk-status` → muda em massa
- `POST /ideas/bulk-delete` → deleta em massa
- Frontend `/ideias` → 200, tabela renderiza, dialog importa

---

### FASE 3 — Prompts & Cliente OpenRouter `[x]`  ✅ concluída 2026-05-18

**Backend**
- [x] `3.B1` Migration: `prompt_templates`, `ai_usage_log`
- [x] `3.B2` `AiModule` global com `OpenRouterClient` (texto via `/chat/completions`). Imagem fica para Fase 4.
- [x] `3.B3` `PromptEngineService` — interpola `{{var}}` e `{{path.dotted}}` (suporta objetos aninhados)
- [x] `3.B4` `AiService.runChat()` registra `ai_usage_log` automaticamente (tokens, custo cents, duração, cached, siteId, promptTemplateId, contentId opcionais)
- [x] `3.B5` Endpoints: `GET /ai/models` (lista pricing), `POST /ai/preview`, `GET /metrics/ai-cost?days=&siteId=`
- [x] `3.B6` Cache **in-memory** por hash sha256 do prompt resolvido (TTL 24h). Redis fica para v1.1 quando precisar de cluster.

**Frontend**
- [x] `3.F1` Página `/prompts` (CRUD com tabela por field + content type)
- [x] `3.F2` `<PromptFormDialog />` (campo, tipo, locale, modelo, system, user, temperature, maxTokens) com pricing visível por modelo
- [x] `3.F3` `<PlaygroundDialog />` — auto-extrai `{{variáveis}}` do prompt, formulário inline por var, badge de aviso de custo, mostra resultado + tokens + custo + cache hit
- [x] `3.F4` Página `/custos` — cards totais, gráfico de barras por dia, tabela por modelo, últimas 20 chamadas. Refetch 15s.

**Saída esperada:** editar prompts, testar em playground, ver custo registrado. ✅ **Verificado 2026-05-18:**
- Smoke test ponta-a-ponta:
  - `POST /prompts` cria template para TITLE/blog
  - `POST /ai/preview` (gemini-2.5-flash) gerou: *"Diabetes: Sinais de alerta da glicemia alta"* — 72 in / 10 out tokens, 2.05s.
  - 2ª chamada idêntica → `cached: true`, sem nova chamada à OpenRouter.
  - `GET /metrics/ai-cost` retornou agregado correto por modelo.

**Nota de precisão:** custos sub-centavo são arredondados para 0 no DB (campo int). Para acumular números pequenos com precisão, mudaríamos `costCents` para `costMicroCents` (×10000). Hoje só relevante quando modelos baratos somam muitas chamadas — atalho conhecido, sem ação imediata.

---

### FASE 4 — Expansão de Conteúdo `[x]`  ✅ concluída 2026-05-18

**Backend**
- [x] `4.B1` Migration: `contents`, `content_versions`, `media_assets`, `tags`, `categories`, `content_tags`, `content_categories` + enums `ContentStatus`, `MediaSource`
- [x] `4.B2` `ContentsModule` CRUD + transitions + versions endpoints
- [x] `4.B3` `ExpansionService.expandIdea({ideaId})` pipeline (texto + imagem):
  1. TITLE (claude-sonnet-4.5, temp 0.5)
  2. BODY (claude-sonnet-4.5, temp 0.7, Markdown ≥800 palavras)
  3. EXCERPT (claude-sonnet-4.5)
  4. META_DESCRIPTION (claude-sonnet-4.5)
  5. TAGS (gemini-2.5-flash) — auto upsert
  6. CATEGORY (gemini-2.5-flash) — auto upsert
  7. Slug (determinístico — `slugify` pt-BR sem stopwords)
  8. JSON-LD (determinístico — BlogPosting/NewsArticle)
  9. IMAGE_PROMPT (gemini-2.5-flash)
  10. Image (gemini-2.5-flash-image) → storage → MediaAsset
- [x] `4.B4` `POST /contents/:id/regenerate` (qualquer field, com `preview:true`)
- [x] `4.B5` Versionamento automático: cada save/transition cria snapshot
- [x] `4.B6` `MediaModule` — upload base64, geração via IA, listagem
- [x] `4.B7` `POST /contents/:id/cover` — regenera só a capa

**Frontend**
- [x] `4.F1` `/conteudos` (lista com thumbnail + filtros)
- [x] `4.F2` `/conteudos/[id]` com 4 abas (Conteúdo / SEO + Preview / Mídia / Histórico)
- [x] `4.F3` `<RegenerateButton field />` por campo
- [x] `4.F4` `<SeoPreview />` (Google SERP + OG + Twitter)
- [x] `4.F5` Aba Mídia com geração de capa + preview do prompt + custo
- [x] `4.F6` Botão **Expandir** nas ideias `PENDING` (com confirmação de custo)
- [ ] **Adiado:** editor MD WYSIWYG (`@uiw/react-md-editor`/`tiptap`). Textarea monoespaçado é suficiente — a LP renderiza.

**Saída esperada:** clicar "Expandir" numa ideia, ver conteúdo gerado, editar, preview SEO. ✅ **Verificado 2026-05-18:**
- Idea *"Hipertensão arterial em idosos: o que fazer"* → 1676 palavras + capa PNG 1.2MB em **80s** total por **US$ 0.10**.
- 9 chamadas IA registradas no log (6× claude-sonnet-4.5 + 3× gemini-2.5-flash + 1× gemini-2.5-flash-image).
- Capa servida via `/uploads/...` (LocalDisk em dev, R2 em prod).

**Correção registrada:** modelo correto de imagem é `google/gemini-2.5-flash-image` (sem `-preview`). FLUX.2 Klein 4B fora do listing da OpenRouter no momento — ver `R10` em PONTOS-ATENCAO.

---

### FASE 5 — Workflow de Revisão `[x]`  ✅ concluída 2026-05-18

**Backend**
- [x] `5.B1` State machine em `state-machine.ts` com `ALLOWED_TRANSITIONS` (matriz declarativa). `canTransition()` valida no service. Edit in-place de PUBLISHED mantém status (gera nova versão na Fase 7 via revalidate).
- [x] `5.B2` `POST /contents/:id/transition` — agora valida transição + role.
- [x] `5.B3` Role gates em `TRANSITION_ROLES`:
  - `EXPANDED → APPROVED`: ADMIN, REVISOR
  - `APPROVED → PUBLISHED`: ADMIN
  - `PUBLISHED → UNPUBLISHED`: ADMIN
  - `* → ARCHIVED`: ADMIN
  - demais: ADMIN ou EDITOR
- [x] `5.B4` `GET /contents/:id/versions` + `GET /contents/:id/diff?from=&to=` (compara snapshots; retorna `{fields: {key: {from, to, changed}}}`).
- [ ] `5.B5` Endpoint `unpublish` dedicado — coberto pelo `transition` para `UNPUBLISHED`. Resposta 410/301 nas rotas públicas vem na Fase 8.

**Frontend**
- [x] `5.F1` Botão de transição já existia (select no editor). Validação backend é o gate real.
- [x] `5.F2` Página `/revisao` — lista status=EXPANDED, 1-click Aprovar/Rejeitar, esconde botões se user não é REVISOR/ADMIN, refetch automático a cada 10s.
- [x] `5.F3` `<VersionDiff />` side-by-side, comparação livre entre quaisquer 2 versões. Card vermelho (from) + verde (to) por campo mudado.

**Saída esperada:** revisar, aprovar, rejeitar, ver histórico. ✅ **Verificado 2026-05-18:**
- Smoke test: EXPANDED → APPROVED ✓ · APPROVED → EXPANDED bloqueia 400 ✓ · APPROVED → PUBLISHED grava `publishedAt` ✓ · PUBLISHED → UNPUBLISHED ✓ · diff v1→v2 detectou mudança em `ogImageId` ✓.

---

### FASE 6 — Agendamento `[x]`  ✅ concluída 2026-05-18

**Backend**
- [x] `6.B1` Migration: `publish_jobs` + enum `PublishJobStatus` (PENDING/RUNNING/SUCCEEDED/FAILED/CANCELLED) + índice `[siteId,scheduledFor]` em Content
- [x] `6.B2` `ScheduleModule` com `BullModule.registerQueue('publish')` — cria jobs delayed (delay = scheduledFor - now), guarda `bullJobId` no DB para correlação. Worker (consumer) fica para Fase 7.
- [x] `6.B3` `ScheduleService.scheduleBulk` com 3 modos: `DAILY`, `WEEKLY`, `EVERY_N_DAYS` (com `everyN`). Flag `skipWeekends`. Horário fixo `HH:MM` (default 09:00). Computa array de datas determinístico.
- [x] `6.B4` Endpoints: `GET /schedule?siteId=&from=&to=`, `POST /schedule` (single), `POST /schedule/bulk`, `PATCH /schedule/:id`, `DELETE /schedule/:id` (cancel + volta content para APPROVED se SCHEDULED).
- Cancel também remove o job da fila BullMQ.

**Frontend**
- [x] `6.F1` Página `/calendario` — **grid mensal customizado** (sem `react-big-calendar`), navegação ← →, marcação do dia atual, até 3 jobs por dia + "+N mais", status com bolinhas coloridas (amarelo/verde/vermelho/cinza). Lista do mês também.
- [x] `6.F2` `<BulkScheduleDialog />` — selector multi de APPROVED, cadência, pré-visualização das datas calculadas em tempo real.
- [ ] `6.F3` Drag-drop para reagendar — **adiado**. Reagendamento via clique no botão (prompt de data ISO). Drag-drop entra na v1.1 com `react-dnd` ou `@dnd-kit`.
- [x] `6.F4` Visão lista (tabela embaixo do calendário) com Aprovar/Rejeitar 1-click… err, cancelar/reagendar.

**Saída esperada:** agendar 365 aprovados em batch, ver no calendário, mover datas. ✅ **Verificado 2026-05-18:**
- Smoke: criou Content dummy → DRAFT → EXPANDED → APPROVED → `POST /schedule/bulk` (DAILY, skipWeekends) → criou job + atualizou Content para `SCHEDULED` com `scheduledFor` correto → `GET /schedule` listou → `PATCH` reagendou para +3 dias → `DELETE` cancelou + voltou Content para APPROVED.
- Jobs entram na fila BullMQ no Redis (validado por `bullJobId` salvo no DB).

---

### FASE 7 — Worker de Publicação `[x]`  ✅ concluída 2026-05-18

**Backend**
- [x] `7.B1` `PublishProcessor extends WorkerHost` (BullMQ consumer da fila `publish`) — marca RUNNING (com `attempts++`), atualiza `Content.status=PUBLISHED` + `publishedAt`, cria `ContentVersion` snapshot, finaliza SUCCEEDED. Trata Content já PUBLISHED como idempotente.
- [x] `7.B2` Webhook outbound — `POST site.revalidateUrl` com `{path, slug, locale, action: 'publish'}` + header `X-Revalidate-Secret`. Falha do webhook **não** quebra publicação (best-effort; LP fica stale até próximo revalidate).
- [x] `7.B3` Retry: `attempts: 3` + `backoff: { type: 'exponential', delay: 30000ms }` (publish-now usa 5s). `@OnWorkerEvent('failed')` grava `lastError` em `publish_jobs` somente quando esgota tentativas.
- [ ] `7.B4` Notificação de falha (email/Slack) — adiado para v1.1; por ora log + visível em `/publicacoes`.
- [x] `7.B5` `POST /contents/:id/publish-now` — enfileira imediatamente (delay 0, attempts 3, backoff 5s). Aceita APPROVED/SCHEDULED/UNPUBLISHED.
- [x] `7.B6` `POST /publish-jobs/:id/retry` — reenfileira jobs FAILED.
- [x] `7.B7` `GET /publish-jobs` (paginado, filtro por status/site).

**Frontend**
- [x] `7.F1` Página `/publicacoes` — tabela paginada com filtro de status, badges, erro inline, botão Retry para FAILED, atalho para o Content. Refetch 10s.
- [x] `7.F2` Botão **"Publicar agora"** no editor do Content (visível em APPROVED/UNPUBLISHED).

**Saída esperada:** agendamento de hoje publica sozinho, status muda, LP atualiza. ✅ **Verificado 2026-05-18:**
- `POST /contents/:id/publish-now` → worker processou em <1s → Content=PUBLISHED, PublishJob=SUCCEEDED, attempts=1.
- Agendamento com delay 5s → BullMQ entregou no tempo certo → Content=PUBLISHED com `publishedAt` correto.
- Site Health Voice ainda sem `revalidateUrl` configurado → webhook pulado silenciosamente (esperado; LP integration vem na Fase 10).

---

### FASE 8 — SEO Output (API Pública) `[x]`  ✅ concluída 2026-05-18

**Backend**
- [x] `8.B1` `PublicModule` com rotas `@IsPublic()` (sem auth) — todas com `Cache-Control` apropriado:
  - `GET /public/:siteSlug/contents?type=blog&locale=pt-BR&page=N` (60s/300s)
  - `GET /public/:siteSlug/contents/:slug?locale=pt-BR` (120s/600s)
  - **410 Gone** para UNPUBLISHED/ARCHIVED (resposta para D8)
- [x] `8.B2` `GET /public/:siteSlug/sitemap.xml` — XML dinâmico, agrupado por slug com `<xhtml:link hreflang>` cross-locale. Cache 1h.
- [x] `8.B3` `GET /public/:siteSlug/rss.xml?type=&locale=` — RSS 2.0 com até 50 itens, default = locale do site. Cache 10min.
- [x] `8.B4` `GET /public/:siteSlug/robots.txt` — gerado dinamicamente, aponta para sitemap. Cache 24h.
- [x] `8.B5` IndexNow ping integrado no `PublishProcessor` — chama `https://api.indexnow.org/IndexNow` com `{host, key, urlList}` quando `site.indexnowKey` está configurado. Best-effort (não rejoga job em falha). Google ping foi descontinuado em 2023 — documentar no runbook (Fase 12).
- [x] `8.B6` Headers `Cache-Control` por endpoint. ETag pode entrar na v1.1 via interceptor.
- [ ] `8.B7` Domínio público dedicado **cross-produto** — em produção/VPS. Hoje servido em `localhost:3333/public/...`.

**Frontend**
- Nada — backend puro consumido pelas LPs (Fase 10).

**Saída esperada:** `curl http://localhost:3333/public/health-voice/sitemap.xml` retorna sitemap válido. ✅ **Verificado 2026-05-18:**
- `/contents` retorna 2 conteúdos PUBLISHED (site, items, total) sem expor campos internos.
- `/sitemap.xml` retorna XML válido com `<urlset>`, `<xhtml:link>` por locale.
- `/rss.xml` retorna RSS 2.0 com 2 items, dates RFC 822.
- `/robots.txt` retorna texto plano com `Sitemap:`.
- `/contents/inexistente` → 404; site não cadastrado → 404; conteúdo UNPUBLISHED → 410 Gone (decisão D8 implementada).

---

### FASE 9 — Linkagem Interna `[x]`  ✅ concluída 2026-05-18

**Backend**
- [ ] `9.B1` Migration `internal_links_index` — **dispensada**: a v1 calcula on-the-fly via Prisma; sem tabela = sem invalidação. Quando volume justificar materializar (>10k posts/site), criar índice.
- [x] `9.B2` `RelatedService.forContent(id)` — busca contents do mesmo site+locale que compartilham ≥1 tag/categoria; score = `tags×1 + categorias×2`; ordena por score desc → publishedAt desc; limita 5/8 (configurável).
- [x] `9.B3` Compute on-fly em vez de job background; o cache HTTP (300s/600s) faz o papel do índice persistido.
- [ ] `9.B4` Aplicação inline no body — **adiado**. Risco de quebrar o markdown. v1 expõe array de relacionados; a LP renderiza como bloco "Posts relacionados" no fim do artigo.
- [x] `9.B5` Endpoints: `GET /contents/:id/related?limit=` (admin) + `GET /public/:siteSlug/contents/:slug/related` (público).

**Frontend**
- [x] `9.F1` Aba **Links Internos** no editor — cards com thumbnail, badges das tags/categorias compartilhadas, score visível, link direto para o conteúdo.
- [ ] `9.F2` Toggle "aplicar automaticamente" — adiado junto com 9.B4.

**Saída esperada:** publicações relacionadas aparecem listadas no editor + via API pública para LP. ✅ **Verificado 2026-05-18:**
- Criei 2 posts dummy (Pressão alta / Glicemia) com tags geradas via IA. O post "Hipertensão em Idosos" detectou `Pressão alta — quando preocupar` como relacionado com **score 1** (tag compartilhada: *saúde cardiovascular*).
- Custo do setup: ~$0.0002 (2 chamadas gemini-2.5-flash).

---

### FASE 10 — SDK & Integração nas LPs `[x]`  ✅ concluída 2026-05-18

**SDK (in-repo, `src/lib/seo-blog.ts` na própria LP)**
- [x] `10.S1` Cliente HTTP tipado (`listBlogPosts`, `getBlogPost`, `getRelated`, `getAllBlogSlugs`, `siteHost`)
- [x] `10.S2` Tipos: `BlogContent`, `BlogListResult`, `BlogDetailResult`, `RelatedItem`, `BlogTag`, `BlogCategory`
- [x] `10.S3` Integra com `fetch` nativo do Next 16 — `next: { revalidate, tags }` para ISR + invalidação por tag
- [ ] **Adiado:** publicar como pacote npm `@seo-blog/sdk`. Por ora cada LP copia `lib/seo-blog.ts`. Quando 3ª LP entrar, extrai pra workspace.

**Componentes (inline na LP — não monorepo na v1)**
- [x] `10.C1` Página `/blog` (grid 3 colunas responsivo, paginação)
- [x] `10.C2` Página `/blog/[slug]` (capa + autor + data + corpo markdown + tags/categorias)
- [x] `10.C3` Seção "Posts relacionados" no fim do artigo
- [x] `10.C4` JSON-LD injetado via `<script type="application/ld+json">` (vem pronto do CMS)
- [x] `10.C5` `generateMetadata` por post + lista (Open Graph + Twitter + canonical)
- [x] `10.M1` `lib/markdown.ts` renderer minimalista (sem deps; H1-3, listas, blockquote, code fence, links, bold/italic/code inline)

**Integração `health-voice-institutional-v2`** (branch `feat/seo-blog-integration`)
- [x] `10.I1` `app/blog/page.tsx` + `app/blog/[slug]/page.tsx`
- [x] `10.I2` `generateMetadata` pulando da API com fallback
- [x] `10.I3` ISR `revalidate = 3600` + webhook `POST /api/revalidate` (valida `X-Revalidate-Secret`, chama `revalidatePath` + `revalidateTag`)
- [x] `10.I4` `sitemap.ts` agora **merge** das URLs estáticas + URLs do blog (via API). Falha silenciosa se backend offline.
- [x] `10.I5` `app/rss.xml/route.ts` — proxy direto pro backend (`/public/:siteSlug/rss.xml`)
- [ ] `10.I6` Lighthouse SEO ≥ 95 — testar em produção. Localmente render-tempo ~1s.
- [x] `10.I7` `.env.local.example` documentando `SEO_BLOG_API_URL`, `SEO_BLOG_SITE_SLUG`, `SEO_BLOG_REVALIDATE_SECRET`, `NEXT_PUBLIC_SITE_HOST`.

**Saída esperada:** publicação no painel aparece em `healthvoice.com.br/blog/[slug]` com SEO completo em < 60s. ✅ **Verificado 2026-05-18:**
- LP em http://localhost:3001/blog renderiza grade com 5 posts publicados (mock data).
- `/blog/hipertensao-idosos-guia-pressao-alta` renderiza título, capa, corpo markdown, tags, JSON-LD e seção "Posts relacionados" (1 relacionado).
- `<title>` final: "Hipertensão em Idosos: Guia para Pressão Alta | Health Voice" ✓
- Webhook `POST /api/revalidate` com secret correto retorna 200 + revalida; sem secret retorna 401.
- Backend (`PublishProcessor`) agora consegue chamar a LP — `site.revalidateUrl` configurado via `PATCH /sites/:id`.

**Importante:** mudanças ficam na branch `feat/seo-blog-integration` na LP. Nada foi committed/pushed ainda — diff disponível via `git status` na pasta da LP.

---

### FASE 11 — Observabilidade & Custos `[x]`  ✅ concluída 2026-05-18

**Backend**
- [x] `11.B1` `GET /metrics/ai-cost` (Fase 3 já tinha) + `GET /metrics/overview` agregando contagens dos pipelines (ideas, contents por status, schedule today/week, publish 7d/30d, success rate, ai cost today/7d/30d).
- [x] `11.B2` `GET /metrics/publish-stats?siteId=&days=` — totais por status + success rate + tempo médio/máximo (computado via `finished_at - started_at` em SQL raw) + breakdown por dia (succeeded/failed).
- [ ] `11.B3` Alertas configuráveis por gasto/dia — **adiado para v1.1**. Dashboard hoje mostra o número; alerta automático via email/Slack quando passa de X exige infra de notificação que ainda não temos (Fase 7.B4 adiada).
- [x] `11.B4` Export CSV: `GET /metrics/ai-cost/export.csv?siteId=&days=` — `Content-Disposition: attachment`, escape de vírgulas, header descritivo.

**Frontend**
- [x] `11.F1` Dashboard `/dashboard` reformulado: 4 cards de cima (ideias pendentes, aguardando revisão, agendados próx 7d, publicados 30d) + 3 cards de baixo (custo IA hoje + agregados, success rate de publicação + tempos, pipeline por status). Refetch 30s.
- [x] `11.F2` Gráfico de barras "publicações por dia" no dashboard (succeeded verde + failed vermelho stack). Gráfico de custo por dia já existia em `/custos` desde a Fase 3.
- [x] `11.F3` Botão **Download CSV** na página `/custos`.

---

### FASE 12 — Hardening & Docs `[x]`  ✅ concluída 2026-05-18

- [ ] `12.1` Testes e2e críticos automatizados — **adiado v1.1**. Smoke scripts manuais (`smoke-fase*.mjs`) cobriram cada fase ponta-a-ponta; o fluxo completo (importar → expandir → aprovar → agendar → publicar → LP) foi exercitado em humano.
- [x] `12.2` Rate limit via `@nestjs/throttler`:
  - Global: 10 req/10s, 60 req/min, 300 req/5min
  - `/auth/login`: 5 req/min (testado: 6ª retorna 429 ✓)
  - `/ai/preview`: 30 req/min
- [x] `12.3` Hardening de uploads em `MediaService.uploadBase64`:
  - Allowlist de MIME (png/jpeg/webp/gif/avif/svg)
  - Max 5MB (`PayloadTooLargeException` se exceder)
  - Validação básica de base64
  - 400/413/415 com mensagens claras
- [x] `12.4` READMEs finais — `/README.md` raiz com arquitetura, `seo-blog-backend/README.md`, `seo-blog-frontend/README.md`. Diagrama ASCII no README raiz.
- [x] `12.5` `docs/RUNBOOK.md` — setup, adicionar site, plugar LP nova, fluxo do operador, debugging de publicação, configuração Google Search Console + IndexNow, backup DB, limites de custo, adiamentos conhecidos.

---

## 6. Convenções de Código

- **Backend:** seguir convenções do `health-voice-api` — `src/modules/<dominio>/` para domínios e `src/shared/<infra>/` para transversais (auth, env, database, storage, ai, email). Services finos; DTOs com `class-validator`. Sempre `prisma.$transaction` para writes multi-tabela. `EnvService` para tudo de env (nunca `process.env` direto). Auth via `@IsPublic` / `@RequiresSecurityToken` / `AdminGuard`.
- **Frontend:** App Router. Components em `components/ui/` (genéricos) e `components/feature/<dominio>/` (específicos). Server Actions para mutações simples; cliente HTTP para fluxos com loading complexo.
- **Variáveis ambiente:** prefixo `SEO_BLOG_`. Schema validado com zod no boot.
- **Commits:** Conventional Commits. Toda PR linkando fase/item (`feat(2.B3): bulk import service`).

---

## 7. Fora de Escopo na v1 (registrar para v2)

- Geração de áudio (podcast TTS).
- A/B testing de títulos.
- Comentários públicos nos posts.
- Analytics integrado no painel (views/cliques por post) — operador consulta GA da LP diretamente.
- Embedding-based internal linking (substituir similaridade por tag).
- Fila pessoal de revisão por usuário (equipe de 5 não justifica na v1).

## 7.1. i18n (pt-BR primário, en/es opcionais)

- Schema preparado: `contents.locale`, `contents.source_content_id`, `sites.supported_locales`.
- Conteúdo é criado primeiro em pt-BR; tradução é uma ação separada via IA (gera novo `content` linkado por `source_content_id`).
- LPs Next.js consomem rotas como `/blog/[slug]` para pt-BR e `/en/blog/[slug]` / `/es/blog/[slug]` para os demais.
- Sitemap inclui `hreflang` cruzado.
- v1 implementa **toda a infra**, mas só pt-BR é obrigatório no go-live; en/es ficam configuráveis por site quando o time decidir traduzir.

---

## 8. Como manter este documento

Ao concluir um item:
1. Marcar `[x]` no item.
2. Registrar entrada em `CHANGELOG.md`.
3. Se desviou do plano, registrar em `CHANGELOG.md` na seção "Mudanças de Rumo" + atualizar a fase aqui.
4. Se surgiu bloqueio/dúvida, mover/criar em `PONTOS-ATENCAO.md`.
