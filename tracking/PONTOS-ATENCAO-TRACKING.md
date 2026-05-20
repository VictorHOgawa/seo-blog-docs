# Pontos de Atenção — Iniciativa de Tracking

> **Documento vivo.** Toda dúvida em aberto, risco, bloqueador, débito técnico previsto entra aqui. Quando resolvido, **mover para "Resolvidos"** com link para a entrada do [`CHANGELOG-TRACKING.md`](./CHANGELOG-TRACKING.md) que decidiu.
> **Última atualização:** 2026-05-19

---

## 🔴 Bloqueadores

_(nenhum no momento)_

---

## 🟠 Decisões pendentes (precisam de resposta antes de avançar a fase referenciada)

### DP1. Biblioteca de gráficos no dashboard — `Fase 4`
- Opções: `recharts` (mais conhecida, comunidade grande), `tremor` (focado em dashboards admin, monta cards prontos), `visx` (mais flexível, mais código), `chart.js` via `react-chartjs-2`.
- **Recomendação:** `recharts` para começar (cobre 90% dos gráficos: line, bar, funnel via custom, pie). Migrar pra `tremor` se o painel crescer muito.
- **Ação:** decidir no início da Fase 4 e registrar no changelog.

### DP2. Geolocalização do visitante — `Fase 1`
- Opções:
  - **`geoip-lite`** (npm): banco MaxMind GeoLite2 local, custo zero, latência ~ms, precisão de país/região; precisa atualizar o DB periodicamente.
  - **Cloudflare** headers (`cf-ipcountry`): grátis se servirmos atrás de CF; só país.
  - **ip-api.com / ipinfo.io**: serviço HTTP, free tier limitado, latência adicional.
- **Trade-off:** `geoip-lite` adiciona ~30MB ao bundle do backend, mas é o mais simples. Cloudflare exige depender de CF na frente.
- **Ação:** decidir no início da Fase 1.

### DP3. Estrutura de `Site.allowedOrigins` — `Fase 1`
- Hoje `Site.domain` é único. LPs com múltiplos hosts (`www.x.com`, `x.com`, `staging.x.com`, `localhost`) precisariam de array.
- **Opção A:** transformar em `String[]`.
- **Opção B:** match por sufixo (`*.x.com`) — mais flexível, mais perigoso.
- **Recomendação:** A. Cadastro explícito.
- **Ação:** definir na migration da Fase 1.

### DP4. Pacote npm interno `@seo-blog/tracking-client` — `Fase 5`
- O [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md) hoje fala em **copiar pasta**. Funciona, mas atualizar 5 LPs vira chato.
- **Plano:** após validar em 2 LPs (Fase 5), extrair `lib/tracking` para pacote `@seo-blog/tracking-client` publicado no GitHub Packages (privado) ou monorepo via workspaces.
- **Ação:** revisar ao concluir Fase 5; criar issue dedicada se evolução for confirmada.

### DP5. Versão de `consentVersion` — operacional contínuo
- Mudou o texto do banner ou as categorias? Precisa nova versão (ex.: `2026-05-19-v1` → `2026-06-01-v2`), o que **invalida consents antigos** e pede consent de novo.
- **Risco:** se mudar sem pensar, todo visitante recorrente vê banner de novo (atrito).
- **Mitigação:** só incrementar versão quando o texto for materialmente diferente. Registrar mudança como `Decision` no changelog explicando o motivo.

### DP6. Tratamento de bots
- Crawlers do Google, Bing e bots maliciosos vão hidratar `tracking_sessions` ruidosas.
- **Opções:**
  - Filtrar UA conhecidos de bots no backend (lista crowdsourced).
  - Adicionar flag `tracking_session.bot: boolean`.
  - Ignorar pageviews quando `navigator.webdriver === true`.
- **Recomendação:** combo do 1 + 2. Bot fica salvo mas excluído por padrão das queries do dashboard.
- **Ação:** definir na Fase 1.

