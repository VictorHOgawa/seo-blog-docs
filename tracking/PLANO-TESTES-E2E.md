# Plano de Testes E2E — Pipeline de Tracking

> **Documento vivo.** Atualizar quando cenários forem adicionados ou o procedimento mudar.
> **Última atualização:** 2026-05-20

---

## Objetivo

Validar o **pipeline completo de tracking** de ponta a ponta: a LP no browser real → backend → Postgres. `curl` valida o endpoint isolado; estes testes validam o que importa de verdade — que o client da LP (`lib/tracking/`) captura sessão, eventos, atribuição e leads e que tudo chega correto nas tabelas `tracking_*`.

A suíte é a base reutilizável de validação de **toda LP nova** — é o passo 8 do [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md).

## Ferramenta

`@playwright/test` (runner do Playwright) — suíte versionada, rodável local e em CI. Decisão registrada no [`CHANGELOG-TRACKING.md`](./CHANGELOG-TRACKING.md) (2026-05-20): suíte versionada em vez de `playwright-mcp` (melhor para regressão e reaproveitável).

A suíte vive no repo da LP piloto: `health-voice-institutional-v2/e2e/`.

## Pré-requisitos

| Requisito | Como garantir |
|---|---|
| Postgres do `seo-blog-backend` no ar | `docker compose up -d` no `seo-blog-backend` |
| Migrations aplicadas | `npx prisma migrate deploy` |
| Backend `seo-blog-backend` rodando (`:3333`) | `yarn start:dev` — ou o Playwright sobe via `webServer` |
| Site `health-voice` com `trackingEnabled = true` e `publicKey` conhecida | o `global-setup` garante isso automaticamente |
| LP rodando (`:3000`) | o Playwright sobe via `webServer` |

## Como rodar

```bash
cd health-voice-institutional-v2
npx playwright test            # roda toda a suíte
npx playwright test --headed   # com browser visível
npx playwright test e2e/lead   # um arquivo
npx playwright show-report     # relatório HTML
```

O `playwright.config.ts` sobe a LP (`yarn dev`) e o backend (`yarn start:dev`) automaticamente se ainda não estiverem no ar (`reuseExistingServer`). O Postgres **não** é gerenciado pela suíte — precisa estar de pé.

## Como as asserções funcionam

Cada teste:
1. Abre a LP num contexto limpo (localStorage/sessionStorage vazios → `anonymousId`/`sessionId` novos).
2. Executa o cenário (navegar, clicar, preencher formulário).
3. Força o flush do client (`window.__trackingClient.flush()`, exposto em modo debug).
4. Lê o `sessionId` real do `sessionStorage` da página.
5. Consulta o Postgres (`tracking_*`) **filtrando por aquele `sessionId`** e valida os registros com `expect.poll` (re-tenta até o write assíncrono chegar).

Isolamento: cada teste tem `sessionId` único; o `afterEach` apaga as linhas daquele `sessionId`. Sem poluição entre testes nem no banco de dev.

## Cenários cobertos

**21 testes**, incluindo caminho feliz, jornada realista, consentimento LGPD e edge cases — para dar confiança de produção.

| Spec | Cenário | Valida |
|---|---|---|
| `session.spec.ts` | Primeira visita | `tracking_session` (device, ipHash, `is_bot=false`) |
| `session.spec.ts` | Navegação entre páginas | `page_view` por rota |
| `session.spec.ts` | Coexistência com GTM | evento espelhado em `window.dataLayer` |
| `session.spec.ts` | **Visitante recorrente** | 2ª sessão reusa o `anonymousId`, gera `sessionId` novo |
| `session.spec.ts` | **User-Agent de bot** | sessão marcada `is_bot = true` |
| `session.spec.ts` | **Unload sem flush explícito** | evento sobrevive via `fetch` keepalive no `pagehide` |
| `cta.spec.ts` | Clique em CTA | `cta_click` com `elementId` correto |
| `lead.spec.ts` | Lead completo (API mockada 200) | `form_view`/`form_submit`/`lead_created` + `tracking_lead`; **telefone normalizado (R11)**; **sem PII em `properties`** |
| `lead.spec.ts` | **Abandono do formulário** | só `form_view` — nada de `form_submit`/lead |
| `lead.spec.ts` | **Validação client-side bloqueia** | submit inválido não dispara `form_submit` |
| `lead.spec.ts` | **Falha da API (500)** | `form_error` dispara; nenhum lead criado |
| `attribution.spec.ts` | Entrada com `?utm_*` | `tracking_attribution` com os UTMs |
| `attribution.spec.ts` | **First-touch persistente** | atribuição não muda ao navegar para página sem UTM |
| `idempotency.spec.ts` | Replay do mesmo `eventId` | 1 só registro |
| `idempotency.spec.ts` | `X-Site-Key` inválida | rejeitada com 401 |
| `journey.spec.ts` | **Jornada realista** anúncio pago → scroll → navegação → conversão | funil inteiro capturado, atribuição first-touch, ordem temporal dos eventos |
| `consent.spec.ts` | Banner aparece e some ao decidir | decisão registrada em `tracking_consent_log` |
| `consent.spec.ts` | "Recusar" | `marketing=false`; analytics segue (opt-out) |
| `consent.spec.ts` | Opt-out por padrão | analytics rastreia sem tocar no banner |
| `consent.spec.ts` | Aceitar | GTM injetado + eventos no `dataLayer` |
| `consent.spec.ts` | Visitante recorrente que já decidiu | banner não reaparece; GTM restaurado |
| `consent.spec.ts` | Opt-out de analytics em `/preferencias-cookies` | tracking para de enviar |

> `consent.spec.ts` **não** usa a fixture `fixtures.ts` — os demais specs usam, para pré-decidir o consentimento e o banner não cobrir a UI durante os cliques.

## Manutenção

- Cenário novo → adicione um spec + uma linha na tabela acima.
- Selector quebrou (mudou a LP) → corrija o spec; se o padrão mudou, reflita no [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md).
- Ao integrar uma LP nova, esta suíte é o molde — os cenários são genéricos, só os selectors e a `publicKey` mudam.
