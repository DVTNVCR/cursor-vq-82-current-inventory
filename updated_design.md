# Current Inventory by Patient — Delivery-Trace API Design (Updated)

Target system: **S/4HANA 2020 on-premise**, future move to **S/4HANA 2023 on RISE**. Output is a **read-only OData GET API**. No `with parameters` — Patient is a regular field; OData consumers filter via `$filter`.

This document supersedes the previous design (which queried `V_EQUI.kund2` directly). The new design traces inventory through actual delivery history with PGI, per the functional team's spec.

---

## 1. Source Function Module (legacy — to be retired)

```abap
FUNCTION zfm_fiori_currentinv
  IMPORTING VALUE(iv_patientid) TYPE bu_partner
  EXPORTING VALUE(et_currentinv) TYPE zcurrentinvtt
            VALUE(es_return) TYPE bapiret2.

  " Logic summary:
  " 1. SELECT v_equi WHERE kund2 = patient
  " 2. Build RMA indicator via OBJK + SER01 (note 175365 fix)
  " 3. Dead VBAK/VBAP/LIPS chain that never feeds the final output
  " 4. Loop-with-SELECT-SINGLE against MAKT/MARA/TWEWT for descriptions
  " 5. Compute nodaypat = sy-datum - datab
  " 6. Sort descending by datab
ENDFUNCTION.
```

