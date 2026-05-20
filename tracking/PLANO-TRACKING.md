# Plano — Hub de Tracking de LPs

> **Documento vivo.** Marcar `[x]` ao concluir item. Atualizar status global em [`README.md`](./README.md) e registrar entrada em [`CHANGELOG-TRACKING.md`](./CHANGELOG-TRACKING.md) ao fim de cada fase.
> **Última atualização:** 2026-05-19

---

## Premissas

- Stack já definida no projeto: NestJS + Prisma + Postgres (back) e Next.js App Router + Tailwind + shadcn (front). Ver [`docs/PLANO.md`](../PLANO.md).
- A LP piloto é o `health-voice-institutional-v2`. Após validar nela, replicamos pela [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md).
- Arquitetura técnica detalhada vive em [`ARQUITETURA-TRACKING.md`](./ARQUITETURA-TRACKING.md). Este doc trata **ordem, escopo e aceite**.

---

## Fase 0 — Planejamento + skills/agents

**Objetivo:** consolidar a arquitetura e criar os artefatos de processo (skills/agents/docs) que vão guiar todas as fases seguintes.

- [x] Auditoria de tracking da Health Voice institucional v2
- [x] Auditoria do Inova (`inova-admin-api`, `inova-institutional`, `inova-admin`)
- [x] Auditoria das LPs `health-lp-v2`, `lp-health-2026`, `health-voice-institutional` (output: [`LICOES-LPS-EXISTENTES.md`](./LICOES-LPS-EXISTENTES.md) com anti-padrões AP1–AP11)
- [x] Criar conjunto de docs vivos desta pasta (README, ARQUITETURA, PLANO, PLAYBOOK, CATALOGO, LICOES, CHANGELOG, PONTOS-ATENCAO)
- [x] Criar skills no `seo-blog-frontend/.claude/skills/` (ver lista em §Fase 0.A)
- [x] Criar skills no `seo-blog-backend/.claude/skills/` (ver lista em §Fase 0.B)
- [x] Criar agents (`.claude/agents/`) para auditoria e revisão (ver §Fase 0.C)
- [x] Atualizar `docs/PLANO.md` raiz com bullet apontando para esta iniciativa
- [x] Criar branch `feat/tracking-hub-integration` na LP piloto `health-voice-institutional-v2`

> **Fase 0 concluída** quanto a planejamento, docs, skills e agents. O único item restante para fechar 100% é validar os critérios de aceite abaixo. A implementação de código começa na Fase 1.

### Fase 0.A — Skills no front
| Skill | Quando dispara | O que entrega |
|---|---|---|
| `lp-tracking-integration` | Usuário pede para integrar tracking em LP nova | Checklist do [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md) aplicado ao repo atual |
| `lp-event-catalog` | Usuário pede para adicionar/renomear evento | Atualiza [`CATALOGO-EVENTOS.md`](./CATALOGO-EVENTOS.md), `events.ts` da LP e (se necessário) DTO do back |
| `analytics-dashboard-page` | Usuário pede nova visualização no dashboard admin | Página em `(admin)/analytics/`, query no backend, chart com recharts |
| `tracking-consent-banner` | Usuário pede customização do banner LGPD | Componente parametrizável, integra com `ConsentManager` |

### Fase 0.B — Skills no back
| Skill | Quando dispara | O que entrega |
|---|---|---|
| `tracking-endpoint` | Usuário pede novo endpoint de tracking | Controller + service + DTO class-validator + testes mínimos + entrada no catálogo |
| `tracking-schema-migration` | Mudança em tabela `tracking_*` | Prisma migration + plano de rollback + bullet no CHANGELOG-TRACKING |
| `analytics-query` | Nova agregação para o dashboard | Método em `AnalyticsService` + endpoint privado + tipo TS compartilhado |

### Fase 0.C — Agents
| Agent | Propósito |
|---|---|
| `lp-tracking-auditor` (front) | Audita LP existente: lista pixels presentes, eventos, captura UTM, sessão, consentimento. Gera relatório com gaps vs [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md) |
| `tracking-pipeline-reviewer` (back) | Revisa PR no módulo `tracking`: valida idempotência, rate-limit, dedup, índices, multi-tenant (siteId), validação de DTO, segurança |
| `analytics-dashboard-reviewer` (front) | Revisa nova página no dashboard: consulta usa siteId? cache? loading/empty/error states? acessibilidade? |

