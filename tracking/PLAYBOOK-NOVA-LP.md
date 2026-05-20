# Playbook — Plugar uma LP nova no Hub de Tracking

> **Documento vivo.** A cada nova LP integrada, **revise este doc**: o passo X precisou ser adaptado? Atualize e registre entrada no [`CHANGELOG-TRACKING.md`](./CHANGELOG-TRACKING.md).
> **Última atualização:** 2026-05-19

---

## Para quem é

Este playbook é o procedimento canônico para integrar **qualquer LP nova** (de qualquer sistema do grupo) ao hub central de tracking do `seo-blog-backend`. Foi desenhado para ser executado por dev (ou pelo Claude operando no repo da LP) sem precisar tomar decisões arquiteturais.

**Pré-requisito:** Fases 0 e 1 do [`PLANO-TRACKING.md`](./PLANO-TRACKING.md) já concluídas. Se não, este playbook ainda não é executável.

---

## Pré-requisitos da LP

| Requisito | Por quê | Como verificar |
|---|---|---|
| Next.js 13+ (App Router preferível) | Hooks de rota e `<Script>` da Next | `package.json` → `"next": "^13"` ou superior |
| Acesso a env vars públicas (`NEXT_PUBLIC_*`) | Cliente precisa ler endpoint + site key | `.env.local` editável |
| Domínio definido (mesmo que `localhost:3000` no dev) | Backend só aceita CORS de origin cadastrada | Conhecido |
| Política de privacidade publicada | LGPD: banner aponta para ela | Página `/lgpd` ou `/privacidade` existe |

> **Se a LP não usa Next.js** (ex.: Vite SPA, Astro, HTML estático): a `lib/tracking/` foi escrita para funcionar em qualquer ambiente browser. O wrapper `<TrackingProvider>` é só conveniência. Em outros stacks, basta importar `client.ts`, `session.ts`, `attribution.ts`, `consent.ts` e chamar `client.track()` direto. Ajustes específicos não estão cobertos aqui — pedir nova entrada neste playbook ao integrar a primeira LP non-Next.

---

## Passo a passo

### 1. Cadastrar a LP no admin

No `seo-blog-frontend` (`/sites`):

1. Criar `Site` com `slug` único (ex.: `juridia`), `name`, `domain` (ex.: `https://juridia.com.br`).
2. Anotar a `publicKey` gerada (mostrada apenas uma vez no card de detalhe).
3. Confirmar `trackingEnabled = true`.

Se o domínio tiver múltiplos hosts (ex.: `www.x.com` + `x.com`), cadastrar ambos em `Site.allowedOrigins` (futuro; por ora usar o canônico).

> ❌ Não pular esta etapa. Sem `publicKey` cadastrada, o backend devolve `401` em todo POST.

### 2. Instalar arquivos do cliente na LP

Copie a pasta padrão para o repo da LP:

```
src/lib/tracking/
├── index.ts
├── client.ts
├── session.ts
├── attribution.ts
├── consent.ts
├── events.ts
├── provider.tsx
├── hooks.ts
└── components/
    ├── TrackedButton.tsx
    └── ConsentBanner.tsx
```

> **Fonte canônica:** após a Fase 2, o código de referência fica em `health-voice-institutional-v2/src/lib/tracking/`. Quando virar pacote npm interno (`@seo-blog/tracking-client`), este passo vira `npm install`.

Se sua LP adicionar eventos próprios (não-canônicos), **não edite `events.ts` diretamente** — crie `src/tracking/events.local.ts` estendendo o tipo, e adicione esses eventos ao [`CATALOGO-EVENTOS.md`](./CATALOGO-EVENTOS.md) com a tag `[lp:juridia]`.

### 3. Configurar env vars

`.env.local`:

```env
NEXT_PUBLIC_TRACKING_ENDPOINT=https://api.seo-blog.example.com
NEXT_PUBLIC_TRACKING_SITE_KEY=pk_xxxxxxxxxxxx
NEXT_PUBLIC_TRACKING_CONSENT_VERSION=2026-05-19-v1
# opcional, mantém pixels existentes funcionando
NEXT_PUBLIC_GTM_ID=GTM-XXXXX
NEXT_PUBLIC_META_PIXEL_ID=
NEXT_PUBLIC_TIKTOK_PIXEL_ID=
```

`.env.example` deve listar todas (sem valores) — checar in.

### 4. Adicionar `<TrackingProvider>` no layout root

`src/app/layout.tsx`:

```tsx
import { TrackingProvider } from '@/lib/tracking';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="pt-BR">
      <body>
        <TrackingProvider
          siteSlug="juridia"
          endpoint={process.env.NEXT_PUBLIC_TRACKING_ENDPOINT!}
          siteKey={process.env.NEXT_PUBLIC_TRACKING_SITE_KEY!}
          consentVersion={process.env.NEXT_PUBLIC_TRACKING_CONSENT_VERSION!}
        >
          {children}
          <ConsentBanner />
        </TrackingProvider>
      </body>
    </html>
  );
}
```

O `<TrackingProvider>` cuida sozinho de:
- Criar/recuperar `anonymousId` + `sessionId`.
- Disparar `page_view` em cada mudança de pathname.
- Capturar UTM no first hit da sessão e gravar em `tracking_attribution`.
- Manter buffer pré-consent.

### 5. Migrar CTAs para `<TrackedButton>` ou `useTracking()`

**Antes (cru):**
```tsx
<button onClick={() => { window.dataLayer.push({ event: 'cta_click' }); router.push('/lead'); }}>
  Quero saber mais
</button>
```

