# Designer Eyes Replenishment Engine — Claude Working Notes

## Project Overview
Single-file HTML replenishment tool for Designer Eyes eyewear retail.
Connects to Shopify POS via Vercel proxy, generates NAV transfer orders.
Product catalog cached in Supabase to avoid proxy timeout on every session.

**File:** `DE_Replenishment.html`
**Proxy:** `https://de-shopify-proxy.vercel.app`
**Shopify store:** `point-of-view-eyewear.myshopify.com`
**Supabase:** `https://zoqihptkwurpqenpjgfw.supabase.co`
**Stores:** 7 active retail POS locations (South Florida, San Juan PR)

---

## Quick Start
- Open `DE_Replenishment.html` directly in Chrome (no build step, no dev server)
- Credentials auto-load from `config.json` on startup; localStorage caches the rest
- First-time flow: Test connection -> Load from cache -> Upload WH -> Run replenishment
- Theme: 🌙/☀ toggle in topbar; persists to `de_theme` localStorage key (light/dark)

## Deployment
- **GitHub:** https://github.com/Roblesmau/de-replen-pro (master branch)
- **Live app (Vercel):** https://de-replen-pro.vercel.app
- **Shopify proxy (Vercel, separate project):** https://de-shopify-proxy.vercel.app/api/shopify
- **Vercel team:** `roblesmau-2975s-projects`
- `vercel.json` rewrites `/` → `/DE_Replenishment` (cleanUrls strips `.html`); deployed file is served at the root domain
- `config.json` is gitignored (local only); contributors copy `config.example.json` and fill in their own
- Proxy source code is NOT in this repo (deployed-only) — `shopify_proxy_vercel/api/` is empty locally; recover via `vercel pull` from the proxy project if needed

### Deploy workflow
```bash
# 1. Commit + push to GitHub (always)
git add -A
git commit -m "..."
git push

# 2. Deploy to Vercel production
#    (Until Vercel→GitHub auto-deploy is connected via Settings → Git)
vercel --prod --yes
```

After connecting GitHub in Vercel project settings, step 2 becomes automatic on every push.

---

## Architecture Rules

### Single File
- Everything in one `.html` file — no build step, opens locally in Chrome
- CDN libs: Chart.js 4.4.1, SheetJS/xlsx 0.18.5
- localStorage: credentials (`de_sb_url`, `de_sb_key`, `de_proxy_url`, `de_shopify_token`) + capacity (`de_capacity_v1`)

### Step Flow (Current — renamed May 2026)
```
Step 1  — Test Connection
Step 2  — Product Catalog (choose ONE):
            loadCatalogFromCache()    -> reads Supabase -> auto-triggers Step 3
            loadCatalogFromShopify()  -> Pass B + Pass A -> Supabase sync -> auto-triggers Step 3
Step 3  — Live Data (auto, always fresh):
            loadLiveData(itemMap)
            -> Locations
            -> Inventory (batch by item IDs)
            -> Orders 90d + Prior Year (parallel, sequential fallback)
            -> processLiveData() -> DATA[] -> apply PY velocity
            -> refreshWHAfterStep2B() -> auto-fires Step 4A or 4B based on S.whSource
Step 4A — WH Bin Contents (Excel upload OR pull last upload from replen_wh_bins)
   OR
Step 4B — Live availability from Supabase inventory_available (read-only)
Step 5  — Inspect Data
```

**Numbering note:** the function `loadLiveData` is still called "Step 2B" in code (variable names, status IDs, log messages), but the **user-visible label** is now `Step 3`. The `setStep()` label dictionary maps internal keys `1/'2a'/'2b'/3/4` to visible prefixes `1/2/3/4/5`. Don't rename the internal keys — too many touchpoints, no functional gain.

### Universal Key: Variant SKU
- `Variant SKU` (from `variants[].sku`) is the ONE universal key across all tables
- Strip leading apostrophe always: `normSku(s) = String(s||'').replace(/^'+/,'').trim()`
- `inventory_item_id` is internal bridge only — never expose in downloads or UI

---

## Theme System (light + dark)
- CSS tokens defined under `:root, [data-theme="light"]` and `[data-theme="dark"]` blocks
- `data-theme` set on `<html>` before paint (anti-flash boot script reads `localStorage.de_theme`)
- Glassmorphism on topbar + sticky tabs via `var(--glass)` + `backdrop-filter: blur(14px)`
- Tokens used everywhere — NO hardcoded `#fff` / hex colors in components (NAV preview ported to tokens, including zebra rows via `var(--surface)` / `var(--surface-2)`)
- Fonts: Inter (body) + JetBrains Mono (`.mono`, `code`, `.metric-val`, `.sim-val`, KPI numbers, SKU columns) loaded from Google Fonts CDN
- `font-variant-numeric: tabular-nums` on body + tables to prevent column jitter on data updates
- Focus rings: `*:focus-visible { outline: 2px solid var(--blue); outline-offset: 2px }`
- `prefers-reduced-motion` honored globally
- Skip link: `<a href="#main-content" class="skip-link">` — appears on first Tab keypress

---

## Vercel Proxy — Critical Constraints (CONFIRMED)

- **`/products.json` pagination always returns `products:[]`** — blocked regardless of params
- **`/variants.json` works** — full `since_id` pagination + `fields=` filtering confirmed
- **`/products.json?ids=id1,id2,...` WORKS** — fetching by specific IDs bypasses the block
- **`/inventory_levels.json?inventory_item_ids=` WORKS** — batch by known item IDs (100/call), all locations at once
- **`/inventory_levels.json?inventory_item_id_gt=` DOES NOT WORK RELIABLY** — proxy ignores/strips cursor, same 250 records returned every page, causes wrap-around cycling. NEVER USE.
- **`/orders.json` works** — paginate with `created_at_max` cursor (NOT `since_id`)
- **`/inventory_items.json?ids=` works** — safety net for unmapped item IDs
- **Auth:** `X-DE-Domain` + `X-DE-Token` headers, or `X-DE-Auth` (base64 `domain:token`). Never URL params

### CRITICAL — Shopify API version is coupled across app + proxy
- Both the app (`DE_Replenishment.html`) and the proxy (`shopify_proxy_vercel/` — separate Vercel project, source NOT in this repo) hardcode the Shopify API version (currently `/admin/api/2024-01/`).
- The proxy has an **allowlist** that rejects any version it doesn't recognize with `403 Endpoint not allowed`.
- **Bumping the app version alone breaks every API call.** You MUST update the proxy's allowlist at the same time.
- When the 2024-01 version eventually sunsets (Shopify deprecates after 24 months — early 2026 is the danger zone), the migration is:
  1. Locate the proxy source code (NOT in this repo — separate Vercel project `de-shopify-proxy`, Vercel team `team_WukDaY2vZ0o3j0S0RwLXdwuV`, project id `prj_Bqbs4ki3fyperVixSWkYhU64Yb0L`). Source may be on dev's machine or another git repo.
  2. Update proxy allowlist + redeploy
  3. Update all 8 `2024-01` references in `DE_Replenishment.html`
  4. Update CLAUDE.md examples (2 places)
  5. Ship together — never the app alone.
- **Transient 404s mid-pagination** (e.g. page 1 of variants works, page 2 returns 404) are usually Shopify rate-limit hiccups or proxy timeouts, NOT version sunset. Retry once before assuming the version is dead.

---

## Products Fetch — CONFIRMED WORKING (Two-Pass Reverse)

```
PASS B FIRST: GET /variants.json?fields=id,product_id,sku,price,barcode,inventory_item_id&limit=250
  -> paginate with since_id (~124 pages, 30,837 variants)
  -> builds variantMeta[inventory_item_id] = {variantSku, cost, barcode, productId}
  -> collects all unique product_ids

PASS A SECOND: GET /products.json?ids=id1,id2,...&limit=250
  -> batches of 250 product_ids (~124 batches, 300ms pause between)
  -> builds productMeta[product_id] = {brand, type, title}

JOIN: variantMeta[itemId].productId -> productMeta[productId]
  -> itemMap[inventory_item_id] = {variantSku, brand, type, desc, cost, barcode}
```

**NEVER attempt:**
- `products.json` with any pagination — always returns empty array
- `products.json` with `fields=` — returns empty array
- Single-pass embedding `variants[]` — proxy times out after ~2 pages

---

## Inventory Fetch — CONFIRMED WORKING (Batch by Item IDs)