**Critérios de aceite Fase 0:**
- Os 7 docs vivos existem e estão linkados entre si.
- Cada skill tem `SKILL.md` com seções: quando-usar, pré-requisitos, passo-a-passo, exemplos, edge cases.
- Cada agent tem `name`, `description`, `tools` e um system prompt focado.
- `docs/PLANO.md` raiz tem bullet apontando para `docs/tracking/`.

---

## Fase 1 — Backend: schema + módulo `tracking` + endpoints públicos

**Objetivo:** o backend aceita eventos de qualquer LP autenticada por `publicKey` e persiste com idempotência.

- [x] Migration: adicionar `publicKey` (unique) e `trackingEnabled` em `Site` (`20260520120000_add_tracking_hub`)
- [x] Migration: criar `tracking_sessions`, `tracking_events`, `tracking_attribution`, `tracking_leads`, `tracking_consent_log`
- [x] Módulo `tracking/`:
  - [x] `tracking.module.ts` (registrado no `AppModule`)
  - [x] `tracking.controller.ts` (público via `@IsPublic`, 4 rotas: session/event/lead/consent)
  - [x] `tracking.service.ts`
  - [x] DTOs (`ingest-session/event/lead/consent.dto.ts`) — **class-validator** (padrão do projeto), não Zod
  - [x] Guard `SiteKeyGuard` — valida `X-Site-Key` contra `Site.publicKey`, popula `req.trackingSite`
  - [~] CORS dinâmico baseado em `Site.domain` — **adiado**: `cors:true` global já cobre; allowlist por site depende de [`DP3`](./PONTOS-ATENCAO-TRACKING.md)
- [x] Rate limit (`@Throttle` no controller, por IP, limites ampliados) — per-`sessionId`/`siteId` é refinamento adiado
- [x] Dedup por `eventId` (upsert no-op) e por hash(`siteId+email+phone`) em janela de 5 min para lead
- [x] Aceitar `Content-Type: text/plain` (para `navigator.sendBeacon`) — middleware no `main.ts`
- [x] IP hashing (`sha256(ip + TRACKING_IP_SALT)`) — IP cru nunca persistido
- [~] Geo lookup (`country`/`city`) — **adiado** (colunas existem, nullable); decisão [`DP2`](./PONTOS-ATENCAO-TRACKING.md)
- [x] Validação manual via `curl`: auth (401), validação (400), `trackingEnabled=false` (403), idempotência, text/plain
- [ ] **Débito:** testes e2e automatizados (`*.e2e-spec.ts`) — validação foi manual; automatizar antes de escalar

**Critérios de aceite Fase 1:** ✅ todos atendidos (validação manual 2026-05-20)
- ✅ `curl` envia evento e ele aparece no banco com `siteId` correto.
- ✅ Replay do mesmo `eventId` não duplica (2 envios → 1 registro).
- ✅ Site sem `trackingEnabled=true` retorna 403; `X-Site-Key` inválida → 401; payload inválido → 400.
- ⚠️ Resta converter a validação manual em suíte e2e automatizada (débito acima).

---

## Fase 2 — LP piloto: `lib/tracking/` na Health Voice

**Objetivo:** a Health Voice manda `page_view`, `cta_click`, `form_submit` e `lead_created` para o nosso backend; o GTM continua rodando em paralelo.

- [x] Criar `src/lib/tracking/` na `health-voice-institutional-v2` (client, session, attribution, consent, events, provider, hooks, TrackedButton)
- [x] Env vars: `NEXT_PUBLIC_TRACKING_ENDPOINT`, `NEXT_PUBLIC_TRACKING_SITE_KEY`, `NEXT_PUBLIC_TRACKING_CONSENT_VERSION`, `NEXT_PUBLIC_GTM_ID`
- [x] `TrackingProvider` no `app/layout.tsx` (root)
- [x] page_view automático em mudança de rota — feito dentro do `TrackingProvider` via `usePathname()`
- [x] Refatorar CTAs da `campaing1`: `openModal`/`scrollToOferta` disparam `cta_click` com `elementId` padronizado
- [x] Refatorar `CampaignLeadModal` → `form_view` ao abrir, `form_submit` ao enviar, `form_error` em falha, `lead_created` após `client.lead()`
- [x] Mover GTM ID hardcoded para env var `NEXT_PUBLIC_GTM_ID` (GTM segue funcionando, agora condicional)
- [x] Substituir `pushLeadWebClick` cru por `track('cta_click', ...)` (o client também espelha no `dataLayer`)
- [x] **Validação E2E automatizada** — suíte `@playwright/test` em `health-voice-institutional-v2/e2e/` (8 testes, todos verdes em 2026-05-20)

