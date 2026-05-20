# Changelog & Mudanças de Rumo

> **Documento vivo.** Toda entrega significativa, decisão técnica ou mudança de plano é registrada aqui.
> Formato: [Keep a Changelog](https://keepachangelog.com/) adaptado.
> **Ordem:** mais recente no topo.

---

## Como registrar

Cada entrada deve ter:
- **Data** (ISO, ex.: `2026-05-18`).
- **Tipo:** `Added` | `Changed` | `Fixed` | `Removed` | `Decision` | `Pivot`.
- **Fase/Item:** referência do `PLANO.md` (ex.: `2.B3`).
- **Resumo curto.**
- **Detalhe / motivo** (especialmente para `Decision` e `Pivot`).

Exemplo:

```
## 2026-05-18 — Decision (Fase 0)
**Fase/Item:** 0 / stack
**Resumo:** Adotado NestJS + Prisma + Supabase + BullMQ + Next.js App Router.
**Motivo:** Stack padrão da casa, equipe já fluente, evita curva de aprendizado.
**Alternativas descartadas:** Strapi/Payload (menos controle sobre fluxo customizado de IA).
```

---

## 2026-05-20 — Added (Hub de Tracking — Fase 1: backend de ingestão)
**Fase/Item:** iniciativa de tracking, Fase 1
**Resumo:** Módulo `tracking` no `seo-blog-backend` com ingestão pública de sessão/evento/lead/consent, idempotente e multi-tenant.
**Detalhe:** migration `add_tracking_hub` (5 tabelas `tracking_*` + `Site.publicKey`/`trackingEnabled`); `TrackingController`/`TrackingService`/`SiteKeyGuard`; rate-limit, dedup por `eventId` e hash de lead, suporte a `text/plain` (sendBeacon), hash de IP. Validado por curl (auth, validação, idempotência). Branch `feat/tracking-hub-backend`.
**Adiado:** testes e2e automatizados, CORS por site, geo lookup. Ver `docs/tracking/CHANGELOG-TRACKING.md`.
**Próximo passo:** Fase 2 — `lib/tracking/` na LP Health Voice.

## 2026-05-19 — Added (iniciativa paralela: Hub de Tracking)
**Fase/Item:** nova iniciativa
**Resumo:** Iniciada a iniciativa paralela "Hub de Tracking de LPs". Criados 8 docs vivos em `docs/tracking/` (README, ARQUITETURA, PLANO, PLAYBOOK, CATALOGO de eventos, LICOES-LPS-EXISTENTES, CHANGELOG e PONTOS-ATENCAO próprios).
**Motivo:** Transformar o `seo-blog` no hub central de tracking + analytics das LPs do grupo. Tentativas anteriores (Health Voice com Supabase próprio, Inova com tracking single-tenant) ficaram fragmentadas — auditadas 5 LPs/sistemas; o `health-lp-v2` capturava bem mas nunca teve visualização útil (~45 tabelas clonadas por LP, dashboard sem funil).
**Detalhe:** v1 prevê 5 tabelas Prisma novas (`tracking_sessions`, `tracking_events`, `tracking_attribution`, `tracking_leads`, `tracking_consent_log`), módulo `tracking` no NestJS com idempotência via `eventId` + rate-limit, `lib/tracking/` no front das LPs com UTM/sessão/consentimento, e dashboard `(admin)/analytics/` separado do dashboard operacional do CMS.
**Próximo passo:** criar skills e agents em `.claude/` (front e back) — Fase 0 da nova iniciativa.
**Ver:** [docs/tracking/](./tracking/README.md), [docs/tracking/CHANGELOG-TRACKING.md](./tracking/CHANGELOG-TRACKING.md)

---

## 2026-05-18 — Added (Fase 12 concluída — hardening + docs) · 🎉 v1 entregue
**Fase/Item:** 12 inteira
**Resumo:** Rate limit nas rotas sensíveis, upload base64 hardened, READMEs finais e runbook operacional.
**Backend:**
- `@nestjs/throttler` instalado. `ThrottlerModule` global (10/10s · 60/min · 300/5min). `@Throttle` específico em `/auth/login` (5/min) e `/ai/preview` (30/min).
- `MediaService.uploadBase64`: allowlist de MIME (image/png/jpeg/webp/gif/avif/svg), limite 5MB, validação básica de base64. Lança `BadRequest`/`PayloadTooLarge`/`UnsupportedMediaType` com mensagens claras.
**Docs:**
- README raiz `/README.md` com diagrama ASCII + quick start + custo estimado.
- `seo-blog-backend/README.md` — stack, estrutura completa, env vars, migrations.
- `seo-blog-frontend/README.md` — mapa de telas + estrutura.
- `docs/RUNBOOK.md` — procedimentos: setup local, adicionar site, plugar LP nova, fluxo do operador, debugging, Search Console + IndexNow, backup DB, limites de custo, adiamentos conhecidos.
- `docs/README.md` atualizado com link pro Runbook.
**Smoke test rate limit:** 5 logins seguidos com senha errada → 401 · 6ª e 7ª request → 429 ✓
**Adiado v1.1:** testes e2e automatizados (Playwright/Vitest). Smoke scripts manuais cobriram cada fase.

## 2026-05-18 — Added (Fase 11 concluída — observabilidade)
**Fase/Item:** 11 inteira
**Resumo:** Endpoints agregados de overview + publish-stats, export CSV, dashboard refeito.
**Backend:**
- `GET /metrics/overview?siteId=` — agrega ideas (pending/total), contents por status, schedule today/nextWeek, publish (publishedLast7/30Days, jobsLast30, succeeded/failed, successRate), aiCost (today/7d/30d). Roda em paralelo (Promise.all) — ~1 req DB para todo o dashboard.
- `GET /metrics/publish-stats?days=&siteId=` — totais por status (SUCCEEDED/FAILED/CANCELLED/PENDING), avg/max duration via SQL raw (`finished_at - started_at`), byDay breakdown com `COUNT(*) FILTER (WHERE status=...)`.
- `GET /metrics/ai-cost/export.csv?days=&siteId=` — CSV com header + linhas (escape de vírgula), `Content-Disposition: attachment`. Limite 50k linhas.
**Frontend:**
- Tipos `OverviewMetrics`, `PublishStats`.
- Dashboard `/dashboard` reformulado:
  - 4 cards (ideias pendentes / aguardando revisão / agendados próx 7d / publicados 30d), com badge "ação" quando EXPANDED > 0.
  - 3 cards detalhados (custo IA + agregados, success rate + tempos, pipeline por status).
  - Gráfico de barras "publicações por dia" (succeeded verde + failed vermelho stacked).
  - Refetch 30s para overview, 60s para publishStats.
- Botão "Download CSV" em `/custos`.
**Adiado:** alertas automáticos por gasto/dia (v1.1, exige notificação email/Slack).

## 2026-05-18 — Added (Fase 10 concluída — integração na LP)
**Fase/Item:** 10 inteira
**Resumo:** LP `health-voice-institutional-v2` consumindo CMS ponta-a-ponta. Branch `feat/seo-blog-integration` (não commitada — diff sob revisão).
**Arquivos novos na LP:**
- `src/lib/seo-blog.ts` — cliente HTTP tipado (`listBlogPosts`, `getBlogPost`, `getRelated`, `getAllBlogSlugs`, `siteHost`) usando `fetch` nativo do Next 16 com `next: { revalidate, tags }` para ISR + invalidação por tag.
- `src/lib/markdown.ts` — renderer markdown minimalista sem deps (H1-3, listas, blockquote, code fence, inline bold/italic/code/links).
- `src/app/blog/page.tsx` — listagem grid 3 colunas, paginação, hero com gradiente Health Voice, `generateMetadata` + canonical.
- `src/app/blog/[slug]/page.tsx` — detalhe com capa, corpo, tags/categorias, seção "Posts relacionados" (até 4), `generateMetadata` com OG/Twitter, `<script type="application/ld+json">` puxado do CMS.
- `src/app/api/revalidate/route.ts` — webhook que valida `X-Revalidate-Secret` (401 se errado), chama `revalidatePath` + `revalidateTag` (com profile `'default'` exigido pelo Next 16).
- `src/app/rss.xml/route.ts` — proxy direto pro backend.
- `.env.local.example` — documenta `SEO_BLOG_API_URL`, `SEO_BLOG_SITE_SLUG`, `SEO_BLOG_REVALIDATE_SECRET`, `NEXT_PUBLIC_SITE_HOST` + dummies de Supabase pra dev local.
**Arquivos modificados:**
- `src/app/sitemap.ts` — agora async, merge URLs estáticas + URLs do blog vindas do CMS, falha silenciosa se backend offline.
**Decisões registradas:**
- SDK fica **inline** na LP (`src/lib/seo-blog.ts`) na v1, não pacote npm. Quando 3ª LP entrar, extrai pra workspace.
- Renderer Markdown **próprio** (sem `marked`/`remark`) — dispensa dep externa.
- Webhook usa `revalidateTag(tag, 'default')` — Next 16 mudou assinatura, requer profile.
**Smoke test ponta-a-ponta:**
- Backend publica → `PublishProcessor` chama webhook da LP → LP retorna 200 e revalida.
- `GET http://localhost:3001/blog` lista 5 posts.
- `GET http://localhost:3001/blog/hipertensao-idosos-guia-pressao-alta` retorna HTML com título correto, JSON-LD, seção de relacionados (1 post linkado por tag *saúde cardiovascular*).

## 2026-05-18 — Added (Fase 9 concluída)
**Fase/Item:** 9 inteira
**Resumo:** Linkagem interna (related posts) on-the-fly + endpoints admin/público + UI no editor.
**Backend:**
- `RelatedService.forContent(id, {limit, publishedOnly})`: candidates do mesmo site+locale com ≥1 tag/categoria compartilhada (Prisma `OR`); score `tags×1 + categorias×2`; ordena por score desc → publishedAt desc.
- `RelatedService.forPublicSlug` para rotas públicas com `publishedOnly: true`.
- Endpoints: `GET /contents/:id/related?limit=` (admin) e `GET /public/:siteSlug/contents/:slug/related?locale=&limit=` (público com `Cache-Control` 300s/600s).
**Frontend:**
- `<RelatedTab />`: cards com thumbnail, score, badges das tags compartilhadas (×1) e categorias (×2), link direto.
- Nova aba "Links Internos" no editor entre Mídia e Histórico.
**Decisões:**
- **Sem tabela `internal_links_index`** materializada na v1. Compute on-fly + cache HTTP. Reavaliar acima de ~10k posts/site.
- **Aplicação inline no body adiada**. LP renderiza bloco "Posts relacionados" no fim usando o endpoint.
**Smoke test:** 2 dummies criados + tags geradas via gemini-flash; related identificou "Pressão alta" como relacionado a "Hipertensão em Idosos" (score 1, tag *saúde cardiovascular*).

## 2026-05-18 — Added (Fase 8 concluída)
**Fase/Item:** 8 inteira
**Resumo:** API pública sem auth para consumo das LPs + sitemap/RSS/robots dinâmicos + IndexNow.
**Backend:**
- `PublicModule` com rotas `@IsPublic()`:
  - `GET /public/:siteSlug/contents?type=&locale=&page=&pageSize=` — paginado, só PUBLISHED, fields filtradas (sem `createdBy`, `version`, `status`).
  - `GET /public/:siteSlug/contents/:slug?locale=` — 200 PUBLISHED, **410 Gone** UNPUBLISHED/ARCHIVED, 404 not found.
  - `GET /public/:siteSlug/sitemap.xml` — XML com `<urlset>` + `<xhtml:link hreflang>` cross-locale agrupado por slug.
  - `GET /public/:siteSlug/rss.xml?type=&locale=` — RSS 2.0 (até 50 items, RFC 822 dates).
  - `GET /public/:siteSlug/robots.txt` — texto plano com `Sitemap:`.
- Headers `Cache-Control` por endpoint (60-86400s s-maxage).
- `PublishProcessor` agora também faz **IndexNow ping** quando `site.indexnowKey` e `site.domain` estão configurados (best-effort, não rejoga em falha).
**Smoke test:**
- `/contents` lista 2 publicados, sem dados internos ✓
- `/sitemap.xml` XML válido com hreflang ✓
- `/rss.xml` RSS 2.0 com pubDate RFC 822 ✓
- `/robots.txt` texto plano ✓
- 404 para site/conteúdo inexistente, 410 para UNPUBLISHED ✓ (cumpre decisão D8 do PONTOS-ATENCAO)
**Adiado:** ETag (v1.1, via interceptor). Domínio próprio (`api.seoblog.<empresa>.com`) entra junto com a VPS.

## 2026-05-18 — Added (Fase 7 concluída)
**Fase/Item:** 7 inteira
**Resumo:** Worker BullMQ consumindo a fila `publish` + retry exponencial + webhook revalidate + tela de histórico.
**Backend:**
- `PublishProcessor extends WorkerHost`: marca RUNNING (attempts++), publica Content (status=PUBLISHED + publishedAt), gera snapshot na ContentVersion, marca SUCCEEDED. Trata idempotência (Content já PUBLISHED).
- Webhook outbound em `site.revalidateUrl` com header `X-Revalidate-Secret`, payload `{path, slug, locale, action: 'publish'}`. Best-effort: falha do webhook não rejoga o job.
- `@OnWorkerEvent('failed')` grava `lastError` só quando esgota tentativas.
- Retry options: `attempts: 3`, `backoff: exponential 30s` (publish-now usa 5s).
- `PublishService.publishNow(contentId)`: cria PublishJob imediato + enfileira.
- `PublishService.retry(jobId)`: reenfileira FAILED.
- Endpoints: `GET /publish-jobs` (paginado, filtro status/site), `POST /contents/:id/publish-now`, `POST /publish-jobs/:id/retry`.
- ScheduleService passa `attempts: 3, backoff exponential 30s` em todas as adições à fila.
**Frontend:**
- `/publicacoes`: tabela paginada com filtro de status, badges, erro inline, Retry para FAILED, atalho para o Content. Refetch 10s.
- Botão **"Publicar agora"** no editor (visível em APPROVED/UNPUBLISHED).
- Sidebar: novo item "Publicações" + remove tag de Fase.
**Smoke test:**
- `publish-now` → SUCCEEDED em <1s (attempts=1).
- Schedule com delay 5s → BullMQ entregou no tempo certo, publishedAt correto.
**Adiado:** notificações email/Slack para falhas (v1.1). Log + dashboard são suficientes.

## 2026-05-18 — Added (Fase 6 concluída)
**Fase/Item:** 6 inteira
**Resumo:** Agendamento ponta-a-ponta — bulk + calendário + reagendar/cancelar.
**Backend:**
- Schema `PublishJob` + enum `PublishJobStatus`. Migration `fase-6-schedule`. Índice `[siteId,scheduledFor]` em Content.
- `ScheduleModule` com `BullModule.registerQueue('publish')`. `ScheduleService` cria jobs delayed e correlaciona `bullJobId` ↔ `PublishJob.id`.
- `ScheduleService.scheduleBulk` com 3 modos (DAILY, WEEKLY, EVERY_N_DAYS), `skipWeekends`, hora fixa. `computeDates()` determinístico.
- `ScheduleService.reschedule` remove o job antigo da fila e cria novo. `cancel` remove + volta Content para APPROVED se ainda SCHEDULED.
- Endpoints: GET list, POST single, POST /bulk, PATCH /:id (reschedule), DELETE /:id (cancel).
**Frontend:**
- Tipos `PublishJob`, `PublishJobStatus`.
- `<BulkScheduleDialog />`: seletor multi de APPROVED, cadência, pré-visualização das datas em tempo real.
- Página `/calendario`: grid mensal customizado (sem libs externas), navegação mês ← →, hoje destacado, até 3 jobs por dia + indicador "+N mais", lista do mês embaixo com reagendar/cancelar.
- Sidebar atualizado (remove badge "Fase 6").
**Smoke test:** bulk schedule → list → reschedule → cancel — todos OK, Content acompanha (SCHEDULED ↔ APPROVED).
**Adiamento:** drag-drop para reagendar fica para v1.1. Reschedule atual via prompt ISO.

## 2026-05-18 — Added (Fase 5 concluída)
**Fase/Item:** 5 inteira
**Resumo:** State machine validada + role gates + diff side-by-side + fila de revisão.
**Backend:**
- `state-machine.ts`: `ALLOWED_TRANSITIONS` (matriz por status) + `TRANSITION_ROLES` (gates por par). `canTransition()` e `rolesForTransition()` helpers.
- `contents.service.transition()`: agora recebe `userRole`, valida com `canTransition` (400 se inválida) e role gate (403 se errada). `publishedAt` setado em PUBLISHED.
- Novo método `contents.service.diffVersions(contentId, from, to)`: compara snapshots e retorna `{fields:{key:{from,to,changed}}}`.
- `contents.controller.ts`: rota de transition usa `@CurrentUser()` (pega role do JWT). Novo endpoint `GET /contents/:id/diff?from=&to=`.
**Frontend:**
- `<VersionDiff />`: select de 2 versões, cards lado a lado (vermelho=from, verde=to) só nos campos mudados.
- Página `/revisao` (sidebar com ícone CheckSquare): lista EXPANDED com thumbnail/badges/contagem de palavras, Aprovar/Rejeitar 1-click, esconde botões se role insuficiente, refetch 10s.
- Editor aba Histórico: tabela de versões + `<VersionDiff />` embaixo.
**Smoke test:** transição válida OK, transição inválida bloqueia 400, publishedAt grava, diff identifica campo mudado.
**Adiamento:** endpoint `unpublish` dedicado fundiu com `transition → UNPUBLISHED`. Resposta 410/301 nas rotas públicas vem na Fase 8.

## 2026-05-18 — Added (Fase 4 concluída)
**Fase/Item:** 4 inteira
**Resumo:** Expansion pipeline + editor de conteúdos + geração de imagem real, ponta-a-ponta.
**Backend:**
- Schema: enum `ContentStatus` (7 estados), enum `MediaSource`, models `Content`/`ContentVersion`/`MediaAsset`/`Tag`/`Category` + joins. Migration `fase-4-contents`.
- `ImageGenerationService`: chama OpenRouter `/chat/completions` com `modalities: ['image','text']`, extrai imagem (image_url data-URI ou base64), faz upload via `STORAGE_TOKEN` (R2/local), cria `MediaAsset` + `AiUsageLog` (kind=IMAGE).
- `ExpansionService`: orquestra TITLE→BODY→EXCERPT→META→TAGS→CATEGORY→IMAGE_PROMPT→IMAGE, com fallback de prompts em `DEFAULT_FIELD_PROMPTS` quando o operador ainda não criou template salvo. Slug e JSON-LD são determinísticos. Auto-upsert de Tag/Category.
- `ContentsModule`: CRUD + `/expand` + `/:id/regenerate` + `/:id/cover` + `/:id/transition` + `/:id/versions`. Versionamento automático em cada save.
- `MediaModule`: GET list, POST link externo, POST `/upload` base64, POST `/generate` IA, DELETE.
**Frontend:**
- Tipos `Content`, `ContentVersionRow`, `MediaAsset`, `Tag`, `Category`, `ContentStatus`.
- `<SeoPreview />`: Google SERP + OG + Twitter.
- `<RegenerateButton field />` integrado em cada campo do editor.
- `/conteudos` (lista + filtros + thumbnail) e `/conteudos/[id]` (4 abas: Conteúdo / SEO + Preview / Mídia / Histórico).
- `/ideias` ganha botão ⚡ Expandir em ideias PENDING, com confirmação de custo estimado.
**Smoke test:** expansão completa de 1 ideia gerou 1676 palavras + capa em ~80s totais por **US$ 0.10**. ✅
**Correção:** model id correto de imagem é `google/gemini-2.5-flash-image` (sem `-preview`). Atualizado em `.env`/`env.ts`. FLUX.2 Klein 4B não está no listing — registrado em PONTOS-ATENCAO R10.

## 2026-05-18 — Added (Fase 3 concluída)
**Fase/Item:** 3 inteira
**Resumo:** Prompts configuráveis + cliente OpenRouter + métricas de custo, fim-a-fim. Primeira chamada paga validada.
**Backend:**
- Schema: enum `PromptField` (10 valores), `PromptTemplate` (unique `[siteId, contentTypeId, field, locale]`), `AiUsageLog`, enum `AiUsageKind`. Migration `fase-3-prompts-ai`.
- `shared/ai/`: `OpenRouterClient` (axios, headers `HTTP-Referer` e `X-Title`), `PromptEngineService` (interpolação `{{var}}` e `{{path.dotted}}`), `AiService.runChat` (cache in-memory sha256, log automático, fallback de pricing estático em `model-pricing.ts`).
- `AiController`: `GET /ai/models`, `POST /ai/preview`.
- `PromptsModule`: CRUD restrito a ADMIN.
- `MetricsModule`: `GET /metrics/ai-cost?days=&siteId=` agrega total + byModel + byDay (raw SQL via `$queryRaw`) + recent 20.
**Frontend:**
- Tipos: `PromptTemplate`, `ModelInfo`, `AiPreviewResult`, `AiUsageMetrics`, `PROMPT_FIELDS`.
- `lib/format.ts`: `formatCostCents` (decimal adaptativo para sub-cent), `formatTokens` (k/M).
- `<PromptFormDialog />`: modelo com pricing visível no select, system/user prompts, temperature, max tokens.
- `<PlaygroundDialog />`: extrai variáveis automaticamente, gera campos inline (textarea para `briefing` e paths com `.`), aviso de custo, toggle `useCache`, exibe resposta + tokens + custo + cache HIT.
- Página `/prompts` (CRUD + ação Playground), página `/custos` (cards + gráfico de barras por dia + tabela por modelo + últimas 20).
**Smoke test:** prompt TITLE → playground (gemini-2.5-flash) → texto válido em 2s. Repetir → cache HIT. Metrics agregados corretos.
**Decisões registradas:**
- Cache em memória (não Redis) na v1. Suficiente para single-instance; Redis quando escalar.
- Pricing fallback estático em `model-pricing.ts` (atualizar manualmente conforme OpenRouter mudar).
- `costCents` int — perda de precisão sub-centavo conhecida; revisitar quando volume justificar.

## 2026-05-18 — Added (Fase 2 concluída)
**Fase/Item:** 2 inteira
**Resumo:** Sistema de ideias com importação em lote ponta-a-ponta.
**Backend:**
- Schema: enum `IdeaStatus (PENDING/EXPANDED/DISCARDED)`, modelo `Idea` com `siteId`, `contentTypeId`, `titleSeed`, `briefing`, `keywords[]`, `locale`, `notes`, `createdBy`. Migration `fase-2-ideas`.
- `IdeasModule`: `GET /ideas` (paginado, filtros: siteId/contentTypeId/status/search), `GET /ideas/:id`, `POST /ideas`, `POST /ideas/bulk` (até 1000), `PATCH /ideas/:id`, `DELETE /ideas/:id`, `POST /ideas/bulk-delete`, `POST /ideas/bulk-status`, `POST /ideas/bulk-move-type`. Validação cross-tenant (`contentTypeId` deve pertencer ao `siteId`).
- Aceita `X-Site-Id` header (via `@CurrentSiteIdOptional`) como fallback de filtro.
**Frontend:**
- `lib/parsers.ts`: parsers próprios (sem dep) para Paste, Markdown (-, *, 1.), JSON (array), CSV (suporta aspas).
- `<BulkImportDialog />` com 4 abas, preview editável (tabela inline), validação live.
- `<IdeaFormDialog />` para create/edit individual.
- Página `/ideias`: tabela paginada, filtros (status/tipo/search), seleção múltipla com bulk actions (mudar status, apagar).
- Sidebar atualizado (remove badge "Fase 2" de Ideias).
**Smoke test:** import 3 ideias → list → bulk-status 2 → bulk-delete 2 → total final = 1. ✅
**Decisão registrada:** parsers ficam no frontend (parse + preview + edição antes de enviar array tipado ao backend). Mais simples que enviar arquivos brutos e o operador valida visualmente.

## 2026-05-18 — Added (Fase 1 concluída)
**Fase/Item:** 1 inteira
**Resumo:** Multi-tenant + auth JWT + CRUDs de sites, content-types, admin-users entregues ponta-a-ponta com frontend.
**Backend:**
- Schema Prisma: enum `AdminRole (ADMIN/EDITOR/REVISOR)`, modelos `AdminUser`, `Site`, `ContentType`. Migration `fase-1-multi-tenant-auth` aplicada.
- `prisma/seed.ts`: cria admin (`admin@seoblog.local` / `123456`), site Health Voice (locales `pt-BR,en,es`, author "Health Voice"), content types `blog` e `noticia`.
- `shared/auth/`: `JwtModule.registerAsync` global (`expiresIn:'7d'`), `AuthService` (login/me/changePassword com bcryptjs), `AuthController` (POST /auth/login, GET /auth/me, POST /auth/change-password), `AuthGuard` global via `APP_GUARD` (suporta `@IsPublic`, `@RequiresSecurityToken`), `AdminGuard` via `APP_GUARD` (consume `@IsAdmin`).
- `modules/sites/`: CRUD completo com DTOs validados (class-validator + Swagger).
- `modules/content-types/`: CRUD com filtro `?siteId=`.
- `modules/admin-users/`: CRUD + `POST /:id/reset-password`, classe inteira `@IsAdmin()`.
- `shared/tenant/current-site.decorator.ts`: `@CurrentSiteId` / `@CurrentSiteIdOptional` extraem `X-Site-Id`.
**Frontend:**
- UI primitives próprias: `Button`, `Input`, `Label`, `Card`, `Table`, `Badge`, `Dialog`, `Select`.
- Providers: `QueryProvider` (TanStack Query), `AuthProvider` (gerencia user + redirect), `CurrentSiteProvider` (lista sites + currentSiteId persistido em localStorage).
- `AppShell`: sidebar (com placeholders das próximas fases marcadas) + topbar com SiteSwitcher + user/logout.
- Páginas: `/login` (form real), `/dashboard` (cards de status), `/sites` (CRUD Dialog), `/content-types` (CRUD por site), `/users` (CRUD restrito a ADMIN).
- Route group `app/(admin)/` com layout protegido.
**Smoke test:** login → me → list sites → list content-types → list admin-users → unauth → 401. tsc clean.
**Adiado:** shadcn/ui CLI propriamente (UI primitives próprias bastam por enquanto); `revoked_tokens` blacklist (JWT 7d stateless por enquanto); `CurrentLocaleProvider` (Fase 7.1); `pino` (Fase 2+).

## 2026-05-18 — Decision (Roles do painel)
**Fase/Item:** 1.B5
**Resumo:** 3 roles: `ADMIN` (tudo), `EDITOR` (cria/edita conteúdo), `REVISOR` (aprova/rejeita). Implementação atual: `AdminGuard` cobre `ADMIN`. Os checks finos de EDITOR/REVISOR ficarão na Fase 5 (workflow de revisão).

## 2026-05-18 — Added (Fase 0 concluída)
**Fase/Item:** 0 inteira
**Resumo:** Scaffold backend NestJS + frontend Next.js entregue e validado ponta-a-ponta.
**Detalhe:**
- Backend (`seo-blog-backend`): NestJS 10, Prisma 6, `EnvModule` tipado com zod, `DatabaseModule` + `PrismaService`, `StorageModule` (R2 + LocalDisk fallback), `QueueModule` (BullMQ), filtro global, decorators (`@IsPublic`, `@RequiresSecurityToken`, `@IsAdmin`, `@CurrentUser`, `@CurrentUserId`), Swagger + Scalar reference, health-check em `/health`. `legacy-peer-deps` ativado via `.npmrc` (mesmo padrão do health-voice-api).
- Frontend (`seo-blog-frontend`): Next.js 15.5 (App Router), Tailwind com tokens CSS prontos para shadcn, `lib/api.ts` (fetch wrapper + JWT cookie/localStorage), `lib/env.ts` zod-validado, páginas `/`, `/login` (placeholder), `/dashboard` (consome health).
- Infra: `docker-compose.yml` com Postgres 16 + Redis 7 (healthchecks). Ambos containers saudáveis.
**Adiado para Fase 1:** pino logger (`0.B5`), shadcn/ui CLI (`0.F2`), AppShell + sidebar (`0.F3`), theme toggle (`0.F5`).
**Verificação:**
- `curl http://localhost:3333/health` → `{"status":"ok","services":{"db":"up"}}`
- Frontend `/`, `/login`, `/dashboard` → 200

## 2026-05-18 — Decision (Storage adapter)
**Fase/Item:** 0.B8
**Resumo:** Storage feito como interface `IStorage` com dois adapters: `R2StorageService` (produção) e `LocalDiskStorageService` (dev). Seleção via env `STORAGE_DRIVER=r2|local`. Em dev sem credenciais R2, app sobe e salva em `./uploads` servido por `/uploads` estático.
**Motivo:** Permitir Fase 4 (geração de imagem) funcionar antes de termos bucket R2 provisionado.

## 2026-05-18 — Decision (npm legacy-peer-deps)
**Fase/Item:** 0
**Resumo:** `.npmrc` com `legacy-peer-deps=true` no backend.
**Motivo:** NestJS 10 vs `@nestjs/swagger@11` têm peer dep conflitante; mesmo workaround usado pelo `health-voice-api`. Sem isso o `npm install` falha.

## 2026-05-18 — Decision (Padrões da casa)
**Fase/Item:** 0 e 1 / arquitetura
**Resumo:** Seguir integralmente o padrão do `health-voice-api`:
- **Estrutura de pastas:** `src/modules/<dominio>/` + `src/shared/<infra>/`
- **Auth:** JWT custom via `@nestjs/jwt` + `bcryptjs`. `JwtModule.registerAsync` global (`expiresIn:'365d'`), `AuthGuard` global via `APP_GUARD`, decorators `@IsPublic()` / `@RequiresSecurityToken()`, `AdminGuard` para rotas privilegiadas. Tabela `admin_users` (campo `password`, role em UPPERCASE).
- **Env:** `EnvModule` + `EnvService` tipado.
- **DB:** Postgres self-hosted + Prisma (`shared/database/prisma/`).
- **Storage:** **Cloudflare R2** via `@aws-sdk/client-s3` (copiar `R2Storage` do health-voice-api).
**Motivo:** consistência com codebase existente; equipe já fluente; código copiável.

## 2026-05-18 — Decision (Storage)
**Fase/Item:** 0.B8
**Resumo:** Cloudflare R2 (não Supabase Storage, não MinIO). Mesmo bucket-pattern do health-voice-api.
**Motivo:** padrão já em uso; sem custo de egress; S3-compatible.

## 2026-05-18 — Decision (DB)
**Fase/Item:** 0.B2
**Resumo:** Postgres self-hosted (Docker em dev, VPS em produção). Sem Supabase.
**Motivo:** decisão do usuário — Supabase usado em outras LPs apenas para tracking, não para banco principal.

## 2026-05-18 — Decision (Fase 0 / Infra)
**Fase/Item:** 0 / hospedagem
**Resumo:** Backend + worker rodam em **Docker local** no PC do dev até o provisionamento de uma **VPS dedicada**. Sem PaaS.
**Motivo:** Decisão do usuário; permite tunar custo e ter controle total.
**Impacto:** docker-compose dev é o setup oficial até VPS pronta; em FASE 12 incluir scripts de deploy para VPS.

## 2026-05-18 — Decision (Fase 8)
**Fase/Item:** 8.B7 / domínio público
**Resumo:** API pública vai usar **um único domínio cross-produto** (ex.: `api.seoblog.<empresa>.com`), não subdomínio por LP.
**Motivo:** Vai atender múltiplos produtos (Health Voice, JuridIA, logística etc.); um domínio só simplifica DNS, CORS, certificado e cache.

## 2026-05-18 — Decision (i18n)
**Fase/Item:** 1 / schema + 7.1
**Resumo:** Infra de i18n implementada na v1 (pt-BR, en, es) via campos `locale` em `contents` e `supported_locales` em `sites`. Go-live obrigatório só pt-BR; en/es ficam disponíveis quando o time decidir traduzir.
**Motivo:** Solicitação do usuário, custo de implementar agora é baixo se for projetado desde já.

## 2026-05-18 — Decision (Autoria)
**Fase/Item:** 1 / schema
**Resumo:** Autor visível dos posts = `sites.author_name` (ex.: "Health Voice"). Sem tabela de autores na v1.
**Motivo:** Decisão do usuário — autor é a marca, não pessoa.

## 2026-05-18 — Decision (Despublicar / edits em published)
**Fase/Item:** 5.B5
**Resumo:** Conteúdo `published` pode receber edits (gera nova versão, dispara revalidate, mantém URL). Ação "Despublicar" muda para `unpublished` e responde 410 (ou 301 configurável) na rota pública.
**Motivo:** Resposta da dúvida D8.

## 2026-05-18 — Decision (Imagem — amostras)
**Fase/Item:** R1
**Resumo:** Validação de qualidade do FLUX.2 Klein 4B = **3 amostras** (reduzido de 10).
**Motivo:** Decisão do usuário.

## 2026-05-18 — Out of Scope (Analytics no painel)
**Fase/Item:** D6
**Resumo:** Views/cliques por post **não** entram no painel na v1. Operadores consultam GA das próprias LPs.
**Motivo:** Reduz escopo; LPs já têm GA configurado.

## 2026-05-18 — Out of Scope (Fila pessoal de revisão)
**Fase/Item:** D1
**Resumo:** Sem fila individual por revisor. Time de 5 trabalha sobre uma lista única filtrável.

## 2026-05-18 — Decision (Imagens externas)
**Fase/Item:** D3
**Resumo:** Permitir upload manual de imagens de banco externo (Unsplash/Freepik) como alternativa à IA, na tela de mídia.

## 2026-05-18 — Decision (Fase 0)
**Fase/Item:** 0 / stack geral
**Resumo:** Stack definida — NestJS + Prisma + Postgres (Supabase) + BullMQ/Redis no backend; Next.js 15 (App Router) + shadcn/ui no frontend admin; OpenRouter como gateway de LLM.
**Motivo:** Manter padrão já dominado pela equipe; reduzir tempo até v1; reaproveitar infra Supabase existente.
**Alternativas avaliadas:** Strapi/Payload/Directus (descartados por engessar fluxo de IA customizado e versionamento de prompts).

## 2026-05-18 — Decision (Fase 4)
**Fase/Item:** 4 / geração de imagem
**Resumo:** Modelo default de imagem = **FLUX.2 Klein 4B** ($0.014/MP). Fallback de qualidade = **Gemini 2.5 Flash Image (Nano Banana)**.
**Motivo:** Custo mais baixo por imagem para volume alto (365+ posts × N sites). Configurável por site.
**Risco:** Qualidade pode ser inferior — validar com amostras reais antes do go-live.

## 2026-05-18 — Decision (Fluxo)
**Fase/Item:** Fluxo de expansão
**Resumo:** Expansão por IA acontece **sob demanda do operador** (botão "Expandir"), não automaticamente no import.
**Motivo:** Controle de custo; permite revisão da lista antes de gastar com geração; alinhado com a preferência do chefe.

## 2026-05-18 — Decision (SEO)
**Fase/Item:** 10 / integração LPs
**Resumo:** SEO entregue via Next.js `generateMetadata` + `<JsonLd />` + ISR com `revalidate` + webhook on-demand.
**Motivo:** LPs já são Next.js (`health-voice-institutional-v2` confirmado). Não há trabalho de migração.

---

## Template em branco para próximas entradas

```
## YYYY-MM-DD — <Tipo> (Fase <N>)
**Fase/Item:**
**Resumo:**
**Detalhe:**
**Impacto:**
```
