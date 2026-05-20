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

| Spec | Cenário | Valida |
|---|---|---|
| `session.spec.ts` | Primeira visita à `campaing1` | 1 `tracking_session` (device, ipHash); `page_view` em `tracking_events` |
| `session.spec.ts` | Navegação entre páginas | novo `page_view` por rota |
| `session.spec.ts` | Coexistência com GTM | evento espelhado em `window.dataLayer` |
| `cta.spec.ts` | Clique em CTA da campanha | `cta_click` com `elementId` correto e `properties` |
| `lead.spec.ts` | Abrir modal de lead | `form_view` |
| `lead.spec.ts` | Enviar formulário (API mockada 200) | `form_submit`; `tracking_lead`; `lead_created` com `leadId` |
| `attribution.spec.ts` | Entrar com `?utm_*` | `tracking_attribution` com os UTMs |
| `idempotency.spec.ts` | Replay do mesmo `eventId` (nível API) | 1 só registro |

## Manutenção

- Cenário novo → adicione um spec + uma linha na tabela acima.
- Selector quebrou (mudou a LP) → corrija o spec; se o padrão mudou, reflita no [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md).
- Ao integrar uma LP nova, esta suíte é o molde — os cenários são genéricos, só os selectors e a `publicKey` mudam.
