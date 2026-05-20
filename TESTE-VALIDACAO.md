# Checklist de Validação — v1 (rodada 2)

> Roteiro **somente do que falta validar** após a 1ª rodada. As seções que
> passaram limpas (auth, sites, ideias, prompts, playground, custos, expansão,
> regerar campo, revisão, despublicar) foram removidas.
> Itens marcados com 🔧 **foram corrigidos** desde a 1ª rodada e precisam re-teste.
> Custo de IA estimado nesta rodada: **~US$ 0,05** (1 expansão + 1 cover).

## Como atualizar antes de começar

```bash
# 1. Pull dos 3 repos
cd seo-blog-backend  && git pull
cd ../seo-blog-frontend && git pull
cd ../health-voice/health-voice-institutional-v2 && git pull

# 2. Reiniciar dev servers (containers Docker já estão de pé)
cd ../../seo-blog-backend  && npm run start:dev   # terminal 1
cd ../seo-blog-frontend    && npm run dev          # terminal 2
cd ../health-voice/health-voice-institutional-v2 && npm run dev   # terminal 3 (LP)
```

---

## 🔧 A. Re-testes (bugs corrigidos)

### A.1 SiteSwitcher aparece no 1º login `[ ]` 🔧
Fix: `AuthProvider` invalida queries pós-login.
- [x] Logout. Login de novo com `admin@seoblog.local` / `123456`.
- [x] **Sem F5**, o SiteSwitcher no topbar deve mostrar "Health Voice".
- [x] Cards do dashboard preenchidos.

### A.2 Tabela de preview do BulkImport `[ ]` 🔧
Fix: dialog `max-w-6xl` + colunas com `min-width` específico.
- [x] `/ideias` → "Importar em lote" → aba "Colar" → colar 3 linhas → pré-visualizar.
- [x] Inputs de Título e Briefing mostram texto completo (não 2 caracteres).

### A.3 Editor: campos persistem ao trocar de aba `[ ]` 🔧
Fix: `useRef` trava inicialização do form por `content.id`. Refetches do React Query não resetam mais.
- [x] Abrir um conteúdo no `/conteudos/[id]`.
- [x] Trocar entre as abas Conteúdo / SEO + Preview / Mídia / Links Internos / Histórico — todas as 4-5 vezes.
- [x] Voltar pra Conteúdo: título, slug, corpo, etc. continuam preenchidos.
- [x] SEO + Preview: meta description e excerpt continuam preenchidos.
- [x] Mídia: capa continua aparecendo.

### A.4 Histórico atualiza após regerar campo `[ ]` 🔧
Fix: RegenerateButton invalida `['content-versions']`.
- [x] No editor, clicar **Regerar** em Meta Description.
- [x] Header mostra `v2` imediatamente.
- [x] Ir para aba Histórico **sem sair da página** — deve mostrar 2 linhas (v1, v2).
- [x] Comparar v1 → v2 mostra mudança só em `metaDescription`.

### A.5 Seletor de status com feedback `[ ]` 🔧
Fix: toast inline verde no sucesso + vermelho com mensagem no erro. Dropdown desabilita durante a transição.
- [x] Editor de um conteúdo APPROVED.
- [x] No dropdown, escolher "→ Publicado" → aparece toast verde **"✓ Status → Publicado"** por ~2.5s; badge muda; dropdown volta a "Mover status…".
- [x] Tentar uma transição inválida (ex.: PUBLISHED → EXPANDED) → toast vermelho **"⚠ Invalid transition: …"** por ~4s.
- [x] Durante a chamada, dropdown mostra "Mudando status…" e fica desabilitado.

### A.6 Prompt customizado com variáveis em PT funciona `[ ]` 🔧
Fix: aliases no `PromptEngine` — `{{titulo}}` resolve como `{{title}}`, `{{tema}}` idem, `{{palavras-chave}}` como `{{keywords}}`.
- [x] Manter (ou recriar) um prompt TITLE com `User: Tema: {{titulo}}\nKeywords: {{keywords}}\nGere 1 título.`
- [x] Expandir uma ideia **"Hipertensão arterial em idosos"**.
- [x] Validar: título gerado é sobre hipertensão (não sobre criar site). Custo ~US$ 0.10.
- [x] Backend não deve logar `prisma:error` durante a expansão.

