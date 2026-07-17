# TDL Expert Reference (voucher/ledger export)

Condensed from deep research. Full field tables + citations below. Gold concrete reference:
**`github.com/dhananjay1405/tally-database-loader` → `tally-export-config.yaml`** (production-verified TDL export routes for every voucher sub-collection). Read it before extending.

## Speed verdict (ranked)
1. **Native Report export → XML / ASCII(CSV)** via export engine (`Alt+E`, `EXPORT` action, or HTTP `EXPORT` XML to port 9000). Fastest for bulk; one streamed pass.
2. **`WRITE FILE LINE` to Text/ASCII** inside a Function (concat whole row, write once). ASCII = single-byte, ~half the I/O of Unicode default.
3. **`WRITE ROW`** (whole row per call) > per-cell `WRITE CELL`.
4. **`OPEN FILE Excel : Write` + `WRITE CELL`** cell-by-cell = SLOWEST at scale. OK only for small formatted sheets.

**The dominant cost is collection re-gathering, not the writer.** Real 10×–100× wins:
- `Keep Source : Yes` — gather once, cache; avoid re-gather per repeated line.
- `Fetch` every method you'll use AT COLLECTION LEVEL (one pass). Push logic into `Compute`, not repeated fields.
- `Search Key` + `$$CollectionFieldByKey` — O(1) object lookup (not at menu level).
- Drop `Is ODBC Table` on collections only WALKed (ODBC surface is a slow fetch source; mark external ones `Client Only`).
- Delta export: `Filter : $AlterID > ##LastAlterID`.

## Sigils
`$method` (field on current object) · `$$func:arg` (built-in) · `##var` (variable, e.g. `##SVFromDate`) · `#fld` (modify) · `@@name` (global System:Formula) · `..method` (parent scope in a walk).

## Voucher methods (confirmed ✔ / [verify])
Header: `Date`✔ `EffectiveDate` `VoucherNumber`✔ `VoucherTypeName`✔ `Guid`✔ `MasterID`✔ `AlterID`✔(delta key) `Reference`✔ `ReferenceDate`✔ `Narration`✔ `EnteredBy` `ClassName` `VoucherKey`[v].
Party/GST: `PartyLedgerName`✔ `PartyName` `PlaceOfSupply`✔ `StateName` `RegistrationType`[v] `PartyGSTIN`[v] `VchGSTClass` `TaxUnitName`[v].
Flags (Yes/No): `IsInvoice`✔ `IsCancelled`✔ `IsOptional`✔ `IsPostDated` `IsDeleted` `IsVoid` `IsOnHold` `HasCashFlow` `HasDiscounts` `DiffActualQty`.
Type test helpers: `$$IsAccountingVch:$VoucherTypeName` `$$IsInventoryVch:…` `$$IsOrderVch:…`.

## Sub-collections to WALK inside a voucher
- **All Ledger Entries** (use for EXPORT; `Ledger Entries` only for IMPORT): `LedgerName`✔ `Amount`✔(signed) `IsDeemedPositive`✔(Yes⇒Dr). Dr/Cr = `if $IsDeemedPositive then "Dr" else "Cr"` or `$$IsDebit:$Amount`.
  - `…BillAllocations`: `Name`(billref) `Amount` `BillType`(New/Agst/Advance/OnAccount) `$$Number:$BillCreditPeriod`.
  - `…CostCentreAllocations`: `Name` `Amount`.
  - `…CategoryAllocations`→`.CostCentreAllocations`: `Category` `Name` `Amount`.
  - `…BankAllocations`: `TransactionType` `InstrumentDate` `InstrumentNumber` `BankName` `Amount` `BankersDate` (parent ledger via `..LedgerName`).
- **All Inventory Entries**: `StockItemName`✔ `ActualQty`✔ `BilledQty`[v] `Rate`✔ `Amount`✔ `AddlAmount`✔ `Discount`✔ `GodownName`✔ `TrackingNumber`✔ `OrderNo`✔ `OrderDueDate`✔.
  - `…BatchAllocations`: `BatchName` `ActualQty` `Amount` `GodownName` `DestinationGodownName` `TrackingNumber`.
- **GST split**: no single header field. Portable = tax-as-ledger-lines (walk AllLedgerEntries, bucket by tax ledger duty head). HSN/rate live on **Stock Item master**: `InfGSTHSNCode` `InfGSTHSNDescription` `InfGSTIGSTRate` `InfGSTTaxablility`. Per-line GST sub-collection names are release-dependent [verify].

## Built-in $$ (export/format)
`$$IsEmpty` `$$String:x[:Fmt]` (`:UniversalDate`→YYYYMMDD) `$$Number:x` `$$IsDebit`/`$$IsDr`/`$$IsCr` `$$FullList:Coll:$M` `$$NumItems:Coll` `$$CollAmtTotal:Coll` `$$Total:$M` `$$StringFindAndReplace:s:a:b` `$$Sprintf` `$$ZeroFill:s:n` `$$Upper`.

## Filtering
Period: `##SVFromDate`/`##SVToDate` (report active period bounds the voucher collection; `Set:` to override). Company: `ChildOf : ##SVCurrentCompany`. Cancelled/optional: `Filter` on System:Formula `NOT $IsCancelled` / `NOT $IsOptional`. Voucher type: `Type : Vouchers : <Name>` + `Child Of : $$VchTypeSales`.

## Procedural keywords
`SET` `LOG`(calc.log) `IF/ELSE/END IF` `WALK COLLECTION/END WALK` `INCREMENT/DECREMENT` `OPEN FILE : "f" : Text|Excel : Read|Write : ASCII|Unicode` `WRITE CELL`/`WRITE ROW`/`WRITE FILE LINE` `CLOSE TARGET FILE` `RETURN`. One action per numbered line.

## Gotchas CONFIRMED at runtime in Tally (not theory)
- **No doubled-quote escape inside a string literal.** Writing `""""` to mean one `"` fails with
  **`Cannot understand. Bad formula!`**. There is no `\"` either. To emit a quote character, build it:
  `SET : q : $$StrByCharCode:34` and concatenate `##q`. (This broke `CsvEsc` in v2–v17; fixed by
  staging `q`/`qq` vars.)
- **Function names invoked with `$$` must have no spaces.** `[Function: Ldr T]` can't be called as
  `$$Ldr T:x`. Spaced names are fine only for `CALL : My Function`.
- **Line labels sort numerically, not textually.** Inserting `1271` between `127` and `128` runs it
  after `190`. There is no integer between 127 and 128 — renumber or stage earlier in the block.
- Precaution (unconfirmed): when passing a call as an argument to a user function, prefer staging it
  in a variable or using a zero-arg wrapper, so the `:` splitting can't be misread.

## Import vs export gotcha
`All Ledger Entries` for reading/export; `Ledger Entries` for create/import (else "Voucher Totals do not match").
