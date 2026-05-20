# Lições das LPs Existentes — o que portar e o que NÃO repetir

> **Documento de referência.** Consolida a auditoria das LPs/sistemas já existentes (Inova + 4 LPs Health Voice). Serve de base para a arquitetura e como anti-checklist. Atualizar se uma nova auditoria revelar algo relevante.
> **Última atualização:** 2026-05-19

---

## 1. O que foi auditado

| Repo | Stack tracking | Maturidade |
|---|---|---|
| `inova-institutional` + `inova-admin-api` + `inova-admin` | Custom backend (Fastify+Prisma), endpoints `/public/track/*` | Média — bom esqueleto, incompleto |
| `health-voice/health-lp-v2` | **Custom Supabase muito rico** + Meta Pixel + webhook | **Alta** — tracking mais completo do grupo |
| `health-voice/lp-health-2026` | Meta Pixel + Meta CAPI (server-side, dedup por `event_id`) | Média — só Meta, sem dado próprio |
| `health-voice/health-voice-institutional` | Microsoft Clarity apenas | Baixa — sem eventos custom |
| `health-voice/health-voice-institutional-v2` | GTM + 1 evento + leads no Supabase | Baixa — minimalista |

---

## 2. O que é BOM e vamos PORTAR

### 2.1. Riqueza de captura do `health-lp-v2` ⭐

`health-lp-v2/src/lib/analytics.ts` é a referência de **o que** rastrear. Captura muito além de pageview/click:

- **Sections com engajamento real:** entrada/saída de cada seção, `time_spent_seconds`, `scroll_position`, `viewport_percentage`, `section_order` (ordem em que o usuário viu as seções). → vira eventos `section_enter` / `section_exit`.
- **Vídeo granular:** play, pause, ended, fullscreen + **heartbeat de progresso a cada 5s** (`analytics_home_video_progress` com `UNIQUE(session_id, video_id)` — upsert, não append). → eventos `video_*` + estado de progresso.
- **FAQ:** abrir/fechar cada pergunta (`faq_toggle`). Mostra quais dúvidas o público tem.
- **Inputs campo-a-campo:** com debounce, registra preenchimento de cada campo com `value_preview`, `next_action` (`next`/`submit`/`blur`). Permite ver **onde o formulário trava**.
- **Cadastro por etapa:** `analytics_home_registration` com `step`, `fields_filled`, `time_on_step_seconds`, `completed`. Funil de cadastro real.
- **Heartbeat de sessão:** ping periódico → permite calcular tempo real na página e detectar abandono.

> **Decisão:** o catálogo de eventos da v1 ([`CATALOGO-EVENTOS.md`](./CATALOGO-EVENTOS.md)) incorpora `section_enter`, `section_exit`, `video_progress`, `faq_toggle`, `field_filled`, `form_step`. Esses eventos não existiam na primeira versão do catálogo — foram adicionados por causa desta auditoria.

### 2.2. Meta CAPI híbrido do `lp-health-2026`

`lp-health-2026/lib/meta-capi.ts` faz Pixel **client-side + Conversions API server-side** com o **mesmo `event_id`** → Meta deduplica nas 48h. Normaliza e hasheia PII (SHA256) corretamente. `MetaPixel.tsx` captura `fbclid` e persiste cookie `_fbc` por 90 dias.

> **Decisão:** o padrão `event_id` compartilhado já está na nossa arquitetura (decisão D2, idempotência). Para LPs que rodam Meta Ads, a integração Pixel+CAPI é mantida na LP como camada paralela; o playbook menciona, mas o hub não substitui isso.

### 2.3. Endpoints públicos do Inova + adapter pattern

Já documentado em [`ARQUITETURA-TRACKING.md§6`](./ARQUITETURA-TRACKING.md). Shape de `/public/track/*` e o `lib/inova-api.ts` (adapter) são a base do nosso módulo `tracking` e do client.

### 2.4. Heartbeat como mecanismo

`health-lp-v2` provou que heartbeat (sessão + vídeo) dá métrica de **tempo real de engajamento**, não só "abriu a página".