### Review findings — to drop in the rewrite
- N+1 selects inside `LOOP AT lt_curinv` against MAKT/MARA/TWEWT.
- VBAK/VBAP/LIPS chain populates `lt_sorder`/`lt_pgi`/`lt_rma` but the final loop overwrites `rmaind` with the OBJK/SER01 result — VBAK chain is dead code.
- `TWEWT-ewbez` select missing `SPRAS` filter.
- Meaningless `SY-SUBRC` checks after `CONVERSION_EXIT_ALPHA_INPUT`.
- Loop-overwriting bug: `lt_sorder` is overwritten every iteration.
- `V_EQUI.kund2` is unreliable (motivated QC note #175365). Replaced by delivery-trace.

---

## 2. Functional team's revised algorithm

| Step | What | Tables |
|---|---|---|
| 1 | All deliveries where **Sold-To = patient** | `LIKP` (header) |
| 2 | Serial numbers from those deliveries where **WADAT_IST is filled** (PGI'd) | `LIKP` + `LIPS` + `VBFA` + `SER03` + `OBJK` |
| 3 | Keep only serials whose **current customer** is one of the **Ship-To IDs** observed across that patient's deliveries | `OBJK` + `SER01` + `LIKP` |

Reinterpretations vs. legacy:
- "Patient ID as Sold-To" → `LIKP-KUNAG` (not `EQUI-KUND2`).
- "`WADAT_IST` from LIPK" → `LIKP` (typo confirmed).
- Step 3 is set-based: a patient may have multiple Ship-To addresses (treatment locations); equipment counts as "with patient" if its current customer is any one of them, regardless of which Ship-To received the original delivery.
- **`SER03` is the serial-number header for goods movements (material documents), not for deliveries directly.** Its keys are `MBLNR / MJAHR / ZEILE` (material doc keys), with no `LIEF_NR` / `VBELN` field. The chain from a delivery to its serials therefore goes **`LIKP` / `LIPS` → `VBFA` (document flow) → `MKPF` / `MSEG` (material doc) → `SER03` → `OBJK`**. SAP's other SER tables: `SER01` = sales documents, `SER04` = purchasing, `SER05` = production, `SER06` = handling units, `SER07` = inspection lots.

---

## 3. Decisions locked

| # | Decision |
|---|---|
| 1 | **Step 2 serial linkage**: delivery → goods-movement flow (`VBFA`) → material-doc serial header (`SER03`) ⋈ `OBJK`. `SER01` reserved for Step 3 current-location. |
| 2 | **Step 3 "with patient"**: any SER01 entry whose `KUNDE` is one of the patient's delivery Ship-Tos. |
| 3 | **No delivery-type filter** — every PGI'd delivery to the patient counts. |
| 4 | **IssueDate = MIN(WADAT_IST)** per (Patient, Equipment) across that patient's deliveries. |
| 5 | **No ReturnIndicator** — appearance in the result is the signal. |
| 6 | **Modern VDM field names** — clean API contract. |
| 7 | **`WADAT_IST` on LIKP** — confirmed typo. |
| 8 | **`SalesOrder` field added** to composite + consumption views; aggregated as `max(ReferenceSDDocument)` = latest/active sales order. |

---

## 4. Architecture — Clean Core layering

### Path decision (revised after empirical validation)

The functional team's original 3-step delivery-trace spec produced 2033 rows for a test patient where the legacy FM produces 23. Diagnosis: the delivery trace correctly identifies every piece of equipment ever delivered to the patient (2033 over an 8-year treatment history); the spec's Step-3 SER01 cross-check returns "any historical SER01 row" — which for medical-device shipments stays positive forever after the first delivery, so essentially never filters anything out. The semantic mismatch is fundamental, not a SQL bug.

**Adopted design (Path C, revised twice)**: `ZI_EquipmentCurrentInstall` (a Z wrap of `EQUI ⋈ EQUZ` mirroring the legacy `V_EQUI`) is the **spine**. The delivery trace is a **LEFT OUTER** enrichment for `SalesOrder` and (optionally) `Batch`.

**Why the wrap and not `I_Equipment`**: a second test patient (1041467) returned 4 rows instead of the FM's 11. Root cause identified from the FM source itself — line 66:

```abap
SELECT equnr, datab, matnr, charge FROM v_equi
  WHERE kund2 = @iv_patientid.
```

The FM reads `kund2` from `V_EQUI`, which projects **`EQUZ.KUND2`** (the time-dependent current customer slice). `I_Equipment.Customer`, by contrast, projects **`EQUI.KUND2`** (the header field). SAP is supposed to keep these two in sync via standard transactions, but for equipment with assignment history (returns, re-issues, manual EQUZ edits) they drift. Patient 200328 had zero drift (23 == 23). Patient 1041467 has 7 rows of drift (EQUZ-current = 1041467 but EQUI-header ≠ 1041467 for those 7 equipment).

`I_Equipment` does not expose `EQUZ.KUND2`, and `V_EQUI` cannot be selected from a `define view entity` (it's a classical DDIC view). The only path to FM parity in 2020 view-entity syntax is a Z wrap of `EQUI ⋈ EQUZ` with `WHERE datbi = '99991231'`. That wrap is `ZI_EquipmentCurrentInstall`. It is structurally a clone of `V_EQUI` and replaces `V_EQUI` 1:1 for our purposes. In a 2023 RISE / on Public Cloud world this view would need to be replaced by whatever future released CDS view eventually exposes the EQUZ time-slice customer, but for 2020 on-prem we have no alternative.

Step-3 in the original spec (the SER01 + Ship-To matching) is **dropped** from the active chain. The two views built for it (`ZI_SerialCurrentCustomer`, `ZI_PatientShipTo`) have been removed from the project. If the functional team revisits "current inventory" semantics in the future and wants the historical SER01 / Ship-To check restored, they can be re-derived from the design history in git.

### View source mapping

| Classical | Released view (2020) | Action |
|---|---|---|
| `LIKP` | `I_DeliveryDocument` | Use |
| `LIPS` | `I_DeliveryDocumentItem` | Use |
| `MARA` | `I_Product` | Use |
| `MAKT` | `I_ProductDescription` | Use |
| `TWEWT` | — | Wrap as `ZI_ExternalProductGroupText` |
| `VBFA` | — | `ZI_DeliveryGoodsMovement` (delivery → material-doc flow; projects `POSNN` as INT4) |
| `SER03` | — | `ZI_GoodsMovementSerial` (projects `ZEILE` as INT4 to align widths) |
| `OBJK` | — | Joined directly inside `ZI_DeliverySerial_Numbers` (no standalone wrap needed) |
| `EQUI` + `EQUZ` (KUND2) | `I_Equipment` was tried but projects header `EQUI.KUND2` only, not the EQUZ time-slice | `ZI_EquipmentCurrentInstall` — direct `EQUI ⋈ EQUZ` wrap (mirrors V_EQUI), spine of the consumption view |
| `LIKP` + `LIPS` (non-FIELDEQUIP batch) | `I_DeliveryDocument` + `I_DeliveryDocumentItem` | `ZI_PatientLatestArrayLot` — finds each patient's most recent consumable-array batch for the OData enrichment field |
| `AUSP` (classification values) | `I_ClfnObjectCharcValForKeyDate` (parameterized released view, P_KeyDate = today) | Joined inline in the consumption view for `ManufactureDate` (characteristic `Z_DATE_OF_MANUFACTURE`, internal ID `0000000151`, class type `002`) |

### View dependency (Path C, revised twice)

```
ZC_CurrentInventoryByPatient                                  (consumption — ZI_EquipmentCurrentInstall is the spine)
   │
   ├── ZI_EquipmentCurrentInstall          [Z wrap — EQUI ⋈ EQUZ, V_EQUI replacement, SPINE, EQUZ.KUND2 carries Patient]
   │
   ├── ZI_PatientEquipmentDelivered                           (LEFT OUTER — delivery enrichment when reachable)
   │      ├── I_DeliveryDocument           [released — LIKP]
   │      ├── I_DeliveryDocumentItem       [released — LIPS]
   │      └── ZI_DeliverySerial_Numbers            [VBFA → SER03 ⋈ OBJK]
   │             ├── ZI_DeliveryGoodsMovement  [VBFA wrap — delivery → mat-doc flow, POSNN as INT4]
   │             └── ZI_GoodsMovementSerial   [SER03 wrap — ZEILE as INT4]
   │
   ├── ZI_PatientLatestArrayLot                              (LEFT OUTER — latest non-FIELDEQUIP batch)
   │      ├── I_DeliveryDocument           [released — LIKP]
   │      └── I_DeliveryDocumentItem       [released — LIPS]
   │
   ├── I_Product                           [released — keyed on I_Equipment.Material]
   ├── I_ProductDescription                [released]
   ├── ZI_ExternalProductGroupText         [TWEWT wrap — TWEWT not released in 2020]
   └── I_ClfnObjectCharcValForKeyDate      [released parameterized view — characteristic Z_DATE_OF_MANUFACTURE for ManufactureDate]
```

### Notes on conventions

- All views use **`define view entity`** syntax. `@AbapCatalog.sqlViewName`, `@AbapCatalog.compiler.compareFilter`, `@AbapCatalog.preserveKey` are omitted — view entities don't generate a separate DDIC SQL view, so these annotations don't apply.
- `@AbapCatalog.viewEnhancementCategory` could be added to basic views if future extensibility is desired; not added here.
- All `Z` wraps of classic tables are tagged `@VDM.viewType: #BASIC` and `@Metadata.allowExtensions: true` where relevant; document them as "classic-table extensions, retire when SAP releases an equivalent."
- **No `select distinct`** anywhere — view entities on 7.55 (S/4 2020) don't support it (added in 7.56). Where dedup is needed, we use `group by` on the projected keys; in the consumption view, non-key fields are wrapped in `max(...)` as placeholder aggregates so the GROUP BY is well-formed. This whole class of code can be cleaned up post-2023 RISE migration if desired.
- **No function/cast in `JOIN ON`** — 7.55 disallows expressions in comparisons. Any width/type normalization happens in basic-view projections (see `ZI_DeliveryGoodsMovement` and `ZI_GoodsMovementSerial`), so all `JOIN ON` conditions reduce to plain field equality.

---

## 5. The eight active CDS views (Path C)

> The spine `ZI_EquipmentCurrentInstall` (Z wrap of `EQUI ⋈ EQUZ`) is documented inline in Section 5.7 as part of the Path C explanation, not in its own subsection — its design is inseparable from why the consumption view is shaped the way it is.

### 5.1. `ZI_DeliveryGoodsMovement` — delivery → material-document flow (VBFA wrap)

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Delivery to Goods Movement Flow'
@VDM.viewType: #BASIC

define view entity ZI_DeliveryGoodsMovement
  as select from vbfa
{
  key vbelv                              as DeliveryDocument,
  key posnv                              as DeliveryDocumentItem,
  key vbeln                              as MaterialDocument,
  key mjahr                              as MaterialDocumentYear,
  key cast( posnn as abap.int4 )         as MaterialDocumentItem
}
where vbtyp_n = 'R'   -- subsequent document category = Goods Movement
```

Why this view exists: `SER03` is keyed on the **material document** (`MBLNR / MJAHR / ZEILE`), not on the delivery. To trace a delivery's serials we first have to find the material document(s) created at PGI — that's what VBFA's document-flow records give us. `VBTYP_N = 'R'` filters to "subsequent document = goods movement."

`POSNN` is projected as `abap.int4` because `VBFA.POSNN` is `POSNR` (NUMC 6) and the matching `SER03.ZEILE` is `MBLPO` (NUMC 4). The 2020 CDS compiler enforces "NUMC casts must preserve length," so we can't narrow NUMC 6 → NUMC 4 directly. It also disallows function/cast expressions in `JOIN ON` clauses. Casting to INT4 in projection sidesteps both rules and gives plain numeric equality downstream.

No `select distinct` here — view entities on 7.55 (S/4 2020) don't support it (added in 7.56). Uniqueness comes naturally from the projected tuple matching VBFA's natural key under the `VBTYP_N = 'R'` filter.

---

### 5.2. `ZI_GoodsMovementSerial` — SER03 wrap with INT4 item position

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'SER03 Wrap (Int Item Position)'
@VDM.viewType: #BASIC

define view entity ZI_GoodsMovementSerial
  as select from ser03
{
  key obknr                          as ObjectNumber,
  key mblnr                          as MaterialDocument,
  key mjahr                          as MaterialDocumentYear,
  key cast( zeile as abap.int4 )     as MaterialDocumentItem
}
```

The mirror of the previous view on the SER03 side: project `ZEILE` (NUMC 4) as `abap.int4` so the join with `ZI_DeliveryGoodsMovement.MaterialDocumentItem` (also INT4) is plain field equality with no expression in `JOIN ON`.

No `select distinct` — view entities on 7.55 (S/4 2020) don't support it. The (`OBKNR`, `MBLNR`, `MJAHR`, `ZEILE`) tuple is naturally unique in SER03 in practice: the remaining SER03 key fields (BLART, BWART, VORGANG, LIEFERANT, KUNDE, WERK, LAGERORT) are attributes of a single goods movement and don't multiply rows for a given material-doc-line / OBKNR pair. Any residual duplicates are absorbed by `ZI_PatientEquipmentDelivered`'s `group by Patient, Equipment` with `min()/max()` aggregations.

---

### 5.3. `ZI_DeliverySerial_Numbers` — serials per delivery item (chain through goods movement)

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Serial Numbers per Delivery Item'
@VDM.viewType: #BASIC
@Metadata.allowExtensions: true

define view entity ZI_DeliverySerial_Numbers
  as select from ZI_DeliveryGoodsMovement as flow

    inner join ZI_GoodsMovementSerial as h
      on  h.MaterialDocument     = flow.MaterialDocument
      and h.MaterialDocumentYear = flow.MaterialDocumentYear
      and h.MaterialDocumentItem = flow.MaterialDocumentItem

    inner join objk as o
      on o.obknr = h.ObjectNumber

{
  key flow.DeliveryDocument        as DeliveryDocument,
  key flow.DeliveryDocumentItem    as DeliveryDocumentItem,
  key o.equnr                      as Equipment,
      o.sernr                      as SerialNumber,
      o.matnr                      as Product
}
where o.taser = 'SER03'
```

`SER03` is the serial-number header for **goods movements / material documents**, not for deliveries directly (the table label literally reads "Document Header for Serial Numbers for Goods Movements"). The chain from a delivery to its serials is therefore:

```
LIKP / LIPS  ──VBFA──▶  MKPF / MSEG (material doc)  ──SER03──▶  OBJK
```

All `JOIN ON` conditions are now plain field equality (no expressions) because both VBFA-side and SER03-side widths are reconciled to `abap.int4` in their respective basic-view projections. `o.taser = 'SER03'` follows the SAP convention that `OBJK.TASER` stores the source SER table name. Some older installations store all serial-object entries with `TASER = 'SER01'` regardless of source — see Section 8 for the verification check and adjustment.

---

### 5.4. `ZI_ExternalProductGroupText` — TWEWT wrap

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'External Product Group Text'
@VDM.viewType: #BASIC

define view entity ZI_ExternalProductGroupText
  as select from twewt
{
  key extwg    as ExternalProductGroup,
  key spras    as Language,
      ewbez    as ExternalProductGroupText
}
```

Retire after the 2023 RISE move if SAP releases an `I_ExternalProductGroupText`.

---

### 5.5. `ZI_PatientEquipmentDelivered` — delivery enrichment spine

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Equipment Delivered to Patient (PGI)'
@VDM.viewType: #COMPOSITE

define view entity ZI_PatientEquipmentDelivered
  as select from I_DeliveryDocument as dlv
    inner join I_DeliveryDocumentItem as dit
      on dit.DeliveryDocument = dlv.DeliveryDocument
    inner join ZI_DeliverySerial_Numbers as dsr
      on  dsr.DeliveryDocument     = dit.DeliveryDocument
      and dsr.DeliveryDocumentItem = dit.DeliveryDocumentItem
{
  key dlv.SoldToParty                              as Patient,
  key dsr.Equipment                                as Equipment,

      min( dlv.ActualGoodsMovementDate )           as EarliestPgiDate,
      max( dsr.SerialNumber )                      as SerialNumber,
      max( dit.Product )                           as Product,
      max( dit.Batch )                             as Batch,
      max( dit.ReferenceSDDocument )               as SalesOrder
}
where dlv.ActualGoodsMovementDate is not initial
group by dlv.SoldToParty, dsr.Equipment
```

Per (Patient, Equipment), this collapses N deliveries (initial ship + re-ships after returns) into a single row, keeping the **earliest** PGI date (IssueDate basis) and the **latest** sales order (active contract). `max()` on SerialNumber/Product/Batch is a deterministic picker — these are 1:1 with Equipment by data design.

Under Path C, this view is no longer the *gating* layer — it's the *enrichment* layer. It overshoots (in our test patient: 2033 rows) because it surfaces every equipment ever delivered. The actual filter happens in the consumption view via `ZI_EquipmentCurrentCustomer`.

---

### 5.6. `ZI_PatientLatestArrayLot` — latest delivered consumable-array lot per patient

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Patient Latest Array Lot Delivered'
@VDM.viewType: #COMPOSITE

define view entity ZI_PatientLatestArrayLot
  as select from    I_DeliveryDocument        as dlv
    inner join      I_DeliveryDocumentItem    as dit
      on dit.DeliveryDocument = dlv.DeliveryDocument
{
  key dlv.SoldToParty                                                                          as Patient,

      max( dlv.ActualGoodsMovementDate )                                                       as LatestArrayDeliveryDate,

      substring(
        max( concat( cast( dlv.ActualGoodsMovementDate as abap.char(8)  ),
                     cast( dit.Batch                   as abap.char(10) ) ) ),
        9, 10 )                                                                                as LatestArrayLot
}
where dlv.ActualGoodsMovementDate is not initial
  and dit.Batch                   is not initial
  and dit.Batch                  <> 'FIELDEQUIP'
group by dlv.SoldToParty
```

One row per patient, carrying the most recent consumable-array lot they were shipped.

The `WHERE` clause keeps only PGI'd delivery items with a non-empty batch and excludes the `FIELDEQUIP` placeholder (Novocure's convention for non-lot-tracked serialized field equipment). Whatever's left is a real production lot — the C-style codes you see in `ZI_ORDER_DELIVERY_BATCH`.

The interesting line is the `substring/max/concat/cast` chain. CDS in 2020 doesn't offer a first-by-date or last-value window function, but the classic SQL trick works fine:

1. `cast( ActualGoodsMovementDate as abap.char(8) )` — DATS is stored internally as 8-char `YYYYMMDD`, so the cast is essentially a no-op type change.
2. `cast( dit.Batch as abap.char(10) )` — normalize to the CHARG field width (`CHAR 10`), padded with trailing spaces.
3. `concat(...)` — produces an 18-char string `YYYYMMDDbbbbbbbbbb`. Lexicographic ordering on this string equals chronological ordering on the date with batch as tiebreaker.
4. `max(concat(...))` — the largest such string per patient = the latest delivery's max batch on that date.
5. `substring(..., 9, 10)` — chop off the date prefix, leaving just the batch.

Result: `LatestArrayLot` is the **actual batch from the most recent array delivery**, not just the lexicographic max batch across the patient's whole shipment history.

`group by dlv.SoldToParty` collapses to one row per patient (replaces `select distinct`, which 7.55 view entities don't support).

---

### 5.7. `ZC_CurrentInventoryByPatient` — consumption view (no parameters)

```abap
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Current Inventory by Patient'
@VDM.viewType: #CONSUMPTION
@ClientHandling.algorithm: #SESSION_VARIABLE
@ObjectModel.usageType: { serviceQuality: #D, sizeCategory: #S, dataClass: #MIXED }
@Search.searchable: true

define view entity ZC_CurrentInventoryByPatient
  as select from ZI_EquipmentCurrentInstall        as ecc

    left outer join ZI_PatientEquipmentDelivered   as ped
      on  ped.Patient   = ecc.Customer
      and ped.Equipment = ecc.Equipment

    left outer join ZI_PatientLatestArrayLot       as lat
      on lat.Patient = ecc.Customer

    left outer join I_Product                      as prd
      on prd.Product = ecc.Material

    left outer join I_ProductDescription           as pdt
      on  pdt.Product  = ecc.Material
      and pdt.Language = $session.system_language

    left outer join ZI_ExternalProductGroupText    as egt
      on  egt.ExternalProductGroup = prd.ExternalProductGroup
      and egt.Language             = $session.system_language

    /* Classification characteristic 'Z_DATE_OF_MANUFACTURE' (internal ID 0000000151)
       on equipment class type 002 — date value lives in CharcFromDate (single-value
       characteristic) or CharcToDate (range upper bound) depending on master-data setup. */
    left outer join I_ClfnObjectCharcValForKeyDate( P_KeyDate: $session.system_date ) as cv
      on  cv.ClfnObjectTable = 'EQUI'
      and cv.ClassType       = '002'
      and cv.ClfnObjectID    = ecc.Equipment
      and cv.CharcInternalID = '0000000151'

{
  @Semantics.businessPartner.id: true
  @Search.defaultSearchElement: true
  key ecc.Customer                                                                 as Patient,

  key ecc.Equipment                                                                as Equipment,

      max( ecc.SerialNumber )                                                      as SerialNumber,

      ltrim( max( ecc.Material ), '0' )                                            as Product,

      @Semantics.batch.batchNumber: true
      coalesce( max( ped.Batch ), max( ecc.Batch ) )                               as LotNumber,

      @Semantics.batch.batchNumber: true
      max( lat.LatestArrayLot )                                                    as LatestArrayLot,

      max( lat.LatestArrayDeliveryDate )                                           as LatestArrayDeliveryDate,

      max( ped.SalesOrder )                                                        as SalesOrder,

      @Semantics.businessDate.from: true
      coalesce( max( ped.EarliestPgiDate ), max( ecc.ValidityStartDate ) )         as IssueDate,

      cast( dats_days_between(
              coalesce( max( ped.EarliestPgiDate ), max( ecc.ValidityStartDate ) ),
              $session.system_date
            ) as abap.int4 )                                                       as DaysWithPatient,

      @Semantics.systemDate.createdAt: true
      max( ecc.CreationDate )                                                      as CreatedDate,

      @Semantics.businessDate.from: true
      max( coalesce( cv.CharcFromDate, cv.CharcToDate ) )                          as ManufactureDate,

      max( prd.ProductType )                                                       as ProductType,
      max( pdt.ProductDescription )                                                as ProductDescription,
      max( prd.ExternalProductGroup )                                              as ExternalProductGroup,
      max( egt.ExternalProductGroupText )                                          as ExternalProductGroupText
}
where ecc.Customer is not initial
group by ecc.Customer, ecc.Equipment
```

#### How Path C (revised twice) lands in the consumption view

`ZI_EquipmentCurrentInstall` is the **spine**, structurally a clone of `V_EQUI`. Each row in it with `Customer = patient` becomes one row in the output — same semantics as the FM (`SELECT v_equi WHERE kund2 = patient`):

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Equipment + current EQUZ time-slice (V_EQUI replacement)'
@VDM.viewType: #BASIC

define view entity ZI_EquipmentCurrentInstall
  as select from    equi as e
    inner join      equz as z
      on e.equnr = z.equnr
{
  key e.equnr          as Equipment,
      e.matnr          as Material,
      e.sernr          as SerialNumber,
      e.charge         as Batch,
      e.erdat          as CreationDate,
      z.kund2          as Customer,
      z.datab          as ValidityStartDate,
      z.datbi          as ValidityEndDate
}
where z.datbi = '99991231'
```

- `ecc.Customer / ecc.Equipment` are the output keys. `SerialNumber`, `Material`, `Batch`, and `CreationDate` come straight from EQUI so they're always populated.
- `ZI_PatientEquipmentDelivered` is LEFT OUTER. Equipment that can be back-traced to a patient-as-Sold-To delivery gets `SalesOrder` from the delivery trace. Equipment that *can't* (older deliveries shipped under a different Sold-To — insurance, parent BP, etc.) gets `null` for `SalesOrder`.
- `LotNumber` and `IssueDate` use `coalesce` to fall back from the delivery trace to the EQUI/EQUZ source (the FM's `charge` and `datab` columns), so all 11 rows are populated for our test patient.
- `CreatedDate` is `EQUI.ERDAT` (equipment master record creation), surfaced through the spine.
- `ManufactureDate` joins `I_ClfnObjectCharcValForKeyDate` — the released parameterized view that resolves the classification value valid as of `$session.system_date`. Hardcoded filters bind to class type `002`, classifiable object `EQUI`, and characteristic internal ID `0000000151` (= `Z_DATE_OF_MANUFACTURE` in this system; verify via `I_ClfnCharacteristic` if needed). The actual date lives in either `CharcFromDate` (single-value characteristic) or `CharcToDate` (range upper bound); `coalesce` picks whichever the master-data setup populated. Equipment without a manufacture-date characteristic value returns `null`.
- Both `CreatedDate` and `ManufactureDate` are exposed as native DATS — OData V4 serializes them as `Edm.Date` (YYYY-MM-DD in JSON). Switch to string-format only if a downstream consumer can't bind `Edm.Date` and needs the literal `cast(... as char(32)) + substring + concat` pattern from `ZI_EQUIPMENT` / `ZI_ORDER_DELIVERY_BATCH`.
- Authorization check on the spine is `#NOT_REQUIRED` because `EQUI/EQUZ` carry no auth-relevant fields beyond what's already checked by the patient `BU_PARTNER` filter at the API layer.

#### Why not use `I_Equipment` (rejected attempt)

`I_Equipment` projects `Customer` from `EQUI.KUND2` (the header field). The FM reads `kund2` from `V_EQUI`, which projects `EQUZ.KUND2` (the time-dependent slice). These two diverge for equipment with assignment history — exactly the older inventory we were missing. There is no released CDS view in 2020 that exposes the EQUZ time-slice customer, so we have to wrap the classical tables ourselves. This wrap is the **only** classical-table dependency in the active chain; in a future RISE release where a released view exists, the wrap is a one-line swap.

The inner join drops every `ped` row whose equipment is no longer registered to the patient. The 2033 → 23 collapse happens here.

`group by ped.Patient, ped.Equipment` and the `max(...)` wrappers remain for defensive uniqueness (each `(Patient, Equipment)` is already 1:1 by design; the wrappers absorb any unexpected fan-out from left outer joins on master data). Reads like a sledgehammer but costs nothing at HANA execution.

---

## 6. Service Definition + Service Binding (OData V4)

### Service Definition

```abap
@EndUserText.label: 'Current Inventory by Patient - API'
define service ZUI_CURRENT_INVENTORY {
  expose ZC_CurrentInventoryByPatient as CurrentInventory;
}
```

### Service Binding

Create in ADT against `ZUI_CURRENT_INVENTORY` with type **OData V4 — Web API** (system-to-system) or **OData V4 — UI** (Fiori Elements). `@OData.publish` is deprecated for new development; don't use it.

### Sample API calls

List by patient:
```
GET /sap/opu/odata4/sap/zui_current_inventory/srvd_a2x/sap/zui_current_inventory/0001/CurrentInventory
    ?$filter=Patient eq '0000012345'
    &$orderby=IssueDate desc
```

Direct read of one equipment for one patient:
```
GET /sap/opu/odata4/sap/zui_current_inventory/srvd_a2x/sap/zui_current_inventory/0001/CurrentInventory(Patient='0000012345',Equipment='000000000010000123')
```

> Consumers must pass `Patient` (and `Equipment` for key access) in **canonical SAP internal form (leading zeros)** — CDS field comparisons don't auto-apply ALPHA conversion. Document this in the API contract.

---

## 7. Decommission plan

- All callers switch to the OData service.
- Delete `ZFM_FIORI_CURRENTINV`.
- Drop `ZCURRENTINV` / `ZCURRENTINVTT` once a project-wide search confirms no remaining references. Consumers should bind to the CDS output shape, not the DDIC structure.
- Delete the obsolete VBAK/VBAP/LIPS RMA chain logic from any auxiliary code — under the new model, returns naturally drop equipment from the result set, so that whole branch is irrelevant.

---

## 8. To verify in ADT before activation

1. **`ReferenceSDDocument`** field name in `I_DeliveryDocumentItem` — confirmed for modern VDM but a small number of 2020 stacks ship it as `OriginSDDocument` or other release-specific names. Check *Released Objects* and adjust the alias if needed.
2. **`I_DeliveryDocument`** field names — `SoldToParty` / `ShipToParty` / `ActualGoodsMovementDate` are the modern names; verify against your specific 2020 SP.
3. **`OBJK.TASER` value for material-doc serials** — `ZI_DeliverySerial_Numbers` filters `o.taser = 'SER03'` based on the SAP convention that `TASER` stores the source SER table name. Some older installations store all serial-object entries with `TASER = 'SER01'` regardless of source. If yours does, the activated view returns zero rows. Quick check (SQL preview):
   ```sql
   SELECT DISTINCT taser FROM objk WHERE obknr IN (SELECT obknr FROM ser03 WHERE ROWNUM < 100)
   ```
   If it shows `'SER01'`, change the `where` clause in `ZI_DeliverySerial_Numbers` to `o.taser = 'SER01'`. If `'SER03'`, leave it.
4. **`VBFA.VBTYP_N = 'R'`** in `ZI_DeliveryGoodsMovement` — standard SAP code for "subsequent document is a goods movement." If your config uses a different VBTYP for some flows (rare with custom doc types), the join misses those movements. Sanity check:
   ```sql
   SELECT DISTINCT vbtyp_n FROM vbfa WHERE vbelv IN (SELECT vbeln FROM likp WHERE ROWNUM < 100)
   ```
   Anything besides `'R'` for material-doc flows means we need to broaden the filter.
5. **Performance** — `Patient` filter pushes down to two indexed paths simultaneously: `LIKP-KUNAG` (delivery spine) and `EQUI/EQUZ-KUND2` (currently-registered gate). HANA's join optimizer should drive the smaller set (typically the EQUI side, since `KUND2 = patient` is highly selective — ~23 rows in the test case) and use that to prune the delivery trace. The VBFA hop is keyed on `VBELV` (delivery number) which is indexed in standard SAP.
6. **`EQUI` / `EQUZ` direct access** — `ZI_EquipmentCurrentInstall` reads `EQUI ⋈ EQUZ` directly (no released CDS view in 2020 projects `EQUZ.KUND2`). ATC will flag this as a classical-table dependency; that's expected. Add the view to the "retire when SAP releases an EQUZ-time-slice-customer view" upgrade backlog. The `WHERE datbi = '99991231'` filter is the standard SAP convention for the currently-valid time slice — verify in your data with `SELECT DISTINCT datbi FROM equz` if you want absolute confidence.
7. **Equipment + batch mismatch across deliveries** — `max(Batch)` picks lexicographically. If your data ever has different batches for the same physical equipment across re-shipments, switch to "batch from the earliest delivery" via a self-join. Uncommon in equipment leasing; flag if it shows up.
8. **Classification characteristic `Z_DATE_OF_MANUFACTURE` (internal ID `0000000151`)** — the consumption view's `ManufactureDate` join binds to a system-specific characteristic internal ID. Internal IDs are stable within a single system landscape but differ across systems. Verify in your system:
   ```sql
   SELECT CharcInternalID, Characteristic FROM I_ClfnCharacteristic WHERE Characteristic = 'Z_DATE_OF_MANUFACTURE'
   ```
   If yours returns something other than `0000000151`, update the literal in the `cv` join's `on` clause. Also confirm the characteristic is on equipment **class type `002`** (the standard SAP equipment class type) and that the equipment objects being queried are assigned to that class. Equipment without an assignment to the relevant class returns `null` for `ManufactureDate` — that's expected behavior, not an error.
9. **`I_ClfnObjectCharcValForKeyDate` parameterized view authorization** — the released classification view performs its own auth checks against `S_CLASS` / `S_CLATTR`. The OData service runs under the calling user's auth context, so users without classification-read auth will see `null` `ManufactureDate` values even when data exists. If the API is consumed by a technical/service user, ensure that user has read access to class type `002`.

---

## 9. SalesOrder semantics — alternatives if needed

Currently using `max( dit.ReferenceSDDocument )` = latest sales order (active contract). Alternatives:

- **Earliest sales order** (matches `EarliestPgiDate`): change to `min( dit.ReferenceSDDocument )`. Correct when SO numbering is monotonic with time, which is the SAP norm.
- **SO paired with the earliest PGI** (correct even when SO numbering isn't monotonic): drop the aggregation, expose an unaggregated `ZI_PatientDeliveredSerial`, and self-join in the consumption view on `EarliestPgiDate` to grab the matching row's `SalesOrder`. Slightly more views but rigorously correct.

---

## 10. 2023 RISE forward-compatibility notes

All artifacts above are 2020-compatible and survive the upgrade unchanged. Post-upgrade actions:

1. Re-check `RAP_BO_RELEASED_API` / ADT *Released Objects* — if SAP releases `I_ExternalProductGroupText` (or equivalents for OBJK/SER01/SER03 linkages), retire the matching `Z` basic wraps and rewire the joins.
2. Run ATC with the Cloud-readiness variant to confirm Cloud language compatibility if any objects need promoting.
3. Confirm edition with basis team — Private Cloud Edition (typical RISE) preserves classic ABAP + `Z*`; Public Cloud Edition would force a different sourcing strategy for any non-released table access.

### Pre-emptive work that pays off
- Develop in ADT (Eclipse), not SE80/SE11.
- Use Standard ABAP language version for now; revisit per-object on 2023.
- Put all new objects in one package (e.g. `Z_CURRENT_INVENTORY_API`) for upgrade scope-checking.
- Tag each `Z` basic view with "classic-table extension, retire when SAP releases an equivalent" so the upgrade review has an explicit retirement candidate list.

---

## 11. Open items / not yet decided

- **External consumer vs. Fiori Elements UI** — drives Service Binding type (Web API vs. UI). Confirm to finalize binding creation.
- **Patient ID alpha formatting at the API consumer** — document required canonical form on the API contract.
- **RISE edition** (Private vs. Public Cloud) — confirm with basis team to lock the retirement plan for `Z` basic wraps.