---

## 🟡 Riscos técnicos identificados

### R1. Volume de eventos pode estourar Postgres `Fase 1+`
- Uma LP ativa com 10k sessões/dia, 20 events cada = **200k inserts/dia, ~6M/mês**.
- ⚠️ Os eventos de engajamento granular adicionados após a auditoria (`section_enter`/`section_exit` por seção, `video_progress` por marco, `faq_toggle`) **multiplicam** isso — uma LP com 8 seções e 2 vídeos pode gerar ~40–60 eventos por sessão. Recalcular: ~12M+/mês por LP ativa.
- Postgres aguenta com índices certos, mas `tracking_events` cresce rápido. Partitioning por mês deve entrar em pauta cedo.
- **Mitigação:** índices em `(site_id, name, occurred_at)` e `(session_id, occurred_at)`. Decisão D11 já evita o pior (heartbeat não vira evento). Plano de retenção: 90–180 dias de eventos crus, agregados mensais permanentes (Fase 6 ou antes se necessário). Eventos de engajamento granular são **opt-in por LP** (feature flag no provider) — LP que não precisa não paga o custo.

### R2. `sendBeacon` payload limit (64KB)
- Se um batch ficar grande (lote de eventos no unload), pode estourar.
- **Mitigação:** client.ts limita batch a ~30 eventos ou 50KB, o que vier primeiro. Acima disso, força flush imediato em vez de esperar unload.

### R3. CORS preflight em `OPTIONS`
- Browsers fazem preflight para `Content-Type: application/json` + headers custom (`X-Site-Key`). Cada evento = 2 requests.
- **Mitigação:** usar `Content-Type: text/plain` no client + serializar JSON na string. Sem preflight, e ainda compatível com `sendBeacon`.

### R4. Adblockers
- ~30% dos visitantes têm uBlock/AdBlock/Brave. Domínios `/tracking/*` podem ser bloqueados.
- **Mitigação parcial:** servir endpoint sob domínio próprio (não `analytics.*` ou `tracking.*`, que estão em listas). Ex.: `api.seo-blog.example.com/t/event`.
- **Aceitar:** sempre haverá undercount; comparar com pixels do GTM ajuda a calibrar.

### R5. LP rodando em `localhost:3000` no dev nunca consegue testar
- CORS de produção exige `Site.domain = https://x.com`.
- **Mitigação:** flag `Site.allowDevOrigin: boolean` (default false). Quando true, libera `http://localhost:*` no CORS. Habilitar manualmente por site só em dev.

### R6. PII vazando em `properties`
- Risco: dev coloca `email` ou `phone` em `properties` por engano.
- **Mitigação:** scanner no backend (validador custom no `TrackingService`) que rejeita strings que parecem email/phone fora de campos esperados. Registrar como warning em dev, hard fail em prod. _Ainda não implementado — entra com a validação de `properties` por nome._

### R7. Schema drift entre LP e backend
- LP atualiza `events.ts` antes do backend (ou vice-versa).
- **Mitigação:** backend em modo "warn" por default (aceita evento desconhecido, loga); modo "strict" via env. Cliente sempre envia `schemaVersion` por evento.

