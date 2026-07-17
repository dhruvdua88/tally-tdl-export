# Tally Voucher-Export TDL — Analysis & Improvement Log

Source (untouched): `original/original.tdl` = Drive file **"new tdl - 05102019WORKING.txt"**
Improved fork: `new/VoucherExport_v1.tdl`

## What the original does
- Adds 2 items to *Gateway of Tally* menu.
- **Export Ledger Voucher** → walks `LVoucher` (Voucher → All Ledger Entries), writes 12 cols to `LedgerVoucher.xlsx` via `OPEN FILE Excel : Write` + `WRITE CELL`.
- **Export Ledger List** → walks `Ledger`, writes 9 cols to `LedgerList.xlsx`.
- Dead code: `Export Total Voucher` function (never wired to menu); `SVoucher` (inventory) and `RTSAllVouchers` collections (never called).

## Problems found (mapped to the 3 goals)

### 1. Accuracy — not ALL fields
Only ~12 voucher fields exported. Missing at voucher level: EffectiveDate, MasterID, AlterID, IsCancelled, IsOptional, PlaceOfSupply (+ GST breakup: HSN, rate, CGST/SGST/IGST/taxable; bill allocations; cost-centre allocations; bank allocations). Inventory lines never exported (SVoucher dead).

### 2. Format
- **No header row** — data starts at row 1, columns unlabelled.
- `Amount` held in a **String** variable → precision/format risk; written via intermediate var.
- `isdeemedpositive` written raw (True/False) instead of readable **Dr/Cr**.
- Date pushed through a variable.

### 3. Speed
- `Is ODBC Table: Yes` on collections that are only WALKed → builds an ODBC index that is never used = wasted cost.
- `RTSAllVouchers` fetches `*, AllLedgerEntries.*, LedgerEntries.*` — extremely heavy, and unused.
- Per-row copy to intermediate variables (`Set: TotalAmount`, `dpm`, `date1`) before writing → avoidable work per row.
- `WRITE CELL` cell-by-cell into an Excel target is the real bottleneck for large data (see v2 plan).

## v1 changes (this fork)
- Header row on every sheet.
- Comprehensive voucher fields (17 cols incl. masterid/alterid/effective date/flags/place of supply) + **Dr/Cr** column + numeric `$Amount` written directly.
- Wired a real **Inventory export** (`InventoryVoucher.xlsx`) using the previously-dead SVoucher walk.
- Removed `Is ODBC Table` from walk-only collections; deleted `RTSAllVouchers` and unwired `Export Total Voucher`.
- Removed per-row intermediate-variable copies — write `$field` directly.
- One action per numbered line (valid TDL); column stepping via explicit `INCREMENT`.

## v2 changes (fast path) — `new/VoucherExport_v2.tdl`
Research verdict: `WRITE CELL` Excel loop is the slowest export at scale; dominant cost is collection re-gathering.
- Switched to **`WRITE FILE LINE` → ASCII CSV** (one concatenated line/row, single flush) instead of per-cell Excel writes.
- Perf levers: **`Keep Source: Yes`** + collection-level **`Fetch`** (one gather pass, cached); `Is ODBC Table` fully removed.
- CSV-safe: `CsvEsc` helper quotes + escapes embedded quotes so commas/quotes in narration/names never break rows.
- Adds **BillRefs** (`$$FullList:BillAllocations:$Name`) and **CostCentres** columns; header row; UniversalDate (YYYYMMDD); numeric `$$Number:$Amount`; clean Dr/Cr.
- Keeps ALL vouchers incl. cancelled (Cancelled column flags them) — no silent drops.
- v1 (.xlsx) retained for users needing a native workbook.