```javascript
// CORRECT: dynamic batch size to never exceed limit=250
// floor(250 / storeCount) guarantees items × stores < 250 per call
const retailLocIds = retailLocs.map(l => String(l.id));
const allItemIds = Object.keys(itemMap); // ~30,839 IDs
const invBatchSize = Math.floor(250 / Math.max(retailLocIds.length, 1)); // = 35 for 7 stores
let allInv = [];
for(let i = 0; i < allItemIds.length; i += invBatchSize){
  const batch = allItemIds.slice(i, i + invBatchSize);
  const url = '/admin/api/2024-01/inventory_levels.json'
    + '?inventory_item_ids=' + batch.join(',')
    + '&location_ids=' + retailLocIds.join(',')
    + '&limit=250';
  const d = await shopifyFetch(url);
  allInv = [...allInv, ...(d.inventory_levels || [])];
}
// Deduplicate by (inventory_item_id, location_id)
const invDedup = new Map();
allInv.forEach(il => {
  const k = String(il.inventory_item_id) + '|' + String(il.location_id);
  if(!invDedup.has(k)) invDedup.set(k, il);
});
allInv = [...invDedup.values()];
```

**CRITICAL — batch size math:**
- `limit=250` is Shopify's hard cap per response
- Each call returns up to `batchSize × storeCount` records
- If `batchSize × storeCount > 250`, Shopify silently truncates — no error, no warning
- With 7 stores: `floor(250/7) = 35` items/batch → max 245 records/call ✓
- Self-adjusting: if stores increase, batch size shrinks automatically

**MUST run after itemMap is built** — needs known item IDs.
**Confirmed result:** ~61,595 records across 7 stores, 0 skipped.

**NEVER use:**
- `inventory_item_id_gt` cursor — proxy ignores it, causes cycling (same 250 records on every page)
- `Promise.all` parallel per-store fetches — causes rate limit storm (429s)
- Per-store sequential loops — replaced by batch-by-ID approach

---

## Orders Fetch — CONFIRMED WORKING (created_at_max cursor)

```javascript
// CORRECT: created_at_max cursor walking backward
async function fetchOrderPages(minDate, maxDate, label){
  let orders = [], cursor = maxDate;
  while(true){
    const url = '/admin/api/2024-01/orders.json'
      + '?status=any&created_at_min=' + minDate
      + '&created_at_max=' + cursor
      + '&limit=250&fields=id,location_id,line_items,refunds,created_at';
    const d = await shopifyFetch(url); const batch = d.orders || [];
    orders = [...orders, ...batch];
    if(batch.length < 250) break;
    cursor = batch[batch.length-1].created_at;
  }
  // Deduplicate by order ID
  const dedup = new Map(); orders.forEach(o => dedup.set(String(o.id), o));
  return [...dedup.values()];
}

// Run 90d + Prior Year in parallel
const [ord90, pyOrders] = await Promise.all([
  fetchOrderPages(ago90, new Date().toISOString(), 'Orders 90d'),
  fetchOrderPages(pyStart, pyEnd, 'Prior Year 120d')
]);
// Sequential fallback if parallel fails
```

**NEVER use `since_id` for orders** — it ignores `created_at_min/max` filters on pages 2+.
Only 3 dates returned (most recent in system) instead of full date window. Root cause confirmed.

### Date Windows
- **90d:** `now − 90 days → now`
- **Prior Year (Option B):** `pyStart = now − 365d − 90d` · `pyEnd = pyStart + 120d`
  - Running Apr 8, 2026: Jan 8, 2025 → May 8, 2025
  - Same calendar start as current 90d window, one year back, extended 120 days forward

### Refunds
All order URLs include `refunds` in `fields=`. Velocity nets `refund_line_items[].quantity`.
`S.raw.returns90d` and `S.raw.returnsPY` stored for inspector stats.

---

## shopifyFetch — Resilient Fetch (8 attempts)

Handles 429 rate limits, 5xx upstream errors (Cloudflare/Vercel/Shopify 502/503/504),
network errors, and non-JSON 200 responses. Exponential backoff capped at 30s.

```javascript
const MAX = 8;
for(let attempt = 0; attempt < MAX; attempt++){
  if(attempt > 0) await new Promise(r => setTimeout(r, Math.min(30000, 1500 * Math.pow(1.7, attempt))));
  let r;
  try { r = await fetch(url, {headers}); }
  catch(netErr){ /* retry network errors */ continue; }
  if(r.status === 429){ await sleep(3000 + attempt * 2000); continue; }       // rate limit
  if(r.status >= 500 && r.status < 600){ /* retry upstream 5xx */ continue; } // 502/503/504
  if(!r.ok){ /* throw with cleaned text snippet */ }
  const txt = await r.text();
  if(!txt.trim()) return null;
  try { return JSON.parse(txt); } catch(_){ /* HTML in 200 — retry */ continue; }
}
```

Without the 5xx retry, a single Cloudflare/Vercel 502 mid-fetch (e.g. on variants page 42 of 124)
kills the entire 5-8 min Shopify load. With it, the run survives transient upstream errors silently.

---

## Pipeline Steps — Full Shopify Load (Step 2A + 2B)

**Step 2A: loadCatalogFromShopify() (~5-8 min)**
1. Pass B: `GET /variants.json` paginated → variantMeta
2. Pass A: `GET /products.json?ids=...` batches → productMeta
3. Join → itemMap; safety net for unmapped IDs via `/inventory_items.json?ids=`
4. Auto-sync itemMap → Supabase (non-blocking)
5. → auto-triggers loadLiveData(itemMap)

**Step 2B: loadLiveData(itemMap) (~2-3 min)**
1. `GET /locations.json` → retail stores
2. `GET /inventory_levels.json?inventory_item_ids=...` batch → allInv
3. `GET /orders.json` 90d + Prior Year in parallel
4. `processLiveData()` → DATA[]
5. Apply PY velocity to DATA

## Pipeline Steps — Cache Load (Step 2A + 2B)

**Step 2A: loadCatalogFromCache() (~30 sec)**
1. Supabase `replen_products` → itemMap (paginated 1000/page, **`order=inventory_item_id` required**)
2. → auto-triggers loadLiveData(itemMap)

> **CRITICAL:** Always include `&order=inventory_item_id` on Supabase paginated reads.
> Without it, PostgreSQL returns rows in arbitrary order — offset pagination skips rows
> silently at page boundaries. Confirmed root cause of missing SKUs in inventory fetch.

**Step 2B: loadLiveData(itemMap)** — identical to above

---

## Supabase

### sbFetch — Empty Body Pattern (CRITICAL)
```javascript
if(!r.ok){ const t = await r.text(); throw new Error('Supabase ' + r.status + ': ' + t); }
if(r.status === 204) return null;
const txt = await r.text();          // READ AS TEXT FIRST
if(!txt || !txt.trim()) return null; // 200 + empty body on upsert with return=minimal
return JSON.parse(txt);
```
Never call `r.json()` directly — throws "Unexpected end of JSON input" on upsert responses.

### Credentials
- Key: `anon public` JWT (`eyJ...`) — NOT `sb_publishable_` (returns 403)
- localStorage: `de_sb_url`, `de_sb_key`

### Sync
- Upsert batches of 1000: `on_conflict=inventory_item_id&resolution=merge-duplicates`
- Non-blocking: `syncCatalogToSupabase(itemMap).catch(e => clog('Supabase sync error: ' + e.message))`

### Tables
```sql
replen_products (inventory_item_id text PK, variant_sku text, brand text, type text,
             title text, price numeric, barcode text, synced_at timestamptz)
replen_sync_log (id serial PK, synced_at timestamptz, product_count int, variant_count int)
```

---

## Warehouse Exclusion — CRITICAL
Apply in loadLiveData AND processLiveData:
```javascript
const EXCLUDE_IDS = new Set(['35933356085','71277510830']);
locs.filter(l => l.active
  && !EXCLUDE_IDS.has(String(l.id))
  && !l.name.toLowerCase().includes('fulfillment')
  && !l.name.toLowerCase().includes('central')
  && !l.name.toLowerCase().includes('corporate parkway'))
```

---

## Known Baseline Numbers (Confirmed Apr 2026)
- Variants: ~30,837 | itemMap entries: ~30,837 | Products: ~30,833
- Active stores: 7 retail | Inventory records: ~51,813 (7,000–11,600/store)
- Brands: 201 | Types: 16 | Unknown brand: 0
- Orders 90d: ~8,572 | Prior Year 120d: ~15,976
- WH SKUs: ~2,621 (86% matched, ~27,217 units)

