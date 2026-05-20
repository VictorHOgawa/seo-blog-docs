# Iniciativa: Hub Central de Tracking de LPs

> **Documento vivo.** Atualize sempre que a estrutura desta pasta mudar (novo doc, novo deliverable, novo dono).
> **Última atualização:** 2026-05-19

---

## Em uma frase

Transformar o `seo-blog` no **hub central de tracking, atribuição e analytics** de todas as LPs do grupo (Health Voice, JuridIA e futuras), com captura padronizada de eventos no backend, dashboard agregado no frontend, e um playbook que permite **plugar uma LP nova em minutos**.

---

## Por que essa iniciativa existe

- Hoje o tracking nas LPs é fragmentado: cada LP grava leads num Supabase próprio (ou nem grava), eventos viram `dataLayer.push` cru, UTM não é capturado, não há sessão persistente.
- Já tentamos várias vezes — a tentativa do **Inova** (`inova-admin-api` + `inova-institutional`) chegou ~80% lá: tem endpoint público de tracking, modelos `PageView`/`TrackEvent`/`CalculatorLead`, cliente `Tracker.tsx`. **Falta:** UTM real, multi-site, consentimento operacional, rate-limit, dedup, dashboard com gráficos, catálogo de eventos.
- O **`health-lp-v2`** chegou ainda mais longe na *captura* (rastreia seção, vídeo, FAQ, input campo-a-campo) mas afundou na *visualização*: ~45 tabelas Supabase clonadas por LP, dashboard com código duplicado por LP, tela de tabela crua sem funil/gráfico. Auditoria completa e anti-padrões em [`LICOES-LPS-EXISTENTES.md`](./LICOES-LPS-EXISTENTES.md).
- O `seo-blog-backend` já tem `Site` multi-tenant, módulo `metrics` (operacional, não comportamental) e o `seo-blog-frontend` tem painel admin pronto para receber uma área de analytics.
- Falta: **infra padronizada de tracking + dashboard que vire base de decisão comercial**.

---

## Princípios não-negociáveis

1. **Multi-site nativo.** Todo evento carrega `siteId`. O backend é hub, não silo por LP.
2. **Backend é a fonte da verdade.** Pixels de terceiros (GTM, Meta, TikTok) seguem coexistindo, mas o dado nosso vive no nosso Postgres.
3. **Type-safe + catálogo.** Eventos têm nome canônico e payload validado no backend (class-validator) + tipos TS (front). Nada de string solta.
4. **LGPD desde o dia 1.** Banner real, `ConsentLog` realmente escrito, gating de tracking antes do opt-in.
5. **Idempotência.** Todo evento traz `eventId` (uuid client-side). Replays/retries não duplicam.
6. **Replicável.** Plugar LP nova = seguir o `PLAYBOOK-NOVA-LP.md`. Sem decisão arquitetural ad-hoc.
7. **Documento vivo.** Cada fase atualiza changelog + pontos de atenção. Decisão não registrada = decisão não tomada.

---

## Estrutura desta pasta

| Arquivo | Conteúdo | Atualização |
|---|---|---|
| [`README.md`](./README.md) | Este índice | Quando a estrutura da pasta mudar |
| [`PLANO-TRACKING.md`](./PLANO-TRACKING.md) | Fases, escopo, critérios de aceite, status | A cada decisão de escopo ou fase concluída |
| [`ARQUITETURA-TRACKING.md`](./ARQUITETURA-TRACKING.md) | Diagrama, schema Prisma, endpoints, módulos client | A cada decisão técnica |
| [`PLAYBOOK-NOVA-LP.md`](./PLAYBOOK-NOVA-LP.md) | Passo-a-passo para integrar tracking em LP nova | Quando o procedimento mudar |
| [`CATALOGO-EVENTOS.md`](./CATALOGO-EVENTOS.md) | Eventos canônicos, payload, convenções | A cada evento novo |
| [`LICOES-LPS-EXISTENTES.md`](./LICOES-LPS-EXISTENTES.md) | Auditoria das LPs existentes: o que portar e os anti-padrões AP1–AP11 | Quando nova auditoria revelar algo |
| [`CHANGELOG-TRACKING.md`](./CHANGELOG-TRACKING.md) | Histórico de entregas e decisões | A cada entrega/decisão |
| [`PONTOS-ATENCAO-TRACKING.md`](./PONTOS-ATENCAO-TRACKING.md) | Riscos, dúvidas, débitos técnicos | A cada novo bloqueio ou risco |

> Os docs raiz (`docs/PLANO.md`, `docs/CHANGELOG.md`, `docs/PONTOS-ATENCAO.md`) tratam do **sistema todo**. Os docs desta pasta tratam **apenas da iniciativa de tracking**, com mais detalhe. Mudanças relevantes ecoam num bullet no doc raiz, sem duplicar conteúdo.

---

## Status global

| Fase | Nome | Status |
|---|---|---|
| 0 | Planejamento + skills/agents | ✅ Concluída |
| 1 | Schema Prisma + módulo `tracking` no backend | ✅ Concluída (resta e2e automatizado) |
| 2 | Cliente `lib/tracking` na LP Health Voice (piloto) | ⬜ Pendente (próxima) |
| 3 | Banner de consentimento LGPD + gating | ⬜ Pendente |
| 4 | Dashboard de analytics no `seo-blog-frontend` | ⬜ Pendente |
| 5 | Playbook validado em segunda LP | ⬜ Pendente |
| 6 | Funil, atribuição multi-touch, coortes | ⬜ Pendente |

Detalhe e critérios de aceite por fase em [`PLANO-TRACKING.md`](./PLANO-TRACKING.md).

---

## Convenções desta pasta

- **Datas em ISO** (`YYYY-MM-DD`).
- **Referências cruzadas com `[[]]`** quando linkando entre docs vivos (ex.: `ver [[ARQUITETURA-TRACKING#schema]]`).
- **Status em emoji:** ⬜ pendente · 🟡 em andamento · ✅ concluído · 🔴 bloqueado · ⏸️ pausado.
- **Decisões importantes** entram no `CHANGELOG-TRACKING.md` com tipo `Decision`, mesmo que não tenham código.

---

## Como contribuir / atualizar

1. Mudou escopo de uma fase? → edita `PLANO-TRACKING.md` + linha em `CHANGELOG-TRACKING.md`.
2. Tomou decisão técnica? → atualiza `ARQUITETURA-TRACKING.md` + linha em `CHANGELOG-TRACKING.md` (tipo `Decision`).
3. Adicionou evento novo? → adiciona em `CATALOGO-EVENTOS.md` + linha em `CHANGELOG-TRACKING.md`.
4. Encontrou risco/dúvida? → adiciona em `PONTOS-ATENCAO-TRACKING.md`.
5. Plugou LP nova? → revisa `PLAYBOOK-NOVA-LP.md` (algo precisou ser adaptado?) + linha no changelog.

## Versionamento e cadência de commits

Estes docs vivem no repositório **[`seo-blog-docs`](https://github.com/VictorHOgawa/seo-blog-docs)** (raiz = a pasta `docs/`). O código fica nos repos `seo-blog-backend`, `seo-blog-frontend` e nas LPs.

**Convenção de checkpoint:** ao concluir uma fase, commitar **e dar push** — nos docs e no(s) repo(s) de código afetados. Durante a fase, commits locais livres como checkpoint; o push fecha a fase. Branches da iniciativa: `feat/tracking-hub-backend`, `feat/tracking-hub-skills` (front), `feat/tracking-hub-integration` (LP).
