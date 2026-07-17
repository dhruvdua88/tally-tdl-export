# Optimizing large-data / all-fields export from Tally — methods & verdict

Goal: flat "one row per ledger entry, all voucher-header + entry + bill/bank/cost-centre
fields", **100k – 1,000,000+ vouchers**. Evidence from official Tally XML docs + the
production loader `dhananjay1405/tally-database-loader` (source-read). `[verify]` = build-dependent.

## Ranking (throughput, largest data first)

| # | Method | Mechanism | Verdict @ 500k+ rows |
|---|---|---|---|
| **1** | **HTTP-XML pull, `TALLYREQUEST=Export` + `TYPE=Collection`** | External client → `:9000`; server-side `[Collection]` with `FETCH` of nested routes; Tally streams XML | **WINNER.** Loader README: **2× throughput, −40% RAM** vs report path. Batched by date → bounded memory, restartable. |
| 2 | HTTP-XML pull, `TYPE=Data` | Same channel, export engine renders a Form/Part/Line report over the socket | Fast, robust; delimited rows come out ready (no client XML-walk). Loader's YAML default. |
| 3 | **In-Tally native Report export** (`EXPORT` action / Alt+E → CSV/SDF/XML) | Same engine, triggered inside Tally, writes local file | Engine-streamed; great up to low-hundred-k. Asks Tally for the WHOLE set at once → memory grows, not restartable, can hang on huge counts. |
| 4 | **TDL Function + `WRITE FILE LINE` loop** (v1–v17 here) | Interpreted per-row loop | Slowest at scale — per-row interpreter + write overhead. Fine for moderate data. |
| 5 | ODBC | Row-cursor over a flat virtual table | Worst. Can't express nested sub-collections; locks UI; limited columns. Ad-hoc master pulls only. |

## Why #1 wins
- `FETCH` pulls named methods **and nested sub-collection handles in ONE collection pass** — no per-field server round-trip, no report layout/measure/paginate stage.
- **Date-range batching (5000–10000 vouchers/batch; do NOT exceed 10k — Tally hangs)** is THE scale lever: bounded RAM, restartable, never asks for an unbounded set. This is what moves millions of rows in production.
- Client flattens the nested XML (`<ALLLEDGERENTRIES.LIST>` inside each `<VOUCHER>`) → one row per entry.

## Perf levers that actually move the needle (ranked)
1. `TYPE=Collection` (raw) over report/`TYPE=Data` when the client can flatten XML → the 2×/−40% win.
2. **`FETCH` the nested route in one pass** (`Voucher.AllLedgerEntries.CostCentreAllocations`) — **far cheaper than per-row cross-object lookups** (`$Method:Ledger:$Name`, `$$CollectionFieldByKey`). Reserve cross-object lookups for the rare unreachable field.
3. Filter server-side, early: `NOT $IsCancelled AND NOT $IsOptional`, `$$NumItems:AllLedgerEntries > 0`.
4. **Batch by date (5000–10k vouchers).** Biggest stability+throughput lever for 1M.
5. **Avoid `Keep Source:Yes` for a one-shot 1M dump** (retains parent source objects = memory blow-up). Keep the collection static/plain. `Keep Source` only pays off when the SAME collection is re-repeated in a live report.
6. `Search Key` + `$$CollectionFieldByKey` only when you must repeatedly join to a master by key — build the index once, not per row.
7. Collection-level `Compute`/aggregation over per-row field re-evaluation.
8. Normalise sign/date in TDL (cheap expressions), not the client; but keep expressions light.
9. Don't mark the dump collection `Is ODBC Table` (publishing overhead). `Client Only` where remote.
10. Minimise per-field cross-object dereferences — route + `FETCH` collapses them.

## The winner's exact request (per date-batch)
Loop `SVFROMDATE`/`SVTODATE` windows over `BooksFrom → last voucher date`, append results:

```xml
<ENVELOPE>
  <HEADER>
    <VERSION>1</VERSION>
    <TALLYREQUEST>Export</TALLYREQUEST>
    <TYPE>Collection</TYPE>
    <ID>VchFlatDump</ID>
  </HEADER>
  <BODY><DESC>
    <STATICVARIABLES>
      <SVEXPORTFORMAT>XML (Data Interchange)</SVEXPORTFORMAT>
      <SVFROMDATE>20240401</SVFROMDATE>
      <SVTODATE>20240430</SVTODATE>
      <SVCURRENTCOMPANY>Your Co Ltd</SVCURRENTCOMPANY>
    </STATICVARIABLES>
    <TDL><TDLMESSAGE>
      <COLLECTION NAME="VchFlatDump">
        <TYPE>Voucher</TYPE>
        <FILTER>fActive</FILTER>
        <FETCH>Date, Guid, VoucherNumber, VoucherTypeName, PartyLedgerName,
               Narration, Reference, PlaceOfSupply,
               AllLedgerEntries,
               AllLedgerEntries.LedgerName, AllLedgerEntries.Amount,
               AllLedgerEntries.BillAllocations,
               AllLedgerEntries.BillAllocations.Name,
               AllLedgerEntries.BillAllocations.Amount,
               AllLedgerEntries.BankAllocations,
               AllLedgerEntries.CostCentreAllocations,
               AllLedgerEntries.CostCentreAllocations.Name,
               AllLedgerEntries.CostCentreAllocations.Amount</FETCH>
      </COLLECTION>
      <SYSTEM TYPE="Formulae" NAME="fActive">NOT $IsCancelled AND NOT $IsOptional</SYSTEM>
    </TDLMESSAGE></TDL>
  </DESC></BODY>
</ENVELOPE>
```
Client: `POST http://<host>:9000` → per `<VOUCHER>` iterate `<ALLLEDGERENTRIES.LIST>` → emit CSV row (header cols repeated + entry cols + allocations). Advance window, repeat.
Enable server: TallyPrime `F1 → Settings → Connectivity → Client/Server = Both, port 9000` (ERP9 `F12 → Advanced Config`).

**For truly large / recurring loads, don't reinvent this — run `tally-database-loader` (config `TYPE=Collection`). It is the reference production puller.**

## In-Tally alternative (no external process): native Report export
If you can't run an external puller, the fastest IN-TALLY method is the **export engine**, not a
`WRITE FILE LINE` loop. Prototype: `new/VchDumpReport_v18.tdl` — a Report/Form/Part/Line/Field
export driven by the `EXPORT` action to comma-delimited CSV. Engine-streamed → faster + flatter
memory than v1–v17's function loop, same menu UX. Trade-off vs the HTTP puller: single-shot (no
date-batching/restart), so best up to low-hundred-thousands.

## Where this repo's v1–v17 fit
`WRITE FILE LINE` loop = rank 4. Correct + complete for **moderate** data (audit periods, a company's
year — tens of thousands of rows) and the simplest to load/run. For **large/recurring** data, prefer
v18 (in-Tally engine export) or the HTTP puller (rank 1). Also, for 1M-row runs, drop the
`Keep Source:Yes` and the cross-object `$…:Ledger:$LedgerName` lookups used in v5+ (they cost at scale).
