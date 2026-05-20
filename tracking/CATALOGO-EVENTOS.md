# Catálogo de Eventos

> **Documento vivo.** A cada evento novo, **adicione aqui ANTES de implementar** e registre entrada no [`CHANGELOG-TRACKING.md`](./CHANGELOG-TRACKING.md). Evento sem entrada no catálogo é evento que não existe — backend pode rejeitar em modo estrito.
> **Última atualização:** 2026-05-19

---

## Convenções de naming

- **Caso:** `snake_case`.
- **Estrutura:** `<objeto>_<verbo>` (ex.: `cta_click`, `form_submit`, `video_play`) ou `<verbo>` quando o objeto é óbvio pelo contexto (`page_view`).
- **Tempo verbal:** sempre passado simples / imperativo curto. `click`, não `clicked`. `submit`, não `submitted`.
- **Sem prefixo de produto/LP** no nome do evento. Esses ficam em `properties`.
- **Versão de schema:** se um evento mudar de shape, **não renomeie** — incremente `schemaVersion` no client e adicione tratamento no backend.
- **Não criar eventos descartáveis.** Se for one-shot pra debug, usar `debug_*` e remover antes do merge.

### Convenções de `elementId`

- **Caso:** `snake_case`.
- **Formato:** `<tipo>_<seção>_<verbo_curto>`. Tipos: `cta`, `link`, `form`, `tab`, `accordion`, `card`.
- **Exemplos:** `cta_hero_demo`, `cta_pricing_assinar`, `link_footer_lgpd`, `form_lead_main`, `accordion_faq_pricing`.
- **Unicidade:** dentro de uma LP, `elementId` é único. Reaparecer = é o mesmo elemento.

### Convenções de `properties`

- **Camelcase** dentro do JSON.
- **Sem PII** (nome, email, telefone) — vai em `tracking_lead`, não em `properties` de event.
- **Tipos primitivos** preferidos. Objetos aninhados só quando essencial.
- **Limite:** ~2KB por evento (rejeitado acima disso).

---

## Eventos canônicos (válidos para todas as LPs)

### `session_start`
Disparado automaticamente pelo `client.ts` ao criar uma sessão nova.
- **`elementId`:** —
- **`properties`:**
  - `firstReferrer: string | null`
  - `firstLandingPath: string`
- **Quem dispara:** `client.ts` (automático)
- **Observações:** Idempotente — `sessionId` já existente no backend reusa.

### `page_view`
Disparado a cada mudança de pathname (ou primeiro load).
- **`elementId`:** —
- **`properties`:**
  - `path: string`
  - `title: string`
  - `referrer?: string`
- **Quem dispara:** `useAutoPageView()` no layout

### `cta_click`
Clique em qualquer chamada para ação.
- **`elementId`:** obrigatório (ex.: `cta_hero_demo`)
- **`properties`:**
  - `label: string` (texto visível do botão)
  - `destination?: string` (URL/rota destino)
  - `variant?: string` (se houver A/B)
- **Quem dispara:** `<TrackedButton>` ou `track('cta_click', ...)`

### `form_view`
Form de lead entra no viewport pela primeira vez na sessão.
- **`elementId`:** obrigatório (ex.: `form_lead_main`)
- **`properties`:** —
- **Quem dispara:** `IntersectionObserver` no componente do form (helper `useFormView()`)

### `form_submit`
Form submetido (antes de saber se a API aceitou).
- **`elementId`:** obrigatório
- **`properties`:**
  - `fields: string[]` (lista de nomes de campos preenchidos — **sem valores**)
- **Quem dispara:** handler de submit do form, **antes** do fetch

### `form_error`
Form rejeitado pelo backend ou validação local.
- **`elementId`:** obrigatório
- **`properties`:**
  - `errorCode?: string`
  - `fields: string[]` (campos com erro)
- **Quem dispara:** catch do submit

### `lead_created`
Lead foi efetivamente persistido (após `client.lead()` retornar OK).
- **`elementId`:** —
- **`properties`:**
  - `leadId: string`
  - `source: string`
  - `destination?: string`
- **Quem dispara:** após `client.lead()` resolver com sucesso
- **Observação:** dispara também o evento equivalente para pixels externos (`fbq('track', 'Lead')`, `gtag('event', 'generate_lead')`), se configurado.

### `scroll_depth`
Marca scroll em 25/50/75/100%. Dispara no máximo 1x por nível por sessão.
- **`elementId`:** —
- **`properties`:**
  - `percent: 25 | 50 | 75 | 100`
  - `path: string`
- **Quem dispara:** `useScrollDepth()` no layout

### `video_play` / `video_pause` / `video_complete`
Marcações de player de vídeo (HTML5 `<video>` ou YouTube embed).
- **`elementId`:** id do player (ex.: `video_hero_demo`)
- **`properties`:**
  - `src: string` (url do vídeo)
  - `position?: number` (segundos)
  - `duration?: number`
- **Quem dispara:** event listeners do player

### `outbound_click`
Clique em link que sai do domínio.
- **`elementId`:** id do link (`link_footer_parceiro_x`)
- **`properties`:**
  - `destination: string` (URL externa)