### Validação E2E automatizada (playwright-mcp)

A partir da Fase 2 a validação do pipeline deixa de ser só `curl` e passa a exercitar o **fluxo real do browser**. Usar o **playwright-mcp** (instalado no ambiente). Quando a Fase 2 tiver código, criar:

1. **Plano de teste** (`docs/tracking/PLANO-TESTES-E2E.md`, novo doc vivo): cenários a cobrir — primeira visita, page_view em navegação, clique em CTA, abertura/submit de formulário, lead, entrada com `?utm_*`, gating de consentimento, replay/idempotência.
2. **Scripts de automação**: roteiros playwright que abrem a LP, executam cada cenário (navegar, clicar, preencher form) e param.
3. **Asserções no banco**: após cada cenário, consultar as tabelas `tracking_*` e validar que os registros esperados existem, com os campos corretos (`siteId`, `name`, `properties`, atribuição, dedup). O script compara o que o browser fez com o que chegou no Postgres.
4. **Relatório**: pass/fail por cenário, com o diff quando algo não bate.

Esses scripts viram a base reutilizável de validação de **toda LP nova** (passo do [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md)) e também servem à Fase 4 (validar que o dashboard reflete os dados).

**Critérios de aceite Fase 2:** ✅ todos atendidos (suíte E2E verde, 2026-05-20)
- ✅ Sessão real mostra `tracking_session`, `page_view`, `cta_click`, `tracking_lead`.
- ✅ `tracking_attribution` registrada com UTM quando entra via `?utm_*`.
- ✅ Eventos espelhados no `dataLayer` (coexistência com GTM).
- ✅ A suíte `@playwright/test` (8 testes) passa em todos os cenários.

---

## Fase 3 — Consentimento LGPD operacional

**Objetivo:** tracking nosso e GTM só rodam após consent; revogação funciona; auditoria fica em `tracking_consent_log`.

- [ ] `ConsentManager` no `lib/tracking/consent.ts` com buffer de eventos pré-consent
- [ ] `<ConsentBanner />` parametrizável (texto, política link, versão)
- [ ] `consentVersion` em env (ex.: `2026-05-19-v1`); mudar versão = pedir consent de novo
- [ ] Gating do GTM: só carrega o script após `consent.analytics === true`
- [ ] POST `/tracking/consent` registra cada decisão em `tracking_consent_log` (com `consentVersion`)
- [ ] Página `/preferencias-cookies` para revisão/revogação (cookie-policy)
- [ ] Default: pré-consent só permite "essenciais" (nenhum tracking nosso, nenhum GTM)

**Critérios de aceite Fase 3:**
- Primeira visita: nenhum POST sai antes do clique.
- Clique em "aceitar todos": flush do buffer + 1 registro em `consent_log` + GTM carrega.
- Clique em "rejeitar": nenhum POST de evento sai; 1 registro em `consent_log` com `false`.
- Revogar via `/preferencias-cookies`: novo registro em `consent_log`, eventos param de sair.

---

## Fase 4 — Dashboard de analytics no `seo-blog-frontend`

**Objetivo:** quem tem acesso ao painel admin consegue ver, por site, o que está acontecendo, e tomar decisão comercial.

> ⚠️ **Antes de codar:** ler [`ARQUITETURA-TRACKING.md§7` (Princípios de visualização)](./ARQUITETURA-TRACKING.md) e os anti-padrões AP2–AP4 de [`LICOES-LPS-EXISTENTES.md`](./LICOES-LPS-EXISTENTES.md). Esta fase corrige o motivo histórico de a visualização sempre falhar — não é só "fazer telas".