> **Decisão de design:** métricas progressivas (progresso de vídeo, dwell time de seção, liveness de sessão) são **estado** — upsert numa linha única — e **não** append de evento. Append de heartbeat a 5s explode a tabela. Ver [[PONTOS-ATENCAO-TRACKING#r1]] e nota em [[ARQUITETURA-TRACKING#heartbeat]].

---

## 3. O que é RUIM — e NÃO vamos repetir

> Esta seção é o anti-checklist. Cada item virou uma restrição de arquitetura. O `tracking-pipeline-reviewer` e o `analytics-dashboard-reviewer` (agents da Fase 0) devem reprovar PRs que reincidam nestes erros.

### ❌ AP1. Uma tabela (ou conjunto de tabelas) por LP/página — **a causa raiz**

`health-lp-v2` tem `analytics_home_*`, `analytics_lp2_*`, `analytics_lp3_*`, `analytics_lp4_*`, `analytics_lp5_*`, `analytics_lp6_*` — **~45 tabelas que são clones com prefixo diferente**. O identificador da LP está **no nome da tabela**, não numa coluna.

Consequências:
- Visão consolidada exige `UNION` de N tabelas — e os schemas divergem (lp3/5/6 só têm 4 tabelas, lp2 tem tabela `quiz` extra).
- Cada LP nova = novo arquivo SQL + novas 10 tabelas. Não escala.
- Impossível responder "quantas sessões em todas as LPs este mês".

**Como evitamos:** uma única `tracking_events` (e `tracking_sessions`, etc.) com **coluna `siteId`**. LP nova = uma linha em `Site`, zero DDL. Decisão D9.

### ❌ AP2. Dashboard com código clonado por LP

`health-lp-v2/src/app/analytics2/` tem `components/2/` e `components/4/`, cada um com `types.ts`, `columns.ts`, `filter-handlers.ts`, `filter-logic.ts`, `expanded-row.tsx`. Adicionar LP = clonar pasta. Endpoints `/api/analytics-lp2`, `/api/analytics-lp4` — um por LP.

**Como evitamos:** dashboard único, parametrizado por `siteId` via seletor. Zero código novo por LP (critério de aceite da Fase 5).

### ❌ AP3. Dashboard = tabela gigante de sessões cruas

O `analytics2` é uma tabela filtrável de sessões individuais. Não tem funil, taxa de conversão, gráfico de tendência, série temporal, comparativo. Para "decidir algo comercial" você precisa ler linha por linha.

**Como evitamos:** dashboard é **insight-first** — funil, conversão, atribuição, tendência primeiro; tabela crua é drill-down, não a tela principal. Ver [[ARQUITETURA-TRACKING#7-princípios-de-visualização]].

### ❌ AP4. Carregar tudo no client, filtrar no browser, sem paginação

`analytics2_structure.md` admite: "Sem Paginação: todos os dados são carregados de uma vez", "Filtros client-side". Morre com volume.

**Como evitamos:** agregação **no backend** (SQL `GROUP BY`, `date_trunc`), endpoints retornam dados já resumidos, tabelas cruas paginadas server-side.

### ❌ AP5. Fragmentação de plataformas

Cada LP num lugar: `health-lp-v2` no Supabase, `lp-health-2026` no Meta Business Manager, `health-voice-institutional` no Clarity. Três logins, três modelos, zero visão unificada.

**Como evitamos:** o hub é a fonte canônica. Pixels de terceiros coexistem para *otimização de ad*, mas a análise de negócio sai de um lugar só. Decisão D1.

### ❌ AP6. Sem `site_id`, sem `anonymous_id`

Tabelas do `health-lp-v2` chaveiam tudo por `session_id` (sessionStorage, morre ao fechar aba). Sem `anonymous_id` persistente → impossível ver visitante voltando. Sem `site_id` → ver AP1.

**Como evitamos:** `anonymousId` (localStorage) + `sessionId` (sessionStorage) + `siteId` em tudo. Decisão D3.

### ❌ AP7. Nomes de evento divergentes entre LPs

`health-lp-v2` usa strings cruas em `event_name`; `lp-health-2026` usa padrão Meta (`PageView`, `Lead`). Sem enum compartilhado → agregação cross-LP quebra (`page_view` ≠ `PageView`).

**Como evitamos:** [`CATALOGO-EVENTOS.md`](./CATALOGO-EVENTOS.md) é fonte única; nomes canônicos; backend valida o DTO (class-validator). Decisão de catálogo.

### ❌ AP8. Sem dado agregado / sem fact tables

Tudo é log cru de evento. Toda query do dashboard varre a tabela inteira.

**Como evitamos:** índices em `(siteId, name, occurredAt)` desde a v1; rollups mensais (`tracking_events_monthly`) na Fase 6 antes do volume doer. Ver [[PONTOS-ATENCAO-TRACKING#r1]].

### ❌ AP9. Sem consentimento LGPD em nenhuma das 4 LPs HV

Nenhuma tem banner. `health-lp-v2` insere dado de todo visitante incondicionalmente, inclusive `value_preview` de inputs (potencial PII).

**Como evitamos:** consent gating obrigatório (Fase 3), `tracking_consent_log` escrito, `properties` sem PII (validação R6). Decisão D8.

### ❌ AP10. Dashboard dentro do repo da LP, protegido por senha solta

`analytics2` vive **dentro** da LP `health-lp-v2`, com auth por senha única em cookie (`analytics_dashboard_auth`), desacoplada de qualquer sistema de usuários.

**Como evitamos:** o dashboard vive no `seo-blog-frontend` (painel admin), com o `AuthGuard`/JWT e os papéis (`ADMIN`/`EDITOR`/`REVISOR`) que já existem no projeto.

### ❌ AP11. PII e IP crus no banco

`health-lp-v2` grava `ip_address` cru e `value_preview` de campos de formulário. `analytics_home_ip_user_link` cruza IP↔email↔telefone abertamente.

**Como evitamos:** IP só como `sha256(ip + salt)`; PII só em `tracking_lead` (entidade com base legal de lead), nunca em `properties` de evento.

---

## 4. Tabela-resumo: porta / não porta

| Item | Origem | Decisão |
|---|---|---|
| Captura rica de seção/vídeo/FAQ/input | `health-lp-v2/src/lib/analytics.ts` | ✅ Portar como eventos canônicos |
| Heartbeat de sessão e de vídeo | `health-lp-v2` | ✅ Portar como **estado** (upsert), não evento |
| Meta Pixel + CAPI com `event_id` dedup | `lp-health-2026/lib/meta-capi.ts` | ✅ Manter na LP, paralelo ao hub |
| Endpoints `/public/track/*` + adapter | `inova-admin-api`, `inova-institutional` | ✅ Base do módulo `tracking` |
| Tabela por LP/página (`analytics_lpN_*`) | `health-lp-v2` | ❌ Substituir por `siteId` em tabela única |
| Dashboard clonado por LP (`components/2`, `/4`) | `health-lp-v2/analytics2` | ❌ Dashboard único parametrizado |
| Dashboard = tabela de sessões cruas | `health-lp-v2/analytics2` | ❌ Insight-first (funil, conversão, tendência) |
| Carregar tudo + filtrar no client | `health-lp-v2/analytics2` | ❌ Agregar no backend, paginar |
| Sem consentimento | todas as 4 LPs HV | ❌ Consent gating obrigatório |
| IP/PII crus no banco | `health-lp-v2` | ❌ Hash de IP, PII só em `tracking_lead` |

---

## 5. Como esta auditoria mudou o plano

- **Catálogo de eventos expandido**: +6 eventos (`section_enter`, `section_exit`, `video_progress`, `faq_toggle`, `field_filled`, `form_step`).
- **Heartbeat como estado**: nova nota de design na arquitetura; métricas progressivas usam upsert.
- **Fase 4 reescrita como insight-first**: critérios de aceite do dashboard agora exigem funil/conversão/tendência antes de qualquer tabela crua, e agregação server-side.
- **Agents de review** ganham o anti-checklist AP1–AP11 como base de reprovação.
