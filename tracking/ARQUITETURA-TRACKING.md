# Arquitetura — Hub de Tracking

> **Documento vivo.** Atualizar a cada decisão técnica. Registrar a entrada correspondente em [`CHANGELOG-TRACKING.md`](./CHANGELOG-TRACKING.md) com tipo `Decision`.
> **Última atualização:** 2026-05-19

---

## 1. Visão geral

```
┌─────────────────────────────┐
│   LP (Next.js)              │
│   ex.: health-voice-v2      │
│                             │
│  ┌───────────────────────┐  │
│  │ lib/tracking/         │  │
│  │  ├─ client.ts         │  │  ── batch + retry + sendBeacon
│  │  ├─ session.ts        │  │  ── anonId (localStorage) + sid (sessionStorage)
│  │  ├─ attribution.ts    │  │  ── parse UTM/gclid/fbclid no first hit
│  │  ├─ consent.ts        │  │  ── gating LGPD + buffer
│  │  └─ events.ts         │  │  ── catálogo type-safe
│  └───────────┬───────────┘  │
└──────────────┼──────────────┘
               │ POST /tracking/event (batched)
               │ + X-Site-Key (publicKey por site)
               ▼
┌─────────────────────────────┐
│   seo-blog-backend (NestJS) │
│                             │
│   modules/tracking/         │
│   ├─ tracking.controller.ts │ ── rate-limit, validação class-validator
│   ├─ tracking.service.ts    │ ── dedup por eventId, persistência
│   ├─ ingest.dto.ts          │ ── shape de payload
│   └─ analytics.service.ts   │ ── agregações para dashboard
└──────────────┬──────────────┘
               │ Prisma
               ▼
┌─────────────────────────────┐
│   Postgres                  │
│   tracking_session          │
│   tracking_event            │
│   tracking_lead             │
│   tracking_consent_log      │
│   tracking_attribution      │
└──────────────┬──────────────┘
               │
               │ Endpoints autenticados:
               │ GET /admin/analytics/overview
               │ GET /admin/analytics/funnel
               │ GET /admin/analytics/attribution
               │ GET /admin/analytics/leads
               ▼
┌─────────────────────────────┐
│   seo-blog-frontend         │
│   (admin)/analytics/        │
│   ├─ visão geral por site   │
│   ├─ funil                  │
│   ├─ atribuição UTM         │
│   ├─ leads recentes         │
│   └─ comparativo            │
└─────────────────────────────┘
```

Pixels de terceiros (GTM, Meta, TikTok) **continuam rodando em paralelo** quando a LP precisar deles. Eles não são removidos — apenas deixam de ser a única fonte. Nosso backend vira a fonte canônica.

---

## 2. Schema Prisma proposto

> Os nomes seguem o padrão já adotado no projeto (`snake_case` no DB via `@map`, camelCase no TS). Todos os modelos têm `siteId` para multi-tenancy.

### 2.1. `tracking_session`

```prisma
model TrackingSession {
  id                String   @id @default(uuid()) @db.Uuid
  siteId            String   @map("site_id") @db.Uuid
  anonymousId       String   @map("anonymous_id")            // persiste em localStorage da LP
  sessionId         String   @unique @map("session_id")      // gerado no cliente, sessionStorage
  startedAt         DateTime @default(now()) @map("started_at")
  lastSeenAt        DateTime @map("last_seen_at")
  endedAt           DateTime? @map("ended_at")
  // First touch (set on session start, immutable)
  firstReferrer     String?  @map("first_referrer")
  firstLandingPath  String?  @map("first_landing_path")
  // Device/context
  userAgent         String?  @map("user_agent")
  deviceType        String?  @map("device_type")             // 'mobile' | 'tablet' | 'desktop'
  locale            String?
  ipHash            String?  @map("ip_hash")                 // sha256(ip + salt)
  country           String?
  city              String?
  // FK
  site              Site     @relation(fields: [siteId], references: [id], onDelete: Cascade)
  events            TrackingEvent[]
  leads             TrackingLead[]
  attribution       TrackingAttribution?

  @@index([siteId, startedAt])
  @@index([anonymousId])
  @@map("tracking_sessions")
}
```