- [ ] Nova rota `(admin)/analytics/` separada da `(admin)/dashboard/` (que continua sendo a operacional do CMS)
- [ ] Layout com tabs/filtros: seletor de site, range de datas (7d, 30d, 90d, custom), comparativo
- [ ] Páginas:
  - [ ] `/analytics` (overview): cards (sessões, page views, eventos, leads, taxa de conversão) + sparkline
  - [ ] `/analytics/funil` (page_view → cta_click → form_view → form_submit → lead_created)
  - [ ] `/analytics/atribuicao` (breakdown por utm_source / medium / campaign; leads atribuídos)
  - [ ] `/analytics/leads` (tabela paginada com filtros; export CSV)
  - [ ] `/analytics/eventos` (top eventos com drill-down)
- [ ] Endpoints back: `GET /admin/analytics/{overview,funnel,attribution,leads,timeseries}` (ver [`ARQUITETURA§3.2`](./ARQUITETURA-TRACKING.md))
- [ ] Charts: `recharts` (decisão; ver [`PONTOS-ATENCAO`](./PONTOS-ATENCAO-TRACKING.md))
- [ ] Estados: loading skeleton, empty state ("sem dados ainda"), error retry
- [ ] Filtro de role: `REVISOR` vê só leitura; `ADMIN`/`EDITOR` exporta

**Critérios de aceite Fase 4:**
- Overview da Health Voice mostra dados reais da Fase 2.
- Funil renderiza as etapas; clique em etapa mostra eventos crus daquela etapa (drill-down, não a tela principal).
- Todo card mostra variação vs. período anterior (▲▼ %).
- Agregação é server-side: nenhuma página baixa log cru pra somar no browser.
- O seletor de site funciona sem nenhum código específico por LP.
- CSV de leads abre no Excel com colunas corretas.
- Estados `loading` / `empty` / `error` tratados em todas as páginas.
- Performance: cada página carrega em <1s com 30 dias de dados (índices em ordem).

---

## Fase 5 — Replicação: segunda LP via playbook

**Objetivo:** validar que [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md) realmente funciona sem decisão arquitetural ad-hoc.

- [ ] Escolher segunda LP (JuridIA ou outra)
- [ ] Seguir o playbook **literalmente** — anotar onde travou ou precisou improvisar
- [ ] Atualizar o playbook com correções/clarificações
- [ ] Adicionar segunda LP no dashboard (sem mudança no back)

**Critérios de aceite Fase 5:**
- Integração levou <2h (medir).
- Playbook foi alterado em ≤3 pontos.
- Dashboard mostra comparativo entre as 2 LPs.

---

## Fase 6 — Atribuição avançada + coortes

**Objetivo:** ir além de first/last touch, ter visão de retorno e ciclo.

- [ ] Modelo de atribuição: linear, time-decay (configurável)
- [ ] Visão de retorno: `anonymousId` voltando em sessões diferentes
- [ ] Coortes: % de sessões da semana X que viraram lead na semana Y
- [ ] LTV inicial: lead → cliente (se houver feed de cliente vindo de CRM via webhook)
- [ ] Alertas: pico de erro 5xx em `/tracking/event`, queda abrupta de sessões

**Critérios de aceite Fase 6:**
- Painel mostra os 3 modelos lado a lado.
- Coorte semanal renderiza.
- Alerta dispara (Slack/email) em quebra simulada.

---

## Anexo — Ordem de dependência

```
Fase 0 ─┬─▶ Fase 1 ─┬─▶ Fase 2 ─┬─▶ Fase 3
        │           │           └─▶ Fase 4 ─▶ Fase 5 ─▶ Fase 6
        │           │
        │           └─▶ (Fase 4 também depende da 1)
        │
        └─▶ skills/agents informam todas as fases
```

Fases 2 e 4 podem rodar em paralelo após a Fase 1, se houver pessoas.

---

## Como esta planilha é mantida

- Ao iniciar uma fase: marca status como 🟡 no README.
- Ao concluir item: `[ ]` → `[x]`.
- Ao concluir fase: status ✅ no README + entrada no `CHANGELOG-TRACKING.md`.
- Mudou escopo? edita aqui + entrada `Decision` no changelog.
