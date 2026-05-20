# Pontos de Atenção, Riscos e Dúvidas em Aberto

> **Documento vivo.** Atualizar a cada nova dúvida, bloqueio ou risco identificado.
> Quando um ponto for resolvido: mover para a seção "Resolvidos" com a resposta + link para o `CHANGELOG.md`.

> 🔗 **Iniciativa Hub de Tracking** tem seu próprio documento de pontos de atenção: [`tracking/PONTOS-ATENCAO-TRACKING.md`](./tracking/PONTOS-ATENCAO-TRACKING.md). Manter os riscos específicos lá, não duplicar aqui. Este documento mantém riscos do sistema SEO Blog v1 original.

---

## 🔴 Bloqueadores (precisam de resposta antes de começar a fase referenciada)

_(nenhum no momento)_

---

## 🟡 Riscos Técnicos

### R10. FLUX.2 Klein 4B indisponível no listing OpenRouter `Fase 4`
- O endpoint `/api/v1/models` da OpenRouter (em 2026-05-18) lista 7 modelos com modalidade image: GPT-5/5.4-image-*, Gemini 2.5/3-flash-image, Gemini 3-pro-image, openrouter/auto. **Não há FLUX.**
- A decisão original era usar FLUX.2 Klein 4B como default (mais barato). Default temporário trocado para `google/gemini-2.5-flash-image` (~$0.04/imagem).
- **Mitigação:** investigar se FLUX está em outro provider/endpoint (fal.ai direto?) ou se OpenRouter adiciona em breve. Custo atual ainda é aceitável para v1.

### R9. Credenciais Cloudflare R2 `Fase 0.B8`
- Precisa: bucket dedicado para o seo-blog (separado dos buckets do health-voice-api), `CLOUDFLARE_ACCOUNT_ID`, access key/secret com permissão apenas neste bucket.
- **Ação:** criar bucket `seo-blog-media` (ou nome a combinar) + credenciais antes da Fase 4.

### R8. Provisionamento da VPS
- Sistema vai rodar em VPS dedicada (a contratar). Enquanto isso, Docker local na máquina do dev.
- **Itens dependentes:** publicação real em produção, certificados SSL, monitoramento, backups Postgres.
- **Mitigação:** Fase 12 deve incluir runbook de deploy + script `docker-compose.production.yml` + checklist de configuração da VPS (firewall, swap, Nginx reverse proxy, Letsencrypt).

### R1. Qualidade da imagem do FLUX.2 Klein 4B `[mitigado]`
- **Mitigação:** validar com **3 amostras** reais antes de definir como default. Fallback Gemini 2.5 Flash Image configurável por site.

### R2. Custo descontrolado de IA `[mitigado]`
- **Mitigação:** rate-limit por operador, cache por hash de prompt, alertas de gasto/dia, painel de custo visível.

### R5. Versionamento de prompts `[mitigado]`
- **Mitigação:** registrar prompt usado em cada `ai_usage_log` (já no schema).

### R6. Tom de voz vs persona por site `[mitigado]`
- **Mitigação:** `prompt_templates` por site + `sites.tone_of_voice` injetado como variável.

### R7. Conteúdo regulado (saúde, jurídico) `[mitigado]`
- **Mitigação:** revisão humana obrigatória; disclaimers padrão por site.

---

## 🟢 Dúvidas a confirmar (não bloqueia, mas precisa antes do go-live)

### D9. Política de URL canônica de tradução
- Quando um post tem versão pt-BR e en, qual é canonical? Mesma página com `hreflang` cruzado entre todas é o padrão correto.
- **Itens dependentes:** Fase 8.B2 (sitemap com hreflang), Fase 10 (LP rendering).

### D10. Configurações a serem documentadas (Convenções a preencher)
- [ ] Padrão de slug (kebab-case, sem stopwords PT?)
- [ ] Tamanho máximo de meta description (155-160 chars)
- [ ] Estrutura H1/H2/H3 obrigatória — quantas H2 mínimo?
- [ ] Tamanho mínimo de corpo (palavras) por content_type
- [ ] Política de imagens (proporção, peso máximo)
- [ ] Padrão de URL: `/blog/[slug]` vs `/blog/[ano]/[mes]/[slug]`?
- [ ] Padrão de URL com locale: `/en/blog/[slug]` (subdir) vs subdomínio?