- **Quem dispara:** delegate listener em `<a target="_blank">` (configurável no provider)

### `download`
Clique em link de download (PDF, planilha).
- **`elementId`:** id do link
- **`properties`:**
  - `file: string`
  - `mime?: string`
- **Quem dispara:** handler explícito ou delegate em `<a download>`

---

## Eventos de engajamento granular

> Adicionados após a auditoria do `health-lp-v2`, que provou o valor de medir engajamento real (não só pageview/click). Ver [`LICOES-LPS-EXISTENTES.md§2.1`](./LICOES-LPS-EXISTENTES.md). Cada LP **opta** por esses eventos — não são obrigatórios; o playbook ativa por feature flag no provider.

### `section_enter`
Uma seção da página entra no viewport.
- **`elementId`:** id da seção (ex.: `section_hero`, `section_pricing`, `section_faq`)
- **`properties`:**
  - `order: number` (a quantésima seção que o usuário viu nesta sessão)
  - `scrollPercent: number` (posição do scroll na página, 0–100)
- **Quem dispara:** `IntersectionObserver` gerenciado pelo provider (`useSectionTracking()`)

### `section_exit`
Uma seção sai do viewport. Carrega o tempo de permanência.
- **`elementId`:** id da seção
- **`properties`:**
  - `dwellMs: number` (tempo com a seção visível, em ms)
  - `maxViewportPercent: number` (maior % da seção que ficou visível, 0–100)
- **Quem dispara:** mesmo observer; emparelhado com o `section_enter`
- **Observação:** é o evento que responde "qual seção segura / perde o usuário".

### `video_progress`
Heartbeat de progresso de vídeo (a cada ~10s assistidos, ou em marcos 25/50/75/100%).
- **`elementId`:** id do player (ex.: `video_hero_demo`)
- **`properties`:**
  - `percent: number` (0–100)
  - `position: number` (segundos)
  - `duration: number`
- **Quem dispara:** listeners do player (`useVideoTracking()`)
- **Observação:** ⚠️ **não** emitir 1 evento a cada segundo. O **estado** "quanto desse vídeo foi assistido nesta sessão" é mantido por upsert (ver [[ARQUITETURA-TRACKING#heartbeat]]); `video_progress` registra apenas marcos discretos para não inflar `tracking_events`.

### `faq_toggle`
Abrir ou fechar um item de FAQ/accordion.
- **`elementId`:** id do item (ex.: `accordion_faq_preco`)
- **`properties`:**
  - `open: boolean`
  - `question?: string` (texto da pergunta, sem PII)
- **Quem dispara:** handler do componente de accordion
- **Observação:** revela as dúvidas reais do público — insumo para copy e produto.

### `field_filled`
Um campo de formulário foi preenchido (dispara no `blur`, com debounce).
- **`elementId`:** id do campo (ex.: `field_lead_email`)
- **`properties`:**
  - `fieldName: string`
  - `fieldType: string` (`text` | `email` | `tel` | `select` | ...)
  - `filled: boolean` (campo ficou com valor ou foi esvaziado)
  - `nextAction?: 'next' | 'submit' | 'blur' | 'abandon'`
- **Quem dispara:** wrapper de input do form de lead
- **Observação:** ⚠️ **NUNCA** enviar o valor do campo. Só metadados. É o que responde "em qual campo o formulário trava".

### `form_step`
Avanço/conclusão de etapa em formulário multi-step (cadastro, quiz).
- **`elementId`:** id do form (ex.: `form_registration`)
- **`properties`:**
  - `step: number`
  - `stepName?: string`
  - `completed: boolean`
  - `timeOnStepMs: number`
- **Quem dispara:** controlador do wizard/multi-step
- **Observação:** alimenta o funil de cadastro etapa-a-etapa.

---

## Eventos específicos por LP

Use prefixo do slug do site para isolar. Adicione aqui antes de implementar.

### `[lp:health-voice]` — nenhum ainda

> Quando uma LP precisa de um evento de domínio (ex.: `calculator_lead` da Inova), registrar aqui com a tag `[lp:slug]`. O backend aceita esses eventos se vierem do `siteId` correspondente; caso contrário, rejeita.

---

## Como adicionar um evento novo

1. **Decida se é canônico ou específico de LP.** Se 80% das LPs futuras vão usar, é canônico.
2. **Adicione neste catálogo** com: nome, elementId esperado, properties (com tipos), quem dispara.
3. **Atualize `lib/tracking/events.ts`** da LP (union type).
4. **Se canônico:** atualize o DTO no backend (`ingest-event.dto.ts`, class-validator) e a validação de `properties` por nome quando ela existir (refinamento previsto — ver ARQUITETURA §4.3).
5. **Registre em [`CHANGELOG-TRACKING.md`](./CHANGELOG-TRACKING.md)** como `Added (event)`.
6. **PR e merge.**

> ⚠️ **Não invente eventos no calor da PR.** Se está em dúvida, abra issue/discussão, alinhe nome, depois codifique. Renomear evento depois quebra dashboard e atribuição histórica.