## Store Location IDs
```
67928096942  R001 - POV Sawgrass (Sunrise)        [inactive in current 7-store set]
67928129710  R002 - Just One Look (Miami)          11,595 inv records
67938123950  R003 - Eyes on Lincoln (Miami Beach)   8,298 inv records
67928162478  R004 - PV2 Dolphin (Miami)             7,683 inv records
67938156718  R005 - DE Mall of San Juan (San Juan)  6,293 inv records
67938189486  R006 - DE Brickell (Miami)             6,887 inv records
67938222254  R008 - DE Florida Mall (Orlando)       [inactive in current 7-store set]
67938255022  R009 - Aventura (Aventura)             6,443 inv records
67928195246  R010 - PV3 iDrive (Orlando)            [inactive in current 7-store set]
67928228014  R011 - PV4 Vineland (Orlando)          [inactive in current 7-store set]
67938287790  R014 - DE Plaza Las Americas (San Juan) 4,614 inv records
35933356085  WAREHOUSE — 530 Sawgrass Corporate Parkway (EXCLUDE)
71277510830  EXCLUDE — Business Central Fulfillment Service
```

---

## UI Structure

### Tab Order (current — Transfers shipped July 2026)
`Live Connect -> Inventory -> Dashboard -> Capacity -> Exceptions -> Store Selection -> Transfers`

Keyboard shortcut Ctrl+7 = Transfers. The Settings tab is hidden by default (`display:none` on the button) and only contains NAV export settings — all data-source/priority/replenishment-rules controls have been moved out.

### Live Connect Steps (Current — renamed May 2026)
1. Test connection
2.  Load from cloud OR Full Shopify load (product catalog)
3.  Auto: Locations + Inventory + Orders 90d + Prior Year (was 2B)
4A. Upload WH Bin Contents (Excel) — pushed to `replen_wh_bins`
4B. Load live availability from Supabase `inventory_available` (read-only)
5.  Inspect data

User picks 4A or 4B via the **WH source segmented toggle** above the steps row (persistence: `localStorage.de_wh_source`). Default is 4A. After Step 3 completes, `refreshWHAfterStep2B()` auto-fires the matching Step 4 path so the pipeline is one click end-to-end.

### WH Source Selection (4A vs 4B)

**Step 4A — Excel upload (`replen_wh_bins` cloud table)**
- User uploads NAV bin contents Excel file via `parseWH()`.
- Filtered at upload time by Allowed Locations + Allowed Zones in **Upload Settings**.
- Only filtered rows pushed to `replen_wh_bins` via `pushWHToSupabase()`.
- Schema: `replen_wh_bins(id, bin_code, item_no, qty, location_code, zone_code, uploaded_at, uploaded_by)` + `replen_wh_meta(id=1, last_uploaded_at, last_uploaded_by, row_count, sku_count)`.
- "Load from cloud" button under Step 4A re-pulls the last upload.
- `S.whRaw` carries the per-bin granularity; `S.wh` is the SKU-aggregated view used by `buildRecos`.

**Step 4B — Supabase live (`inventory_available` table)**
- Populated by an external Python script (SQL Server → Supabase). App is read-only against this table.
- Schema: `inventory_available(item_no, location_code, total_inventory, reserved_for_picking, reserved_for_ato, available_qty, number_of_bins, sample_zone, synced_at, row_hash)`. PK `(item_no, location_code)`.
- Replenishment uses `available_qty` (excludes reservations).
- Filters applied: `?available_qty=gt.0` + `?location_code=in.(...)` server-side, `sample_zone` filter client-side.
- "Load from Supabase" button under Step 4B re-pulls fresh.
- Auto-refreshes whenever Step 3 (loadLiveData) completes — no separate timer.

**WH overview panel** (`#whOverviewPanel`) — single shared panel that swaps content based on the active source. Mirrors the SQL:
```sql
-- Step 4A:
select location_code, count(distinct item_no) as skus, sum(qty) as qty
  from replen_wh_bins group by location_code order by location_code;
-- Step 4B:
select location_code, count(*) as skus, sum(available_qty) as available, max(synced_at)
  from inventory_available group by location_code order by location_code;
```
Located in the top-of-Live-Connect 2-column grid, paired with the always-visible **Upload Settings** panel. The settings panel hides the column-mapping section when 4B is active (column mapping only matters for Excel uploads).

### Raw Data Inspector
- Dropdown: Unified Join View | Locations | Products | Inventory Levels | Orders 90d | Orders Prior Year | WH Bin Contents | ⚠ WH SKUs without Shopify match | ⚠ Warnings — Unknown SKUs
- Buttons: Copy JSON | Download (Excel of current selection)
- Stats bar: date range badge + units sold · returned · net
- Orders 90d badge: blue · Orders Prior Year badge: amber
- "WH SKUs without Shopify match" — lists `S.wh.filter(w => !w.matched)` with bin/location/zone cross-ref + heuristic "Likely cause" (whitespace, leading apostrophe, missing from catalog). Auto-shown when unmatched count > 0; hidden after Clear WH.
- Appears BEFORE "Data Loaded from Shopify" card

### Filter Architecture — CENTRALIZED (as of commit 2302c55, Apr 24 2026)
**ALL filtering is driven exclusively from the Inventory tab.** No filter controls exist on Dashboard or Simulator.

- **Inventory tab** owns: `invStoreFilter` (multi-select), `invBrandFilter` (multi-select), `invTypeFilter` (multi-select), forecast model selector
- **Dashboard tab** shows only: hint text `"Filters controlled in Inventory tab"` + Units/$ toggle. NO dfStore/dfBrand/dfType elements exist.
- **Simulator tab** reads directly from Inventory filters via `getFilterValues('invStoreFilter')` etc.
- `getFD()` reads only from `invStoreFilter` / `invBrandFilter` / `invTypeFilter`
- `getActiveBrands()` reads only from `invBrandFilter`

**REMOVED functions** (do not re-add):
- `populateDashFilters()` — was syncing Dashboard dropdowns from Inventory
- `populateSimSelects()` — was populating Simulator dropdowns
- `syncInvToDash()` / `syncDashToInv()` — were keeping Dashboard ↔ Inventory in sync
- `applySettings()` — old Settings-tab "Apply & recalculate" button; replaced by velocity tiers + auto-rebuild
- `savePriority()` — old Settings-tab priority dropdown handler; Store Selection priority dropdown calls `syncPriority()` directly

### Inventory Tab Filter Layout (current)
Labeled column sections in a flex row, `align-items:flex-start`, `gap:14px`:
```
[STORES]  [BRANDS]  [TYPES]  [MODEL]  [ACTIONS]  [COUNT]
 multi-    multi-    multi-   forecast  Select      N combos
 select    select    select   dropdown  All btn     shown
 260px h   260px h   260px h            (resets
 8px pad   8px pad   8px pad             to all)
```
Each column: `flex-direction:column`, label is `font-size:11px, uppercase, letter-spacing:0.5px, color:var(--muted)`

Inventory filter changes propagate to: Dashboard widgets (`updateDash`), Capacity tab (rendered on tab switch), Replenishment scope (`buildRecos` re-runs).

### Capacity — Cloud sync (Supabase, shared across users)
- Two new Supabase tables: `replen_capacity` (rows: store_id, brand, type, max_capacity, updated_at, updated_by) and `replen_capacity_meta` (single-row table holding last_uploaded_at, last_uploaded_by, cell_count for the badge)
- SQL DDL documented in commit message — run once in Supabase SQL Editor
- Pull on boot: `pullCapacityFromSupabase()` fires after `processLiveData` populates S.CAP from localStorage; cloud rows are then overlaid (cloud = source of truth). Also fires on first Capacity tab activation
- Push on save: `pushCapacityToSupabase()` called from `exportCapXLSXAndSave` (rebadged "Save & Sync") AND from `importCapFromFile` success. Upserts all non-zero cells in batches of 500 with `on_conflict=store_id,brand,type&resolution=merge-duplicates`, then bumps the meta row
- Clear all: also calls `wipeCloudCapacity()` (deletes all rows in `replen_capacity` and the meta row)
- Toolbar buttons: `⬇ Save & Sync` (was `Export & Save`) · `☁ Pull` (manual re-fetch) · existing Reset / Clear all
- Cloud status box at top of Capacity tab (`#capCloudBox`): shows `Last sync: 5m ago by mauricio · 1,245 cells in cloud`; updates after every push/pull. Helper: `setCloudStatus`, `fmtRelTime`, `getCurrentUserLabel` (best-effort: domain part of Shopify domain → 'anonymous' if missing)
- Graceful degradation: if Supabase table doesn't exist (404), badge shows "Cloud table not found — run the SQL DDL"; localStorage continues working