### A.7 Backend sem `prisma:error` ao mover status `[ ]` 🔧
Fix: `transition()` incrementa `version` → snapshot único por transição.
- [x] Salvar manualmente um campo no editor (criando v2 via save).
- [x] Mudar status pelo dropdown → ver no terminal do backend: **nenhum `prisma:error`** sobre `Unique constraint failed on the fields (content_id, version)`.

### A.8 Timezone: agendar no dia certo `[ ]` 🔧
Fix: backend e frontend agora parseiam `startDate` como LOCAL (não UTC).
- [x] `/calendario` → "Agendar em lote" → data = **amanhã**, hora = `09:00`, modo DAILY.
- [x] Pré-visualização do dialog mostra a data correta de amanhã.
- [x] Confirmar. No calendário, o ponto deve cair em **amanhã**, não hoje.
- [x] Reagendar clicando no ícone: o prompt agora mostra **horário local** (ex.: `2026-05-19T09:00`), não UTC.
- [x] Editar para `2026-05-20T15:30` e confirmar → calendário move o job para o dia 20 às 15:30.

---

## ⏳ B. Cenários ainda não testados

### B.1 Agendar: cancelar e voltar pra APPROVED `[ ]`
- [x] No `/calendario`, clicar 🗑 num job PENDING.
- [x] Confirmar. Status do job vira **CANCELLED** na lista.
- [x] Em `/conteudos`, esse conteúdo voltou de SCHEDULED para **APPROVED**.

### B.2 Publish-now (publicação manual instantânea) `[ ]`
- [x] Editor de um APPROVED → botão **"Publicar agora"** (aparece só em APPROVED/UNPUBLISHED).
- [x] Confirma → alerta "Publicação enfileirada".
- [x] Em <5s, recarregar o editor → status = **PUBLISHED**.
- [x] `/publicacoes` mostra job SUCCEEDED com `attempts=1`.

### B.3 Schedule com delay curto (worker BullMQ) `[ ]`
- [ ] Garantir 1 conteúdo APPROVED.
- [ ] `/calendario` → "Agendar em lote" com data = **hoje** e hora = **3 min no futuro** (ex.: agora são 14:10 → marcar 14:13).
- [ ] Voltar à página em ~3 min: status do Content é **PUBLISHED**.
- [ ] `/publicacoes` tem um job SUCCEEDED novo.

### B.4 API pública (consumida pelas LPs) `[ ]`
Tudo via `curl` (sem token):
- [x] `curl http://localhost:3333/public/health-voice/contents` → JSON com array de PUBLISHED.
- [x] `curl http://localhost:3333/public/health-voice/contents/<slug-real>` → detalhe completo.
- [x] `curl http://localhost:3333/public/health-voice/contents/abc-nao-existe` → **404**.
- [x] `curl http://localhost:3333/public/site-x/contents` (site inexistente) → **404**.
- [x] `curl http://localhost:3333/public/health-voice/sitemap.xml` → XML válido (`<urlset>` + `<xhtml:link hreflang>`).
- [x] `curl http://localhost:3333/public/health-voice/rss.xml` → RSS 2.0 com items e `<pubDate>`.
- [x] `curl http://localhost:3333/public/health-voice/robots.txt` → texto plano com `Sitemap:`.
- [x] Despublicar um post (UNPUBLISHED) → `GET /contents/<slug>` retorna **410 Gone**.

### B.5 LP integrada (`/blog` em http://localhost:3001) `[ ]`
**Pré-requisito:** ter pelo menos 1 post PUBLISHED e a LP rodando.
- [ ] `http://localhost:3001/blog` → grade com posts (capa, categoria, título, meta, data).
- [ ] Paginação se houver >12 posts.
- [ ] Clicar num card → `/blog/<slug>` carrega com: hero, capa, corpo Markdown, tags+categoria no rodapé, seção "Posts relacionados" (se houver).
- [ ] **Ver source (Ctrl+U) do detalhe:**
  - `<title>` correto
  - `<meta name="description">` correto
  - `<meta property="og:title">`, `og:image`
  - `<script type="application/ld+json">` com BlogPosting
- [ ] `http://localhost:3001/sitemap.xml` inclui URLs estáticas **+** `/blog/*` (do CMS).
- [ ] `http://localhost:3001/rss.xml` retorna 200 com RSS válido.