**Depois (com tracking padronizado):**
```tsx
<TrackedButton elementId="cta_hero_quero_saber_mais" eventProperties={{ label: 'Quero saber mais' }}>
  Quero saber mais
</TrackedButton>
```

Ou em código imperativo:
```tsx
const track = useTracking();
const onClick = () => {
  track('cta_click', { elementId: 'cta_hero_quero_saber_mais', properties: { label: 'Quero saber mais' } });
  router.push('/lead');
};
```

Padrão para `elementId`:
- Formato: `<tipo>_<seção>_<verbo_curto>` em snake_case.
- Exemplos: `cta_hero_demo`, `cta_pricing_assinar`, `link_footer_lgpd`, `form_lead_submit`.
- Cada elemento clicável crítico deve ter um id único. Use o catálogo do site (criar `src/tracking/elements.ts` enum se a LP for grande).

### 6. Integrar formulário de lead

Antes do submit:
```ts
track('form_submit', { elementId: 'form_demo_main', properties: { fields: Object.keys(values) } });
```

Após resposta da API do form (server-side):
```ts
const lead = await fetch('/api/lead', { method: 'POST', body: JSON.stringify(values) }).then(r => r.json());

await trackingClient.lead({
  name: values.name,
  email: values.email,
  phone: values.phone,
  source: 'campaing1',
  buttonId: 'cta_hero_demo',
  destination: '/obrigado',
  payload: { /* campos extras */ },
});

track('lead_created', { properties: { leadId: lead.id, source: 'campaing1' } });
```

> A API do form da LP **continua existindo** (salva no banco da LP se for o caso — ex.: `campaign_leads` da Health Voice). O `client.lead()` envia o lead para o hub **em paralelo**. Não migrar dado, espelhar.

### 7. Manter pixels existentes (GTM, Meta, TikTok) coexistindo

O `client.ts` já dispara `dataLayer.push({ event: name, ...properties })` quando `window.dataLayer` existe. Não há trabalho extra.

Para Meta/TikTok: o handler de evento canônico pode chamar também `fbq('track', ...)` / `ttq.track(...)` — isso vive na LP, não no client de tracking.

### 8. Validação manual

Antes de mergear:

1. Abrir a LP em janela anônima.
2. **Verificar console:** sem erros do `TrackingClient`.
3. **Verificar Network:** primeira ação → `POST /tracking/session` (200 ou 204). Depois `POST /tracking/event`.
4. **Verificar no dashboard admin** (`(admin)/analytics?siteId=juridia`): sessão aparece em até 10s, eventos aparecem.
5. **Verificar UTM:** acessar `https://lp.com.br/?utm_source=teste&utm_campaign=playbook` → na tabela `tracking_attribution` deve constar.
6. **Verificar consent gating:** com banner ainda aberto, console.log do client deve mostrar eventos no buffer, nada no Network. Aceitar → flush.
7. **Verificar replay:** desconectar Wi-Fi, clicar 3x num botão tracked, reconectar. No banco: 1 sessão, 3 eventos com 3 `eventId` diferentes (não 1 duplicado nem 3 do mesmo).

### 9. Atualizar este playbook

Se você precisou improvisar em algum passo, **edite este arquivo** e registre a alteração no `CHANGELOG-TRACKING.md` como `Changed (playbook)`. O playbook só serve se reflete a realidade.

---

## Troubleshooting

| Sintoma | Causa provável | Ação |
|---|---|---|
| `POST /tracking/event` retorna 401 | `X-Site-Key` ausente/errada | Conferir env `NEXT_PUBLIC_TRACKING_SITE_KEY` e `Site.publicKey` no admin |
| Retorna 403 | `Site.trackingEnabled = false` | Habilitar no admin |
| Retorna 429 | Burst (testes locais com loop) | Esperar 1min; ajustar rate-limit se for tráfego real |
| CORS bloqueia | `Site.domain` não cobre origin atual | Cadastrar todos os hosts (incl. `localhost:3000` no dev) em `allowedOrigins` |
| Eventos vão para o banco mas não aparecem no dashboard | Filtro de `siteId` no dashboard difere | Conferir seletor de site no painel |
| `page_view` duplicado | Layout monta 2x (Strict Mode em dev) | Esperado em `npm run dev`; production não duplica |
| UTM não capturado | Cliente entrou via deeplink sem UTM, ou cookie já tinha sessão ativa | Limpar storage ou abrir aba anônima |
| Tudo aparece, mas sem `country` / `city` | Geo lookup desativado/desconfigurado | Ver decisão em [`PONTOS-ATENCAO`](./PONTOS-ATENCAO-TRACKING.md) |

---

## Anexo — Checklist de aceite para considerar a LP integrada

- [ ] Site cadastrado no admin com `publicKey`
- [ ] `lib/tracking/` instalada
- [ ] Env vars setadas em `.env.local` e listadas em `.env.example`
- [ ] `<TrackingProvider>` no layout root
- [ ] `<ConsentBanner />` renderizado
- [ ] Todos os CTAs principais usam `<TrackedButton>` ou `useTracking()`
- [ ] Form de lead dispara `form_submit` + `lead_created`
- [ ] Validação manual (passo 8) passou nos 7 sub-itens
- [ ] Dashboard `(admin)/analytics` mostra essa LP no seletor e tem dados reais
- [ ] Entrada em [`CHANGELOG-TRACKING.md`](./CHANGELOG-TRACKING.md) registrando integração

Após esse checklist, a LP está oficialmente integrada.
