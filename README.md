# Designer Eyes — Replenishment Engine

Single-file HTML replenishment tool for Designer Eyes eyewear retail.
Connects to Shopify POS via a Vercel proxy and generates NAV transfer orders.

**Stores:** 7 active retail POS locations (South Florida, San Juan PR)

## Quick Start (local)

1. Clone this repo
2. Copy `config.example.json` to `config.json` and fill in your values:
   - `supabase.url` and `supabase.anonKey` (Supabase project anon public key)
   - `shopify.proxyUrl` (your Vercel proxy URL)
3. Open `DE_Replenishment.html` directly in Chrome — no build step, no dev server
4. First-time flow inside the app: Test connection → Load from cache → Upload WH bin contents → Run replenishment

The app stores credentials and operational state in `localStorage` (capacity, exceptions, store selection, velocity tiers, NAV settings, theme).

## Architecture

- **Frontend:** Single `DE_Replenishment.html` (~236 KB) — vanilla JS + CSS tokens
  - Light + dark theme via CSS custom properties (toggle in topbar)
  - Charts: Chart.js 4.4.1 (CDN)
  - Excel I/O: SheetJS 0.18.5 (CDN)
  - Fonts: Inter + JetBrains Mono (Google Fonts CDN)
- **Proxy:** `shopify_proxy_vercel/` — Vercel Function that proxies Shopify Admin API requests with auth headers
- **Cache:** Supabase Postgres (`de_products` table) — avoids 5-8 min full Shopify load on every session

## Tabs

| Tab | Purpose |
|---|---|
| Live Connect | Test credentials, load catalog (cache or full Shopify), upload WH bin contents, inspect raw data |
| Inventory | Brand × Type × Store fill-rate matrix; central source for all filters |
| Dashboard | KPIs, fill-rate by store, units on hand, replenish-ready vs stockout combos |
| Capacity | Editable Brand × Type × Store capacity matrix with row/column totals |
| Exceptions | SKUs that should never be sent to specific stores |
| Store Selection | Pick stores, run replenishment, review recommendations, export NAV |

## Deployment

- **HTML app:** Auto-deploys to Vercel from this repo (`de-replen-pro` project)
- **Proxy:** Separate Vercel project (`shopify_proxy_vercel`) — see `shopify_proxy_vercel/`

## Development Notes

See [`CLAUDE.md`](./CLAUDE.md) for:
- Architecture rules + step flow
- Vercel proxy constraints (e.g. don't use `since_id` for orders, batch by `inventory_item_ids`)
- Filter centralization rules (Inventory tab is sole source)
- Replenishment logic (gap-fill, WH pool allocation by priority)
- Common bugs and root causes table

## License

Proprietary — Designer Eyes internal tooling.
