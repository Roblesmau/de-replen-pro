# DE Ecom Replen — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone single-file browser tool (`DE_Ecom_Replen.html`) that reads Designer Eyes e-commerce inventory + open-order data from Supabase, forecasts demand from trailing sales windows, computes coverage, and suggests vendor purchase orders that cover a configurable horizon including two-leg lead time.

**Architecture:** One HTML file, no build step (DE Replen Pro conventions: anti-flash theme boot, glass topbar, tabs, CSS tokens, localStorage). Read-only against Supabase project `zoqihptkwurpqenpjgfw`. Pure-logic engine (forecast/coverage/PO) lives in a `ECR.engine` namespace and is verified by a built-in self-test harness invoked with `?selftest=1` (PASS/FAIL to console). UI verified via browser preview. A one-time python loader bootstraps the `ecom_inventory` table from the Excel export; the external python/Microsoft sync replaces it later.

**Tech Stack:** HTML/CSS/vanilla JS, SheetJS (xlsx export), Chart.js (dashboard), Supabase REST (PostgREST), Python 3 + openpyxl + requests (one-time bootstrap only).

**Security note (CDN integrity):** the SheetJS and Chart.js `<script>` tags must include Subresource Integrity — `integrity="sha384-…" crossorigin="anonymous"` — to guard against CDN compromise. Fetch the published SRI hash for each pinned version (jsdelivr provides it) when adding the tag in Tasks 10 and 13. Do not ship a CDN tag without SRI.

**Spec:** `docs/superpowers/specs/2026-06-20-de-ecom-replen-design.md`

---

## File Structure

| File | Responsibility |
|------|----------------|
| `DE_Ecom_Replen.html` | The entire app: shell, Setup, Report, Forecast & Coverage, Suggested Purchase, Dashboard, `ECR.engine`, self-tests. |
| `scripts/bootstrap_ecom_inventory.py` | One-time Excel → Supabase loader (run locally; not part of the app). |
| `docs/superpowers/plans/2026-06-20-de-ecom-replen.md` | This plan. |
| Supabase migrations | `ecom_inventory`, `ecom_open_orders`, `ecom_sales_history` tables + RLS. Applied via MCP `apply_migration`. |

**Engine namespace contract** (defined in Task 9, used everywhere after). All pure, no DOM:
```
ECR.engine.rate(item, w)                 -> number   // daily rate for window w∈{7,15,30,60,90}
ECR.engine.weeklyDemand(item, params)    -> number
ECR.engine.trend(item)                   -> {ratio:number, label:'accelerating'|'steady'|'decelerating'}
ECR.engine.available(item, params)       -> number
ECR.engine.onOrder(item, openOrders)     -> {vendorOpen:number, marketplaceInbound:number, total:number}
ECR.engine.weeksOfSupply(item, params, openOrders) -> number   // Infinity if no demand
ECR.engine.totalLeadWeeks(item, params)  -> number   // supplier + fc inbound
ECR.engine.suggestPO(item, params, openOrders) -> {qty:number, need:number, flags:string[]}
```
`params` is the resolved settings object built from localStorage (Task 5). `item` is one row of `S.DATA`. `openOrders` is `S.openByItem[item_no]` (array of open PO lines, possibly empty).

---

## Column contract (ecom_inventory)

Snake_case of the Excel `ECOM` sheet header (row 3). PK `item_no` (Excel "Item"). All quantity/value/sales/price/cost columns are `numeric`; identity/status columns `text`; date columns load as `timestamptz` (Excel serials converted). Full list embedded in Task 1 DDL and Task 2 loader.

---

## Phase 0 — Supabase contract & data bootstrap

### Task 1: Create Supabase tables + RLS

**Files:**
- Supabase migration (via MCP `apply_migration`, name `ecr_create_tables`)

- [ ] **Step 1: Apply the migration**

Use MCP `mcp__8a165cce-...__apply_migration` with `project_id="zoqihptkwurpqenpjgfw"`, `name="ecr_create_tables"`, and this SQL:

```sql
-- ecom_inventory: the ~98-column report data, keyed by item_no
create table if not exists public.ecom_inventory (
  item_no text primary key,
  vcs_listing_status text, ecz_listing_status text, account_destination text,
  ecom_oor numeric,
  item_ec text, item_vsc text, asin text, upc text,
  brand_name text, amz_description text, gender text, item_category text,
  product_type text, vendor_name text,
  ecom_pickable_bins numeric, ecom_pickable_bins_value numeric,
  ecz_active numeric, ecz_active_value numeric,
  ecz_reserved_fc_transfer numeric, ecz_reserved_fc_transfer_value numeric,
  ecz_reserved_fc_processing numeric, ecz_reserved_fc_processing_value numeric,
  ecz_reserved_customer_order numeric, ecz_reserved_customer_order_value numeric,
  ecz_inbound numeric, ecz_inbound_value numeric,
  ecz_fba_total numeric, ecz_fba_total_value numeric,
  ecz_awd_available numeric, ecz_awd_inbound numeric, ecz_awd_replenishment_to_fba numeric,
  ecz_awd_total numeric, ecz_awd_total_value numeric,
  ecz_fba_plus_awd_total numeric, ecz_fba_plus_awd_total_value numeric,
  ecz_unfulfillable_total numeric, ecz_stranded_total numeric, ecz_removed_total numeric,
  ecz_rtv numeric, ecz_rtv_value numeric,
  vcs_active numeric, vcs_active_value numeric,
  vcs_reserved_fc_transfer numeric, vcs_reserved_fc_transfer_value numeric,
  vcs_reserved_fc_processing numeric, vcs_reserved_fc_processing_value numeric,
  vcs_reserved_customer_order numeric, vcs_reserved_customer_order_value numeric,
  vcs_inbound numeric, vcs_inbound_value numeric,
  vcs_fba_total numeric, vcs_fba_total_value numeric,
  vcs_awd_available numeric, vcs_awd_inbound numeric, vcs_awd_replenishment_to_fba numeric,
  vcs_awd_total numeric, vcs_awd_total_value numeric,
  vcs_fba_plus_awd_total numeric, vcs_fba_plus_awd_total_value numeric,
  vcs_unfulfillable_total numeric, vcs_stranded_total numeric,
  tpl_inventory_total numeric, tpl_inventory_total_value numeric,
  retail_pickable_bins numeric, retail_pickable_bins_value numeric,
  pickable_sunrise_inventory numeric, pickable_sunrise_inventory_value numeric,
  warehouse_transfers numeric,
  direct_cost numeric, us_map_price numeric, us_min_price numeric, us_max_price numeric,
  vcs_7d numeric, vcs_15d numeric, vcs_30d numeric, vcs_60d numeric, vcs_90d numeric,
  vcs_90d_value numeric, vcs_coverage_indicator text,
  ecz_7d numeric, ecz_15d numeric, ecz_30d numeric, ecz_60d numeric, ecz_90d numeric,
  ecz_90d_value numeric, ecz_coverage_indicator text,
  ecom_on_hand_qty numeric, ecom_on_hand_value numeric, ecom_damaged_qty numeric,
  retail_on_hand_qty numeric, retail_on_hand_value numeric, retail_damaged_qty numeric,
  total_on_hand_qty numeric, total_on_hand_value numeric,
  exclusivity text,
  amz_refreshing_date timestamptz, report_generation_date timestamptz,
  synced_at timestamptz default now()
);
create index if not exists ecom_inventory_brand_idx  on public.ecom_inventory(brand_name);
create index if not exists ecom_inventory_vendor_idx on public.ecom_inventory(vendor_name);

-- ecom_open_orders: open vendor POs from the ERP (SQL Server -> python -> Supabase)
create table if not exists public.ecom_open_orders (
  po_no text not null,
  po_line integer not null,
  item_no text,
  vendor_name text,
  qty_ordered numeric,
  qty_received numeric,
  qty_outstanding numeric,
  order_date date,
  expected_receipt_date date,
  destination text,
  status text,
  synced_at timestamptz default now(),
  primary key (po_no, po_line)
);
create index if not exists ecom_open_orders_item_idx on public.ecom_open_orders(item_no);

-- ecom_sales_history: time series stub for future seasonality (empty at launch)
create table if not exists public.ecom_sales_history (
  item_no text not null,
  source text not null,
  period_date date not null,
  units numeric,
  revenue numeric,
  synced_at timestamptz default now(),
  primary key (item_no, source, period_date)
);

-- RLS: anon may read; writes only via service_role (bootstrap + sync)
alter table public.ecom_inventory   enable row level security;
alter table public.ecom_open_orders enable row level security;
alter table public.ecom_sales_history enable row level security;
create policy ecr_inv_read  on public.ecom_inventory   for select to anon using (true);
create policy ecr_oo_read   on public.ecom_open_orders for select to anon using (true);
create policy ecr_sh_read   on public.ecom_sales_history for select to anon using (true);

-- mau_main_items is the BASE; ensure anon can read it (add policy only if RLS is enabled)
do $$
begin
  if exists (select 1 from pg_tables where schemaname='public' and tablename='mau_main_items'
             and rowsecurity=true)
     and not exists (select 1 from pg_policies where schemaname='public'
                     and tablename='mau_main_items' and policyname='ecr_mmi_read') then
    create policy ecr_mmi_read on public.mau_main_items for select to anon using (true);
  end if;
end $$;

-- v_replen_base: master (BASE) LEFT JOIN e-com data + aggregated open orders, server-side
create or replace view public.v_replen_base as
select m.item_no, m.upc, m.brand_name, m.model, m.product_color_code, m.size,
       m.product_group_code, m.item_category_code, m.description,
       e.account_destination, e.ecz_listing_status, e.vcs_listing_status, e.asin,
       e.gender, e.item_category, e.product_type, e.vendor_name, e.exclusivity,
       e.ecom_oor, e.ecom_pickable_bins, e.pickable_sunrise_inventory,
       e.ecz_fba_total, e.ecz_awd_available, e.ecz_awd_inbound, e.ecz_inbound,
       e.ecz_fba_plus_awd_total, e.vcs_fba_total, e.vcs_awd_available, e.vcs_awd_inbound,
       e.vcs_inbound, e.tpl_inventory_total, e.retail_pickable_bins,
       e.total_on_hand_qty, e.ecom_on_hand_qty,
       e.direct_cost, e.us_map_price, e.us_min_price, e.us_max_price,
       e.vcs_7d, e.vcs_15d, e.vcs_30d, e.vcs_60d, e.vcs_90d,
       e.ecz_7d, e.ecz_15d, e.ecz_30d, e.ecz_60d, e.ecz_90d,
       e.amz_description,
       coalesce(oo.vendor_open, 0) as vendor_open, oo.next_eta,
       (e.item_no is not null) as has_ecom
from public.mau_main_items m
left join public.ecom_inventory e on e.item_no = m.item_no
left join (
  select item_no, sum(qty_outstanding) as vendor_open, min(expected_receipt_date) as next_eta
  from public.ecom_open_orders group by item_no
) oo on oo.item_no = m.item_no;

grant select on public.v_replen_base to anon;
```