### Capacity Tab (current)
- **Top section:** Capacity matrix upload card (file input + "⬇ Download template (5 examples)" button) — moved here from Settings
- **Toolbar:** `+ Add brand to stores` · hint text "Filters controlled in Inventory tab" · Import · Template · Export & Save · Reset
- **Local filter dropdowns hidden** (`capFType` / `capFBrand` are `display:none` with default values that pass through) — Inventory tab is sole filter source
- **Reset vs Clear all** — two distinct buttons:
  - `↺ Reset` (resetCap) — restores defaults from `S.ORIG` (keeps matrix populated; clears localStorage too)
  - `🗑 Clear all` (wipeAllCapacities) — deletes EVERY capacity value from memory + localStorage so you can import a fresh Excel containing only the brands you carry. Workflow: Clear all → Import fresh Excel (additive). Import remains additive — only adds rows present in the file.
- **Table:** Brand/Type rows × Store columns (filtered to Inventory selection)
- **Bottom summary:** scope label · total units · changed-cells count (no Export Excel button — top toolbar is sole action source)

**Persistence:**
- Auto-save to localStorage (`de_capacity_v1`) on every cell edit (`onCapChange`)
- Restored in `processLiveData` after rebuild via `loadCapFromLocalStorage()` + `applyCapToData()`
- Reset clears both in-memory `S.CAP` AND localStorage (true "start over")

**Filter inheritance** (on tab switch only — not reactive):
- Rows filtered to Inventory `invBrandFilter` × `invTypeFilter`
- Per-store columns hidden for stores not in `invStoreFilter`
- Edits while filtered still write to full `S.CAP` (filter is view-only)
- Import/Export ignore the filter (operate on full dataset)

### Exceptions Tab (current — between Capacity and Store Selection)
Single card: "Exception list — SKUs to NEVER send to specific stores"
- Upload `.xlsx` with columns `Store Code` · `Variant SKU` (numeric SKUs, written as text cells)
- Buttons: Download template (5 examples) · Export current list · Clear list
- Counter: `N exceptions loaded`
- Stored in `S.exceptions` (Set of `storeId|sku` keys, normalized lowercase)
- Persisted to `localStorage.de_exceptions_v1` as JSON array
- Loaded on boot AND after every Shopify/cache load (`loadExcFromLocalStorage`)
- Active in `buildRecos()` — `isExcepted(d.storeId, d.sku)` check skips matching pairs entirely (never enter `S.recos`, never consume warehouse units)
- **Visible list table** columns: `Store Code · Store Name · Variant SKU · Brand · Type · [Remove]` — Brand/Type looked up from `S.raw.itemMap` at render time (lowercase SKU keys); shows `—` if SKU not in catalog

### Store Selection Tab (current)
**Toolbar:** Select all · Clear · `N/Y stores selected` · scope badges (`N stores · N brands (all) · N types (all)`) · Priority dropdown · ⚙ Send qty rules · Run replenishment · Export NAV

**Velocity tier panel** (`⚙ Send qty rules` toggles `velTierPanel`):
- Editable list of tiers `{minPerWeek, qty}`. Defaults: `[{0,1},{1,2},{3,3},{7,5}]` in `VEL_TIERS_DEFAULT`
- "Allow over-capacity sends" checkbox (default OFF)
- Replaces the old `defQty`/`hotQty` system. `getQtyForRow(d)` returns tier qty for `d.velocity90 / 12.857` (per-store per-SKU weekly velocity)

**Priority order preview** (`priorityListWrap` below the store grid):
- Numbered ordered list reflecting current priority mode (Urgency/Velocity/Manual)
- Shows store rank · short name · location ID · % full pill · metric (DOS / 90d units / "Selected #N")
- Refreshed on `renderStoreGrid()` and `syncPriority()`

**Persistence:**
- `de_store_sel_set` — selected store IDs
- `de_store_sel_order` — selection sequence (for Manual priority)
- `de_priority_mode` — `urgency` | `velocity` | `manual`
- `de_vel_tiers` — JSON of tier array
- `de_allow_overcap` — `'1'` or `'0'`
- On reload, saved IDs are intersected with current STORES (silently skipped if a store no longer exists)

**Replenishment scope filtering:**
- `buildRecos()` reads `invBrandFilter` / `invTypeFilter` from Inventory tab — rows whose brand/type isn't selected are skipped (in addition to exception list)
- `S.storeStats.pct` (powers store tile + priority list `% full`) also honors the same filter, with capacity dedup by `(brand, type)` Set

**Run summary card** (after `runRun()` fires):
- Header includes `· Priority: X · HH:MM:SS` stamp so user can confirm a fresh run
- 6 KPI cards in `row3` grid: Stores · NAV lines · Total units · Hot sellers · WH qty before · WH qty after (with `−N` red delta)
- **Per-store allocation table** below KPIs — ordered by current priority via `getStorePriorityOrder()`. Shows `# · Store · Lines · Units · % of total` with progress bars. This is where priority-mode differences become visible (aggregate KPIs stay stable when WH is the bottleneck).

**Replenishment Impact Report** (visible after run, between Run summary and Recommendations):
- Per-(store, brand, type) breakdown — one row per visible combo
- Columns: `Store · Brand · Type · Capacity · On Hand Before · + Sent · On Hand After · % Fulfill Before · % Fulfill After`
- Respects Inventory filters (`invStoreFilter`, `invBrandFilter`, `invTypeFilter`)
- Sortable (click any column header) · Searchable (text filter) · Excel export
- Built by `buildImpactRows()` aggregating DATA `storeQty` (per-SKU) into per-(store, brand, type) sums, then layering recos `transferQty` onto `+ Sent`
- Default sort: fill % before ascending (most under-stocked first)
- Hides combos that are entirely zero (cap=0 AND on-hand=0 AND sent=0)
- Functions: `buildImpactRows`, `renderImpactReport`, `sortImpactReport`, `exportImpactReport`
- localStorage state: `S._impactSort` (in-memory only, not persisted)

**Recommendations card** (visible after run):
- Header includes plain-language description and a pill legend explaining `N lines` / `N units` / `before → after` / `📋 NAV` pills
- Per-store collapsed cards; each header pill shows total store on-hand `before → after`
- Table columns: `SKU (NAV No.) · Brand · Type · 90d vel · Store qty · After (green) · WH qty · QTY · Signals`

### Transfers Tab (current — shipped July 2026)
Store-to-store dead-stock reallocation. Mirror-image of replenishment: instead of WH → store, it moves sitting inventory from a low-rotation store to a store where the SKU (or brand+type) is actively selling.

**Card:** "Store-to-store transfer suggestions" · cloud-status pill in the header (`transfersCloudStatus`).

**Controls (row above results):**
- `transferMinDays` — number input, "Flag as sitting after N days without movement" (default 120, persisted `de_transfer_min_days`)
- `transferMatchLevel` — select: `both` (exact SKU first, then Brand+Type · default) · `sku` · `brandtype` (persisted `de_transfer_match`)
- `btnBuildTransfers` — `▶ Find transfers` runs `buildTransfers()`
- `btnExportTransfers` — `📋 Export` writes xlsx + logs ledger (disabled until results exist)

**Sitting-clock resolution (`buildLastInboundMap`):**
1. `S._transfersLatest` — pulled from Supabase view `replen_transfers_latest` (authoritative; direction-aware inbound date per `to_loc|sku`)
2. Fallback: `S.raw.inventory[].updated_at` joined via `itemMap[inventory_item_id].variantSku` — key `location_id|sku`
Ledger always wins when both are present. `daysSitting = floor((now − ms) / 86_400_000)`.

**Dead-source rule** (in `buildTransfers`):
`storeQty > 0` AND `velocity90 === 0` AND row is within source scope AND `daysSitting ≥ transferMinDays`. No clock evidence (no ledger row and no `updated_at`) → skip conservatively. All SKU join keys use `normSku(...).toLowerCase()` on BOTH producer and consumer sides — raw `d.sku` may carry a leading apostrophe (Supabase cache path stores `variant_sku` un-normalized).

