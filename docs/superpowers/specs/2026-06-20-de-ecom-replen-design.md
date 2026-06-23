# DE Ecom Replen — Design Spec

**Date:** 2026-06-20
**Author:** Mau Rock (with Claude)
**Status:** Approved design → ready for implementation plan

---

## 1. Purpose

Recreate the Designer Eyes **E-Commerce Inventory Report** (today an Excel export from the SQL
data warehouse) as an interactive, browser-based **demand-forecasting and purchasing tool**.

The tool answers three questions per SKU:

1. **What will sell?** — forecast e-commerce demand from trailing sales windows (trend now,
   seasonality later).
2. **Can I cover it?** — compare forecast demand over the replenishment horizon against
   on-hand + on-order stock.
3. **What should I buy?** — suggest a purchase quantity per item/vendor that covers a
   configurable number of weeks of demand, accounting for the full pipeline time (supplier
   lead time **plus** inbound transit/check-in to the marketplace fulfillment centers).

This is a **separate problem** from the existing `DE_Replenishment.html` (retail store
transfers). It ships as its own standalone single-file tool.

---

## 2. Scope & decomposition

The full vision is three independent systems. **This spec covers system #3 only** (the app),
built against a defined Supabase contract (#2). System #1 (ingestion) is out of scope and
owned by the data-warehouse/Microsoft/python jobs.

| # | System | Owner | In this spec? |
|---|--------|-------|---------------|
| 1 | Ingestion pipelines: NAV/SQL DWH → Supabase; **ERP open purchase orders (SQL Server → python → Task Scheduler → Supabase)**; SaaS demand sources (GA4, Keepa, Vendorati, Helium 10, SmartScout, Sellerboard) → Supabase | DWH / python / Microsoft | No (external) |

> **"SQL Server" = the ERP databases.** Designer Eyes' ERP (NAV) lives on an in-house SQL
> Server. The open-orders feed (and any other ERP-sourced data) is pulled by a python job on
> Windows Task Scheduler and pushed to Supabase. The app only reads the resulting Supabase
> table — it never connects to SQL Server directly.
| 2 | Supabase data contract (tables the app reads) | Defined here; bootstrapped by Claude | Schema only |
| 3 | **DE Ecom Replen app** (single-file HTML) | This spec | **Yes** |

**Build order:** app first. Claude does a **one-time `All_Inventory` Excel → Supabase load**
to bootstrap real data. The python/Microsoft sync replaces that load later — same tables, no
app change.

---

## 3. Data contract (Supabase)

Project: `zoqihptkwurpqenpjgfw` (shared with DE Replen Pro).

### 3.1 `mau_main_items` — exists (product master, 337,276 rows) — **THE BASELINE**
PK `item_no`. Columns: `item_no, upc, no_2, description_3, description, model,
product_color_code, size, product_group_code, item_category_code, brand_name, synced_at`.

**This is the base / row driver.** Every other feed left-joins onto it by `item_no`. The
item universe of the tool is the master; e-com stock/sales/forecast/open-orders are
*attached* to master items. Items in the master with no e-com data simply show blank e-com
columns (and no forecast/PO).

**Data-quality rule:** every e-com item *should* exist in the master. E-com items absent from
`mau_main_items` (e.g. item `367906`) are treated as a defect: they are **excluded from the
working set** (strict left-join from master) **and surfaced in the "Unmatched e-com items"
report** (§6) so they get added to the master upstream. Nothing is silently merged in.

### 3.2 `ecom_inventory` — NEW (the report data)
PK `item_no`. Mirrors the ~98 ECOM columns from the Excel `ECOM` sheet, normalized to
snake_case. Bootstrapped from `All_Inventory_20260618.xlsx`; later overwritten by the
external sync. Key column groups:

- **Identity / listing:** `item_no` (Excel "Item"), `item_ec`, `item_vsc`, `asin`, `upc`,
  `account_destination`, `ecz_listing_status`, `vcs_listing_status`, `exclusivity`.
- **Attributes:** `brand_name`, `gender`, `item_category`, `product_type`, `vendor_name`.
- **NAV ecom stock:** `ecom_pickable_bins` (+ value), `pickable_sunrise_inventory` (+ value),
  `warehouse_transfers`.
- **Amazon ECZ stock:** `ecz_active`, `ecz_inbound`, `ecz_fba_total`, `ecz_awd_available`,
  `ecz_awd_inbound`, `ecz_awd_total`, `ecz_fba_awd_total`, reserved/unfulfillable/stranded/
  removed/rtv (+ their value columns).
- **Amazon VCS stock:** same field set prefixed `vcs_`.
- **Retail / 3PL:** `retail_pickable_bins` (+ value), `retail_on_hand_qty` (+ value/damaged),
  `tpl_inventory_total` (+ value).
- **On-order (in-transit):** `ecom_oor` (PO Qty − Qty Received for ECOM POs).
- **Sales windows (units, cumulative trailing):** `vcs_7d/15d/30d/60d/90d` (+ `vcs_90d_value`),
  `ecz_7d/15d/30d/60d/90d` (+ `ecz_90d_value`).
- **Coverage indicators (as exported):** `vcs_coverage_indicator`, `ecz_coverage_indicator`.
- **On-hand rollups:** `ecom_on_hand_qty` (+ value/damaged), `total_on_hand_qty` (+ value).
- **Financials:** `direct_cost`, `us_map_price`, `us_min_price`, `us_max_price`.
- **Dates:** `amz_refreshing_date`, `report_generation_date`, `synced_at`.

> Excel serial dates (e.g. `46190.96…`) are converted to ISO timestamps on load.

### 3.2b `v_replen_base` — NEW (server-side join view) + smart load
Because the master is 337,276 rows, the app does **not** load it all into the browser. A
Postgres **view** does the join server-side; the app loads only what it needs.

```sql
create or replace view public.v_replen_base as
select m.item_no, m.upc, m.brand_name, m.model, m.product_color_code, m.size,
       m.product_group_code, m.item_category_code, m.description,
       e.*,                               -- all ecom_inventory columns (e.item_no may be null)
       oo.vendor_open, oo.next_eta,        -- aggregated open orders
       (e.item_no is not null) as has_ecom
from public.mau_main_items m
left join public.ecom_inventory e on e.item_no = m.item_no
left join (
  select item_no, sum(qty_outstanding) as vendor_open, min(expected_receipt_date) as next_eta
  from public.ecom_open_orders group by item_no
) oo on oo.item_no = m.item_no;
```

**Load strategy (smart):**
- **Actionable working set (loaded fully, ~20–35k):** rows where `has_ecom` is true OR
  `vendor_open > 0` — i.e. anything with sales, stock, or an open PO. Forecast/coverage/PO
  run only here. Query: `v_replen_base?or=(has_ecom.eq.true,vendor_open.gt.0)`.
- **Catalog browse (server-side, ~300k):** master items with no e-com data are **not**
  preloaded. A "Show non-selling catalog items" toggle browses them via server-side
  paginated/filtered queries on `v_replen_base` (no forecast needed — they have no demand).
- Both paths read `&order=item_no` for stable pagination (DE Replen Pro lesson).

### 3.3 `ecom_sales_history` — NEW (empty for now)
Time series stub for genuine seasonality. PK `(item_no, source, period_date)`.
Columns: `item_no, source (ga4|keepa|helium10|smartscout|sellerboard|vendorati|amazon),
period_date, units, revenue, synced_at`. The SaaS connectors populate it later; the app's
forecast auto-upgrades to seasonality once enough history exists. **No app feature depends on
it being populated at launch.**

### 3.5 `ecom_open_orders` — NEW (open vendor POs, in-transit from supplier)
Sourced from the **ERP on SQL Server** by a python job on Task Scheduler → Supabase. Gives
line-level visibility into incoming vendor stock (richer than the flat `ecom_oor` rollup).
PK `(po_no, po_line)`; indexed on `item_no`. Columns: `po_no, po_line, item_no, vendor_name,
qty_ordered, qty_received, qty_outstanding, order_date, expected_receipt_date, destination
(amazon|walmart|de_wh|…), status, synced_at`.

- The app reads this table when present and aggregates `qty_outstanding` per `item_no` as the
  authoritative **vendor open-order** quantity, with `expected_receipt_date` enabling
  time-phased coverage (§5).
- **Graceful fallback:** if the table is empty/absent, the app falls back to
  `ecom_inventory.ecom_oor`. No app feature hard-depends on it at launch.

### 3.4 `app_settings` — exists (`key, value jsonb, updated_at, updated_by`)
Optional cloud sync of **shared, non-secret** default parameters (forecast model, lead times,
coverage targets). Secrets never go here — they stay in the browser.

---

## 4. Forecast engine (in-browser)

Demand signal = **e-commerce units = ECZ + VCS** (the only sales columns available). The
trailing windows are cumulative (`30d ≥ 15d ≥ 7d`).

**Daily demand rate (per item):**
- `rate(w) = (ecz_w + vcs_w) / days(w)` for w ∈ {7,15,30,60,90}.

**Forecast model (default = Smoothed blend):**
- `daily = 0.5·rate(30) + 0.3·rate(60) + 0.2·rate(90)`
- Alternatives selectable in Setup: **Recent** (`rate(30)`), **Conservative** (`min` of
  30/60/90 rates).
- **Trend multiplier (toggle, default on):** `trend = rate(30) / rate(90)`; classify
  Accelerating (>1.1) / Steady / Decelerating (<0.9); optionally scale `daily` by a clamped
  `trend` (clamp 0.5–2.0).
- `weekly_demand = daily × 7`.

**Seasonality:** deferred. Surface a "needs history" state; auto-activate when
`ecom_sales_history` has ≥ 1 comparable prior period. Not faked from a single snapshot.

---

## 5. Coverage & suggested purchase

**Available to cover demand (default = all sellable ecom pools, no 3PL).** Counts only
stock that is *sellable now*, and must NOT overlap with the on-order pool below:
```
available = ecom_pickable_bins
          + pickable_sunrise_inventory
          + ecz_fba_total              (ECZ FBA on-hand, excl. AWD)
          + ecz_awd_available          (AWD units ready, excl. AWD inbound)
          + vcs_fba_total
          + vcs_awd_available
# excluded: reserved, damaged, unfulfillable, stranded, retail, 3PL (3PL toggle, default OFF)
```

**On-order / in pipeline (units arriving, not yet sellable):**
```
vendor_open = Σ ecom_open_orders.qty_outstanding by item_no   # ERP feed; fallback ecom_oor
marketplace_inbound = ecz_inbound + ecz_awd_inbound + vcs_inbound + vcs_awd_inbound
on_order = vendor_open + marketplace_inbound
```
- **Vendor open orders** = POs already placed and in transit from the supplier. Counting them
  prevents re-ordering stock that's already coming. Prefer the line-level `ecom_open_orders`
  feed; fall back to the flat `ecom_oor` when the feed is absent.
- **Time-phasing (when `expected_receipt_date` is present):** a PO expected to arrive *before*
  the projected stockout counts toward covering near-term demand and clears the stockout flag;
  a PO arriving *after* the horizon still reduces `reorder_need` but does **not** clear a
  near-term stockout risk. Without dates, all `on_order` is treated as covering the horizon
  (simple mode).

> **Double-count guard:** `*_fba_awd_total` and `*_awd_total` are roll-ups that bundle AWD
> *inbound* together with available stock. The split above deliberately uses the leaf fields
> (`fba_total`, `awd_available`, `*_inbound`) so a unit is counted once — never in both
> `available` and `on_order`. The exact AWD roll-up definition will be confirmed against the
> bootstrapped data during implementation.

**Two-leg lead time (the full pipeline before a PO is sellable):**
```
total_lead_weeks = supplier_lead_weeks      # PO placed → received at DE warehouse
                 + fc_inbound_weeks          # DE WH → marketplace FC available-to-sell
```
- `supplier_lead_weeks`: default **6**, adjustable globally and **per vendor**.
- `fc_inbound_weeks`: transit + check-in to destination FC, adjustable globally and
  **per destination** (Amazon, Walmart, …). Default **2**.

**Reorder math:**
```
weeks_of_supply = (available + on_order) / weekly_demand        # ∞ if weekly_demand = 0
horizon_weeks   = total_lead_weeks + target_weeks + safety_weeks
reorder_need    = weekly_demand × horizon_weeks − (available + on_order)
suggested_po    = max(0, ceil(reorder_need))                    # optional case-pack rounding
```
- `target_weeks`: default **4**, configurable.
- `safety_weeks`: default **2**, configurable.

**Flags:**
- **Stockout risk:** `weeks_of_supply < total_lead_weeks` (will run out before a reorder
  could arrive).
- **Overstock:** `weeks_of_supply > horizon_weeks + target_weeks` (informational).
- **No demand:** `weekly_demand = 0` → never suggest a PO; list under "no sales".

**PO output:** rows where `suggested_po > 0`, grouped by `vendor_name`; columns Item, ASIN,
Brand, Description, weekly demand, weeks of supply, available, on-order, suggested qty,
direct cost, extended cost. Export to Excel/CSV.

---

## 6. UI structure

Single-file `DE_Ecom_Replen.html`, DE Replen Pro conventions: anti-flash `data-theme` boot
script, glass topbar, sticky tabs, Inter + JetBrains Mono, CSS tokens (no hardcoded hex),
`tabular-nums`, focus-visible rings, reduced-motion, skip link. CDN: SheetJS (export +
one-time parse), optionally Chart.js (dashboard).

**Tabs:**
1. **Setup** — credentials + parameters (see §7).
2. **Report** — grid over `v_replen_base` (master ⟕ e-com). Defaults to the **actionable
   working set**; a **"Show non-selling catalog items"** toggle adds the ~300k master-only
   rows via server-side pagination. Filter by brand, vendor, category, product type, listing
   status, account destination; text search; column visibility presets; export current view.
   Sticky header, `.table-scroll`. Includes an inspector dropdown with a **"⚠ Unmatched
   e-com items"** view: e-com SKUs present in `ecom_inventory` but **absent from
   `mau_main_items`** (query: `ecom_inventory` left-join master where master `item_no` is
   null). Shows item_no, ASIN, brand, sales, on-hand; downloadable to Excel; auto-shown when
   count > 0. This is the data-quality report that drives fixing the master upstream.
3. **Forecast & Coverage** — per item: rates by window, model output, trend pill,
   weekly demand, available, on-order (split: vendor-open vs marketplace-inbound), nearest
   open-PO ETA, weeks of supply, coverage status. Honors filters. Expandable row shows the
   item's open `ecom_open_orders` lines (PO #, qty outstanding, expected receipt, destination).
4. **Suggested Purchase** — reorder list grouped by vendor (§5). Editable qty before export.
5. **Dashboard** — KPIs: # at stockout risk, # to reorder, total reorder units & value,
   coverage distribution, top vendors by reorder value. (Chart.js.)

Load flow: Setup → Test connection → Load from Supabase (`ecom_inventory` paginated 1000/row
with `&order=item_no`, enrich from `mau_main_items`) → compute forecast/coverage client-side
→ tabs populate.

---

## 7. Setup tab — secrets persisted in the browser

**Persistence:** all values in `localStorage` (browser-persistent, per the existing pattern).
Credentials never leave the browser / never written to `app_settings`.

**Credentials (live):**
- `ecr_sb_url` — Supabase URL (default the known project URL)
- `ecr_sb_key` — Supabase anon public JWT

**Future SaaS keys (fields present, inert until pipelines exist):**
`ecr_ga4_*`, `ecr_keepa_key`, `ecr_helium10_key`, `ecr_smartscout_key`,
`ecr_sellerboard_key`, `ecr_vendorati_key`. Stored, not yet used; labeled "not yet active".

**Replenishment parameters (persisted; optional cloud sync to `app_settings`):**
- `ecr_forecast_model` (smoothed | recent | conservative) — default **smoothed**
- `ecr_trend_on` (bool) — default **true**
- `ecr_supplier_lead_weeks` (default **6**) + per-vendor override map
- `ecr_fc_inbound_weeks` (default **2**) + per-destination override map (Amazon, Walmart, …)
- `ecr_target_weeks` (default **4**)
- `ecr_safety_weeks` (default **2**)
- `ecr_include_3pl` (bool) — default **false**
- `ecr_case_pack_round` (bool) — default **false**

localStorage keys are prefixed `ecr_` to avoid collision with the existing `de_` tool.

---

## 8. Out of scope (explicit)

- Building the NAV/SQL → Supabase or any SaaS ingestion pipeline (external, owned by DWH).
- Writing back to Supabase live data from the app (read-only on `ecom_inventory`).
- True seasonality modeling until `ecom_sales_history` is populated.
- Multi-user auth / RLS changes beyond reading the contract tables.
- Editing/sending real POs into NAV (export only for now).

---

## 9. Risks & notes

- **UPC join mismatch:** Excel UPC drops the leading zero (`97963850292`). Join on `item_no`
  first; normalize UPC if used as fallback.
- **Cumulative windows:** confirmed monotonic; rates use window length, not differencing,
  to stay robust to noise.
- **Single snapshot:** trend is reliable; seasonality is not — do not overstate forecast
  confidence in the UI.
- **One-time bootstrap volume:** ~20k ECOM rows; upsert in batches of 1000, `synced_at`
  stamped.
- **API version coupling** (DE Replen Pro lesson) does not apply — this tool talks only to
  Supabase, not the Shopify proxy.