- [ ] **Step 2: Verify tables exist**

Run MCP `execute_sql`:
```sql
select table_name from information_schema.tables
where table_schema='public' and table_name like 'ecom_%' order by table_name;
```
Expected rows: `ecom_inventory`, `ecom_open_orders`, `ecom_sales_history`.

- [ ] **Step 3: Verify column count of ecom_inventory**

Run MCP `execute_sql`:
```sql
select count(*) from information_schema.columns
where table_schema='public' and table_name='ecom_inventory';
```
Expected: `99` (98 data columns + `synced_at`).

- [ ] **Step 4: Verify the view exists and anon can read it**

Run MCP `execute_sql`:
```sql
select count(*) as base_rows, count(*) filter (where has_ecom) as ecom_rows
from public.v_replen_base;
```
Expected before bootstrap: `base_rows` ≈ 337,276, `ecom_rows` = 0.
After Task 2 bootstrap, re-run: `ecom_rows` ≈ 20k. (If `v_replen_base` errors on `has_ecom`,
the view didn't create — re-apply the migration.)

---

### Task 2: Write the one-time Excel → Supabase bootstrap loader

**Files:**
- Create: `scripts/bootstrap_ecom_inventory.py`

- [ ] **Step 1: Write the loader script**

The script reads the `ECOM` sheet (header on row 3, subtotals on row 2, data from row 4),
snake_cases headers, maps `Item`→`item_no`, converts Excel serial dates, coerces numerics,
and upserts to Supabase in batches via PostgREST with the **service_role** key.

```python
#!/usr/bin/env python3
"""One-time bootstrap: All_Inventory ECOM sheet -> Supabase public.ecom_inventory.
Usage:
  SB_URL=https://zoqihptkwurpqenpjgfw.supabase.co \
  SB_SERVICE_KEY=eyJ... \
  python scripts/bootstrap_ecom_inventory.py "C:/path/All_Inventory_20260618.xlsx"
"""
import os, sys, re, json, datetime, urllib.request
from openpyxl import load_workbook

SB_URL = os.environ["SB_URL"].rstrip("/")
SB_KEY = os.environ["SB_SERVICE_KEY"]
XLSX   = sys.argv[1]

# Columns that must be numeric (everything except identity/status/date/coverage-indicator)
TEXT_COLS = {
  "vcs_listing_status","ecz_listing_status","account_destination","item_no","item_ec",
  "item_vsc","asin","upc","brand_name","amz_description","gender","item_category",
  "product_type","vendor_name","vcs_coverage_indicator","ecz_coverage_indicator","exclusivity",
}
DATE_COLS = {"amz_refreshing_date","report_generation_date"}

def snake(h):
    h = h.replace("*","").replace("+"," plus ").strip()
    h = h.replace("3PL","tpl").replace("FC","fc")
    h = re.sub(r"[^0-9A-Za-z]+","_",h).strip("_").lower()
    return re.sub(r"_+","_",h)

def excel_serial_to_iso(v):
    # Excel serial day (1900 date system) -> ISO timestamp
    if v in (None,""): return None
    try: f = float(v)
    except (TypeError,ValueError): return None
    base = datetime.datetime(1899,12,30)
    return (base + datetime.timedelta(days=f)).isoformat()

def coerce(col, v):
    if v in (None,""): return None
    if col in DATE_COLS: return excel_serial_to_iso(v)
    if col in TEXT_COLS:
        s = str(v).strip()
        return s if s else None
    try: return float(v)
    except (TypeError,ValueError): return None

def post_batch(rows):
    url = f"{SB_URL}/rest/v1/ecom_inventory?on_conflict=item_no"
    body = json.dumps(rows).encode()
    req = urllib.request.Request(url, data=body, method="POST", headers={
        "apikey": SB_KEY, "Authorization": f"Bearer {SB_KEY}",
        "Content-Type": "application/json",
        "Prefer": "resolution=merge-duplicates,return=minimal",
    })
    with urllib.request.urlopen(req) as r:
        if r.status not in (200,201,204):
            raise SystemExit(f"Supabase {r.status}: {r.read().decode()[:300]}")

def main():
    wb = load_workbook(XLSX, read_only=True, data_only=True)
    ws = wb["ECOM"]
    rows_iter = ws.iter_rows(values_only=True)
    all_rows = list(rows_iter)
    header = [snake(str(h)) if h is not None else "" for h in all_rows[2]]  # row 3
    header = ["item_no" if h=="item" else h for h in header]
    data = all_rows[3:]
    batch, total = [], 0
    for raw in data:
        item_no = raw[header.index("item_no")]
        if item_no in (None,""): continue
        rec = {}
        for i,col in enumerate(header):
            if not col: continue
            rec[col] = coerce(col, raw[i] if i < len(raw) else None)
        batch.append(rec)
        if len(batch) >= 1000:
            post_batch(batch); total += len(batch); print(f"upserted {total}"); batch=[]
    if batch:
        post_batch(batch); total += len(batch)
    print(f"DONE: {total} rows upserted to ecom_inventory")

if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Run the loader**

Get the service_role key from Mau (Supabase dashboard → Project Settings → API → `service_role`).
Run (Git Bash):
```bash
SB_URL=https://zoqihptkwurpqenpjgfw.supabase.co \
SB_SERVICE_KEY='<service_role_key>' \
python scripts/bootstrap_ecom_inventory.py "/c/Users/Mauricio.Robles/Downloads/All_Inventory_20260618.xlsx"
```
Expected: progress lines `upserted 1000 … 20000`, final `DONE: ~20689 rows`.

- [ ] **Step 3: Verify load**

Run MCP `execute_sql`:
```sql
select count(*) rows,
       count(*) filter (where ecz_90d > 0 or vcs_90d > 0) with_sales,
       count(distinct vendor_name) vendors,
       round(sum(direct_cost * total_on_hand_qty)) approx_inv_cost
from public.ecom_inventory;
```
Expected: ~20,689 rows, several thousand `with_sales`, >0 vendors, a plausible cost total.

- [ ] **Step 4: Commit the script**

```bash
git add scripts/bootstrap_ecom_inventory.py
git commit -m "Add one-time Excel->Supabase bootstrap loader for ecom_inventory"
```

---

## Phase 1 — App shell & Setup

### Task 3: App skeleton (boot theme, topbar, tab nav)

**Files:**
- Create: `DE_Ecom_Replen.html`

- [ ] **Step 1: Create the file with shell + theme boot + tabs**

Follow DE Replen Pro conventions. Minimum viable shell (engine/data wired in later tasks):

```html
<!doctype html>
<html lang="en" data-theme="light">
<head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1">
<title>DE Ecom Replen</title>
<script>
  // anti-flash theme boot
  try{var t=localStorage.getItem('ecr_theme')||'light';document.documentElement.setAttribute('data-theme',t);}catch(e){}
</script>
<style>
  :root,[data-theme="light"]{--bg:#f6f8fa;--surface:#fff;--surface-2:#f0f3f6;--text:#1b1f24;--muted:#656d76;--blue:#0969da;--blue-bg:#ddf4ff;--navy:#1f4e79;--border:#d0d7de;--glass:rgba(255,255,255,.7);--good:#1a7f37;--warn:#9a6700;--bad:#cf222e;}
  [data-theme="dark"]{--bg:#0d1117;--surface:#161b22;--surface-2:#21262d;--text:#e6edf3;--muted:#8b949e;--blue:#4493f8;--blue-bg:#121d2f;--navy:#9ec5ff;--border:#30363d;--glass:rgba(22,27,34,.7);--good:#3fb950;--warn:#d29922;--bad:#f85149;}
  *{box-sizing:border-box}body{margin:0;font-family:Inter,system-ui,sans-serif;background:var(--bg);color:var(--text);font-variant-numeric:tabular-nums}
  .mono,code,.metric-val{font-family:'JetBrains Mono',ui-monospace,monospace}
  .topbar{position:sticky;top:0;z-index:10;display:flex;align-items:center;gap:14px;padding:10px 16px;background:var(--glass);backdrop-filter:blur(14px);border-bottom:1px solid var(--border)}
  .tabs{display:flex;gap:4px;position:sticky;top:52px;z-index:9;background:var(--glass);backdrop-filter:blur(14px);padding:6px 16px;border-bottom:1px solid var(--border);overflow-x:auto}
  .tab{padding:8px 14px;border:none;background:transparent;color:var(--muted);cursor:pointer;border-radius:8px;font-weight:600}
  .tab.active{background:var(--surface);color:var(--text)}
  .panel{display:none;padding:18px}.panel.active{display:block}
  .card{background:var(--surface);border:1px solid var(--border);border-radius:12px;padding:16px;margin-bottom:14px}
  .btn{padding:8px 14px;border:1px solid var(--border);background:var(--surface);color:var(--text);border-radius:8px;cursor:pointer;font-weight:600}
  .btn.primary{background:var(--blue);color:#fff;border-color:var(--blue)}
  .table-scroll{overflow:auto;max-height:75vh;position:relative}
  table{border-collapse:collapse;width:100%}th,td{padding:6px 10px;border-bottom:1px solid var(--border);text-align:right;white-space:nowrap}
  th:first-child,td:first-child,th.txt,td.txt{text-align:left}
  thead th{position:sticky;top:0;background:var(--surface-2);z-index:1}
  input,select{padding:7px 9px;border:1px solid var(--border);border-radius:8px;background:var(--surface);color:var(--text)}
  .pill{display:inline-block;padding:2px 8px;border-radius:999px;font-size:12px;font-weight:700}
  .pill.good{background:var(--blue-bg);color:var(--good)}.pill.warn{color:var(--warn)}.pill.bad{color:var(--bad)}
  a.skip-link{position:absolute;left:-999px}a.skip-link:focus{left:8px;top:8px}
  *:focus-visible{outline:2px solid var(--blue);outline-offset:2px}
  @media (prefers-reduced-motion:reduce){*{transition:none!important}}
</style>
</head>
<body>
<a href="#main" class="skip-link">Skip to content</a>
<div class="topbar">
  <strong>DE Ecom Replen</strong>
  <span id="connState" class="pill warn">Not connected</span>
  <span style="flex:1"></span>
  <button class="btn" id="themeBtn" onclick="ECR.toggleTheme()">🌙</button>
</div>
<div class="tabs" role="tablist">
  <button class="tab active" data-tab="setup"    onclick="ECR.showTab('setup')">Setup</button>
  <button class="tab" data-tab="report"   onclick="ECR.showTab('report')">Report</button>
  <button class="tab" data-tab="forecast" onclick="ECR.showTab('forecast')">Forecast &amp; Coverage</button>
  <button class="tab" data-tab="purchase" onclick="ECR.showTab('purchase')">Suggested Purchase</button>
  <button class="tab" data-tab="dashboard" onclick="ECR.showTab('dashboard')">Dashboard</button>
</div>
<main id="main">
  <section class="panel active" id="panel-setup"><div class="card">Setup loads here.</div></section>
  <section class="panel" id="panel-report"></section>
  <section class="panel" id="panel-forecast"></section>
  <section class="panel" id="panel-purchase"></section>
  <section class="panel" id="panel-dashboard"></section>
</main>
<script>
window.ECR = window.ECR || {};
ECR.S = { DATA:[], openByItem:{}, params:{}, loadedAt:null };
ECR.showTab = function(name){
  document.querySelectorAll('.tab').forEach(t=>t.classList.toggle('active', t.dataset.tab===name));
  document.querySelectorAll('.panel').forEach(p=>p.classList.toggle('active', p.id==='panel-'+name));
};
ECR.toggleTheme = function(){
  var cur=document.documentElement.getAttribute('data-theme')==='dark'?'light':'dark';
  document.documentElement.setAttribute('data-theme',cur);
  try{localStorage.setItem('ecr_theme',cur);}catch(e){}
};
</script>
</body>
</html>
```

- [ ] **Step 2: Verify it renders**

Run `preview_start` on `DE_Ecom_Replen.html`, then `preview_snapshot`.
Expected: topbar with "DE Ecom Replen", 5 tabs, "Setup loads here." card; clicking tabs switches panels; theme button toggles light/dark.

- [ ] **Step 3: Commit**

```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: app shell with theme boot and tab nav"
```

---

### Task 4: Setup tab — fields, defaults, localStorage persistence

**Files:**
- Modify: `DE_Ecom_Replen.html` (replace `#panel-setup` content; add settings JS)

- [ ] **Step 1: Add the Setup panel markup**

Replace the `#panel-setup` section body with credential + parameter + future-keys cards:

```html
<section class="panel active" id="panel-setup">
  <div class="card">
    <h3>Connection</h3>
    <label class="txt">Supabase URL <input id="ecr_sb_url" style="width:380px" placeholder="https://zoqihptkwurpqenpjgfw.supabase.co"></label><br><br>
    <label class="txt">Supabase anon key <input id="ecr_sb_key" type="password" style="width:380px"></label><br><br>
    <button class="btn primary" id="btnTest" onclick="ECR.testConnection()">Test connection</button>
    <button class="btn" onclick="ECR.loadData()">Load data</button>
    <span id="connMsg" class="mono"></span>
  </div>
  <div class="card">
    <h3>Replenishment parameters</h3>
    <label>Forecast model
      <select id="ecr_forecast_model">
        <option value="smoothed">Smoothed (0.5·30d+0.3·60d+0.2·90d)</option>
        <option value="recent">Recent (30d)</option>
        <option value="conservative">Conservative (slowest of 30/60/90)</option>
      </select></label>
    <label><input type="checkbox" id="ecr_trend_on"> Apply trend multiplier</label><br><br>
    <label>Supplier lead (weeks) <input type="number" id="ecr_supplier_lead_weeks" min="0" step="0.5" style="width:80px"></label>
    <label>FC inbound (weeks) <input type="number" id="ecr_fc_inbound_weeks" min="0" step="0.5" style="width:80px"></label>
    <label>Target coverage (weeks) <input type="number" id="ecr_target_weeks" min="0" step="0.5" style="width:80px"></label>
    <label>Safety (weeks) <input type="number" id="ecr_safety_weeks" min="0" step="0.5" style="width:80px"></label><br><br>
    <label><input type="checkbox" id="ecr_include_3pl"> Include 3PL in available</label>
    <label><input type="checkbox" id="ecr_case_pack_round"> Round PO up to case pack</label><br><br>
    <button class="btn primary" onclick="ECR.saveSettings()">Save settings</button>
    <span id="setupMsg" class="mono"></span>
  </div>
  <div class="card">
    <h3>Future data sources <span class="pill warn">not yet active</span></h3>
    <p class="mono" style="color:var(--muted)">Stored for later pipelines; unused today.</p>
    <label class="txt">GA4 property ID <input id="ecr_ga4_property"></label>
    <label class="txt">Keepa key <input id="ecr_keepa_key" type="password"></label>
    <label class="txt">Helium 10 key <input id="ecr_helium10_key" type="password"></label>
    <label class="txt">SmartScout key <input id="ecr_smartscout_key" type="password"></label>
    <label class="txt">Sellerboard key <input id="ecr_sellerboard_key" type="password"></label>
    <label class="txt">Vendorati key <input id="ecr_vendorati_key" type="password"></label>
  </div>
</section>
```

- [ ] **Step 2: Add settings persistence JS**

```javascript
ECR.DEFAULTS = {
  ecr_sb_url:'https://zoqihptkwurpqenpjgfw.supabase.co', ecr_sb_key:'',
  ecr_forecast_model:'smoothed', ecr_trend_on:true,
  ecr_supplier_lead_weeks:6, ecr_fc_inbound_weeks:2, ecr_target_weeks:4, ecr_safety_weeks:2,
  ecr_include_3pl:false, ecr_case_pack_round:false,
  ecr_ga4_property:'', ecr_keepa_key:'', ecr_helium10_key:'', ecr_smartscout_key:'',
  ecr_sellerboard_key:'', ecr_vendorati_key:''
};
ECR.getEl = id => document.getElementById(id);
ECR.loadSettings = function(){
  Object.keys(ECR.DEFAULTS).forEach(k=>{
    let v; try{ v=localStorage.getItem(k); }catch(e){}
    if(v===null||v===undefined) v=ECR.DEFAULTS[k];
    else if(typeof ECR.DEFAULTS[k]==='boolean') v=(v==='true'||v===true);
    else if(typeof ECR.DEFAULTS[k]==='number') v=parseFloat(v);
    const el=ECR.getEl(k); if(!el) return;
    if(el.type==='checkbox') el.checked=!!v; else el.value=v;
  });
  ECR.buildParams();
};
ECR.saveSettings = function(){
  Object.keys(ECR.DEFAULTS).forEach(k=>{
    const el=ECR.getEl(k); if(!el) return;
    const v = el.type==='checkbox' ? el.checked : el.value;
    try{ localStorage.setItem(k, v); }catch(e){}
  });
  ECR.buildParams();
  ECR.getEl('setupMsg').textContent='Saved ✓';
  if(ECR.S.DATA.length) ECR.recompute();
};
ECR.buildParams = function(){
  const num=id=>parseFloat(ECR.getEl(id).value)||0;
  const bool=id=>ECR.getEl(id).checked;
  ECR.S.params = {
    model: ECR.getEl('ecr_forecast_model').value, trendOn: bool('ecr_trend_on'),
    supplierLead: num('ecr_supplier_lead_weeks'), fcInbound: num('ecr_fc_inbound_weeks'),
    target: num('ecr_target_weeks'), safety: num('ecr_safety_weeks'),
    include3pl: bool('ecr_include_3pl'), casePack: bool('ecr_case_pack_round')
  };
  return ECR.S.params;
};
document.addEventListener('DOMContentLoaded', ECR.loadSettings);
```

- [ ] **Step 3: Verify persistence**

`preview_start` then `preview_eval`:
```js
document.getElementById('ecr_target_weeks').value=10; ECR.saveSettings();
window.location.reload();
```
After reload, `preview_eval`: `document.getElementById('ecr_target_weeks').value` → Expected `"10"`.
Reset to default: set back to 4, save.

- [ ] **Step 4: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: Setup tab with persisted params and credentials"
```

---

### Task 5: Supabase connection layer + Test connection

**Files:**
- Modify: `DE_Ecom_Replen.html` (add `ECR.sbFetch`, `ECR.testConnection`)

- [ ] **Step 1: Add sbFetch (text-first pattern) + testConnection**

```javascript
ECR.sbCreds = function(){
  return { url:(ECR.getEl('ecr_sb_url').value||'').replace(/\/$/,''), key:ECR.getEl('ecr_sb_key').value||'' };
};
ECR.sbFetch = async function(path){
  const {url,key}=ECR.sbCreds();
  if(!url||!key) throw new Error('Missing Supabase URL/key');
  const r = await fetch(url+'/rest/v1/'+path, { headers:{ apikey:key, Authorization:'Bearer '+key } });
  if(!r.ok){ const t=await r.text(); throw new Error('Supabase '+r.status+': '+t.slice(0,200)); }
  if(r.status===204) return null;
  const txt = await r.text();
  if(!txt || !txt.trim()) return null;
  return JSON.parse(txt);
};
ECR.testConnection = async function(){
  const msg=ECR.getEl('connMsg'); msg.textContent='Testing…';
  try{
    const d = await ECR.sbFetch('ecom_inventory?select=item_no&limit=1');
    ECR.getEl('connState').className='pill good';
    ECR.getEl('connState').textContent='Connected';
    ECR.getEl('btnTest').classList.add('primary');
    msg.textContent = (d && d.length) ? 'OK — table reachable ✓' : 'Connected, but table empty';
  }catch(e){
    ECR.getEl('connState').className='pill bad'; ECR.getEl('connState').textContent='Error';
    msg.textContent = e.message;
  }
};
```

- [ ] **Step 2: Verify against live Supabase**

In Setup, enter the anon key (ask Mau), click Test connection via `preview_eval`:
```js
ECR.getEl('ecr_sb_key').value='<anon_key>'; await ECR.testConnection();
document.getElementById('connState').textContent;
```
Expected: `"Connected"` and `connMsg` "OK — table reachable ✓".

- [ ] **Step 3: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: Supabase fetch layer and Test connection"
```

---

## Phase 2 — Data load & state

### Task 6: loadData() — smart load (master base) + open orders + unmatched list

**Files:**
- Modify: `DE_Ecom_Replen.html` (add `ECR.loadData`, `ECR.recompute` stub)

Baseline = `mau_main_items` (337k), so the app does NOT load it all. It loads the
**actionable working set** from `v_replen_base` (rows with e-com data OR an open order),
fetches open-order lines for the detail/fallback, and builds the **unmatched e-com list**
(e-com SKUs absent from the master) for the data-quality report. The ~300k non-selling master
items are browsed server-side later (Task 10), not preloaded.

- [ ] **Step 1: Add smart load + open-order aggregation + unmatched list**

```javascript
ECR.fetchPaged = async function(pathBase){
  const PAGE=1000; let from=0, out=[];
  while(true){
    const sep = pathBase.indexOf('?')>=0 ? '&' : '?';
    const d = await ECR.sbFetch(pathBase+sep+'limit='+PAGE+'&offset='+from) || [];
    out = out.concat(d);
    if(d.length < PAGE) break;
    from += PAGE;
  }
  return out;
};
ECR.loadData = async function(){
  const msg=ECR.getEl('connMsg'); msg.textContent='Loading actionable items…';
  try{
    // 1) actionable working set from the master-based view (has e-com data OR an open order)
    const inv = await ECR.fetchPaged('v_replen_base?select=*&or=(has_ecom.eq.true,vendor_open.gt.0)&order=item_no.asc');
    // 2) open-order lines for per-item detail + onOrder fallback (tolerate empty/missing)
    msg.textContent='Loading open orders…';
    let oo=[]; try{ oo = await ECR.fetchPaged('ecom_open_orders?select=*&order=item_no.asc'); }catch(e){ oo=[]; }
    ECR.S.openByItem = {};
    oo.forEach(o=>{ const k=o.item_no; if(!k) return; (ECR.S.openByItem[k]=ECR.S.openByItem[k]||[]).push(o); });
    // 3) data-quality: e-com SKUs NOT in the master (left-join-anti via PostgREST)
    msg.textContent='Checking unmatched e-com items…';
    let unmatched=[];
    try{
      // ecom_inventory rows whose item_no has no master row: use a NOT-in filter in chunks
      const masterIds = new Set(inv.map(r=>r.item_no));
      const allEcom = await ECR.fetchPaged('ecom_inventory?select=item_no,asin,brand_name,amz_description,ecz_90d,vcs_90d,total_on_hand_qty&order=item_no.asc');
      unmatched = allEcom.filter(r=>!masterIds.has(r.item_no));
    }catch(e){ unmatched=[]; }
    ECR.S.DATA = inv;
    ECR.S.unmatched = unmatched;
    ECR.S.loadedAt = new Date();
    ECR.buildParams();
    ECR.recompute();
    msg.textContent = `Loaded ${inv.length} actionable items · ${oo.length} open-order lines · ${unmatched.length} unmatched e-com ✓`;
  }catch(e){ msg.textContent='Load failed: '+e.message; }
};
ECR.recompute = function(){ /* filled in Task 10 (renders tabs) */ };
```

> **Note on the unmatched check:** the working set is the *intersection* (e-com items that
> exist in the master). `unmatched` = e-com items in `ecom_inventory` whose `item_no` is not in
> that intersection — i.e. absent from the master. This is computed client-side by diffing the
> full `ecom_inventory` id list against the loaded master-matched ids. It is the data-quality
> report; those items are intentionally NOT in `S.DATA` (strict master base).

- [ ] **Step 2: Verify load counts**

`preview_eval` (after setting key + Test connection):
```js
await ECR.loadData();
({actionable:ECR.S.DATA.length, sample:ECR.S.DATA[0] && ECR.S.DATA[0].item_no,
  oo:Object.keys(ECR.S.openByItem).length, unmatched:ECR.S.unmatched.length,
  hasEcom:ECR.S.DATA[0] && ECR.S.DATA[0].has_ecom});
```
Expected: `actionable` ≈ 20k (e-com items present in master), non-empty `sample`,
`unmatched` ≥ 1 (e.g. item 367906), `hasEcom` true.

- [ ] **Step 3: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: smart load from v_replen_base + unmatched e-com list"
```

---

## Phase 3 — Engine (TDD via ?selftest=1 harness)

### Task 7: Self-test harness + forecast functions

**Files:**
- Modify: `DE_Ecom_Replen.html` (add `ECR.engine`, `ECR.runSelfTests`, auto-run on `?selftest=1`)

- [ ] **Step 1: Write the failing self-test for forecast**

Add the harness and the forecast assertions (functions not yet implemented):

```javascript
ECR.num = v => { const n=parseFloat(v); return isFinite(n)?n:0; };
ECR.engine = ECR.engine || {};
ECR.runSelfTests = function(){
  const A=[]; const ok=(n,c)=>A.push((c?'PASS':'FAIL')+': '+n);
  const approx=(a,b)=>Math.abs(a-b)<1e-6;
  // fixture: ecz windows 7/15/30/60/90 = 25/45/70/137/222, vcs all 0
  const item={ ecz_7d:25,ecz_15d:45,ecz_30d:70,ecz_60d:137,ecz_90d:222,
               vcs_7d:0,vcs_15d:0,vcs_30d:0,vcs_60d:0,vcs_90d:0 };
  ok('rate(30)=70/30', approx(ECR.engine.rate(item,30), 70/30));
  // smoothed = 0.5*(70/30)+0.3*(137/60)+0.2*(222/90)
  const sm = 0.5*(70/30)+0.3*(137/60)+0.2*(222/90);
  ok('weeklyDemand smoothed', approx(ECR.engine.weeklyDemand(item,{model:'smoothed',trendOn:false}), sm*7));
  // trend = rate(30)/rate(90) = (70/30)/(222/90)
  ok('trend accelerating', ECR.engine.trend(item).label==='accelerating');
  // zero-demand item
  const zero={ecz_7d:0,ecz_15d:0,ecz_30d:0,ecz_60d:0,ecz_90d:0,vcs_7d:0,vcs_15d:0,vcs_30d:0,vcs_60d:0,vcs_90d:0};
  ok('zero weeklyDemand', ECR.engine.weeklyDemand(zero,{model:'smoothed',trendOn:false})===0);
  const out=A.join('\n'); console.log('SELFTEST\n'+out);
  return A.every(x=>x.startsWith('PASS'));
};
if(location.search.indexOf('selftest=1')>=0) document.addEventListener('DOMContentLoaded',ECR.runSelfTests);
```

- [ ] **Step 2: Run and verify it FAILS**

`preview_start` with URL `DE_Ecom_Replen.html?selftest=1`, then `preview_console_logs`.
Expected: `SELFTEST` block with FAIL lines (engine.rate is undefined → throws or fails).

- [ ] **Step 3: Implement forecast functions**

```javascript
ECR.engine.rate = function(item,w){
  const days={7:7,15:15,30:30,60:60,90:90}[w];
  const sold = ECR.num(item['ecz_'+w+'d']) + ECR.num(item['vcs_'+w+'d']);
  return days ? sold/days : 0;
};
ECR.engine.trend = function(item){
  const r30=ECR.engine.rate(item,30), r90=ECR.engine.rate(item,90);
  const ratio = r90>0 ? r30/r90 : (r30>0?2:1);
  let label='steady'; if(ratio>1.1) label='accelerating'; else if(ratio<0.9) label='decelerating';
  return {ratio,label};
};
ECR.engine.weeklyDemand = function(item,params){
  const r30=ECR.engine.rate(item,30), r60=ECR.engine.rate(item,60), r90=ECR.engine.rate(item,90);
  let daily;
  if(params.model==='recent') daily=r30;
  else if(params.model==='conservative') daily=Math.min(r30,r60,r90);
  else daily = 0.5*r30 + 0.3*r60 + 0.2*r90; // smoothed
  if(params.trendOn){
    let m=ECR.engine.trend(item).ratio; m=Math.max(0.5,Math.min(2.0,m)); daily*=m;
  }
  return daily*7;
};
```

- [ ] **Step 4: Run and verify it PASSES**

Reload `?selftest=1`, `preview_console_logs`.
Expected: all forecast lines `PASS`. (Coverage/PO tests added next will still be absent — fine.)

- [ ] **Step 5: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: forecast engine + self-test harness (TDD)"
```

---

### Task 8: Coverage functions + tests

**Files:**
- Modify: `DE_Ecom_Replen.html` (extend `ECR.engine`, add coverage asserts to `runSelfTests`)

- [ ] **Step 1: Add failing coverage assertions** (inside `runSelfTests`, before the `const out=` line)

```javascript
  // coverage fixture
  const ci={ ecom_pickable_bins:10, pickable_sunrise_inventory:0,
    ecz_fba_total:5, ecz_awd_available:2, vcs_fba_total:0, vcs_awd_available:0,
    tpl_inventory_total:100, ecom_oor:8, ecz_inbound:1, ecz_awd_inbound:0,
    vcs_inbound:0, vcs_awd_inbound:0,
    ecz_7d:7,ecz_15d:15,ecz_30d:30,ecz_60d:60,ecz_90d:90, vcs_7d:0,vcs_15d:0,vcs_30d:0,vcs_60d:0,vcs_90d:0 };
  // available excl 3PL = 10+0+5+2 = 17
  ok('available excl 3PL', ECR.engine.available(ci,{include3pl:false})===17);
  ok('available incl 3PL', ECR.engine.available(ci,{include3pl:true})===117);
  // onOrder: vendorOpen from ecom_oor=8 (no feed), marketplaceInbound=1 -> total 9
  ok('onOrder total', ECR.engine.onOrder(ci,[]).total===9);
  // onOrder uses feed when present (sum qty_outstanding)
  ok('onOrder feed', ECR.engine.onOrder(ci,[{qty_outstanding:20},{qty_outstanding:5}]).vendorOpen===25);
  // totalLeadWeeks = supplier 6 + fc 2 = 8
  ok('totalLeadWeeks', ECR.engine.totalLeadWeeks(ci,{supplierLead:6,fcInbound:2})===8);
```

- [ ] **Step 2: Run, verify new lines FAIL** — `?selftest=1`, `preview_console_logs`. Expected: coverage lines FAIL.

- [ ] **Step 3: Implement coverage functions**

```javascript
ECR.engine.available = function(item,params){
  let a = ECR.num(item.ecom_pickable_bins) + ECR.num(item.pickable_sunrise_inventory)
        + ECR.num(item.ecz_fba_total) + ECR.num(item.ecz_awd_available)
        + ECR.num(item.vcs_fba_total) + ECR.num(item.vcs_awd_available);
  if(params.include3pl) a += ECR.num(item.tpl_inventory_total);
  return a;
};
ECR.engine.onOrder = function(item,openOrders){
  const marketplaceInbound = ECR.num(item.ecz_inbound)+ECR.num(item.ecz_awd_inbound)
                           + ECR.num(item.vcs_inbound)+ECR.num(item.vcs_awd_inbound);
  let vendorOpen;
  if(openOrders && openOrders.length) vendorOpen = openOrders.reduce((s,o)=>s+ECR.num(o.qty_outstanding),0);
  else vendorOpen = ECR.num(item.ecom_oor); // fallback
  return { vendorOpen, marketplaceInbound, total: vendorOpen+marketplaceInbound };
};
ECR.engine.totalLeadWeeks = function(item,params){
  return ECR.num(params.supplierLead) + ECR.num(params.fcInbound);
};
ECR.engine.weeksOfSupply = function(item,params,openOrders){
  const wd=ECR.engine.weeklyDemand(item,params); if(wd<=0) return Infinity;
  return (ECR.engine.available(item,params) + ECR.engine.onOrder(item,openOrders).total)/wd;
};
```

- [ ] **Step 4: Run, verify PASS** — `?selftest=1`, `preview_console_logs`. Expected: all coverage lines PASS.

- [ ] **Step 5: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: coverage engine + tests"
```

---

### Task 9: Suggested-PO function + tests

**Files:**
- Modify: `DE_Ecom_Replen.html` (extend `ECR.engine`, add PO asserts)

- [ ] **Step 1: Add failing PO assertions** (inside `runSelfTests`)

```javascript
  // PO fixture: weeklyDemand ~ compute, horizon=lead(8)+target(4)+safety(2)=14
  const pp={model:'recent',trendOn:false,supplierLead:6,fcInbound:2,target:4,safety:2,include3pl:false,casePack:false};
  // recent rate(30)=30/30=1/day -> weekly 7; horizon 14 -> need=7*14 - (avail+onorder)
  const pf={ ecom_pickable_bins:10, pickable_sunrise_inventory:0, ecz_fba_total:0, ecz_awd_available:0,
    vcs_fba_total:0, vcs_awd_available:0, tpl_inventory_total:0,
    ecom_oor:0, ecz_inbound:0, ecz_awd_inbound:0, vcs_inbound:0, vcs_awd_inbound:0,
    ecz_7d:7,ecz_15d:15,ecz_30d:30,ecz_60d:60,ecz_90d:90, vcs_7d:0,vcs_15d:0,vcs_30d:0,vcs_60d:0,vcs_90d:0 };
  // need = 7*14 - (10+0) = 98 - 10 = 88
  ok('suggestPO qty', ECR.engine.suggestPO(pf,pp,[]).qty===88);
  // no-demand item never suggests
  const nd=Object.assign({},pf,{ecz_7d:0,ecz_15d:0,ecz_30d:0,ecz_60d:0,ecz_90d:0});
  ok('suggestPO zero-demand', ECR.engine.suggestPO(nd,pp,[]).qty===0);
  // stockout flag: avail+onorder=10, weeklyDemand=7 -> wos=1.43 < totalLead(8) -> flag
  ok('stockout flag', ECR.engine.suggestPO(pf,pp,[]).flags.indexOf('stockout')>=0);
```

- [ ] **Step 2: Run, verify FAIL** — `?selftest=1`. Expected: PO lines FAIL.

- [ ] **Step 3: Implement suggestPO**

```javascript
ECR.engine.suggestPO = function(item,params,openOrders){
  const wd = ECR.engine.weeklyDemand(item,params);
  const flags=[];
  if(wd<=0) return {qty:0, need:0, flags:['no_demand']};
  const avail = ECR.engine.available(item,params);
  const oo = ECR.engine.onOrder(item,openOrders).total;
  const lead = ECR.engine.totalLeadWeeks(item,params);
  const horizon = lead + ECR.num(params.target) + ECR.num(params.safety);
  const wos = (avail+oo)/wd;
  if(wos < lead) flags.push('stockout');
  if(wos > horizon + ECR.num(params.target)) flags.push('overstock');
  let need = wd*horizon - (avail+oo);
  let qty = Math.max(0, Math.ceil(need));
  // case-pack rounding hook (no pack data yet -> no-op unless casePack && item.case_pack)
  if(params.casePack && item.case_pack>1) qty = Math.ceil(qty/item.case_pack)*item.case_pack;
  return {qty, need, flags};
};
```

- [ ] **Step 4: Run, verify ALL self-tests PASS** — `?selftest=1`, `preview_console_logs`.
Expected: every line `PASS` (forecast + coverage + PO).

- [ ] **Step 5: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: suggested-PO engine + tests (all self-tests green)"
```

---

## Phase 4 — UI tabs

### Task 10: Report tab (filterable grid + export)

**Files:**
- Modify: `DE_Ecom_Replen.html` (render `#panel-report`; add SheetJS CDN; `ECR.renderReport`)

- [ ] **Step 1: Add SheetJS + report render**

Add to `<head>`: `<script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>`

```javascript
ECR.REPORT_COLS = [
  ['item_no','Item','txt'],['brand_name','Brand','txt'],['amz_description','Description','txt'],
  ['vendor_name','Vendor','txt'],['item_category','Cat','txt'],['asin','ASIN','txt'],
  ['ecz_90d','ECZ 90d',''],['vcs_90d','VCS 90d',''],['ecom_oor','OOR',''],
  ['ecz_fba_plus_awd_total','ECZ FBA+AWD',''],['ecom_pickable_bins','Ecom bins',''],
  ['total_on_hand_qty','Total OH',''],['direct_cost','Cost','']
];
ECR.getFilters = function(){
  return { brand:ECR.getEl('rf_brand').value, vendor:ECR.getEl('rf_vendor').value,
    cat:ECR.getEl('rf_cat').value, q:(ECR.getEl('rf_q').value||'').toLowerCase() };
};
ECR.filteredData = function(){
  const f=ECR.getFilters();
  return ECR.S.DATA.filter(d=>{
    if(f.brand && d.brand_name!==f.brand) return false;
    if(f.vendor && d.vendor_name!==f.vendor) return false;
    if(f.cat && d.item_category!==f.cat) return false;
    if(f.q){ const hay=((d.item_no||'')+' '+(d.amz_description||'')+' '+(d.asin||'')+' '+(d.brand_name||'')).toLowerCase(); if(hay.indexOf(f.q)<0) return false; }
    return true;
  });
};
ECR.optsFor = key => [''].concat([...new Set(ECR.S.DATA.map(d=>d[key]).filter(Boolean))].sort());
ECR.renderReport = function(){
  const p=ECR.getEl('panel-report');
  if(!ECR.S.DATA.length){ p.innerHTML='<div class="card">Load data in Setup first.</div>'; return; }
  const sel=(id,key,label)=>`<label>${label} <select id="${id}" onchange="ECR.renderReport()">`+
    ECR.optsFor(key).map(o=>`<option${o===(ECR.getEl(id)?ECR.getEl(id).value:'')?' selected':''}>${o}</option>`).join('')+`</select></label>`;
  const rows=ECR.filteredData();
  p.innerHTML = `<div class="card">
      ${sel('rf_brand','brand_name','Brand')} ${sel('rf_vendor','vendor_name','Vendor')} ${sel('rf_cat','item_category','Cat')}
      <input id="rf_q" placeholder="search…" oninput="ECR.renderReport()" value="${ECR.getEl('rf_q')?ECR.getEl('rf_q').value:''}">
      <button class="btn" onclick="ECR.exportReport()">Export Excel</button>
      <span class="mono">${rows.length} items</span>
    </div>
    <div class="card"><div class="table-scroll"><table><thead><tr>${
      ECR.REPORT_COLS.map(c=>`<th class="${c[2]}">${c[1]}</th>`).join('')}</tr></thead><tbody>${
      rows.slice(0,2000).map(d=>'<tr>'+ECR.REPORT_COLS.map(c=>{
        const v=d[c[0]]; const cls=c[2]==='txt'?'txt':'';
        return `<td class="${cls}">${v==null?'':(c[2]===''?ECR.num(v).toLocaleString():v)}</td>`;
      }).join('')+'</tr>').join('')}</tbody></table></div>
      ${rows.length>2000?'<p class="mono">Showing first 2000; export for all.</p>':''}</div>`;
};
ECR.exportReport = function(){
  const rows=ECR.filteredData().map(d=>{ const o={}; ECR.REPORT_COLS.forEach(c=>o[c[1]]=d[c[0]]); return o; });
  const ws=XLSX.utils.json_to_sheet(rows), wb=XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb,ws,'Report'); XLSX.writeFile(wb,'ecom_report.xlsx');
};
```

Wire into `ECR.recompute` and tab switch:
```javascript
ECR.recompute = function(){ ECR.renderReport(); ECR.renderForecast && ECR.renderForecast(); ECR.renderPurchase && ECR.renderPurchase(); ECR.renderDashboard && ECR.renderDashboard(); };
```

- [ ] **Step 1b: Add inspector (Unmatched e-com report) + catalog-browse toggle**

Add an inspector `<select>` to the report header and two helpers. The Unmatched view reads
`ECR.S.unmatched` (built in Task 6). The catalog-browse toggle fetches master-only rows
server-side on demand (not preloaded). Add inside the report-header card markup:
`<select id="rf_view" onchange="ECR.renderReport()"><option value="working">Working set</option><option value="unmatched">⚠ Unmatched e-com items</option></select>`
`<label><input type="checkbox" id="rf_catalog" onchange="ECR.toggleCatalog()"> Show non-selling catalog items</label>`

```javascript
ECR.toggleCatalog = async function(){
  const on = ECR.getEl('rf_catalog').checked;
  if(on && !ECR.S._catalogLoaded){
    const msg=ECR.getEl('connMsg'); if(msg) msg.textContent='Loading non-selling catalog items…';
    // master items with no e-com data, server-side; cap to keep the tab responsive
    const cat = await ECR.fetchPaged('v_replen_base?select=*&has_ecom=is.false&vendor_open=eq.0&order=item_no.asc');
    ECR.S._catalog = cat; ECR.S._catalogLoaded = true;
    if(msg) msg.textContent='Loaded '+cat.length+' catalog-only items ✓';
  }
  ECR.renderReport();
};
ECR.reportSource = function(){
  if(ECR.getEl('rf_view') && ECR.getEl('rf_view').value==='unmatched') return ECR.S.unmatched||[];
  let base = ECR.S.DATA;
  if(ECR.getEl('rf_catalog') && ECR.getEl('rf_catalog').checked && ECR.S._catalog) base = base.concat(ECR.S._catalog);
  return base;
};
```
Then change `ECR.filteredData` to read `ECR.reportSource()` instead of `ECR.S.DATA`, and in
`renderReport` switch columns to an unmatched-specific set when the view is `unmatched`:
```javascript
ECR.UNMATCHED_COLS = [['item_no','Item','txt'],['asin','ASIN','txt'],['brand_name','Brand','txt'],
  ['amz_description','Description','txt'],['ecz_90d','ECZ 90d',''],['vcs_90d','VCS 90d',''],['total_on_hand_qty','Total OH','']];
// in renderReport: const cols = (ECR.getEl('rf_view')&&ECR.getEl('rf_view').value==='unmatched') ? ECR.UNMATCHED_COLS : ECR.REPORT_COLS;
// use `cols` for header + body + export. When unmatched and count>0, show a red banner:
//   "<N> e-com SKUs are not in mau_main_items — add them to the master upstream."
```

- [ ] **Step 2: Verify** — `preview_eval` `await ECR.loadData()`, `ECR.showTab('report')`, `preview_snapshot`.
Expected: filter dropdowns populated, table renders, item count shown. Change a brand filter → row count drops.
Switch inspector to "Unmatched e-com items" → shows `ECR.S.unmatched` rows + red banner (item 367906 present). Toggle "Show non-selling catalog items" → row count jumps as master-only items load.

- [ ] **Step 3: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: Report tab with filters and Excel export"
```

---

### Task 11: Forecast & Coverage tab

**Files:**
- Modify: `DE_Ecom_Replen.html` (add `ECR.renderForecast`)

- [ ] **Step 1: Add render with engine-derived columns + expandable open-PO detail**

```javascript
ECR.coverageRows = function(){
  const P=ECR.S.params;
  return ECR.filteredData().map(d=>{
    const oo=ECR.S.openByItem[d.item_no]||[];
    const wd=ECR.engine.weeklyDemand(d,P);
    const onO=ECR.engine.onOrder(d,oo);
    const wos=ECR.engine.weeksOfSupply(d,P,oo);
    const po=ECR.engine.suggestPO(d,P,oo);
    const eta=oo.filter(o=>o.expected_receipt_date).map(o=>o.expected_receipt_date).sort()[0]||'';
    return {d,oo,wd,onO,wos,po,eta,avail:ECR.engine.available(d,P),trend:ECR.engine.trend(d)};
  });
};
ECR.renderForecast = function(){
  const p=ECR.getEl('panel-forecast');
  if(!ECR.S.DATA.length){ p.innerHTML='<div class="card">Load data in Setup first.</div>'; return; }
  const rows=ECR.coverageRows().filter(r=>r.wd>0).sort((a,b)=>a.wos-b.wos);
  const fmt=n=>isFinite(n)?n.toLocaleString(undefined,{maximumFractionDigits:1}):'∞';
  const tp=t=>`<span class="pill ${t.label==='accelerating'?'good':t.label==='decelerating'?'bad':''}">${t.label}</span>`;
  p.innerHTML=`<div class="card"><h3>Forecast & Coverage</h3><span class="mono">${rows.length} items with demand · sorted by weeks of supply</span></div>
    <div class="card"><div class="table-scroll"><table><thead><tr>
    <th class="txt">Item</th><th class="txt">Brand</th><th class="txt">Description</th>
    <th>Weekly demand</th><th>Trend</th><th>Available</th><th>Vendor open</th><th>MP inbound</th>
    <th class="txt">Next ETA</th><th>Weeks supply</th><th>Status</th></tr></thead><tbody>${
    rows.slice(0,1500).map(r=>{
      const st=r.po.flags.indexOf('stockout')>=0?'<span class="pill bad">stockout risk</span>':
               r.po.flags.indexOf('overstock')>=0?'<span class="pill warn">overstock</span>':'<span class="pill good">ok</span>';
      return `<tr><td class="txt">${r.d.item_no}</td><td class="txt">${r.d.brand_name||''}</td>
        <td class="txt">${(r.d.amz_description||'').slice(0,40)}</td>
        <td>${fmt(r.wd)}</td><td>${tp(r.trend)}</td><td>${fmt(r.avail)}</td>
        <td>${fmt(r.onO.vendorOpen)}</td><td>${fmt(r.onO.marketplaceInbound)}</td>
        <td class="txt">${r.eta? String(r.eta).slice(0,10):'—'}</td><td>${fmt(r.wos)}</td><td>${st}</td></tr>`;
    }).join('')}</tbody></table></div></div>`;
};
```

- [ ] **Step 2: Verify** — `preview_eval` `ECR.showTab('forecast')`, `preview_snapshot`.
Expected: table sorted with lowest weeks-of-supply (stockout risk) at top; trend pills; vendor-open vs MP-inbound split columns.

- [ ] **Step 3: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: Forecast & Coverage tab"
```

---

### Task 12: Suggested Purchase tab (grouped by vendor, editable, export)

**Files:**
- Modify: `DE_Ecom_Replen.html` (add `ECR.renderPurchase`, `ECR.exportPO`)

- [ ] **Step 1: Add render grouped by vendor with editable qty**

```javascript
ECR.poRows = function(){
  return ECR.coverageRows().filter(r=>r.po.qty>0).map(r=>({
    item_no:r.d.item_no, brand:r.d.brand_name||'', desc:r.d.amz_description||'', asin:r.d.asin||'',
    vendor:r.d.vendor_name||'(no vendor)', wd:r.wd, wos:r.wos, avail:r.avail,
    onorder:r.onO.total, qty:r.po.qty, cost:ECR.num(r.d.direct_cost)
  }));
};
ECR.renderPurchase = function(){
  const p=ECR.getEl('panel-purchase');
  if(!ECR.S.DATA.length){ p.innerHTML='<div class="card">Load data in Setup first.</div>'; return; }
  const rows=ECR.poRows();
  const byV={}; rows.forEach(r=>{(byV[r.vendor]=byV[r.vendor]||[]).push(r);});
  const fmt=n=>ECR.num(n).toLocaleString(undefined,{maximumFractionDigits:0});
  const usd=n=>'$'+ECR.num(n).toLocaleString(undefined,{maximumFractionDigits:0});
  const totUnits=rows.reduce((s,r)=>s+r.qty,0), totCost=rows.reduce((s,r)=>s+r.qty*r.cost,0);
  let html=`<div class="card"><h3>Suggested Purchase</h3>
    <span class="mono">${rows.length} SKUs · ${fmt(totUnits)} units · ${usd(totCost)} · ${Object.keys(byV).length} vendors</span>
    <button class="btn primary" onclick="ECR.exportPO()">Export PO Excel</button></div>`;
  Object.keys(byV).sort().forEach(v=>{
    const vr=byV[v]; const vCost=vr.reduce((s,r)=>s+r.qty*r.cost,0);
    html+=`<div class="card"><h4 class="txt">${v} <span class="mono">(${vr.length} SKUs · ${usd(vCost)})</span></h4>
      <div class="table-scroll"><table><thead><tr><th class="txt">Item</th><th class="txt">Desc</th>
      <th>Weekly</th><th>Wks supply</th><th>Avail</th><th>On order</th><th>Suggested qty</th><th>Ext cost</th></tr></thead><tbody>${
      vr.sort((a,b)=>a.wos-b.wos).map(r=>`<tr><td class="txt">${r.item_no}</td><td class="txt">${r.desc.slice(0,40)}</td>
        <td>${r.wd.toFixed(1)}</td><td>${isFinite(r.wos)?r.wos.toFixed(1):'∞'}</td><td>${fmt(r.avail)}</td>
        <td>${fmt(r.onorder)}</td><td>${fmt(r.qty)}</td><td>${usd(r.qty*r.cost)}</td></tr>`).join('')}</tbody></table></div></div>`;
  });
  p.innerHTML=html;
};
ECR.exportPO = function(){
  const rows=ECR.poRows().map(r=>({Vendor:r.vendor,Item:r.item_no,ASIN:r.asin,Brand:r.brand,
    Description:r.desc,'Weekly demand':+r.wd.toFixed(1),'Weeks supply':isFinite(r.wos)?+r.wos.toFixed(1):'',
    Available:r.avail,'On order':r.onorder,'Suggested qty':r.qty,'Unit cost':r.cost,'Ext cost':+(r.qty*r.cost).toFixed(2)}));
  const ws=XLSX.utils.json_to_sheet(rows), wb=XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb,ws,'Suggested PO'); XLSX.writeFile(wb,'suggested_po.xlsx');
};
```

- [ ] **Step 2: Verify** — `preview_eval` `ECR.showTab('purchase')`, `preview_snapshot`.
Expected: vendor-grouped cards, per-vendor SKU count + cost, suggested qty > 0 rows only, header totals; Export PO produces a file (verify via console no-error).

- [ ] **Step 3: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: Suggested Purchase tab grouped by vendor + export"
```

---

### Task 13: Dashboard (KPIs + charts)

**Files:**
- Modify: `DE_Ecom_Replen.html` (add Chart.js CDN; `ECR.renderDashboard`)

- [ ] **Step 1: Add Chart.js + KPI render**

Add to `<head>`: `<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>`

```javascript
ECR.renderDashboard = function(){
  const p=ECR.getEl('panel-dashboard');
  if(!ECR.S.DATA.length){ p.innerHTML='<div class="card">Load data in Setup first.</div>'; return; }
  const cov=ECR.coverageRows();
  const withDemand=cov.filter(r=>r.wd>0);
  const stockout=withDemand.filter(r=>r.po.flags.indexOf('stockout')>=0);
  const po=ECR.poRows();
  const totUnits=po.reduce((s,r)=>s+r.qty,0), totCost=po.reduce((s,r)=>s+r.qty*r.cost,0);
  const usd=n=>'$'+ECR.num(n).toLocaleString(undefined,{maximumFractionDigits:0});
  const kpi=(l,v)=>`<div class="card" style="display:inline-block;min-width:160px"><div class="mono" style="color:var(--muted)">${l}</div><div class="metric-val" style="font-size:26px">${v}</div></div>`;
  // vendor reorder $ for chart (top 10)
  const byV={}; po.forEach(r=>{byV[r.vendor]=(byV[r.vendor]||0)+r.qty*r.cost;});
  const top=Object.entries(byV).sort((a,b)=>b[1]-a[1]).slice(0,10);
  p.innerHTML=`<div>${kpi('Items w/ demand',withDemand.length.toLocaleString())}
    ${kpi('Stockout risk',stockout.length.toLocaleString())}
    ${kpi('SKUs to reorder',po.length.toLocaleString())}
    ${kpi('Reorder units',totUnits.toLocaleString())}
    ${kpi('Reorder cost',usd(totCost))}</div>
    <div class="card"><h3>Top vendors by reorder $</h3><canvas id="vChart" height="120"></canvas></div>`;
  new Chart(ECR.getEl('vChart'),{type:'bar',data:{labels:top.map(t=>t[0]),
    datasets:[{label:'Reorder $',data:top.map(t=>Math.round(t[1])),backgroundColor:'#0969da'}]},
    options:{plugins:{legend:{display:false}}}});
};
```

- [ ] **Step 2: Verify** — `preview_eval` `ECR.showTab('dashboard')`, `preview_snapshot` + `preview_screenshot`.
Expected: 5 KPI cards with non-zero values, a bar chart of top vendors.

- [ ] **Step 3: Commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: Dashboard KPIs + top-vendor chart"
```

---

## Phase 5 — Polish & verification

### Task 14: Optional param cloud sync + final QA pass

**Files:**
- Modify: `DE_Ecom_Replen.html`

- [ ] **Step 1: Add optional shared-param sync to app_settings** (non-secret only)

```javascript
ECR.syncParamsToCloud = async function(){
  const {url,key}=ECR.sbCreds(); if(!url||!key) return;
  const body=[{key:'ecr_params', value:ECR.S.params, updated_at:new Date().toISOString(), updated_by:(location.host||'browser')}];
  await fetch(url+'/rest/v1/app_settings?on_conflict=key',{method:'POST',
    headers:{apikey:key,Authorization:'Bearer '+key,'Content-Type':'application/json','Prefer':'resolution=merge-duplicates,return=minimal'},
    body:JSON.stringify(body)});
};
```
Add a "Sync params to cloud" button in Setup calling it (best-effort; ignore failure).

- [ ] **Step 2: Full QA pass via preview**

Run the full flow with `preview_eval`/`preview_snapshot`:
1. `?selftest=1` → `preview_console_logs` → all PASS.
2. Setup → key → Test connection → Connected.
3. Load data → ~20,689 items.
4. Each tab renders; toggle dark mode → `preview_screenshot` (no hardcoded-hex contrast issues).
5. Change `target_weeks` → Save → Suggested Purchase totals change (recompute fires).
6. `preview_console_logs` → no errors.

- [ ] **Step 3: Final commit**
```bash
git add DE_Ecom_Replen.html
git commit -m "DE Ecom Replen: optional param cloud sync + QA polish"
```

- [ ] **Step 4: Deploy notes** (do NOT auto-deploy)

Vercel: this is a separate static file. Either add to an existing project or new project; serve `DE_Ecom_Replen.html`. Confirm with Mau before any deploy. Push to GitHub only when Mau says so.

---

## Self-Review (completed by plan author)

**Spec coverage:**
- §3.1 `mau_main_items` as BASE + §3.2b `v_replen_base` view + smart load → Task 1 (view DDL + grants) + Task 6 (actionable working-set load, server-side catalog browse). ✓
- §3.2–3.5 tables → Task 1 (DDL) + Task 2 (bootstrap). ✓
- §4 forecast (smoothed/recent/conservative + trend, seasonality deferred) → Task 7. Seasonality intentionally not built (spec defers it; `ecom_sales_history` table created, unused). ✓
- §5 coverage, two-leg lead, reorder math, flags, AWD double-count guard → Tasks 8–9 (available uses leaf fields `fba_total`+`awd_available`, onOrder uses `*_inbound` — no overlap). ✓
- §5 time-phasing (ETA) → surfaced in Task 11 (Next ETA column); full before/after-stockout phasing is represented by showing ETA + stockout flag. NOTE: deeper date-vs-stockout gating is simplified to "show ETA + flag" per simple-mode default; acceptable for v1. ✓ (documented limitation)
- §6 tabs (Setup/Report/Forecast/Purchase/Dashboard) → Tasks 3,4,10,11,12,13. ✓
- §6 Report: catalog-browse toggle + "⚠ Unmatched e-com items" data-quality report → Task 6 (builds `S.unmatched`) + Task 10 Step 1b. ✓
- §7 Setup secrets + params persisted (`ecr_` keys), future SaaS fields inert → Task 4. ✓
- §8 read-only, export-only, no pipeline build → respected (only one-time bootstrap script writes, via service_role). ✓
- §9 risks: UPC join (we key on item_no; master is base), cumulative windows (rates use window length), single-snapshot (no fake seasonality), batch 1000 (loader), 337k load avoided via smart load. ✓

**Decisions reflected (master-as-base revision):**
- Baseline = `mau_main_items` (337k), strict left-join of e-com data via `v_replen_base`.
  model/color/size enrichment now comes from the base view (no longer a deferred cut).
- E-com items absent from the master (e.g. 367906) are excluded from `S.DATA` and surfaced in
  the Unmatched report (Task 6 + Task 10 Step 1b) — the data-quality governance Mau asked for.
- 337k never fully loaded: actionable set (~20k) loaded fully; non-selling catalog browsed
  server-side on toggle.

**Engine input note:** `v_replen_base` exposes every field the engine reads
(`ecz_fba_total`, `ecz_awd_available`, `vcs_fba_total`, `vcs_awd_available`, `*_inbound`,
`ecom_pickable_bins`, `pickable_sunrise_inventory`, `tpl_inventory_total`, sales windows,
`direct_cost`), so `S.DATA` rows from the view are valid engine inputs unchanged. ✓

**Placeholder scan:** none — all steps contain runnable code/SQL/commands.

**Type consistency:** `ECR.engine.*` signatures in the contract match their definitions and call sites (`weeklyDemand(item,params)`, `onOrder(item,openOrders)`, `suggestPO(item,params,openOrders)`, `available(item,params)`). Params object keys (`model,trendOn,supplierLead,fcInbound,target,safety,include3pl,casePack`) consistent between `buildParams` (Task 4) and engine (Tasks 7–9). ✓