### 2.2. `tracking_event`

```prisma
model TrackingEvent {
  id          String   @id @default(uuid()) @db.Uuid
  eventId     String   @unique @map("event_id")              // uuid gerado no cliente, idempotency key
  siteId      String   @map("site_id") @db.Uuid
  sessionId   String   @map("session_id")
  anonymousId String   @map("anonymous_id")
  name        String                                          // canonical event name (ver CATALOGO-EVENTOS.md)
  path        String?                                         // window.location.pathname no momento do evento
  elementId   String?  @map("element_id")                    // CTA / form ID
  properties  Json     @default("{}")                         // payload livre (validado por nome do evento no backend)
  occurredAt  DateTime @map("occurred_at")                    // client-side timestamp
  receivedAt  DateTime @default(now()) @map("received_at")    // server-side timestamp
  schemaVersion Int    @default(1) @map("schema_version")

  site        Site             @relation(fields: [siteId], references: [id], onDelete: Cascade)
  session     TrackingSession  @relation(fields: [sessionId], references: [sessionId], onDelete: Cascade)

  @@index([siteId, name, occurredAt])
  @@index([sessionId, occurredAt])
  @@map("tracking_events")
}
```

### 2.3. `tracking_attribution`

```prisma
model TrackingAttribution {
  sessionId    String   @id @map("session_id")               // 1:1 com session
  siteId       String   @map("site_id") @db.Uuid
  // First touch — fixado na criação da sessão, nunca atualizado
  utmSource    String?  @map("utm_source")
  utmMedium    String?  @map("utm_medium")
  utmCampaign  String?  @map("utm_campaign")
  utmTerm      String?  @map("utm_term")
  utmContent   String?  @map("utm_content")
  gclid        String?
  fbclid       String?
  referrer     String?
  landingPath  String?  @map("landing_path")
  // Last touch — atualizado a cada nova sessão com UTM
  lastUtmSource    String? @map("last_utm_source")
  lastUtmMedium    String? @map("last_utm_medium")
  lastUtmCampaign  String? @map("last_utm_campaign")
  createdAt    DateTime @default(now()) @map("created_at")

  session      TrackingSession @relation(fields: [sessionId], references: [sessionId], onDelete: Cascade)

  @@index([siteId, utmSource, utmCampaign])
  @@map("tracking_attribution")
}
```

### 2.4. `tracking_lead`

```prisma
model TrackingLead {
  id          String   @id @default(uuid()) @db.Uuid
  siteId      String   @map("site_id") @db.Uuid
  sessionId   String?  @map("session_id")                    // pode ser null se enviado server-to-server
  anonymousId String?  @map("anonymous_id")
  // Identificadores
  name        String?
  email       String?
  phone       String?
  // Contexto
  source      String?                                        // origem informada pela LP (ex.: 'campaing1', 'app_store')
  buttonId    String?  @map("button_id")
  destination String?                                        // URL para onde o CTA levou
  sourceUrl   String?  @map("source_url")
  // Captura snapshot da atribuição no momento da conversão
  utmSource   String?  @map("utm_source")
  utmMedium   String?  @map("utm_medium")
  utmCampaign String?  @map("utm_campaign")
  // Status comercial
  status      String   @default("NEW")                       // NEW | CONTACTED | QUALIFIED | CONVERTED | LOST
  notes       String?
  payload     Json     @default("{}")                        // campos extras específicos da LP
  consentLgpd Boolean  @default(false) @map("consent_lgpd")
  ipHash      String?  @map("ip_hash")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  site        Site             @relation(fields: [siteId], references: [id], onDelete: Cascade)
  session     TrackingSession? @relation(fields: [sessionId], references: [sessionId], onDelete: SetNull)

  @@index([siteId, createdAt])
  @@index([siteId, status])
  @@index([email])
  @@map("tracking_leads")
}
```

### 2.5. `tracking_consent_log`