**Filter scoping (hardened Jul 2026):** Brand/Type filters scope BOTH sources and destinations; the Store filter scopes **sources only** — destinations always consider all stores. (Otherwise a single-store filter makes matches structurally impossible.)

**Destination logic:**
- Exact-SKU: other stores with a DATA row for the same SKU, `velocity90 > 0`, not excepted, sorted by dest velocity desc
- Brand+Type: stores whose aggregate `sum(velocity90) for (brand,type)` > 0, not excepted for the source SKU, sorted by aggregate vel desc
- ONE shared `alloc()` closure implements the room math for both levels: `room = getEffectiveCap(sid,b,t) − onHand − committed`, where `committed` is pre-seeded with the current replenishment plan's `S.activeRecos` transferQty per bucket — transfers never re-allocate headroom replenishment already claimed, and the over-capacity tolerance applies identically to both engines.
- Suggestions store `matchType` (`'sku'`/`'brandtype'`) only; display labels derive from `TRANSFER_CONF_LABEL` at render/export time.
- `buildTransfers` is **async**: it awaits `S._transfersPullPromise` if a ledger pull is in flight (never silently builds from the updated_at fallback while the ledger is seconds away).

**Ledger — Supabase shared source of truth:**
- Table `replen_transfers(id, exported_at, from_loc, to_loc, variant_sku, qty, source, user_label)` — anon RLS (select/insert/update/delete); indexes on `(to_loc, variant_sku, exported_at desc)` and `(exported_at desc)`
- View `replen_transfers_latest` — `DISTINCT ON (to_loc, variant_sku) ... ORDER BY ... exported_at DESC` (one row per store+SKU), `security_invoker = true` (Supabase lint: SECURITY DEFINER views bypass RLS)
- Paginated pull MUST use `order=to_loc,variant_sku` — the view's unique pair. `variant_sku` alone repeats across stores → unstable offset pagination silently skips rows (same bug class as the replen_products `order=` rule)
- `logTransferExport(rows, source)` — non-blocking POST in 500-row batches. Self-guards (do not remove): **live-mode only** (`S.mode!=='live'` → return; sample plans carry synthetic SKUs with REAL store IDs), **session idempotency** via `S._ledgerLogged` Set keyed `source|to_loc|sku|qty` (re-exporting the same plan — full workbook or per-store — never appends duplicates / resets sitting-clocks; the Set is cleared when a NEW plan is built in `runRun`/`buildTransfers`), **no prompt** (reads `de_user_label` directly, defaults `'anonymous'` — never calls `getCurrentUserLabel()` which would pop a dialog mid-export). After writing it merges the payload into `S._transfersLatest` locally — no network re-pull.
- **Called from** `exportNAV` (source=`'replenishment'`, from_loc=`'WAREHOUSE'`) AND `exportTransfers` (source=`'transfer'`, from_loc=source store's location_id)
- Pulled after `processLiveData` (which sets `S._transfersCloudPulled=true` first so the tab-switch pull doesn't double-fetch) and lazily on first Transfers tab visit. The no-credentials early-return inside the pull resets `S._transfersCloudPulled=false` so a later visit retries (mirrors WH).

**Plan invalidation (`invalidateTransferPlan`):** clears `S._transferPlan`, wipes `transfersResult`/`transfersStatus`, disables `btnExportTransfers`. Called from `processLiveData` (DATA rebuilt), `handleInvFilter` (scope changed), and `saveTransferMinDays` (settings changed). A stale plan must never remain exportable — it would write phantom rows to the shared ledger.

**Persistence:** `de_transfer_min_days`, `de_transfer_match` (localStorage). In-memory only: `S._transfersLatest`, `S._transferPlan`, `S._transfersCloudPulled`, `S._transfersPullPromise`, `S._ledgerLogged`.

**Rendering:** `renderTransfersTable` caps the DOM at 500 rows (footer shows "Showing 500 of N — Export includes the full list"); uses the global `hesc()` escaper, not a local copy.

**Excel export (`exportTransfers`):** one sheet "Transfers" with columns `From Store · From Loc ID · To Store · To Loc ID · SKU · Brand · Type · Description · Qty · Days Sitting (source) · Dest 90d Vel · Confidence`. SKU pushed as `String(p.sku)` so `aoa_to_sheet` emits a text cell (no scientific notation, no post-hoc cell loop).

**Design notes:**
- The ledger date is "sent" (when the NAV file was exported), not "confirmed received" — leads true receipt by transit lag. Acceptable and clearly labeled.
- `S.raw.inventory[].updated_at` is direction-blind (any adjustment resets it — including a sale). The ledger overrides it precisely for that reason.
- Sample mode is hard-blocked from the ledger (`logTransferExport` returns early) and won't produce meaningful Transfers output — expected.

### Inventory Tab Table
Brand x Type grouped: `Brand | Type | On Hand | 90d Sales | Capacity | Fill% | [per-store cols]`

### Simulator Tab (current — commit 2302c55)
- Title card: `"Send-units simulator"` + badge `"Uses selected Inventory filters"`
- `simFilterDisplay` div: shows active filter summary text (store names | brands | types) — updated on Simulate click
- Number input `simUnitsInput` (min=0, max=999999) + `▶ Simulate` button calling `simulate()`
- `simResult` div: 8-card result grid rendered after simulate()
- NO filter dropdowns, NO slider

**simulate() logic:**
1. Reads `invStoreFilter` / `invBrandFilter` / `invTypeFilter` — errors if any are empty
2. Capacity: sums `S.CAP[storeId][brand][type]` with Set dedup on `storeId|brand|type`
3. WH qty: deduped by SKU (`r.sku || r.brand+'|'+r.type`) across all matching rows
4. Other stores critical: counts **distinct storeIds** (not row count) with fill < 30%
5. Units input validated: must be ≤ available WH qty

---

## Download Tables

| Table | Columns |
|---|---|
| Variants | Variant SKU, Brand, Type, Product Title, Price, Barcode |
| Inventory Levels | Variant SKU, Brand (vendor), Type (product_type), Store Name, Location ID, On Hand, Updated At |
| Orders 90d / Prior Year | Order ID, Date, Location ID, Store Name, Variant SKU, Qty Sold, Returned Qty, Net Qty, Description, Unit Price |
| Unified Join View | Store, City, Variant SKU, Brand, Type, Desc, On Hand, Cost, 90d Sales, [Prior Year 120d, Trend%] |
| WH Bin Contents | Variant SKU, WH Qty, Brand, Type, Description, Matched |

---

## Replenishment Logic

### Core Rule (gap-fill to tier target)
```
tierQty = getQtyForRow(d)              // velocity-tier TARGET stock level (1, 2, 3, 5...)
tierGap = max(0, tierQty - storeQty)   // missing units to reach target
room    = max(0, cap - storeQty)        // shelf capacity room
desiredQty = allowOverCap ? tierGap : min(tierGap, room)
Send IF: desiredQty > 0 AND whQty > 0 AND SKU in itemMap AND not isExcepted
```

**Key rule:** If a store already has >= tierQty on hand, do NOT send (regardless of capacity room).
Slow seller (tierQty=1) + storeQty=1 → send 0 ✅
Hot seller (tierQty=5) + storeQty=1 → send 4 (top up to target)
Gap-fill synthetic row (storeQty=0) + tierQty=1 → send 1

### Gap-Fill (DO NOT REMOVE from top of buildRecos)
Injects synthetic DATA rows for WH items at zero store stock. Only injects if store capacity > 0 for that brand+type.

### WH Pool Allocation (filterActive — DO NOT CHANGE)
Sequential by store priority. Store 1 filled first, remainder flows to Store 2, etc.

---

## Capacity
```javascript
// localStorage key: de_capacity_v1, format: [{sid, b, t, v}, ...]
initCap()                  // seeds defaults from DCAP
loadCapFromLocalStorage()  // called after initCap() in processLiveData
saveCapToLocalStorage()    // called on Export & Save + Import
importCapFromFile(input)   // Excel -> load -> save
```
Capacity table renders only combos in `catalogCombos` (from itemMap) — no phantom rows.

---

## WH Bin Contents

### NAV File Format
Columns: `Bin Code | Item No. | Available Qty. to Take`
- `Item No.` = Variant SKU (strip leading apostrophe)
- `Available Qty. to Take` = qty — sum per Item No. (multiple rows per SKU, one per bin)
- Field name must match exactly — NOT `Quantity`, NOT `Qty`

### Match Logic
Match against `S.raw.itemMap` (full 30k catalog), NOT `DATA` (stocked items only).
itemMap -> ~86% match rate. DATA -> ~1% match rate.

---

## State Object
```javascript
S = {
  raw: {
    locations[],        // ALWAYS array not {}
    inventory[],        // ALWAYS array not {}
    inventoryItems[],   // ALWAYS array not {}
    products[],
    orders[],
    ordersPY[],
    itemMap{},          // inventory_item_id -> {variantSku, brand, type, desc, cost, barcode}
    returns90d{},       // locId|sku -> returned qty (90d)
    returnsPY{}         // locId|sku -> returned qty (prior year)
  },
  wh: [],              // WH bin contents after upload
  CAP: {},             // S.CAP[storeId][brand][type] = capacity number
  ORIG: {},            // baseline for reset
  _pyLoading: false,   // guard flag (stub, kept for safety)
}
```

---

## Before Every File Delivery — Checklist
- [ ] No duplicate function definitions
- [ ] Key functions present: `loadCatalogFromCache`, `loadCatalogFromShopify`, `loadLiveData`, `loadFromCache` (alias), `loadAllData` (alias), `loadPriorYearOrders` (stub), `syncCatalogToSupabase`, `sbFetch`, `processLiveData`, `shopifyFetch`, `setProgress`, `buildRecos`, `renderStoreGrid`, `renderPriorityList`, `renderScopeBadges`, `exportNAV`, `exportRawTable`, `buildUnifiedRows`, `renderInspector`, `renderSchema`, `processWHRows`, `renderInv`, `renderCap`, `importCapFromFile`, `wipeAllCapacities`, `saveCapToLocalStorage`, `loadCapFromLocalStorage`, `downloadCapTemplate`, `downloadInspectorXLSX`, `simulate`, `getFD`, `getActiveBrands`, `getQtyForRow`, `saveStoreSelection`, `loadStoreSelection`, `saveVelTiers`, `loadVelTiers`, `toggleVelTierPanel`, `renderVelTiers`, `parseExcFile`, `loadExcFromLocalStorage`, `saveExcToLocalStorage`, `isExcepted`, `downloadExcTemplate`, `exportExcList`, `clearExcList`, `buildImpactRows`, `renderImpactReport`, `sortImpactReport`, `exportImpactReport`, `toggleTheme`, `buildTransfers`, `renderTransfersTable`, `exportTransfers`, `buildLastInboundMap`, `pullTransfersFromSupabase`, `logTransferExport`, `loadTransferMinDays`, `saveTransferMinDays`, `setTransfersCloudStatus`, `invalidateTransferPlan`
- [ ] File size ~400 KB (July 2026 baseline, post-Transfers tab) — sudden drop still indicates truncation
- [ ] `<div class="app">` count = 1
- [ ] No `since_id` in orders URLs
- [ ] No `inventory_item_id_gt` anywhere
- [ ] Inventory batch uses dynamic size: `Math.floor(250/Math.max(retailLocIds.length,1))`
- [ ] Supabase cache read has `&order=inventory_item_id` on paginated query
- [ ] `inventory_item_ids=` batch fetch present (1 occurrence in loadLiveData)
- [ ] `await loadLiveData(itemMap)` called from both 2A paths (2 occurrences)
- [ ] `Promise.all` in loadLiveData for parallel orders
- [ ] `step2BStatus` div in HTML
- [ ] `btn2B` display div in HTML
- [ ] `S.raw.returns90d` and `S.raw.returnsPY` stored
- [ ] sbFetch reads body as text before JSON.parse
- [ ] `setStep('2a',...)` and `setStep('2b',...)` in setStep labels
- [ ] btnWH uses pointer-events not disabled attr
- [ ] btnTest turns green on step 1 ok
- [ ] btn2B turns green on step 2b ok
- [ ] Dashboard tab has NO `dfStore` / `dfBrand` / `dfType` elements — only hint text + Units/$ toggle
- [ ] Simulator has NO `simStore` / `simBrand` / `simType` dropdowns / slider — only `simFilterDisplay`, `simUnitsInput`, Simulate button
- [ ] `getFD()` reads only `invStoreFilter` / `invBrandFilter` / `invTypeFilter` — no Dashboard fallback
- [ ] `populateDashFilters`, `populateSimSelects`, `syncInvToDash`, `syncDashToInv`, `applySettings`, `savePriority` are ABSENT
- [ ] Tab DOM order: `tab-connect`, `tab-inventory`, `tab-dashboard`, `tab-capacity`, `tab-exceptions`, `tab-stores`, `tab-transfers` (Settings hidden)
- [ ] `panel-exceptions` present with `excFile` upload + `excCount` + Download/Export/Clear buttons
- [ ] `panel-transfers` present with `transferMinDays` + `transferMatchLevel` + `btnBuildTransfers` + `btnExportTransfers` + `transfersCloudStatus` + `transfersStatus` + `transfersResult` (no "Coming soon" placeholder)
- [ ] `showTab('transfers')` hydrates the input via `loadTransferMinDays()` and lazily pulls `replen_transfers_latest` once (guarded by `S._transfersCloudPulled`)
- [ ] `exportNAV` fires `logTransferExport(...,'replenishment')` after `XLSX.writeFile` — feeds sitting-clock
- [ ] `exportTransfers` fires `logTransferExport(...,'transfer')` after `XLSX.writeFile`
- [ ] `processLiveData` calls `pullTransfersFromSupabase()` non-blocking after `pullExceptionsFromSupabase`, sets `S._transfersCloudPulled=true` first, and calls `invalidateTransferPlan()`
- [ ] `logTransferExport` self-guards intact: `S.mode!=='live'` early-return · `S._ledgerLogged` dedup Set · no `getCurrentUserLabel()` call (reads `de_user_label` directly)
- [ ] Ledger pull uses `order=to_loc,variant_sku` (unique pair — NEVER `variant_sku` alone) and resets `S._transfersCloudPulled=false` in its no-credentials branch
- [ ] `S._ledgerLogged=new Set()` reset present in BOTH `runRun()` and `buildTransfers()` (new plan → may log again)
- [ ] `invalidateTransferPlan` called from `handleInvFilter` and `saveTransferMinDays`
- [ ] `buildTransfers` uses `getEffectiveCap` (not raw `S.CAP`) and pre-seeds `committed` from `S.activeRecos`
- [ ] `--card` CSS token defined in BOTH `:root/[data-theme="light"]` and `[data-theme="dark"]` blocks
- [ ] Boot line calls `loadTransferMinDays()` (guarded `typeof ==='function'`) at end of the buildSample+load hooks chain
- [ ] `replen_transfers` table + `replen_transfers_latest` view + anon RLS (select/insert/update/delete) exist in Supabase
- [ ] `S.exceptions` is a `Set` (not array) initialized in S
- [ ] `isExcepted(storeId, sku)` check at the TOP of the `DATA.forEach` in `buildRecos`
- [ ] `velTierPanel` toggleable in Store Selection tab; `S.velTiers` defaulted from `VEL_TIERS_DEFAULT`
- [ ] `priorityListWrap` rendered after store grid; `renderPriorityList()` called from `renderStoreGrid()` and `syncPriority()`
- [ ] `scopeBadges` populated by `renderScopeBadges()` showing stores · brands · types counts
- [ ] Capacity tab top section: capacity matrix upload card with `downloadCapTemplate` button
- [ ] `loadCapFromLocalStorage()` + `applyCapToData()` called in `processLiveData` AFTER S.CAP rebuild
- [ ] `loadVelTiers()` + `loadExcFromLocalStorage()` + `loadStoreSelection()` + `loadWHSettings()` + `loadWHSource()` called in both boot path (after `buildSample`) and live-data path (in `processLiveData`)
- [ ] Settings tab ONLY contains NAV export settings card (no Data source / Store priority / Replenishment rules / Capacity upload)
- [ ] localStorage keys: `de_capacity_v1`, `de_capacity_saved_at`, `de_exceptions_v1`, `de_store_sel_set`, `de_store_sel_order`, `de_priority_mode`, `de_vel_tiers`, `de_overcap_pct`, `de_wh_source`, `de_wh_col_map`, `de_wh_filter_locations`, `de_wh_filter_zones`, `de_wh_all_locations`, `de_wh_all_zones`, `de_wh_all_sheets`, `de_wh_all_columns`, `de_user_label`, `de_vel_panel_open`, `de_transfer_min_days`, `de_transfer_match`, plus existing `de_inv_*_filter`

### Step 4A/4B + WH source toggle invariants
- [ ] WH source segmented toggle present (`#whSrcBtnUpload`, `#whSrcBtnSupabase`) with `selectWHSource(target)` handlers
- [ ] `S.whSource` persisted in `de_wh_source`; defaults to `'upload'`
- [ ] `S.whRaw` carries per-bin granularity with `location_code` + `zone_code` populated; `S.wh` is SKU-aggregated
- [ ] `replen_wh_bins` schema includes `location_code` + `zone_code` columns (DDL ran)
- [ ] `pushWHToSupabase` writes both columns; `pullWHFromSupabase` reads them
- [ ] `processWHRows` RESETS every `DATA[].whQty=0` before applying new `S.wh` values (prevents stale residue when filter narrows)
- [ ] `pullFromInventoryAvailable` includes `?available_qty=gt.0` + `&order=item_no` + server-side `location_code=in.(...)` filter
- [ ] `refreshWHAfterStep2B()` routes to the correct path based on `S.whSource` (both 4A and 4B paths covered)
- [ ] `showTab('connect')` checks `S.whSource==='upload'` before firing `pullWHFromSupabase` (otherwise overwrites 3B data)
- [ ] Unified `#whOverviewPanel` swaps content via neutral DOM IDs (`whOverviewTitle`, `whOverviewSubtitle`, `whOverviewQtyHeader`, `whOverviewBody`, `whOverviewFoot`, `whOverviewNote`) — no separate panels per source
- [ ] Refresh functions `refreshWHUploadOverview`/`refreshInvAvailOverview` guard with `if(S.whSource !== expected) return` on non-manual calls
- [ ] Column-mapping section in Upload Settings hides when 4B is active (`updateWHSourcePill` toggles `#whColMappingSection`)
- [ ] Raw Data Inspector exposes "⚠ WH SKUs without Shopify match" option, auto-shown when `S.wh.filter(w=>!w.matched).length > 0`

### Replenishment math invariants (per current `buildRecos`)
- [ ] Aggregate room check: `aggOnHand[bucket]` pre-computed from real DATA, `aggUsed[bucket]` accumulates during loop. `aggRoom = effectiveCap - baseOnHand - aggUsed`. Multiple SKUs in same (store, brand, type) share one budget
- [ ] Synthetic gap-fill rows process FIRST: `DATA = injected.concat(realRows)` after gap-fill injection (new-to-store wins aggregate room)
- [ ] No brand+type fallback for `wh.whQty` on real rows — if SKU not in `S.wh`, `whQty=0` (prevents phantom recos eating budget)
- [ ] `getEffectiveCap(sid, b, t) = base × (1 + S.overCapPct/100)` — over-cap tolerance configurable in Send Qty Rules
- [ ] `tierGap = max(0, tierQty - storeQty)` (NOT `tierQty` alone) — top up to target, don't blindly fill

### Inventory tab invariants
- [ ] `renderInv()` uses memoization via `_invRenderCache` keyed on data/cap/wh version tokens + filter selections + forecast model
- [ ] Single-pass aggregation loop builds `perStore` + `aggByBT` in one traversal (replaces 5+ separate DATA scans)
- [ ] Version-bump helpers exist: `bumpDataVersion()`, `bumpCapVersion()`, `bumpWhVersion()`
- [ ] SKU detail table: WH column is leftmost; TOTAL row sits under header with `class="sticky-total"`; first 3 columns (WH, Brand, Type) are sticky-left
- [ ] `.table-scroll` wrapper class used on inventory/capacity/store-selection tables (max-height:75vh, overflow:auto, sticky thead via CSS)

---

## Known Issues (Open)
- **Capacity bucket over-allocation**: When N SKUs share a (brand,type) bucket, each row's
  `room = cap - storeQty` is computed independently — combined sends can exceed bucket cap.
  Fix path: track running consumption inside `buildRecos` per (storeId, brand, type) Map.
  Workaround until fixed: leave "Allow over-capacity sends" OFF.

---

## Git Workflow
- `master` is the shipping branch; in-flight work uses worktrees under `.claude/worktrees/`
- Commit style: imperative subject (e.g. "Fix capacity persistence on reload")
- Push only when user says "push" / "commit pending" — never auto-push
- Stale `CLAUDE.md` may exist in old worktrees; the root copy is authoritative

---

## Common Bugs & Root Causes

| Symptom | Root Cause | Fix |
|---|---|---|
| All brands Unknown | Using `products.json` pagination | Use variants-first + products-by-ID-batch |
| Only 3 order dates returned | `since_id` on orders — ignores date filters on pages 2+ | Use `created_at_max` cursor |
| Inventory shows 250 rows per store | `inventory_item_id_gt` cursor cycling — proxy strips it | Use `inventory_item_ids=` batch by known IDs |
| Inventory shows exactly N×250 rows | Old `pg<N` cap hit | Batch-by-ID approach has no cap |
| SKUs in Shopify missing from inventory download | `limit=250` truncation: 100 items × 7 stores = 700 potential records but limit caps at 250 silently | Batch size = `floor(250 / storeCount)` — guarantees records per call never exceeds limit |
| SKUs in Products download but missing from inventory | Supabase cache pagination without `order=` — rows skipped at page boundaries due to unstable sort | Add `&order=inventory_item_id` to Supabase query |
| Prior year fires twice | Button had multiple unlock calls + no guard | `S._pyLoading` guard (now stub since PY in loadLiveData) |
| Supabase "Unexpected end of JSON" | `r.json()` on empty 200 body | Use text-first pattern |
| Supabase 403 | Wrong key — `sb_publishable_` instead of anon JWT | Use `eyJ...` anon public key |
| WH 1% match rate | Matching against DATA (stocked only) | Match against itemMap (full catalog) |
| Capacity phantom rows | Using BRANDS × TYPES instead of catalogCombos | Drive from itemMap catalogCombos |
| R006/R009/R014 show 0 inventory | Was: `inventory_item_id_gt` cycling. Fixed by batch-by-ID | Confirmed fixed — all stores now show real counts |
| Dashboard filters out of sync with Inventory | Separate dfStore/dfBrand/dfType elements required sync functions | Removed Dashboard filters entirely — filtering centralized to Inventory tab |
| Simulator using wrong filter values | simStore/simBrand/simType were independent DOM elements | Removed; simulate() now reads directly from invStoreFilter/invBrandFilter/invTypeFilter |
| Capacity double-counting in simulator | Summing row.cap across all rows (one per SKU) | Use Set dedup on storeId\|brand\|type, read from S.CAP authoritatively |
| `simulate()` errors on removed DOM elements | Old code still referenced simStore/simBrand/simSlider after HTML removal | Fully rewrote simulate() to use Inventory filters + text input |
| `getFD()` references removed dfStore elements | Dashboard filter elements removed but getFD() still had fallback reads | Replaced with Inventory-only reads |
| Capacity edits lost on page reload | `processLiveData` was rebuilding `S.CAP` from defaults but never overlaying saved values | Call `loadCapFromLocalStorage()` + `applyCapToData()` after S.CAP rebuild |
| Capacity not auto-saving | `onCapChange` only updated memory; localStorage write was gated to Export/Save button | Added `saveCapToLocalStorage()` to `onCapChange` |
| Reset capacity values come back on reload | `resetCap()` only reset memory, never cleared localStorage | Reset now removes `de_capacity_v1` + `de_capacity_saved_at` |
| `% full` in store priority list doesn't match selected filters | `S.storeStats.pct` summed all DATA rows + summed `b.cap` across SKU rows (inflated denominator) | Filter rows by `invBrands`/`invTypes`; dedup capacity by (brand,type) Set reading from `S.CAP` |
| Replenishment ignores Inventory brand/type filters | `buildRecos` iterated all DATA without filter | Read `getFilterValues('invBrandFilter'/'invTypeFilter')` and skip non-matching rows at top of loop |
| Exception list cleared on page reload | No persistence layer | `de_exceptions_v1` JSON array of `storeId\|sku` keys, loaded on boot + after `processLiveData` |
| Variant SKU column reformatted by Excel (scientific notation) | Cells written as numbers, Excel auto-converts | Set cell `t='s'` and stringify value when writing template/export |
| Settings tab still shows old Data source / Priority / Replenishment rules cards | Cards not removed during consolidation | Removed; functionality lives in Store Selection tab (priority dropdown, ⚙ Send qty rules); orphaned `applySettings`/`savePriority` deleted |
| Shopify load dies on transient 502 mid-pagination | `shopifyFetch` only retried 429, threw on 5xx | 8-attempt retry on 5xx + network errors + non-JSON 200 bodies with exponential backoff (1.5s → 30s) |
| Run summary appears static when changing priority mode | Aggregate KPIs (total units, NAV lines) are WH-bottlenecked → identical regardless of priority order. UI gave no visual hint a re-run happened | Add `· Priority: X · HH:MM:SS` stamp to header + per-store breakdown table (priority effect surfaces visibly per-store) |
| NAV Transfer Order Preview unreadable in dark mode | Hardcoded hex colors (`#F0F6FC`, `#1F4E79`, `#EBF3FB`) bypassed theme tokens | Port all colors to `var(--surface)` / `var(--surface-2)` / `var(--navy)` / `var(--blue-bg)` etc. |
| Capacity tab had local brand/type filters compounding with Inventory filters | Legacy `capFType` / `capFBrand` dropdowns left in toolbar | Hide via `display:none` with default values; renderCap continues to read Inventory filters |
| Top understocked combos showed only dead/discontinued combos | Filter was `c.cap > 0` only — sort ascending by fill% put 0% / 0qty / 0wh at top | Split into TWO widgets: `Replenish-ready` (cap>0, wh>0, vel>0) and `Stockout — WH empty` (cap>0, wh=0, vel>0) |
| SKU sent again even when store already has it in stock | `desiredQty = min(tierQty, room)` — used full tier qty instead of the gap to tier target. A store with 1 unit and tier=1 still got another unit because shelf room>0 | `desiredQty = min(tierQty - storeQty, room)` — only ship the missing units to reach the tier target. tier=1 + storeQty=1 → send 0; tier=5 + storeQty=1 → send 4 |
| Capacity Excel export had phantom rows (e.g. ABATTA × BACKPACK / CLIPS / GOGGLES that don't exist in catalog) | `exportCapXLSX` iterated `BRANDS × TYPES` Cartesian product instead of catalog combos | Use the same `catalogCombos` source as `renderCap` (driven from `S.raw.itemMap`); fall back to non-zero S.CAP cells if no catalog loaded yet |
| Capacity total on screen disagrees with re-exported Excel (e.g. footer says 28,921 but Excel exports 29,000+) | Two distinct causes: (1) Cells where brand has a value in `S.CAP` but isn't in `S.STORE_BRANDS[storeId]` — table shows "+" placeholder, footer skips, export emits. (2) `(brand, type)` pairs not in live catalog — filtered from `catalogCombos` so excluded from table view but still in `S.CAP` | (1) `renderCap` now self-heals at render time: any brand with a non-zero value in `S.CAP` is auto-added to `S.STORE_BRANDS[sid]` and to `BRANDS` if missing. (2) Footer shows BOTH `visible total` and `stored grand total` (true sum across `S.CAP`); when they differ, an amber pill surfaces hidden units + count of orphaned combos with a `show` button (`showOrphanedCapacity`) listing the offending pairs |
| WH TOTAL in SKU detail > what's actually in `S.wh` after switching Location filter | `processWHRows` only wrote new `d.whQty` values for SKUs that matched the new `S.wh`; SKUs that fell out of scope kept stale values from the previous load | Reset `DATA.forEach(d=>{d.whQty=0;})` FIRST inside `processWHRows`, THEN apply new values. Single source of truth: only SKUs in the current `S.wh` have non-zero `whQty` |
| Run replenishment produces empty plan immediately after Clear results | `runRun()` called `filterActive()` against the wiped `S.recos` — never rebuilt it. Only filter changes were rebuilding `S.recos` via `handleInvFilter` | `runRun` calls `buildRecos()` first so the Run is stateless w.r.t. prior clears |
| All brands selected → only 2 Ray-Ban units ship despite 233 units of room | Per-SKU `room = d.cap - d.storeQty` check let any one SKU "see" full bucket headroom; brand+type aggregate was never enforced | Pre-compute `aggOnHand[bucket]` from real DATA, track `aggUsed[bucket]` during the loop, `aggRoom = effectiveCap - baseOnHand - aggUsed`. Multiple SKUs in same bucket share one budget |
| Gap-fill ships 0 new-to-store SKUs even when room exists | Synthetic gap-fill rows were appended at the END of DATA; under-target real rows iterated first and ate `aggUsed` budget | `DATA = injected.concat(realRows)` — synthetic rows process FIRST so new-to-store wins aggregate room |
| Real DATA row with no WH-match inflates `aggUsed` with phantom recos | `whQty` fallback used the brand+type SUM when SKU not in `S.wh`, then `filterActive` dropped the phantom rec at WH pool stage — budget already wasted | Drop the brand+type fallback: real rows w/o exact SKU match get `whQty=0`, no reco pushed, no budget eaten |
| Switching to Connect tab in 3B mode silently reloads stale 3A data | `showTab('connect')` fired `pullWHFromSupabase()` (reads `replen_wh_bins` = 3A source) unconditionally; `_whCloudPulled` flag never set in the supabase boot branch | Gate by `S.whSource==='upload'` in `showTab('connect')`. The 4B path is handled separately by `refreshLiveConnectCloudLabels` |
| Inventory tab `% full` and "X recs" pill stale after Clear results | `clearReplenishmentResults` wiped `S.recos` but `S.storeStats[id].recos` was last written during `buildRecos` and kept its old count | `clearReplenishmentResults` zeroes `S.storeStats[id].recos` for every store before re-rendering. `pct` is left alone since it reflects inventory state, not run output |
| Inventory tab filter row orphans columns awkwardly at ~1100px | Uncapped `flex-wrap` with `min-width:180px` allowed two-row reflows with the "Actions" column dangling | New `.inv-filter-row` uses `grid-template-columns: repeat(auto-fit, minmax(180px, 1fr))`. Mobile breakpoint at 600px stacks vertically. |
| Sticky headers don't pin on tall tables in Inventory / Capacity / Store Selection | Tables were inside `overflow-x:auto` wrappers — `position:sticky` only sticks within nearest scroll ancestor, which existed but had no `max-height` so it never engaged | New `.table-scroll` class: `overflow:auto; max-height:75vh; position:relative;`. `<thead> th` gets `position:sticky; top:0`. SKU detail's TOTAL row gets a second sticky row at `top:34px` |
| WH SKUs not in Shopify catalog silently ignored | No surfaced report of unmatched SKUs — operator has no way to fix the mapping | New "⚠ WH SKUs without Shopify match" option in Raw Data Inspector. Lists `S.wh.filter(w=>!w.matched)` with bin/loc/zone + heuristic root-cause (whitespace, apostrophe, missing catalog entry). Download to XLSX wired |
| Bumping Shopify API version in the app alone broke every API call with `403 Endpoint not allowed` | The proxy has its own allowlist hardcoding `2024-01`. App + proxy versions are coupled; updating app alone breaks both | App + proxy must be updated together. Proxy source is NOT in this repo (separate Vercel project `de-shopify-proxy`) — find/update it first, THEN bump the app, THEN commit both together |
| Transient 404 mid-Shopify-pagination (page 1 worked, page 2 returns 404) | Not necessarily API version sunset — often a brief Shopify or proxy hiccup. `shopifyFetch` doesn't currently retry on 404 | Try the operation again before assuming the version is dead. If reproducible, capture the failing URL + response body from DevTools Network tab to diagnose properly |
| Both WH overview panels (3A and 3B) showed at once during source switches | Two separate panels (`#whUploadOverview` + `#whInvAvailOverview`) raced — async refresh callbacks could both fire before the toggle hid one | Consolidated into a single `#whOverviewPanel` whose content swaps based on `S.whSource`. Refresh functions guard with `if(S.whSource !== expected) return;` to prevent cross-mode rendering |

---

## File Management
- Working file: `DE_Replenishment.html` in this project directory
- Use the Edit tool for targeted replacements; Read the file before editing
- Always read the actual code before diagnosing — never speculate
- For large replacements: verify the old_string is unique before applying
- Credentials stored in `config.json` (loaded on startup); never hardcode keys in source

---

## User Preferences
- Address user as **Mau Rock**
- Communicate in English
- No emoji in code comments
- Read the actual code before diagnosing — never speculate
- Fix root cause, not symptoms
- Checklist before every file delivery
- **MQ** prefix on a request = ask 3-5 clarifying questions before coding (scope, behavior, UI placement, defaults, edge cases). Wait for confirmation before executing.
- **MC** prefix = respond ultra-concisely; minimum tokens, no preamble