### R8. Pacote `lib/tracking` divergindo entre LPs (cópia)
- Cada LP terá uma cópia da pasta. Bug fix em uma não propaga.
- **Mitigação curto prazo:** registrar versão no header (`X-Tracker-Version`) e mostrar no dashboard. Mitigação real: ver [`DP4`](#dp4-pacote-npm-interno-seo-blogtracking-client--fase-5).

### R9. Migration Prisma na fase 1 vai mexer no model `Site`
- Adicionar `publicKey` e `trackingEnabled` em `Site` toca tabela existente em produção.
- **Mitigação:** `publicKey` com default `gen_random_uuid()`. `trackingEnabled` default `false` (opt-in explícito por site no admin antes de começar a rastrear).

### R10. Race entre `session_start` e primeiro `event`
- Se `event` chega antes da sessão estar persistida, FK quebra.
- **Mitigação:** `tracking_event` referencia `sessionId` por valor (não por id da row); upsert da sessão acontece no service para ambos os endpoints.

---

## 🟢 Dúvidas em aberto (não bloqueiam, mas precisam de resposta antes da fase respectiva)

### D1. Quem é "admin" no contexto de analytics? `Fase 4`
- Hoje `AdminRole` é `ADMIN | EDITOR | REVISOR`. Para analytics:
  - ADMIN: tudo.
  - EDITOR: tudo da analytics?
  - REVISOR: read-only? só vê o site do `siteAccess` dele?
- **Proposta:** REVISOR não vê analytics (não é foco dele). EDITOR vê só os sites do `siteAccess`. ADMIN vê tudo. Confirmar antes da Fase 4.

### D2. Webhook de novo lead para CRM externo? `Fase 2 ou Fase 6`
- A Health Voice já tem `WEBHOOK_LEADS_URL` / `N8N_WEBHOOK_URL`.
- **Pergunta:** o hub central também deve disparar webhook configurável por site quando `tracking_lead` é criado?
- **Proposta:** sim, em `Site.leadWebhookUrl`. Mas isso é Fase 6 (não bloqueia v1).

### D3. Quanto tempo de retenção para `tracking_events`? `Fase 1`
- Eventos crus crescem rápido. 90 dias suficiente? Indefinido (com particionamento)?
- **Proposta:** 180 dias crus + agregados mensais permanentes em `tracking_events_monthly` (Fase 6).

### D4. Como nomear UTMs vs `source`/`buttonId` no `tracking_lead`? `Fase 2`
- Risco de confusão: `lead.source = 'campaing1'` (origem interna) vs `lead.utmSource = 'facebook'` (origem da campanha).
- **Proposta:** documentar claramente em [`CATALOGO-EVENTOS.md`](./CATALOGO-EVENTOS.md) e renomear `source` → `internalSource` se causar confusão na prática.

### D5. Dashboard mostra dados do banco direto ou de cache (Redis)? `Fase 4`
- Queries de agregação 30d podem ser lentas com milhões de linhas.
- **Proposta:** queries diretas até dor real (>500ms). Se precisar, cache de 1min em Redis (que já está no projeto pra BullMQ).

### D6. Migrar dados históricos das LPs antigas? `Fase 5+`
- `health-lp-v2` tem dados em ~45 tabelas Supabase (`analytics_home_*`, `analytics_lpN_*`); `lp-health-2026` tem histórico no Meta; `health-voice-institutional` no Clarity.
- **Pergunta:** vale escrever um script de ETL para trazer o histórico para o schema novo, ou começamos a contagem do zero a partir da integração?
- **Proposta:** começar do zero. O histórico antigo tem schema inconsistente, sem `siteId`, sem `anonymousId` — normalizar custa caro e o dado é de qualidade duvidosa. Se houver necessidade comercial específica (ex.: comparar campanha antiga), fazer ETL pontual só daquelas tabelas. Confirmar com o time comercial antes da Fase 5.

### D7. Eventos de engajamento granular: ligados por padrão? `Fase 2`
- `section_*`, `video_progress`, `faq_toggle`, `field_filled`, `form_step` são poderosos mas geram volume (ver [R1](#r1-volume-de-eventos-pode-estourar-postgres-fase-1)).
- **Proposta:** desligados por padrão; LP liga via flag no `<TrackingProvider features={{ sections: true, video: true }}>`. A Health Voice piloto (Fase 2) liga todos para validar o pipeline ponta-a-ponta.

---

## ✅ Resolvidos

_(nenhum ainda)_

> Quando um item daqui for resolvido, **mova para esta seção** com:
> - referência ao item original (ex.: `DP1 → recharts`),
> - link para a entrada do `CHANGELOG-TRACKING.md` que decidiu,
> - data da resolução.