```prisma
model TrackingConsentLog {
  id                String   @id @default(uuid()) @db.Uuid
  siteId            String   @map("site_id") @db.Uuid
  anonymousId       String   @map("anonymous_id")
  sessionId         String?  @map("session_id")
  consentAnalytics  Boolean  @map("consent_analytics")
  consentMarketing  Boolean  @map("consent_marketing")
  consentVersion    String   @map("consent_version")          // ex.: "2026-05-19-v1" — versão do texto do banner
  ipHash            String?  @map("ip_hash")
  userAgent         String?  @map("user_agent")
  createdAt         DateTime @default(now()) @map("created_at")

  site              Site     @relation(fields: [siteId], references: [id], onDelete: Cascade)

  @@index([siteId, anonymousId])
  @@map("tracking_consent_log")
}
```

### 2.6. Adição em `Site`

```prisma
model Site {
  // ... campos existentes
  publicKey         String   @unique @map("public_key")        // chave pública usada pelo client da LP
  trackingEnabled   Boolean  @default(true) @map("tracking_enabled")
  // ...
}
```

`publicKey` é gerada na criação do site, fica visível no painel admin, e o cliente da LP envia em todo POST como header `X-Site-Key`. Não é secret — é apenas para o backend saber qual site está enviando e para barrar tráfego inválido com 401.

### 2.7. Métricas progressivas (heartbeat) — estado, não evento {#heartbeat}

O `health-lp-v2` provou o valor de medir engajamento contínuo (tempo real na página, % do vídeo assistido). Mas registrar isso como **evento append a cada 5s** explode a `tracking_events` (1 vídeo de 3min = ~36 linhas só de progresso × cada visitante).

**Regra:** métrica progressiva é **estado** — atualiza uma linha única por upsert. Não é evento.

- **Liveness de sessão:** já coberto por `TrackingSession.lastSeenAt`. O client manda um ping leve (`POST /tracking/session` com o `sessionId`) a cada ~15s enquanto a aba está visível; o backend só faz `UPDATE ... SET last_seen_at = now()`. Tempo na página = `lastSeenAt - startedAt`.
- **Progresso de vídeo:** o evento `video_progress` ([`CATALOGO-EVENTOS`](./CATALOGO-EVENTOS.md)) registra **apenas marcos** (25/50/75/100%). O "% máximo assistido nesta sessão" é mantido pelo client e enviado como propriedade do marco — não como stream contínuo.
- **Dwell time de seção:** o `section_exit` carrega `dwellMs` calculado no client (diferença entre enter e exit). Um evento por par enter/exit, não heartbeat.

Se no futuro precisarmos de granularidade de segundo-a-segundo, isso vai para uma tabela de estado dedicada (`tracking_engagement_state`, upsert por `sessionId+target`), nunca para `tracking_events`. Fora do escopo da v1.

---

## 3. Endpoints

### 3.1. Públicos (consumidos pela LP)

| Método | Rota | Propósito |
|---|---|---|
| `POST` | `/tracking/session` | Criar/atualizar sessão. Idempotente por `sessionId`. |
| `POST` | `/tracking/event` | Enviar evento (ou batch de eventos). Idempotente por `eventId`. |
| `POST` | `/tracking/lead` | Criar lead. Idempotente por hash(`email+phone+siteId`) dentro de janela de 5min. |
| `POST` | `/tracking/consent` | Registrar decisão de consentimento. |

Todos exigem `X-Site-Key`. CORS aberto para o `domain` do site (lookup `Site.domain` no allowlist).

**Rate limit:** 60 req/min por `sessionId` + 1000 req/min por `siteId` (NestJS `@nestjs/throttler` já está no projeto).

**Dedup:** Upsert por `eventId` (PK natural). Replay do mesmo `eventId` retorna 200 sem inserir.

**sendBeacon:** o client deve preferir `navigator.sendBeacon` no `unload`. Endpoint precisa aceitar `Content-Type: text/plain` quando vindo de sendBeacon.

### 3.2. Privados (consumidos pelo dashboard)

Autenticação JWT padrão do projeto (`AuthGuard` global). Filtro por `siteId` obrigatório (ou `all`, se admin).

