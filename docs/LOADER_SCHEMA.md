# tally-database-loader — export schema (for TDL replication)

Source: `tally-export-config.yaml` (branch `main`) + type→`<SET>` generator in `src/tally.mts`.

## The key rule
YAML `field:` = INNER expr only. Final TDL `<SET>` generated per `type`, BUT only when the
inner expr is a bare identifier `^(\.\.)?[A-Za-z0-9_]+$`. Anything else (has `if`/`$$`/`$`/space)
is emitted VERBATIM, unwrapped.

Per-type wrapper (`X` = the YAML field value):
| type | final TDL `<SET>` |
|---|---|
| text | `$X` |
| logical | `if $X then 1 else 0` |
| date | `if $$IsEmpty:$X then $$StrByCharCode:241 else $$PyrlYYYYMMDDFormat:$X:"-"` |
| number | `if $$IsEmpty:$X then "0" else $$StringFindAndReplace:($$String:$X):"(-)":"-"` |
| amount | `$$StringFindAndReplace:(if $$IsDebit:$X then -$$NumValue:$X else $$NumValue:$X):"(-)":"-"` |
| quantity | `$$StringFindAndReplace:(if $$IsInwards:$X then $$Number:$$String:$X:"TailUnits" else -$$Number:$$String:$X:"TailUnits"):"(-)":"-"` |
| rate | `if $$IsEmpty:$X then 0 else $$Number:$X` |

## Global conventions
- **Field separator = TAB**, **row separator = CRLF**, header row = column `name`s tab-joined.
- **Empty-date sentinel** = `$$StrByCharCode:241` = `ñ` (0xF1). (Loader later maps ñ→NULL.)
- Text cleaned: embedded TAB/CR/LF → space (`\r` removed, `\n`/`\t`→space).
- Date out = `YYYY-MM-DD` via `$$PyrlYYYYMMDDFormat:$X:"-"`.
- Amount/qty sign-normalised, `(-)`→`-`.
- **Parent/child link:** every child (Derived) table repeats the parent voucher **`guid`** (`$Guid`) as col 1. Cross-level back-ref via `..` (e.g. `trn_bank.ledger = ..LedgerName`).
- **Transaction filters:** `NOT $IsCancelled` ; `NOT $IsOptional` ; derived add `$$NumItems:<Coll> > 0`.

## Tables (collection · ordered cols → expr[type])
Masters (nature Primary):
- **mst_group** [Group]: guid Guid[t]; name Name[t]; parent `if $$IsEqual:$Parent:$$SysName:Primary then "" else $Parent`[t]; primary_group _PrimaryGroup[t]; is_revenue IsRevenue[l]; is_deemedpositive IsDeemedPositive[l]; is_reserved IsReserved[l]; affects_gross_profit AffectsGrossProfit[l]; sort_position SortPosition[n]
- **mst_ledger** [Ledger]: guid Guid[t]; name Name[t]; parent `if $$IsEqual:$Parent:$$SysName:Primary then "" else $Parent`[t]; alias OnlyAlias[t]; description Description[t]; notes Narration[t]; is_revenue IsRevenue[l]; is_deemedpositive IsDeemedPositive[l]; opening_balance OpeningBalance[a]; closing_balance ClosingBalance[a]; mailing_name MailingName[t]; mailing_address `if $$IsEmpty:$Address then "" else $$FullList:Address:$Address`[t]; mailing_state LedStateName[t]; mailing_country CountryName[t]; mailing_pincode PinCode[t]; email Email[t]; mobile `if NOT $$IsEmpty:$LedgerMobile then $$Sprintf:"%s %s":$LedgerCountryISDCode:$LedgerMobile else ""`[t]; it_pan IncomeTaxNumber[t]; gstn `if $$IsEmpty:$PartyGSTIN then $LedGSTRegDetails[Last].GSTIN else $PartyGSTIN`[t]; gst_registration_type `if $$IsEmpty:$Gstregistrationtype then $LedGSTRegDetails[Last].Gstregistrationtype else $Gstregistrationtype`[t]; gst_supply_type Gsttypeofsupply[t]; gst_duty_head Gstdutyhead[t]; bank_account_holder Bankaccholdername[t]; bank_account_number BankDetails[t]; bank_ifsc Ifscode[t]; bank_swift Swiftcode[t]; bank_name Bankingconfigbank[t]; bank_branch BankBranchname[t]; bill_credit_period `$$Number:$BillCreditPeriod`[n]
- **mst_vouchertype** [VoucherType]: guid Guid[t]; name Name[t]; parent Parent[t]; numbering_method NumberingMethod[t]; is_deemedpositive IsDeemedPositive[l]; affects_stock AffectsStock[l]
- **mst_stock_item** [StockItem] fetch GstDetails,PartNo: guid Guid[t]; name Name[t]; parent `if $$IsEqual:$Parent:$$SysName:Primary then "" else $Parent`[t]; category `if $$IsEqual:$Category:$$SysName:NotApplicable then "" else $Category`[t]; alias OnlyAlias[t]; description Description[t]; notes Narration[t]; part_number PartNo[t]; uom `if $$IsEqual:$BaseUnits:$$SysName:NotApplicable then "" else $BaseUnits`[t]; alternate_uom `if $$IsEqual:$AdditionalUnits:$$SysName:NotApplicable then "" else $AdditionalUnits`[t]; conversion Conversion[n]; opening_balance OpeningBalance[q]; opening_rate OpeningRate[r]; opening_value OpeningValue[a]; closing_balance ClosingBalance[q]; closing_rate ClosingRate[r]; closing_value ClosingValue[a]; costing_method CostingMethod[t]; gst_type_of_supply GSTMSTTypeofSupply[t]; gst_hsn_code InfGSTHSNCode[t]; gst_hsn_description InfGSTHSNDescription[t]; gst_rate InfGSTIGSTRate[n]; gst_taxability InfGSTTaxablility[t]
- **mst_godown** [Godown]: guid Guid[t]; name Name[t]; parent `if $$IsEqual:$Parent:$$SysName:Primary then "" else $Parent`[t]; address `if $$IsEmpty:$Address then "" else $$FullList:Address:$Address`[t]
- **mst_cost_category** [CostCategory]: guid Guid[t]; name Name[t]; allocate_revenue AllocateRevenue[l]; allocate_non_revenue AllocateNonRevenue[l]
- **mst_cost_centre** [CostCentre]: guid Guid[t]; name Name[t]; parent `if $$IsEqual:$Parent:$$SysName:Primary then "" else $Parent`[t]; category Category[t]