### B.6 Webhook de revalidate ponta-a-ponta `[ ]`
- [ ] No painel admin: `/sites` → editar Health Voice → setar `Revalidate URL = http://localhost:3001/api/revalidate` e `Revalidate Secret = change-me-must-match-cms`.
- [ ] No editor de um conteúdo APPROVED, clicar **Publicar agora**.
- [ ] No terminal da LP, ver log `POST /api/revalidate 200`.
- [ ] Recarregar `http://localhost:3001/blog/<slug-novo>` — deve aparecer já (sem esperar ISR de 1h).
- [ ] **Secret errado:** mudar secret no painel para outro valor, publicar de novo. Backend deve logar warning de **401 no webhook**, mas a publicação no CMS acontece mesmo assim. Voltar o secret.

### B.7 Dashboard agregado `[ ]`
Voltar para `/dashboard`:
- [ ] 4 cards principais com números reais (ideias pendentes, aguardando revisão, agendados próx 7d, publicados 30d).
- [ ] **Custo IA hoje** > 0.
- [ ] **Success rate** = 100% (todos os jobs OK).
- [ ] **Pipeline**: contagem por status faz sentido.
- [ ] **Gráfico publicações por dia** mostra barra verde em hoje.
- [ ] Refetch a cada 30s (deixa aberto, faça uma publicação em outra aba, volta — cards atualizam).

### B.8 Custos: download CSV `[ ]`
- [ ] `/custos` → botão **CSV** baixa `ai-cost-30d.csv`.
- [ ] Arquivo tem header `created_at,site_id,content_id,kind,model,...` + linhas com dados reais.

### B.9 Linkagem interna (related posts) `[ ]`
Pré-requisito: ≥2 posts PUBLISHED no mesmo site com pelo menos 1 tag em comum.
- [ ] Editor de um post → aba **Links Internos** mostra cards com score, badges de tag/categoria compartilhada.
- [ ] `curl http://localhost:3333/public/health-voice/contents/<slug>/related` → array com os mesmos posts.
- [ ] Em `http://localhost:3001/blog/<slug>` deve aparecer a seção "Posts relacionados" no fim.

### B.10 Hardening — uploads `[ ]`
Via Scalar (http://localhost:3333/reference) → `POST /media/upload`:
- [ ] Mandar payload com `mimeType: "application/pdf"` → **415 Unsupported Media Type**.
- [ ] Mandar payload com `base64` de uma imagem **>5MB** (pode gerar com `dd if=/dev/zero bs=1M count=6 | base64`) → **413 Payload Too Large**.

### B.11 Smoke final — fluxo realista `[ ]` ⚠️ ~US$ 0.10
Esse é o "happy path" do operador:
1. [ ] Importar 3 ideias via Paste.
2. [ ] Expandir 1 (esperar ~80s).
3. [ ] No editor, ajustar manualmente o título → Salvar (v2).
4. [ ] `/revisao` → Aprovar.
5. [ ] `/calendario` → agendar para 3 min no futuro.
6. [ ] Aguardar publicação automática.
7. [ ] `/conteudos` mostra PUBLISHED.
8. [ ] `http://localhost:3001/blog/<slug>` carrega com SEO completo.
9. [ ] `/dashboard` mostra +1 publicado e custo refletido.

---

## Anotações da rodada anterior (preservadas para histórico)

- **3.1 preview pequeno** — corrigido em A.2.
- **5 título sem sentido** — corrigido em A.6 (aliases `{{titulo}}`).
- **5 campos sumindo entre abas** — corrigido em A.3.
- **5.1 histórico não atualizava** — corrigido em A.4.
- **6.2 seletor de status sem feedback** — corrigido em A.5.
- **7.2 ponto laranja confuso quando data passou** — comportamento esperado (job já rodou). Sem ação.
- **7.3 reschedule +3h e dia errado** — corrigido em A.8.
- **Backend `prisma:error` sobre constraint** — corrigido em A.7.

---

## Se algo falhar

Ver [`RUNBOOK.md`](./RUNBOOK.md) seção "Quando uma publicação falha". Erros mais comuns:

| Erro | Causa | Ação |
|---|---|---|
| `EADDRINUSE :3333` | Backend ainda rodando | `(Get-NetTCPConnection -LocalPort 3333).OwningProcess \| Stop-Process -Force` |
| `EPERM Prisma` | Backend segurando `query_engine.dll` | Matar processo Node → `npx prisma generate` → restart |
| `Supabase URL required` na LP | `.env.local` sem dummies | Copiar de `.env.local.example` |
| Webhook 401 no backend log | Secret divergente CMS ↔ LP | Sincronizar valor em `/sites` e `.env.local` |