| Método | Rota | Retorno |
|---|---|---|
| `GET` | `/admin/analytics/overview?siteId=...&days=30` | Cards: sessions, pageViews, events, leads, conversion rate, top events |
| `GET` | `/admin/analytics/funnel?siteId=...&days=30` | Funil padrão: page_view → cta_click → form_view → form_submit → lead_created |
| `GET` | `/admin/analytics/attribution?siteId=...&days=30` | Breakdown por utm_source/medium/campaign + leads atribuídos |
| `GET` | `/admin/analytics/leads?siteId=...&page=1` | Lista paginada de leads com filtros (status, source, utm) |
| `GET` | `/admin/analytics/timeseries?siteId=...&metric=pageviews&days=30` | Série temporal por dia |
| `GET` | `/admin/analytics/leads.csv?siteId=...&days=30` | Export CSV |

---

## 4. Cliente da LP — `lib/tracking/`

Estrutura proposta (válida tanto para Next.js App Router quanto Pages Router):

```
lib/tracking/
├── index.ts          // re-exports + setup do provider
├── client.ts         // singleton TrackingClient (batch, retry, sendBeacon)
├── session.ts        // SessionManager: anonId (localStorage), sid (sessionStorage com TTL 30min)
├── attribution.ts    // parse de UTM/gclid/fbclid no first hit; persiste em sessionStorage
├── consent.ts        // ConsentManager: estado, buffer de eventos pré-consent, persiste em localStorage
├── events.ts         // catálogo type-safe: tipo discriminado por `name`
├── provider.tsx      // <TrackingProvider siteSlug publicKey endpoint>
├── hooks.ts          // useTracking(), useAutoPageView()
└── components/
    ├── TrackedButton.tsx
    └── ConsentBanner.tsx
```

### 4.1. SessionManager

```ts
// session.ts (resumo)
const ANON_KEY = 'sb:anon';                  // localStorage, nunca expira
const SID_KEY  = 'sb:sid';                   // sessionStorage, expira em 30min de inatividade

export function getOrCreateAnonymousId(): string {
  const existing = localStorage.getItem(ANON_KEY);
  if (existing) return existing;
  const id = crypto.randomUUID();
  localStorage.setItem(ANON_KEY, id);
  return id;
}

export function getOrCreateSessionId(): string {
  // Verifica TTL; se expirou, cria nova sessão (e dispara session_start no client.ts)
}
```

### 4.2. Consentimento LGPD — modelo híbrido (Fase 3)

A LGPD admite mais de uma base legal. Em vez de opt-in para tudo, o tracking
usa **bases distintas por categoria** (decisão 2026-05-20):

| Categoria | Cobre | Base legal | Modelo |
|---|---|---|---|
| `analytics` | hub próprio (dado fica conosco, IP hasheado, storage first-party) | legítimo interesse | **opt-out** (default ligado) |
| `marketing` | GTM / Meta / TikTok (compartilham com terceiros) | consentimento | **opt-in** (default desligado) |

```ts
// consent.ts (resumo) — duas categorias independentes
const DEFAULTS = { analytics: true, marketing: false }; // híbrido

hasAnalyticsConsent(): boolean   // default true  → hub roda já
hasMarketingConsent(): boolean   // default false → GTM espera opt-in
hasDecided(): boolean            // controla a exibição do banner
setConsent({ analytics, marketing })  // persiste + notifia listeners
```

- O **client** gateia `sendSession` / `lead` / `flush` em `hasAnalyticsConsent()`
  e `mirrorToDataLayer` em `hasMarketingConsent()`.
- `<GtmLoader>` injeta o GTM **só** quando `hasMarketingConsent()` vira `true`.
- `<ConsentBanner>` é informativo (Aceitar / Recusar / Preferências) e some após
  a decisão; `/preferencias-cookies` permite revisar/revogar a qualquer momento.
- `client.updateConsent()` persiste, faz `POST /tracking/consent` (auditoria em
  `tracking_consent_log`) e reage — flush se analytics foi religado, descarte da
  fila se foi desligado.
- Não há "buffer pré-consent" para analytics: como o default é opt-out, o hub
  rastreia desde o primeiro hit. O represamento só ocorreria se o usuário fizer
  opt-out — aí a fila é descartada.

### 4.3. Catálogo de eventos type-safe