---

## ✅ Resolvidos

### ~~Q. Stack do backend e frontend admin~~ (resolvido 2026-05-18)
**Resposta:** NestJS + Prisma + Next.js + shadcn/ui. Ver `CHANGELOG.md`.

### ~~Q. LPs atuais são SPA puro ou já têm SSR?~~ (resolvido 2026-05-18)
**Resposta:** São Next.js (confirmado em `health-voice-institutional-v2`). SSR/SSG nativo.

### ~~Q. Quando expandir via IA?~~ (resolvido 2026-05-18)
**Resposta:** Sob demanda do operador, após import.

### ~~Q. Gateway de LLM~~ (resolvido 2026-05-18)
**Resposta:** OpenRouter.

### ~~Q. Modelo de imagem mais barato~~ (resolvido 2026-05-18)
**Resposta:** FLUX.2 Klein 4B; fallback Gemini 2.5 Flash Image.

### ~~B1. Auth do painel admin~~ (resolvido 2026-05-18)
**Resposta:** JWT custom no NestJS seguindo o padrão exato do `health-voice-api` (`@nestjs/jwt` + `bcryptjs`, `AuthGuard` global, decorators `@IsPublic` / `@RequiresSecurityToken`, `AdminGuard`). Tabela `admin_users`.

### ~~B4. "Sem Supabase" é total ou só Auth?~~ (resolvido 2026-05-18)
**Resposta:** Sem Supabase em **nenhuma camada**. Postgres self-hosted (Docker dev → VPS prod). Storage = **Cloudflare R2** (mesmo padrão do health-voice-api). Supabase só era usado em outras LPs para tracking, não é banco principal da casa.

### ~~B2. Hospedagem~~ (resolvido 2026-05-18)
**Resposta:** Docker local na máquina do dev até VPS dedicada estar provisionada. Virou R8.

### ~~B3. Domínio API pública~~ (resolvido 2026-05-18)
**Resposta:** Domínio único cross-produto (ex.: `api.seoblog.<empresa>.com`), com `:siteSlug` no path. Atende todas as LPs.

### ~~D1. Quem revisa~~ (resolvido 2026-05-18)
**Resposta:** Equipe de 5; sem fila individual; lista única filtrável.

### ~~D2. Idiomas~~ (resolvido 2026-05-18)
**Resposta:** pt-BR primário; infra de en/es implementada na v1 mas opcional no go-live.

### ~~D3. Imagens — banco interno permitido?~~ (resolvido 2026-05-18)
**Resposta:** Sim, upload manual permitido como alternativa à IA.

### ~~D4. Comentários~~ (resolvido 2026-05-18)
**Resposta:** Não na v1.

### ~~D5. Autoria~~ (resolvido 2026-05-18)
**Resposta:** Autor = nome do site (`sites.author_name`), ex.: "Health Voice". Sem tabela de autores.

### ~~D6. Analytics no painel~~ (resolvido 2026-05-18)
**Resposta:** Fora do escopo v1. Operador usa GA da LP.

### ~~D7. Categoria vs tag~~ (resolvido 2026-05-18)
**Resposta:** Ambos suportados.

### ~~D8. Voltar published para draft~~ (resolvido 2026-05-18)
**Resposta:** Edit em `published` mantém status (nova versão + revalidate). Ação "Despublicar" → `unpublished` (URL 410 ou 301 configurável). Ver 5.B5.

### ~~R3. Indexação Google~~ (esclarecido 2026-05-18)
**Resposta:** Não há ping automático (Google descontinuou em 2023). IndexNow atinge só Bing/Yandex. Para Google, cadastrar cada site no Search Console + submeter sitemap manualmente. Documentar em runbook (Fase 12).

### ~~R4. Cache/invalidação nas LPs~~ (esclarecido 2026-05-18)
**Resposta:** Next.js ISR + webhook on-demand. Cada site armazena `revalidate_url` + `revalidate_secret`; backend chama após publicar/editar/despublicar para forçar `revalidatePath()`. Ver 7.B2.

---

## Como manter

- Quando criar entrada: dar ID (`B<N>`, `R<N>`, `D<N>`) e referenciar a fase do `PLANO.md`.
- Quando resolver: mover para `✅ Resolvidos` com data + resposta + link cruzado para o changelog.
- Revisar este documento no início de cada fase nova.
