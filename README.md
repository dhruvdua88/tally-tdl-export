# Tally TDL Voucher / Ledger Export

**🌐 Download page (non-technical install guide): https://dhruvdua88.github.io/tally-tdl-export/**

Fast, complete export of vouchers, ledgers, inventory, bills, cost-centres and
masters from **Tally.ERP 9 / TallyPrime** — entirely in **TDL** (Tally
Definition Language). No external tools, no ODBC, no scripting.

Two ready-to-load add-ons (pick per need):

| File | What you get | Use when |
|---|---|---|
| **`new/DDVoucherExport_v17.tdl`** | Clean **CSV** exports via a *DD Data Export* menu: Accounting, Accounting-Delta, Inventory, Bill-wise, Cost-Centre, Ledger Master — with headers, Dr/Cr, GST classification, optional AlterID delta + date-range scope. | You want readable CSVs to open in Excel / feed a recon. |
| **`new/DDLoaderExport_v19.tdl`** | **14 tables** (7 `mst_*` + 7 `trn_*`) exported as `.tsv` in the **exact format of [tally-database-loader](https://github.com/dhananjay1405/tally-database-loader)** (same column names, order, value-transforms, GUID linkage). | You want a normalized, DB-loadable dataset / loader-compatible files without running the Node loader. |

Everything is a **fork** of an original working TDL — the original is kept
untouched in `original/`.

---

## Install (load a TDL in Tally)

**TallyPrime**
1. `F1 (Help) → TDL & Add-On → F4 (Manage Local TDLs)`.
2. *Load selected TDL files on startup* → **Yes**.
3. Add the full path to the `.tdl` you want (e.g. `DDVoucherExport_v17.tdl` or `DDLoaderExport_v19.tdl`) → **Accept**.
4. A new item appears on the **Gateway of Tally** (*DD Data Export* / *DD Loader Export*).

**Tally.ERP 9**: `F12 → Product & Features → F4 (Manage Local TDLs)` → add the path.

> **Create the folder `C:\TallyExport` before your first export.** All files are
> written there. (A bare filename would land in Tally's own program folder,
> which Windows protects — the file then silently disappears into a VirtualStore
> copy. An explicit folder avoids that.)
> To use a different folder, edit the `DDExportPath` default at the top of the
> `.tdl` and keep the trailing backslash:
> `[Variable: DDExportPath]  Default : "D:\MyExports\"`
>
> Set the period with `F2` / `Alt+F2` before exporting — exports honour the
> current company and active period.

You can load **both** files at once (their menu names and definitions don't clash).

---

## A. `DDVoucherExport_v17.tdl` — CSV exports

**Gateway of Tally → DD Data Export** (key `D`) → submenu:

| Item (key) | Output | One row per |
|---|---|---|
| Accounting Vouchers (`A`) | `LedgerVoucher.csv` | ledger entry — 29 cols: dates, ids, party, GST duty-head classification, bill refs, cost centres, bank/instrument, Dr/Cr |
| Accounting Vouchers – Delta (`E`) | `LedgerVoucher.csv` + `.maxalter.txt` | changed ledger entry (AlterID cursor) |
| Inventory Vouchers (`I`) | `InventoryVoucher.csv` | stock line + HSN / GST-rate from item master + batches |
| Bill-wise (`B`) | `Billwise.csv` | bill allocation (amount / type / credit period → aging) |
| Cost-Centre (`C`) | `CostCentre.csv` | cost-centre allocation (category / centre / amount) |
| Ledger Master (`L`) | `LedgerList.csv` | ledger master (25 cols) |

**Delta / incremental:** the *Delta* item exports only vouchers with
`AlterID > DDFromAlterID` (default `0` = full) and writes the high-water mark to
`LedgerVoucher.maxalter.txt`. Automation: `SET : DDFromAlterID : <lastMax>` before
the call; consumer merges by GUID.

**Date-range scope:** set `DDFromDate` / `DDToDate` to restrict any export to a
period without changing F2 (both empty = no restriction).

## B. `DDLoaderExport_v19.tdl` — tally-database-loader format

**Gateway of Tally → DD Loader Export** (key `L`) → submenu with one item per
table + **Export ALL** (`X`). Produces (TAB-delimited `.tsv`, loader-exact):

`mst_group`, `mst_ledger`, `mst_vouchertype`, `mst_stock_item`, `mst_godown`,
`mst_cost_category`, `mst_cost_centre`, `trn_voucher`, `trn_accounting`,
`trn_inventory`, `trn_cost_centre`, `trn_bill`, `trn_bank`, `trn_batch`.

Faithful to the loader: same column names/order, per-`type` value transforms
(amount sign-fix, `YYYY-MM-DD` dates, `ñ` empty-date NULL sentinel, 1/0
logicals), and every child table repeats the parent voucher `guid`. Full mapping
in [`docs/LOADER_SCHEMA.md`](docs/LOADER_SCHEMA.md).

---

## Large data (100k – 1,000,000+ rows)

The `.tdl` files use an in-Tally `WRITE FILE LINE` loop — great for an audit
year. For very large / recurring loads see
[`docs/METHODS_LARGE_DATA.md`](docs/METHODS_LARGE_DATA.md): the ranked methods and
the fastest architecture (external **HTTP-XML puller**, `TYPE=Collection`,
batched 5–10k vouchers by date — i.e. run `tally-database-loader` itself). An
in-Tally engine-streamed prototype is in `new/VchDumpReport_v18.tdl`.

---

## Verify field names on your build

A few TDL method names vary by Tally release (flagged `[verify]` in the docs).
Load **`new/FieldProbe_v10.tdl`**, run *DD Field Probe* once → it writes
`Probe.csv` listing each candidate field's value for a few sample objects.
Populated = the method works on your build; blank for every object = adjust that
one line.

---

## Repository layout

```
original/    original.tdl                     (untouched source)
new/         DDVoucherExport_v17.tdl          (CSV exports — recommended)
             DDLoaderExport_v19.tdl           (tally-database-loader format)
             VchDumpReport_v18.tdl            (engine-export prototype, large data)
             FieldProbe_v10.tdl               (one-time field verifier)
             *_v1..v16.tdl                    (focused single-purpose references)
docs/        TDL_REFERENCE.md                 (TDL export cheat-sheet)
             ANALYSIS.md                      (original-vs-new + full version log)
             LOADER_SCHEMA.md                 (tally-database-loader schema map)
             METHODS_LARGE_DATA.md            (export-method ranking + verdict)
```

## Why it's faster than a naive export

- Streams rows with `WRITE FILE LINE` to an **ASCII** file instead of cell-by-cell
  `WRITE CELL` into an Excel/OLE target (the slowest path).
- Fetches everything at **collection level** in one pass; drops the unused
  `Is ODBC Table` indexes and heavy `Fetch *, AllLedgerEntries.*` collections
  from the original.

## Credits & license

- Output schema and value-transform conventions in `DDLoaderExport_v19.tdl`
  faithfully follow **[dhananjay1405/tally-database-loader](https://github.com/dhananjay1405/tally-database-loader)** (MIT). Please star that project.
- This repository: MIT License (see `LICENSE`).

*Not affiliated with Tally Solutions Pvt. Ltd. "Tally" is a trademark of its owner.*