## Iteration log (15-min loop)
- **v1** — xlsx, headers, all-fields, inventory export, dead-code/ODBC removed.
- **v2** — fast CSV, perf levers, bill/cost-centre, CSV-escaping.
- **v3** — +bank/instrument cols (TxnType/InstrumentNo/InstrumentDate/BankName), +IsInvoice/EnteredBy. Static-checked: all confirmed BankAllocations methods.
- **v4** — fast Inventory CSV (`InventoryExport_v4.tdl`): one row/stock line, qty/rate/amount/addlamount/discount + godown/order/tracking + batches, ASCII CSV + Keep Source. Replaces the slow v1 xlsx inventory path.
- **v5** — GST/ledger classification cols (LedgerGroup, GSTDutyHead, LedgerGSTIN) via cross-object master lookup `$Method:Ledger:$LedgerName`, staged into temp vars (avoids `:` arg-split; caught+fixed a line-ordering bug where labels 1271-1273 sorted after 190). Tax rows now identifiable → pivot tax by head. GSTDutyHead casing flagged [verify].
- **v6** — Inventory CSV (`InventoryExport_v6.tdl`) + HSN/GST from Stock Item master: HSNCode, HSNDesc, GSTRate(IGST%), Taxability, BaseUnit via `$InfGST*:StockItem:$StockItemName` staged into temp vars. Each stock line now GST-ready for e-invoice/return reco. InfGST* method names flagged [verify] (may sit under GstDetails[Last] on some builds).
- **v7** — Ledger Master CSV (`LedgerMasterExport_v7.tdl`): fast path replacing slow v1 xlsx ledger master. 25 cols — identity/group, GSTIN/regtype/supplytype/PAN, address(FullList)/state/country/pin, contact/phone/mobile/email, bank/ac/IFSC, credit period, opening/closing/cashflow. Method existence flagged [verify] (names case-insensitive in TDL, so only presence matters per build).
- **v8 — RECOMMENDED single file** (`DDVoucherExport_v8.tdl`): consolidates best of v5+v6+v7 behind ONE "DD Data Export" Gateway item → submenu (Accounting / Inventory / Ledger Master CSV). Shared CsvEsc, 5 uniquely-named collections, all fast-path. v1-v7 kept as focused references; **load v8 in Tally**.
- **v9 — AlterID delta/incremental** (`VoucherExport_v9_delta.tdl`): system var `DDFromAlterID` (default 0=full) + filter `$AlterID > ##DDFromAlterID`; tracks high-water mark → `LedgerVoucher.maxalter.txt` for the next run. Automation feed = export only changed-since vouchers, consumer merges by GUID. Static-checked: filter in voucher context, IF/SET ordering, 2nd file write clean.
- **v10 — Field probe / diagnostic** (`FieldProbe_v10.tdl`): emits tall CSV (ObjectType,ObjectName,Field,Value) for first 5 vouchers/ledgers/stock-items, listing every `[verify]` method. Run ONCE on the target build → populated=method OK, all-blank=wrong name. Closes the verification loop (was blocked by no-Tally-on-mac). Assumption: OPEN FILE target is ambient across CALL to ProbeRow (standard TDL); fallback = inline writes.
- **v11 — Bill-wise allocation CSV** (`BillwiseExport_v11.tdl`): explodes Voucher→AllLedgerEntries→BillAllocations to ONE row per bill ref with amount/type/credit-period + voucher/ledger context. Replaces names-only `$$FullList` flattening → real receivables/payables aging keyed by (Ledger,BillRef). Static-checked: 3-level walk chain, parent context fetched each level. `$BillDate` [verify].
- **v12 — DEFINITIVE single file** (`DDVoucherExport_v12.tdl`): supersedes v8. One "DD Data Export" Gateway item → submenu of 5 exports (Accounting / Accounting-Delta / Inventory / Bill-wise / Ledger Master). Folds v5+v6+v7+v9+v11. Shared CsvEsc, DDFromAlterID var+DDDeltaFlt filter, 10 uniquely-named collections, all fast-path. **This is the file to load in Tally.** v1-v11 kept as focused references; v10 probe run separately to confirm [verify] fields.
- **v13 — User guide** (`README.md`): end-user doc — install steps (Manage Local TDLs), menu→output map, column dictionary per file, delta-pipeline recipe, probe workflow, why-faster, repo layout. (Tool is code-complete; this closes the usability gap.)
- **v14 — Cost-centre allocation CSV** (`CostCentreExport_v14.tdl`): explodes Voucher→AllLedgerEntries→CategoryAllocations→CostCentreAllocations to ONE row per allocation (Category, CostCentre, Amount, DrCr + voucher/ledger context) → department/project costing. Replaces names-only flattening. [verify]: uses category route (standard TallyPrime); flat `Cost Centre Allocations` walk needed if a build stores centres directly under the LE.
- **v15 — DEFINITIVE single file** (`DDVoucherExport_v15.tdl`): supersedes v12. Folds cost-centre (v14) into the submenu → SIX exports (Accounting / Accounting-Delta / Inventory / Bill-wise / Cost-Centre / Ledger Master). Built by cp v12 + inject fn/collections/menu-item.
- **v16 — DEFINITIVE + date-range** (`DDVoucherExport_v16.tdl`): supersedes v15. Adds optional programmatic period scope via `DDFromDate`/`DDToDate` vars + `DDDateFlt` filter (`$$IsEmpty` guarded → empty = no restriction, F2 period still applies) on all voucher-root collections. Backlog exhausted.
- **v17 — DEFINITIVE (menu-action fix)** (`DDVoucherExport_v17.tdl`): CORRECTNESS pass. v8-v16 opened exports with plain `[Menu] Item : Call : fn` — an UNVERIFIED menu-item action. v17 switches submenu items to the original's PROVEN `Key Item + <key> + CALL : Function` form (keys D→menu, A/E/I/B/C/L→exports). No feature change; de-risks whether the recommended file actually fires in Tally.
- **v18 — LARGE-DATA fast path** (`VchDumpReport_v18.tdl`) + `docs/METHODS_LARGE_DATA.md`: research-driven. Native **Report-export engine** (`EXPORT` action → comma-delimited CSV) instead of the `WRITE FILE LINE` loop → engine-streamed, flat RAM for 100k+ rows. Perf: no Keep Source, no cross-object lookups, single-pass route walk, server-side cancelled/optional filter. Methods doc ranks all 5 export methods; verdict for 500k-1M+ = external HTTP-XML puller `TYPE=Collection` batched by date (= tally-database-loader); v18 is the best IN-Tally option. `$$SysName:ASCIICommaDelimited` + CSV-width truncation flagged [verify]. **v17 stays the recommended load for moderate data; v18/HTTP-puller for large.**
- **v19 — LOADER-FORMAT export** (`DDLoaderExport_v19.tdl`) + `docs/LOADER_SCHEMA.md`: replicates tally-database-loader's output IN PURE TDL (no external puller). Submenu "DD Loader Export" → 14 tables (7 masters + 7 transactions) each to a `.tsv` with loader-exact column names/order/expressions. Key finding integrated: loader's per-`type` value wrappers (text/logical/date/number/amount/quantity/rate) reproduced as 7 formatter fns `LdrT/L/D/N/A/Q/R` character-for-character (amount `$$IsDebit` sign-fix, date `$$PyrlYYYYMMDDFormat`+`ñ`(chr241) sentinel, TAB delim, child tables repeat voucher GUID). Static fixes: formatter fns must be space-free for `$$` invocation. [verify]: `$$NumValue`, `$$PyrlYYYYMMDDFormat`, `$LedGSTRegDetails[Last]` are loader-sourced. **This is the answer to "same format as tally-database-loader, using TDL".**

## Known open items / v2+ backlog
- [DONE v2] fast CSV via WRITE FILE LINE + Keep Source (research-confirmed).
- [DONE v2] bill-wise + cost-centre allocation columns.
- [DONE v3] bank/instrument allocation columns.
- [DONE v4] fast inventory CSV.
- **GST detail** (top remaining accuracy gap): per-voucher CGST/SGST/IGST/Cess by aggregating tax ledger lines (portable, version-agnostic); HSN + GST rate from Stock Item master (`InfGSTHSNCode`,`InfGSTIGSTRate`) joined onto inventory rows.
- **AlterID delta export**: `Filter : $AlterID > ##LastAlterID` to export only changed vouchers.
- **Period params**: expose SVFromDate/SVToDate so export honours a chosen date range.
- Consolidate v1–v4 into ONE recommended .tdl with a submenu.
- Verify exact identifiers against Tally build ([verify] items: BilledQty, EnteredBy, RegistrationType).

## Testing constraint
No Tally install on this machine → cannot runtime-test the TDL here. Iteration is **static**: correctness against the official TDL reference + structural/perf review each loop. Runtime validation must be done by loading the `.tdl` in Tally (F1 > TDL & Add-Ons / Manage Local TDLs).