```ts
// events.ts (resumo — catálogo completo em CATALOGO-EVENTOS.md)
export type TrackingEvent =
  | { name: 'page_view'; properties: { path: string; title: string } }
  | { name: 'cta_click'; elementId: string; properties: { label: string; destination?: string } }
  | { name: 'form_view'; elementId: string }
  | { name: 'form_submit'; elementId: string; properties: { fields: string[] } }
  | { name: 'lead_created'; properties: { leadId: string; source: string } }
  | { name: 'scroll_depth'; properties: { percent: 25 | 50 | 75 | 100 } }
  | { name: 'video_play'; elementId: string; properties: { src: string } };
```

No backend, os DTOs (`ingest-*.dto.ts`) validam o envelope com **class-validator** (padrão do projeto — ver [[CHANGELOG-TRACKING]] 2026-05-20). Na v1 o `properties` é aceito como objeto genérico; a validação estrita de `properties` **por nome de evento** (contra o catálogo) é refinamento previsto — quando entrar, fica no `TrackingService`, modo `warn` por default e `strict` via env.

---

## 5. Decisões de design (com motivo)

| # | Decisão | Motivo |
|---|---|---|
| D1 | Backend próprio como fonte da verdade (não só GTM/GA4) | Sem ele, decisões comerciais ficam reféns de dashboards externos e exports manuais |
| D2 | `eventId` gerado no cliente (uuid v4) | Idempotência: retry/sendBeacon nunca duplicam |
| D3 | `anonymousId` em localStorage + `sessionId` em sessionStorage | Permite ver visitante voltando (anon) e agrupar pageviews por sessão (sid) — padrão da indústria |
| D4 | UTM first-touch imutável + last-touch atualizável | Métrica de aquisição precisa ser estável; otimização de campanha precisa do último |
| D5 | `properties: Json` no event + validação por nome (refinamento) | Flexibilidade pra adicionar eventos sem migration; validação estrita por nome vem depois |
| D6 | `tracking_lead` separado de `tracking_event` | Lead é entidade comercial, tem ciclo de vida (status), merece tabela própria |
| D7 | `X-Site-Key` pública (não secret) + CORS allowlist | Tracking é por natureza público; segurança é via rate-limit + dedup + IP hash |
| D8 | `ConsentLog` realmente escrito (corrige falha do Inova) | Auditoria LGPD; sem isso, não há como provar consentimento |
| D9 | `siteId` em TUDO | Multi-tenancy nativa. Inova falhou aqui — single-tenant complica reaproveitamento |
| D10 | Não remover GTM/pixels existentes | Meta Ads / Google Ads precisam dos pixels deles pra otimização — coexistência |
| D11 | Métrica progressiva é estado (upsert), não evento append | Heartbeat de 5s como evento explode `tracking_events`. Ver [[#heartbeat]] e [[LICOES-LPS-EXISTENTES#ap8]] |

---

## 6. Referências às implementações que servem de base

| Arquivo do Inova | O que reaproveitar | O que adaptar |
|---|---|---|
| `inova-admin-api/src/modules/tracking/tracking.routes.ts` | Shape dos endpoints `pageview` / `event` | Trocar Fastify → NestJS, adicionar rate-limit, dedup, `siteId` |
| `inova-admin-api/prisma/schema.prisma` (models `PageView`, `TrackEvent`, `CalculatorLead`, `ConsentLog`) | Estrutura das tabelas | Adicionar `siteId`, `eventId`, `anonymousId`, separar `TrackingAttribution` |
| `inova-institutional/src/components/Tracker.tsx` | Padrão de cliente fire-and-forget + sessionId | Modularizar em arquivos por responsabilidade, adicionar UTM, consent, batch |
| `inova-institutional/lib/inova-api.ts` | Adapter pattern + ISR | Direto, sem grandes mudanças |

| Arquivo da Health Voice | O que aproveitar |
|---|---|
| `health-voice-institutional-v2/src/app/api/campaign-lead/route.ts` | Validação de form (regex email, phone) — virar utility |
| `health-voice-institutional-v2/src/components/CampaignLeadModal.tsx` | Modal reutilizável com `source`, `buttonId`, `destination` |
| `health-voice-institutional-v2/.../campaing1/page.tsx` (CTA_BUTTON_IDS) | Padrão de enum de buttonId — formalizar como convenção |
| **`health-lp-v2/src/lib/analytics.ts`** | **Referência de _o que_ rastrear**: seção (dwell/scroll/viewport), vídeo (heartbeat), FAQ, input campo-a-campo, cadastro por etapa | Reescrever como eventos canônicos; **não** copiar o modelo de tabela-por-LP (ver [[LICOES-LPS-EXISTENTES#ap1]]) |
| `lp-health-2026/lib/meta-capi.ts` | Padrão Pixel + CAPI com `event_id` compartilhado + normalização/hash de PII | Mantém na LP como camada paralela ao hub |

> ⚠️ A análise completa do que portar e dos **anti-padrões AP1–AP11** vive em [`LICOES-LPS-EXISTENTES.md`](./LICOES-LPS-EXISTENTES.md). Leitura obrigatória antes da Fase 1 e da Fase 4.

---

## 7. Princípios de visualização (o dashboard que dá decisão)

> Esta seção existe porque a *captura* de dados nas LPs antigas era boa, mas a *visualização* sempre falhou. O `health-lp-v2` tem dados ricos e mesmo assim ninguém consegue decidir nada com eles. Os princípios abaixo são a correção direta dos anti-padrões AP2–AP4 de [`LICOES-LPS-EXISTENTES.md`](./LICOES-LPS-EXISTENTES.md).

1. **Insight-first, tabela por último.** A tela principal responde perguntas de negócio ("a conversão caiu?", "qual campanha traz lead?", "onde o funil vaza?") com número grande + tendência + comparação. Tabela de sessões cruas é **drill-down**, acessível mas nunca a primeira coisa.

2. **Toda métrica tem comparação.** Número sozinho não decide. Todo card mostra variação vs. período anterior (▲▼ %) e/ou contra a meta. "1.200 leads" não diz nada; "1.200 leads, ▼18% vs. mês passado" diz.

3. **Agregação no backend, nunca no browser.** Endpoints `/admin/analytics/*` retornam dados **já resumidos** (SQL `GROUP BY`, `date_trunc`, `FILTER`). O front recebe pronto pra plotar. Nada de baixar 100k linhas e `reduce` no client (AP4).

4. **Um dashboard, N sites.** Zero código por LP. Site novo aparece no seletor automaticamente porque o filtro é `siteId` (AP2). Comparativo entre sites é first-class.

5. **O funil é a tela mais importante.** `page_view → section engajada → cta_click → form_view → form_submit → lead_created`. Cada etapa mostra volume + % de passagem + maior ponto de vazamento destacado.

6. **Atribuição responde "de onde vem o dinheiro".** Breakdown de leads por `utm_source`/`medium`/`campaign`, com first-touch e last-touch lado a lado. Linka campanha → lead → status comercial.

7. **Estados explícitos.** Toda página trata `loading` (skeleton), `empty` ("site sem dados ainda — verifique a integração"), `error` (retry). Dado parcial nunca é mostrado como se fosse completo.

8. **Cada tela cabe numa decisão.** Se uma tela não muda o que alguém vai fazer na segunda-feira, ela não entra. Métrica de vaidade (ex.: "total de eventos") fica no rodapé ou fora.

A implementação dessas telas é a **Fase 4** do [`PLANO-TRACKING.md`](./PLANO-TRACKING.md).

---

## 8. O que esta arquitetura **deliberadamente não faz** (escopo controlado)

- Não implementa fingerprint de dispositivo (privacidade + LGPD; `anonymousId` é suficiente).
- Não implementa server-side rendering de pixel (GTM server-side container) — fica para evolução futura.
- Não implementa atribuição multi-touch ponderada (linear, time-decay) na v1 — só first/last touch. Modelos avançados ficam para a Fase 6.
- Não implementa A/B testing na v1 — pode ser construído em cima do `properties.variant` depois.
- Não substitui CRM. `tracking_lead.status` é minimalista; integração com CRM (HubSpot/Pipedrive) é hook de webhook futuro.
