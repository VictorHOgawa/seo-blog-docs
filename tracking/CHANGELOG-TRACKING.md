# Changelog — Iniciativa de Tracking

> **Documento vivo.** Toda entrega, decisão técnica, mudança de escopo ou novo evento entra aqui.
> Formato: [Keep a Changelog](https://keepachangelog.com/) adaptado.
> **Ordem:** mais recente no topo.

---

## Como registrar

Cada entrada:

- **Data** ISO (`YYYY-MM-DD`).
- **Tipo:** `Added` | `Changed` | `Fixed` | `Removed` | `Decision` | `Pivot` | `Added (event)` | `Changed (playbook)`.
- **Fase/Item:** referência ao [`PLANO-TRACKING.md`](./PLANO-TRACKING.md) (ex.: `Fase 1`, `Fase 2.CTA`).
- **Resumo curto** (1 linha).
- **Detalhe / motivo** (especialmente para `Decision` e `Pivot`).
- **Arquivos afetados** (quando aplicável).

Exemplo:

```
## 2026-05-20 — Decision (Fase 1)
**Fase/Item:** Fase 1 / schema
**Resumo:** Geo lookup via `geoip-lite` em vez de serviço externo.
**Motivo:** Custo zero, latência menor; precisão suficiente pra dashboard. Re-avaliar se precisar de cidade granular.
**Arquivos:** `seo-blog-backend/src/modules/tracking/tracking.service.ts`
```

---

## 2026-05-20 — Added (Fase 1 concluída — backend de ingestão)
**Fase/Item:** Fase 1 inteira
**Resumo:** Schema Prisma + módulo `tracking` no `seo-blog-backend`. Ingestão funcional e validada.
**Branch:** `feat/tracking-hub-backend`.
**Schema:**
- Migration `20260520120000_add_tracking_hub`: `Site.publicKey` (UUID unique, default `gen_random_uuid()`) + `Site.trackingEnabled` (default `false`); tabelas `tracking_sessions`, `tracking_events`, `tracking_attribution`, `tracking_leads`, `tracking_consent_log`.
- `siteId` é coluna indexada (padrão `ai_usage_log`), sem relação Prisma; eventos↔sessão ligam por valor de `sessionId`, sem FK (evita race R10).
**Módulo `src/modules/tracking/`:**
- `TrackingController` — rotas públicas `POST /tracking/{session,event,lead,consent}`, `@IsPublic` + `SiteKeyGuard`, `@Throttle` ampliado.
- `TrackingService` — upsert idempotente (sessão por `sessionId`, evento por `eventId`, lead por hash em janela de 5 min). Atribuição first-touch imutável.
- `SiteKeyGuard` — valida `X-Site-Key` contra `Site.publicKey` (401/403).
- DTOs em **class-validator** (padrão do projeto — ver Decision abaixo).
- `tracking.util.ts` — hash de IP, detecção de device/bot, hash de dedup.
- `main.ts` — middleware para parsear `text/plain` (sendBeacon).
- `Site` (DTO + service) ganhou `trackingEnabled` editável pelo admin.
**Validação manual (curl, 2026-05-20):** 401 sem/má key · 400 payload inválido · 403 `trackingEnabled=false` · 204/202/200 caminho feliz · idempotência (2× `eventId` → 1 linha; replay de lead → `deduplicated:true`) · `text/plain` · IP hasheado, e-mail normalizado, atribuição persistida. Dados de teste removidos do banco após validação.
**Adiado / débito:** testes e2e automatizados; CORS allowlist por site (DP3); geo lookup `country`/`city` (DP2); rate-limit por `sessionId`/`siteId` (hoje por IP).
**Próximo passo:** Fase 2 — `lib/tracking/` na LP `health-voice-institutional-v2` (branch `feat/tracking-hub-integration`).

## 2026-05-20 — Decision (Fase 2+)
**Fase/Item:** Fase 2 / validação
**Resumo:** A validação do pipeline a partir da Fase 2 será E2E automatizada via **playwright-mcp** (browser real → backend → asserção no Postgres), não só `curl`.
**Motivo:** `curl` valida o endpoint isoladamente; o pipeline real envolve o client da LP (sessão, UTM, consent, batch). Scripts playwright que exercitam a LP e conferem as tabelas `tracking_*` viram a base de validação de toda LP nova. Plano detalhado entra em `PLANO-TESTES-E2E.md` quando a Fase 2 tiver código.

## 2026-05-20 — Decision (Fase 1)
**Fase/Item:** Fase 1 / DTOs
**Resumo:** DTOs do módulo `tracking` usam **class-validator**, não Zod.
**Motivo:** O `seo-blog-backend` valida DTOs com class-validator + `ValidationPipe` global (`whitelist`/`transform`); Zod no projeto é só para env. Manter class-validator é consistência — Zod isolado no tracking criaria dois padrões. `ARQUITETURA-TRACKING.md §4.3`, §5 e a skill `tracking-endpoint` foram corrigidas.

## 2026-05-19 — Added (Fase 0)
**Fase/Item:** Fase 0 / skills + agents
**Resumo:** Criados os `.claude/` do front e do back com as skills e agents da iniciativa.
**Frontend** (`seo-blog-frontend/.claude/`):
- skills: `lp-tracking-integration`, `lp-event-catalog`, `analytics-dashboard-page`, `tracking-consent-banner`.
- agents: `lp-tracking-auditor`, `analytics-dashboard-reviewer`.
**Backend** (`seo-blog-backend/.claude/`):
- skills: `tracking-endpoint`, `tracking-schema-migration`, `analytics-query`.
- agents: `tracking-pipeline-reviewer`.
**Detalhe:** cada skill aponta para os docs vivos de `docs/tracking/` como fonte da verdade e embute o anti-checklist AP1–AP11. README em cada `.claude/`. Branch `feat/tracking-hub-integration` criada na LP piloto `health-voice-institutional-v2` para receber a implementação da Fase 2.
**Próximo passo:** Fase 1 — schema Prisma + módulo `tracking` no backend.

## 2026-05-19 — Audit + Added (Fase 0)
**Fase/Item:** Fase 0 / diagnóstico
**Resumo:** Auditoria das LPs `health-lp-v2`, `lp-health-2026` e `health-voice-institutional`. Criado [`LICOES-LPS-EXISTENTES.md`](./LICOES-LPS-EXISTENTES.md).
**Achados:**
- `health-lp-v2` tem o tracking mais rico do grupo (`src/lib/analytics.ts`): seção com dwell/scroll/viewport, vídeo com heartbeat, FAQ, input campo-a-campo, cadastro por etapa. → vira base do catálogo de eventos.
- **Causa raiz da falha de visualização identificada:** `health-lp-v2` clona ~45 tabelas Supabase por LP (`analytics_home_*`, `analytics_lp2_*` … `analytics_lp6_*`), com o id da LP no *nome da tabela*. Dashboard `analytics2` clona código por LP (`components/2/`, `components/4/`), é tabela crua sem funil/gráfico, carrega tudo no client sem paginação.
- Catalogados 11 anti-padrões (AP1–AP11) que viram restrições de arquitetura e base de reprovação dos agents de review.

## 2026-05-19 — Added (event)
**Fase/Item:** Fase 0 / catálogo
**Resumo:** 6 eventos de engajamento granular adicionados ao [`CATALOGO-EVENTOS.md`](./CATALOGO-EVENTOS.md): `section_enter`, `section_exit`, `video_progress`, `faq_toggle`, `field_filled`, `form_step`.
**Motivo:** A auditoria do `health-lp-v2` provou o valor de medir engajamento real (tempo em seção, % de vídeo, onde o form trava), não só pageview/click.

## 2026-05-19 — Decision (Fase 0)
**Fase/Item:** Fase 0 / arquitetura
**Resumo:** Métricas progressivas (liveness de sessão, progresso de vídeo, dwell de seção) são **estado** (upsert numa linha), não evento append. Decisão D11.
**Motivo:** O `health-lp-v2` registrava heartbeat de 5s como linha de evento — isso explode a tabela. `tracking_events` guarda só marcos discretos; estado contínuo vai em `lastSeenAt` / propriedade de marco.
**Referência:** [[ARQUITETURA-TRACKING#heartbeat]]

## 2026-05-19 — Decision (Fase 0)
**Fase/Item:** Fase 4 / visualização
**Resumo:** Adicionada seção "Princípios de visualização" à arquitetura (§7) e Fase 4 reescrita como insight-first.
**Motivo:** Captura de dados nunca foi o problema das LPs antigas — a visualização foi. Os 8 princípios (insight-first, comparação obrigatória, agregação server-side, dashboard único multi-site, funil como tela central, etc.) corrigem diretamente os anti-padrões AP2–AP4.
**Referência:** [[ARQUITETURA-TRACKING#7-princípios-de-visualização]]

## 2026-05-19 — Added (Fase 0)
**Fase/Item:** Fase 0 / docs
**Resumo:** Criação da pasta `docs/tracking/` com 8 documentos vivos da iniciativa.
**Detalhe:**
- `README.md` — índice + status global.
- `ARQUITETURA-TRACKING.md` — diagrama, schema Prisma, endpoints, módulos cliente, decisões D1–D10.
- `PLANO-TRACKING.md` — fases 0 a 6 com critérios de aceite.
- `PLAYBOOK-NOVA-LP.md` — passo-a-passo replicável para integrar LP nova.
- `CATALOGO-EVENTOS.md` — eventos canônicos + convenções.
- `LICOES-LPS-EXISTENTES.md` — auditoria das LPs existentes + anti-padrões.
- `CHANGELOG-TRACKING.md` (este).
- `PONTOS-ATENCAO-TRACKING.md` — riscos e dúvidas iniciais.

## 2026-05-19 — Decision (Fase 0)
**Fase/Item:** Fase 0 / arquitetura
**Resumo:** Backend `seo-blog-backend` será o hub central de tracking; LPs param de gravar leads em Supabase próprio e passam a enviar tudo via `POST /tracking/*`.
**Motivo:** Eliminar fragmentação. Tentativas anteriores (Inova, Health Voice) sempre falharam em ter uma visão agregada porque cada LP tinha seu próprio banco.
**Alternativas descartadas:**
- Supabase central — descartado por estar fora do padrão da casa (Postgres self-hosted).
- Serviço externo (Segment, PostHog) — custo recorrente e dado fora do nosso domínio.
**Referência:** [[ARQUITETURA-TRACKING#1-visão-geral]]

## 2026-05-19 — Decision (Fase 0)
**Fase/Item:** Fase 0 / arquitetura
**Resumo:** Schema Prisma usará 5 tabelas: `tracking_sessions`, `tracking_events`, `tracking_attribution`, `tracking_leads`, `tracking_consent_log`.
**Motivo:** Separar sessão (contexto de visita), evento (ação granular), atribuição (UTM first/last), lead (entidade comercial) e consent (auditoria LGPD) facilita queries, evita NULLs em massa e atende LGPD.
**Alternativa descartada:** uma única tabela `tracking_events` com tudo em `properties: Json` — flexível, mas dificulta indexação e separação semântica.
**Referência:** [[ARQUITETURA-TRACKING#2-schema-prisma-proposto]]

## 2026-05-19 — Decision (Fase 0)
**Fase/Item:** Fase 0 / cliente
**Resumo:** `anonymousId` em `localStorage` (persistente) + `sessionId` em `sessionStorage` (TTL 30min de inatividade).
**Motivo:** Padrão da indústria (Segment, Amplitude). Permite ver visitante voltando e agrupar pageviews por sessão sem fingerprint.
**Referência:** [[ARQUITETURA-TRACKING#41-sessionmanager]] · Decisão D3 da arquitetura.

## 2026-05-19 — Decision (Fase 0)
**Fase/Item:** Fase 0 / idempotência
**Resumo:** `eventId` (uuid v4) gerado no cliente; backend usa como PK natural + upsert.
**Motivo:** Replays (retry, sendBeacon, conexão instável) jamais duplicam. Sem essa proteção, contagem de eventos é fundamentalmente não confiável.
**Referência:** Decisão D2.

## 2026-05-19 — Decision (Fase 0)
**Fase/Item:** Fase 0 / segurança
**Resumo:** Autenticação via `X-Site-Key` pública (não secret) + CORS allowlist por `Site.domain` + rate-limit por sessão e por site.
**Motivo:** Tracking client roda no browser; toda key que sai do server é pública. Segurança real está em rate-limit, dedup por `eventId`, hash de IP e CORS.
**Referência:** Decisão D7.

## 2026-05-19 — Decision (Fase 0)
**Fase/Item:** Fase 0 / atribuição
**Resumo:** UTM first-touch fixo na criação da sessão (`tracking_attribution`); last-touch atualizável a cada sessão nova com UTM.
**Motivo:** First-touch é métrica de aquisição; last-touch é métrica de otimização de campanha. Ter ambos cobre análise de aquisição e de conversão sem inventar atribuição multi-touch ainda (fica pra Fase 6).
**Referência:** Decisão D4.

## 2026-05-19 — Decision (Fase 0)
**Fase/Item:** Fase 0 / consentimento
**Resumo:** `tracking_consent_log` recebe um registro por decisão; cliente mantém buffer de eventos pré-consent; GTM e demais pixels só carregam após `consent.analytics === true`.
**Motivo:** O Inova tem `ConsentLog` órfão (zero código escreve nele). Corrigimos isso desde o dia 1 para sustentar auditoria LGPD.
**Referência:** Decisão D8.

## 2026-05-19 — Audit
**Fase/Item:** Fase 0 / diagnóstico
**Resumo:** Auditoria da Health Voice (`health-voice-institutional-v2`) e do Inova (`inova-admin-api`, `inova-institutional`, `inova-admin`).
**Output:** seções "Pontos negativos" e "Pontos positivos" alimentaram a arquitetura; falhas do Inova ([sem UTM, sem multi-site, sem dedup, sem consent operacional, sem charts](./PONTOS-ATENCAO-TRACKING.md)) viraram critérios obrigatórios da v1.