Transactions (Derived unless noted; all filter NOT cancelled/optional):
- **trn_voucher** [Voucher] fetch Narration,PartyLedgerName,PlaceOfSupply,Reference,ReferenceDate: guid Guid[t]; date Date[d]; voucher_type VoucherTypeName[t]; voucher_number VoucherNumber[t]; reference_number Reference[t]; reference_date ReferenceDate[d]; narration Narration[t]; party_name PartyLedgerName[t]; place_of_supply PlaceOfSupply[t]; is_invoice IsInvoice[l]; is_accounting_voucher `if $$IsAccountingVch:$VoucherTypeName then 1 else 0`[l]; is_inventory_voucher `if $$IsInventoryVch:$VoucherTypeName then 1 else 0`[l]; is_order_voucher `if $$IsOrderVch:$VoucherTypeName then 1 else 0`[l]
- **trn_accounting** [Voucher.AllLedgerEntries] +NumItems AllLedgerEntries>0: guid Guid[t]; ledger LedgerName[t]; amount Amount[a]; amount_forex `if $$IsEmpty:$$ForexValue:$Amount then 0 else $$StringFindAndReplace:(if $$IsDebit:$Amount then -$$ForexValue:$Amount else $$ForexValue:$Amount):"(-)":"-"`[a]; currency `$$Currency:$Amount`[t]
- **trn_inventory** [Voucher.AllInventoryEntries] +NumItems AllInventoryEntries>0: guid Guid[t]; item StockItemName[t]; quantity ActualQty[q]; rate Rate[r]; amount Amount[a]; additional_amount AddlAmount[a]; discount_amount Discount[n]; godown GodownName[t]; tracking_number `if ($$IsEmpty:$TrackingNumber or $$IsNotApplicable:$TrackingNumber) then "" else $TrackingNumber`[t]; order_number `if ($$IsEmpty:$OrderNo or $$IsNotApplicable:$OrderNo) then "" else $OrderNo`[t]; order_duedate OrderDueDate[d]
- **trn_cost_centre** [Voucher.AllLedgerEntries.CostCentreAllocations] +NumItems AllLedgerEntries>0: guid Guid[t]; ledger LedgerName[t]; costcentre Name[t]; amount Amount[a]
- **trn_bill** [Voucher.AllLedgerEntries.BillAllocations] +NumItems AllLedgerEntries>0: guid Guid[t]; ledger LedgerName[t]; name Name[t]; amount Amount[a]; billtype BillType[t]; bill_credit_period `$$Number:$BillCreditPeriod`[n]
- **trn_bank** [Voucher.AllLedgerEntries.BankAllocations] +NumItems AllLedgerEntries>0: guid Guid[t]; ledger `..LedgerName`[t]; transaction_type TransactionType[t]; instrument_date InstrumentDate[d]; instrument_number InstrumentNumber[t]; bank_name BankName[t]; amount Amount[a]; bankers_date BankersDate[d]
- **trn_batch** [Voucher.AllInventoryEntries.BatchAllocations] +NumItems AllInventoryEntries>0: guid Guid[t]; item StockItemName[t]; name BatchName[t]; quantity ActualQty[q]; amount Amount[a]; godown GodownName[t]; destination_godown DestinationGodownName[t]; tracking_number `if ($$IsEmpty:$TrackingNumber or $$IsNotApplicable:$TrackingNumber) then "" else $TrackingNumber`[t]

Extras present (not built in v19): mst_gst_effective_rate [StockItem.GstDetails.StateWiseDetails.RateDetails], trn_closingstock_ledger [Ledger.LedgerClosingValues], trn_cost_category_centre, trn_cost_inventory_category_centre, trn_inventory_additional_cost, payroll (trn_employee/payhead/attendance), mst_uom/stock_category/stock_group/employee/payhead.
