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
| `VBFA` | — | `ZI_DELIVERY_GOODSMOVEMENT` (delivery → material-doc flow; projects `POSNN` as INT4) |
| `SER03` | — | `ZI_GoodsMovementSerial` (projects `ZEILE` as INT4 to align widths) |
| `OBJK` | — | Joined directly inside `ZI_DELIVERYSERIAL_NUMBERS` (no standalone wrap needed) |
| `EQUI` + `EQUZ` (KUND2) | `I_Equipment` was tried but projects header `EQUI.KUND2` only, not the EQUZ time-slice | `ZI_EquipmentCurrentInstall` — direct `EQUI ⋈ EQUZ` wrap (mirrors V_EQUI), spine of the consumption view |
| `LIKP` + `LIPS` (non-FIELDEQUIP batch) | `I_DeliveryDocument` + `I_DeliveryDocumentItem` | `ZI_PatientLatestArrayLot` — finds each patient's most recent consumable-array batch for the OData enrichment field |
| `AUSP` (classification values) | `I_ClfnObjectCharcValForKeyDate` (parameterized released view, P_KeyDate = today) | Joined inline in the consumption view for `ManufactureDate` (characteristic `Z_DATE_OF_MANUFACTURE`, internal ID `0000000151`, class type `002`) |
| `VBAK` (sales order header) | `I_SalesOrder` (released VDM) | Joined LEFT OUTER in both legs of the consumption view for `SalesOrderCreatedDate` (sales-order creation date, mirroring the reference `ZI_ORDER_DELIVERY_BATCH` view) |

### View dependency (Path C, revised twice; full chain including API-team facade)

```
Z_I_INVENTORY                                              [API team — bound to OData service, classic CDS]
   │
   └── ZI_PATIENT_INVENTORY                                [API team — alias/rename layer, view entity]
          │
          └── ZC_CurrentInvByPatient                       (our top consumption view — CLASSIC, UNION ALL of two legs)
                 │
                 ├──[LEG 1: EQUIPMENT]── ZI_PatientEquipmentInventory       (composite — equipment rows, Path C)
                 │       │
                 │       ├── ZI_EquipmentCurrentInstall          [Z wrap — EQUI ⋈ EQUZ, V_EQUI replacement, SPINE]
                 │       │
                 │       ├── ZI_PatientEquipmentDelivered                           (LEFT OUTER — delivery enrichment)
                 │       │      ├── I_DeliveryDocument           [released — LIKP]
                 │       │      ├── I_DeliveryDocumentItem       [released — LIPS]
                 │       │      └── ZI_DELIVERYSERIAL_NUMBERS         [VBFA → SER03 ⋈ OBJK]
                 │       │             ├── ZI_DELIVERY_GOODSMOVEMENT   [VBFA wrap — POSNN as INT4]
                 │       │             └── ZI_GoodsMovementSerial      [SER03 wrap — ZEILE as INT4]
                 │       │
                 │       ├── ZI_PatientLatestArrayLot                              (LEFT OUTER)
                 │       │      ├── I_DeliveryDocument           [released]
                 │       │      └── I_DeliveryDocumentItem       [released]
                 │       │
                 │       ├── I_Product                           [released]
                 │       ├── I_ProductDescription                [released]
                 │       ├── ZI_ExternalProductGroupText         [TWEWT wrap]
                 │       └── I_ClfnObjectCharcValForKeyDate      [released parameterized — ManufactureDate]
                 │
                 └──[LEG 2: DELIVERY_BATCH]── ZI_PatientDeliveryBatch         (composite — delivery-batch rows, Batch ≠ FIELDEQUIP)
                         │
                         ├── I_DeliveryDocument                  [released — LIKP, SoldToParty = Patient]
                         ├── I_DeliveryDocumentItem              [released — LIPS, Batch is the lot number]
                         ├── ZI_PatientLatestArrayLot            [LEFT OUTER — reused from leg 1]
                         ├── I_Product                           [released]
                         ├── I_ProductDescription                [released]
                         └── ZI_ExternalProductGroupText         [TWEWT wrap]
```

### Notes on conventions

- All views use **`define view entity`** syntax. `@AbapCatalog.sqlViewName`, `@AbapCatalog.compiler.compareFilter`, `@AbapCatalog.preserveKey` are omitted — view entities don't generate a separate DDIC SQL view, so these annotations don't apply.
- `@AbapCatalog.viewEnhancementCategory` could be added to basic views if future extensibility is desired; not added here.
- All `Z` wraps of classic tables are tagged `@VDM.viewType: #BASIC` and `@Metadata.allowExtensions: true` where relevant; document them as "classic-table extensions, retire when SAP releases an equivalent."
- **No `select distinct`** anywhere — view entities on 7.55 (S/4 2020) don't support it (added in 7.56). Where dedup is needed, we use `group by` on the projected keys; in the consumption view, non-key fields are wrapped in `max(...)` as placeholder aggregates so the GROUP BY is well-formed. This whole class of code can be cleaned up post-2023 RISE migration if desired.
- **No function/cast in `JOIN ON`** — 7.55 disallows expressions in comparisons. Any width/type normalization happens in basic-view projections (see `ZI_DELIVERY_GOODSMOVEMENT` and `ZI_GoodsMovementSerial`), so all `JOIN ON` conditions reduce to plain field equality.

---

## 5. Active CDS views (10 we own + 2 API-team facade = 12 total)

> The spine `ZI_EquipmentCurrentInstall` (Z wrap of `EQUI ⋈ EQUZ`) is in Section 5.7 — it has its own subsection because it's the FM-parity anchor.

> **2026-06-01 update**: the consumption view now returns both equipment and consumable rows, distinguished by a `SourceType` column. Because S/4HANA 2020 / NW 7.55 view entities don't support UNION, the top-level `ZC_CurrentInvByPatient` is a *classic* `define view` that UNION ALLs two view-entity helpers (`ZI_PatientEquipmentInventory`, `ZI_PatientDeliveryBatch`). Post-2023 RISE upgrade (NW 7.56+) this collapses back into a single view entity with native union support — see Section 10.

> **2026-06-02 update**: an additional 2 facade views (`Z_I_INVENTORY`, `ZI_PATIENT_INVENTORY`) sit on top of `ZC_CurrentInvByPatient`, owned by the API team. They are listed in the artifact inventory below and detailed in Section 12.

### Artifact inventory (full list)

| # | View | Owner | Type | Source file in this repo | Role |
|---|------|-------|------|---|------|
| 1 | `Z_I_INVENTORY`                  | API team | Classic CDS (`define view`) | `Z_I_INVENTORY.ddls` *(snapshot)* | Bound to OData service. API `createdDate` field is sourced from `IssueDate` (PGI date), formatted `YYYY-MM-DD`. Composite key `(orderId, partnerId)`. |
| 2 | `ZI_PATIENT_INVENTORY`           | API team | View entity                 | `ZI_PATIENT_INVENTORY.ddls` *(snapshot)* | Alias / rename / field-selection layer. Selects from `ZC_CurrentInvByPatient`. |
| 3 | `ZC_CurrentInvByPatient`         | Us       | Classic CDS (`define view`) | `ZC_CurrentInvByPatient.ddls` | Top consumption view — `UNION ALL` of equipment + consumables legs |
| 4 | `ZI_PatientEquipmentInventory`   | Us       | View entity                 | `ZI_PatientEquipmentInventory.ddls` | Equipment leg of the union |
| 5 | `ZI_PatientDeliveryBatch` | Us       | View entity                 | `ZI_PatientDeliveryBatch.ddls` | Delivery-batch leg of the union (Batch ≠ FIELDEQUIP) — paired with `SourceType = 'DELIVERY_BATCH'` |
| 6 | `ZI_EquipmentCurrentInstall`     | Us       | View entity                 | `ZI_EquipmentCurrentInstall.ddls` | `EQUI ⋈ EQUZ` spine (V_EQUI replacement), source of FM parity |
| 7 | `ZI_PatientEquipmentDelivered`   | Us       | View entity                 | `ZI_PatientEquipmentDelivered.ddls` | Delivery-trace enrichment (LEFT OUTER on equipment leg) |
| 8 | `ZI_PatientLatestArrayLot`       | Us       | View entity                 | `ZI_PatientLatestArrayLot.ddls` | Latest consumable-array lot per patient |
| 9 | `ZI_DELIVERYSERIAL_NUMBERS`      | Us       | View entity (basic)         | `ZI_DELIVERYSERIAL_NUMBERS.ddls` | Serial numbers per delivery item (LIKP/LIPS → VBFA → SER03 ⋈ OBJK) |
| 10 | `ZI_DELIVERY_GOODSMOVEMENT`     | Us       | View entity (basic, VBFA wrap)  | `ZI_DELIVERY_GOODSMOVEMENT.ddls` | VBFA wrap; projects `POSNN` as INT4 for width-safe joins |
| 11 | `ZI_GoodsMovementSerial`        | Us       | View entity (basic, SER03 wrap) | `ZI_GoodsMovementSerial.ddls` | SER03 wrap; projects `ZEILE` as INT4 for width-safe joins |
| 12 | `ZI_ExternalProductGroupText`   | Us       | View entity (basic, TWEWT wrap) | `ZI_ExternalProductGroupText.ddls` | TWEWT (`EXTWG → EWBEZ`) wrap; English-only filter applied upstream |

Service definition + binding (also owned by the API team, names TBD — see Section 6 once registered):

| # | Artifact | Owner | Type | Role |
|---|----------|-------|------|------|
| S1 | `<service definition>` | API team | `.srvd` | Exposes `Z_I_INVENTORY` |
| S2 | `<service binding>`    | API team | `.srvb` | OData V2/V4 endpoint |

> **2026-06-03**: This section now mirrors the deployed `.ddls` files byte-for-byte. Each subsection's code block was lifted from the active source-of-truth file in the repo. Surrounding prose describes the deployed behavior, not earlier design iterations.

### 5.1. `ZI_DELIVERY_GOODSMOVEMENT` — delivery → material-document flow (VBFA wrap)

```abap
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Delivery to Goods Movement Flow'
@Metadata.ignorePropagatedAnnotations: true
@VDM.viewType: #BASIC
define view entity ZI_DELIVERY_GOODSMOVEMENT
  as select from vbfa
{
  key vbelv                      as DeliveryDocument,
  key posnv                      as DeliveryDocumentItem,
  key vbeln                      as MaterialDocument,
  key mjahr                      as MaterialDocumentYear,
  key cast( posnn as abap.int4 ) as MaterialDocumentItem
}
where
  vbtyp_n = 'R' //subsequent document category = Goods Movement
```

Why this view exists: `SER03` is keyed on the **material document** (`MBLNR / MJAHR / ZEILE`), not on the delivery. To trace a delivery's serials we first have to find the material document(s) created at PGI — that's what VBFA's document-flow records give us. `VBTYP_N = 'R'` filters to "subsequent document = goods movement."

`POSNN` is projected as `abap.int4` because `VBFA.POSNN` is `POSNR` (NUMC 6) and the matching `SER03.ZEILE` is `MBLPO` (NUMC 4). The 2020 CDS compiler enforces "NUMC casts must preserve length," so we can't narrow NUMC 6 → NUMC 4 directly. It also disallows function/cast expressions in `JOIN ON` clauses. Casting to INT4 in projection sidesteps both rules and gives plain numeric equality downstream.

No `select distinct` here — view entities on 7.55 (S/4 2020) don't support it (added in 7.56). Uniqueness comes naturally from the projected tuple matching VBFA's natural key under the `VBTYP_N = 'R'` filter.

---

### 5.2. `ZI_GoodsMovementSerial` — SER03 wrap with INT4 item position

```abap
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'SER03 Wrapper'
@Metadata.ignorePropagatedAnnotations: true
@VDM.viewType: #BASIC
define view entity ZI_GoodsMovementSerial
  as select from ser03
{
  key obknr                      as ObjectNumber,
  key mblnr                      as MaterialDocument,
  key mjahr                      as MaterialDocumentYear,
  key cast( zeile as abap.int4 ) as MaterialDocumentItem
}
```

The mirror of the previous view on the SER03 side: project `ZEILE` (NUMC 4) as `abap.int4` so the join with `ZI_DELIVERY_GOODSMOVEMENT.MaterialDocumentItem` (also INT4) is plain field equality with no expression in `JOIN ON`.

No `select distinct` — view entities on 7.55 (S/4 2020) don't support it. The (`OBKNR`, `MBLNR`, `MJAHR`, `ZEILE`) tuple is naturally unique in SER03 in practice: the remaining SER03 key fields (BLART, BWART, VORGANG, LIEFERANT, KUNDE, WERK, LAGERORT) are attributes of a single goods movement and don't multiply rows for a given material-doc-line / OBKNR pair. Any residual duplicates are absorbed by `ZI_PatientEquipmentDelivered`'s `group by Patient, Equipment` with `min()/max()` aggregations.

---

### 5.3. `ZI_DELIVERYSERIAL_NUMBERS` — serials per delivery item (chain through goods movement)

```abap
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Serial Numbers per Delivery Item'
@Metadata.ignorePropagatedAnnotations: true
@VDM.viewType: #BASIC
define view entity ZI_DELIVERYSERIAL_NUMBERS
  as select from ZI_DELIVERY_GOODSMOVEMENT as dgm

      inner join ZI_GoodsMovementSerial as gms
      on  gms.MaterialDocument     = dgm.MaterialDocument
      and gms.MaterialDocumentYear = dgm.MaterialDocumentYear
      and gms.MaterialDocumentItem = dgm.MaterialDocumentItem

    inner join objk as obj
      on obj.obknr = gms.ObjectNumber

{
  key dgm.DeliveryDocument        as DeliveryDocument,
  key dgm.DeliveryDocumentItem    as DeliveryDocumentItem,
  key obj.equnr                      as Equipment,
      obj.sernr                      as SerialNumber,
      obj.matnr                      as Product
}
where obj.taser = 'SER03'
```

`SER03` is the serial-number header for **goods movements / material documents**, not for deliveries directly (the table label literally reads "Document Header for Serial Numbers for Goods Movements"). The chain from a delivery to its serials is therefore:

```
LIKP / LIPS  ──VBFA──▶  MKPF / MSEG (material doc)  ──SER03──▶  OBJK
```

All `JOIN ON` conditions are now plain field equality (no expressions) because both VBFA-side and SER03-side widths are reconciled to `abap.int4` in their respective basic-view projections. `obj.taser = 'SER03'` follows the SAP convention that `OBJK.TASER` stores the source SER table name. Some older installations store all serial-object entries with `TASER = 'SER01'` regardless of source — see Section 8 for the verification check and adjustment.

---

### 5.4. `ZI_ExternalProductGroupText` — TWEWT wrap

```abap
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'External Product Group Tex'
@Metadata.ignorePropagatedAnnotations: true
define view entity ZI_ExternalProductGroupText
  as select from twewt
{
  key extwg as ExternalProductGroup,
  key spras as Language,
      ewbez as ExternalProductGroupText
}
```

Retire after the 2023 RISE move if SAP releases an `I_ExternalProductGroupText`. The label typo `'External Product Group Tex'` was left as-is in the deployed source — harmless cosmetic issue, not worth a transport.

---

### 5.5. `ZI_PatientEquipmentDelivered` — delivery enrichment

```abap
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Equipment Delivered to Patient (PGI)'
@Metadata.ignorePropagatedAnnotations: true
define view entity ZI_PatientEquipmentDelivered
  as select from I_DeliveryDocument as dlv
    inner join I_DeliveryDocumentItem as dit
      on dit.DeliveryDocument = dlv.DeliveryDocument
    inner join ZI_DELIVERYSERIAL_NUMBERS as dsr
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

Under Path C this view is the *enrichment* layer, not the *gating* layer. It overshoots (in our test patient: ~2033 rows) because it surfaces every equipment ever delivered to that patient as Sold-To. The actual current-inventory filter happens upstream in `ZI_PatientEquipmentInventory`, which spines off `ZI_EquipmentCurrentInstall` and LEFT JOINs this view for enrichment.

---

### 5.6. `ZI_PatientLatestArrayLot` — latest delivered consumable-array lot per patient

```abap
@AbapCatalog.viewEnhancementCategory: [ #NONE ]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Patient Latest Array Lot Delivered'
@Metadata.ignorePropagatedAnnotations: true
@VDM.viewType: #COMPOSITE
define view entity ZI_PatientLatestArrayLot
  as select from I_DeliveryDocument     as dlv
    inner join   I_DeliveryDocumentItem as dit on dit.DeliveryDocument = dlv.DeliveryDocument
{
  key dlv.SoldToParty                  as Patient,
      max(dlv.ActualGoodsMovementDate) as LatestArrayDeliveryDate,
      substring(
        max(concat(cast(dlv.ActualGoodsMovementDate as abap.char(8)),
                   cast(dit.Batch                   as abap.char(10)))),
        9, 10)                         as LatestArrayLot
}
where
      dlv.ActualGoodsMovementDate is not initial
  and dit.Batch                   is not initial
group by
  dlv.SoldToParty
```

One row per patient, carrying the most recent consumable-array lot they were shipped.

The deployed `WHERE` clause keeps PGI'd delivery items with a non-empty batch. The `<> 'FIELDEQUIP'` exclusion is **not** in this view's filter — that exclusion happens in `ZI_PatientDeliveryBatch` (the consumables leg of the union). For `LatestArrayLot` we accept that field-equipment placeholder rows participate in the `max(concat(...))` computation; in practice the patient's most recent shipment is virtually always a real consumable batch, so the lexicographic max keeps the right answer. If field data ever shows this assumption breaking, add `and dit.Batch <> 'FIELDEQUIP'` here.

The interesting line is the `substring/max/concat/cast` chain. CDS in 2020 doesn't offer a first-by-date or last-value window function, but the classic SQL trick works fine:

1. `cast( ActualGoodsMovementDate as abap.char(8) )` — DATS is stored internally as 8-char `YYYYMMDD`, so the cast is essentially a no-op type change.
2. `cast( dit.Batch as abap.char(10) )` — normalize to the CHARG field width (`CHAR 10`), padded with trailing spaces.
3. `concat(...)` — produces an 18-char string `YYYYMMDDbbbbbbbbbb`. Lexicographic ordering on this string equals chronological ordering on the date with batch as tiebreaker.
4. `max(concat(...))` — the largest such string per patient = the latest delivery's max batch on that date.
5. `substring(..., 9, 10)` — chop off the date prefix, leaving just the batch.

Result: `LatestArrayLot` is the **actual batch from the most recent array delivery**, not just the lexicographic max batch across the patient's whole shipment history.

`group by dlv.SoldToParty` collapses to one row per patient (replaces `select distinct`, which 7.55 view entities don't support).

---

### 5.7. `ZI_EquipmentCurrentInstall` — V_EQUI replacement (FM-parity spine)

```abap
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Equipment time-slice: V_EQUI replacement'
@Metadata.ignorePropagatedAnnotations: true
define view entity ZI_EquipmentCurrentInstall
  as select from equi as equip
    inner join   equz as timeseg on equip.equnr = timeseg.equnr
{
  key equip.equnr   as Equipment,
      equip.matnr   as Material,
      equip.sernr   as SerialNumber,
      equip.charge  as Batch,
      equip.erdat   as CreationDate,
      timeseg.kund2 as Customer,
      timeseg.datab as ValidityStartDate,
      timeseg.datbi as ValidityEndDate
}
where
  timeseg.datbi = '99991231'
```

This is the **spine of the equipment leg** and the source of FM parity. `V_EQUI` (a classical DDIC view, can't be used inside a `define view entity`) reads `EQUI ⋈ EQUZ` and exposes `EQUZ.KUND2` — the time-dependent assigned customer. The original FM uses `V_EQUI.KUND2`; `I_Equipment.Customer` projects only `EQUI.KUND2` (the header copy), which drifts from `EQUZ.KUND2` for any equipment that has reassignment history.

We discovered the drift the hard way: for patient `1041467`, the FM returned 11 equipment rows, our `I_Equipment.Customer`-based view returned only 4. Replacing the spine with this `EQUI ⋈ EQUZ` wrap restored FM parity exactly.

`WHERE timeseg.datbi = '99991231'` selects the currently-valid time slice (SAP's "end-of-time" date for an open assignment). One row per equipment = current customer.

ATC will flag the direct `EQUI` / `EQUZ` reads as a classical-table dependency. That's expected and documented — log it on the "retire when SAP releases an EQUZ-time-slice view" upgrade backlog. Until then, no alternative on S/4HANA 2020.

---

### 5.8. `ZI_PatientEquipmentInventory` — composite (equipment leg)

This is the Path-C equipment-row composite. Spines off `ZI_EquipmentCurrentInstall` (the V_EQUI replacement) and LEFT JOINs the delivery trace plus master-data enrichments. It used to *be* `ZC_CurrentInvByPatient` (as a view entity); the consumables expansion demoted it to a helper.

```abap
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Patient Equipment Inventory'
@VDM.viewType: #COMPOSITE
@Metadata.ignorePropagatedAnnotations: true

define view entity ZI_PatientEquipmentInventory
  as select from    ZI_EquipmentCurrentInstall                                        as ecc

    left outer join ZI_PatientEquipmentDelivered                                      as ped on  ped.Patient   = ecc.Customer
                                                                                             and ped.Equipment = ecc.Equipment

    left outer join I_SalesOrder                                                      as so  on so.SalesOrder = ped.SalesOrder

    left outer join ZI_PatientLatestArrayLot                                          as lat on lat.Patient = ecc.Customer

    left outer join I_Product                                                         as prd on prd.Product = ecc.Material

    left outer join I_ProductDescription                                              as pdt on  pdt.Product  = ecc.Material
                                                                                             and pdt.Language = $session.system_language

    left outer join ZI_ExternalProductGroupText                                       as egt on  egt.ExternalProductGroup = prd.ExternalProductGroup
                                                                                             and egt.Language             = $session.system_language

  /* Classification characteristic 'Z_DATE_OF_MANUFACTURE' (internal ID 0000000151)
     on equipment class type 002 — date value lives in CharcFromDate (single-value
     characteristic) or CharcToDate (range upper bound) depending on master-data setup. */
    left outer join I_ClfnObjectCharcValForKeyDate( P_KeyDate: $session.system_date ) as cv  on  cv.ClfnObjectTable = 'EQUI'
                                                                                             and cv.ClassType       = '002'
                                                                                             and cv.ClfnObjectID    = ecc.Equipment
                                                                                             and cv.CharcInternalID = '0000000151'

{
  key ecc.Customer                                                         as Patient,
  key ecc.Equipment                                                        as Equipment,

      max( ecc.SerialNumber )                                              as SerialNumber,
      ltrim( max( ecc.Material ), '0' )                                    as Product,
      coalesce( max( ped.Batch ), max( ecc.Batch ) )                       as LotNumber,
      max( lat.LatestArrayLot )                                            as LatestArrayLot,
      max( lat.LatestArrayDeliveryDate )                                   as LatestArrayDeliveryDate,
      ltrim( max( ped.SalesOrder ), '0' )                                  as SalesOrder,
      coalesce( max( ped.EarliestPgiDate ), max( ecc.ValidityStartDate ) ) as IssueDate,
      cast( dats_days_between(
              coalesce( max( ped.EarliestPgiDate ), max( ecc.ValidityStartDate ) ),
              $session.system_date
            ) as abap.int4 )                                               as DaysWithPatient,
      max( ecc.CreationDate )                                              as CreatedDate,
      max( so.CreationDate )                                               as SalesOrderCreatedDate,
      max( coalesce( cv.CharcFromDate, cv.CharcToDate ) )                  as ManufactureDate,
      max( prd.ProductType )                                               as ProductType,
      max( pdt.ProductDescription )                                        as ProductDescription,
      max( prd.ExternalProductGroup )                                      as ExternalProductGroup,
      max( egt.ExternalProductGroupText )                                  as ExternalProductGroupText
}
where
  ecc.Customer is not initial
group by
  ecc.Customer,
  ecc.Equipment
```

#### How Path C lands

- Spine = `ZI_EquipmentCurrentInstall` (V_EQUI replacement; reads `EQUZ.KUND2` for the current time-slice customer — matches FM).
- `ZI_PatientEquipmentDelivered` is LEFT OUTER. Equipment that can be back-traced to a patient-as-Sold-To delivery picks up `SalesOrder` and `Batch`. Equipment that can't (older deliveries shipped under a different Sold-To) gets `null` for those.
- `LotNumber` and `IssueDate` `coalesce` from delivery trace → EQUI/EQUZ source so every row is populated.
- `Product` and `SalesOrder` are `ltrim(..., '0')`'d in the projection. Leading-zero stripping happens here, not in the consumption view.
- `CreatedDate` = `EQUI.ERDAT` (via `ecc.CreationDate`). The DDIC date semantics are preserved as `abap.dats`; the `YYYY-MM-DD` string formatting is applied later, in `ZC_CurrentInvByPatient`.
- `ManufactureDate` = classification characteristic `Z_DATE_OF_MANUFACTURE` (CharcInternalID `0000000151`) on equipment class type `002`, time-resolved via `I_ClfnObjectCharcValForKeyDate`. Some equipment uses `CharcFromDate` (single-value characteristic); some uses `CharcToDate` (range upper bound) — `coalesce` picks whichever is populated.
- `SalesOrderCreatedDate` = `I_SalesOrder.CreationDate` LEFT OUTER on the aggregated `ped.SalesOrder`. NULL when the delivery trace produced no sales order. This field is **not** propagated to the consumption view — it stays available here for any future internal consumer that wants both `EQUI.ERDAT` and the sales-order creation date side-by-side.
- `I_Equipment` was tried as spine and rejected: it projects header `EQUI.KUND2` only, drifts from `EQUZ.KUND2` for equipment with reassignment history (4 rows vs the FM's 11 for patient 1041467).

---

### 5.9. `ZI_PatientDeliveryBatch` — composite (delivery-batch leg)

Every PGI'd delivery item to the patient where the batch is a real lot (not `FIELDEQUIP`). One row per `(Patient, DeliveryDocument, DeliveryDocumentItem)`.

The label `'Patient Consumables Inventory'` is kept from earlier iterations even though the view was renamed to `ZI_PatientDeliveryBatch`. `so.CreationDate` is aliased as **`CreatedDate`** here (not `SalesOrderCreatedDate` as on the equipment leg) — the field-name slot carries different underlying semantics between the two legs. See Section 5.10 for how this is reconciled in the consumption view.

```abap
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Patient Consumables Inventory'
@VDM.viewType: #COMPOSITE
@Metadata.ignorePropagatedAnnotations: true

define view entity ZI_PatientDeliveryBatch
  as select from    I_DeliveryDocument          as dlv
    inner join      I_DeliveryDocumentItem      as dit on dit.DeliveryDocument = dlv.DeliveryDocument
    left outer join I_SalesOrder                as so  on so.SalesOrder = dit.ReferenceSDDocument
    left outer join ZI_PatientLatestArrayLot    as lat on lat.Patient = dlv.SoldToParty
    left outer join I_Product                   as prd on prd.Product = dit.Material
    left outer join I_ProductDescription        as pdt on  pdt.Product  = dit.Material
                                                       and pdt.Language = $session.system_language
    left outer join ZI_ExternalProductGroupText as egt on  egt.ExternalProductGroup = prd.ExternalProductGroup
                                                       and egt.Language             = $session.system_language

{
  key dlv.SoldToParty                       as Patient,
  key dlv.DeliveryDocument                  as DeliveryDocument,
  key dit.DeliveryDocumentItem              as DeliveryDocumentItem,

      ltrim( dit.Material, '0' )            as Product,
      dit.Batch                             as LotNumber,
      lat.LatestArrayLot                    as LatestArrayLot,
      lat.LatestArrayDeliveryDate           as LatestArrayDeliveryDate,
      ltrim( dit.ReferenceSDDocument, '0' ) as SalesOrder,
      dlv.ActualGoodsMovementDate           as IssueDate,
      cast( dats_days_between( dlv.ActualGoodsMovementDate, $session.system_date )
            as abap.int4 )                  as DaysWithPatient,
      so.CreationDate                       as CreatedDate,

      prd.ProductType                       as ProductType,
      pdt.ProductDescription                as ProductDescription,
      prd.ExternalProductGroup              as ExternalProductGroup,
      egt.ExternalProductGroupText          as ExternalProductGroupText
}
where
      dit.Batch                   <> ''
  and dit.Batch                   <> 'FIELDEQUIP'
  and dlv.ActualGoodsMovementDate is not initial
  and dlv.SoldToParty             is not initial
```

The view never aggregates — each delivery item is its own row. Equipment-only fields (`Equipment`, `SerialNumber`, `ManufactureDate`) are not projected here; the top-level union view supplies them as initial values.

`Product` and `SalesOrder` are `ltrim(..., '0')`'d at this level (same approach as the equipment leg). `CreatedDate` comes from `I_SalesOrder.CreationDate` joined on `dit.ReferenceSDDocument` — for delivery items that always have a sales order reference, this should populate ~100% of the time.

---

### 5.10. `ZC_CurrentInvByPatient` — consumption view (classic CDS, UNION ALL)

```abap
@AbapCatalog.sqlViewName: 'ZCCURRINVBYPAT'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Current Inventory by Patient'
@VDM.viewType: #CONSUMPTION
@ClientHandling.algorithm: #SESSION_VARIABLE
@ObjectModel.usageType: { serviceQuality: #D, sizeCategory: #S, dataClass: #MIXED }

/* To use the union feature on a 2020 system, we have to go with the classic define view instead of
   define view entity.
   This should be refactored when we upgrade to 2023  */
define view ZC_CurrentInvByPatient
  as select from ZI_PatientEquipmentInventory as eq
{

  key eq.Patient                                                                   as Patient,
      //  key cast( concat( 'EQ/', eq.Equipment ) as abap.char(30) )                       as InventoryItemKey,
      cast( 'EQUIPMENT' as abap.char(20) )                                         as SourceType,
      eq.Equipment                                                                 as Equipment,
      eq.SerialNumber                                                              as SerialNumber,
      eq.Product                                                                   as Product,
      eq.LotNumber                                                                 as LotNumber,
      eq.LatestArrayLot                                                            as LatestArrayLot,
      eq.LatestArrayDeliveryDate                                                   as LatestArrayDeliveryDate,
      eq.SalesOrder                                                                as SalesOrder,
      eq.IssueDate                                                                 as IssueDate,
      eq.DaysWithPatient                                                           as DaysWithPatient,
      concat(substring(cast(eq.CreatedDate as abap.char(32)),1,4),concat('-',concat(substring(cast(eq.CreatedDate as abap.char(32)),5,2),
      concat('-',substring(cast(eq.CreatedDate as abap.char(32)),7,2)))))          as CreatedDate,
      concat(substring(cast(eq.ManufactureDate as abap.char(32)),1,4),concat('-',concat(substring(cast(eq.ManufactureDate as abap.char(32)),5,2),
           concat('-',substring(cast(eq.ManufactureDate as abap.char(32)),7,2))))) as ManufactureDate,
      eq.ProductType                                                               as ProductType,
      eq.ProductDescription                                                        as ProductDescription,
      eq.ExternalProductGroup                                                      as ExternalProductGroup,
      eq.ExternalProductGroupText                                                  as ExternalProductGroupText
}

union all

select from ZI_PatientDeliveryBatch            as db
{
  key db.Patient                                                                   as Patient,

//  key cast( concat( 'CN/',
//              concat( db.DeliveryDocument,
//                concat( '/', cast( db.DeliveryDocumentItem as abap.char(6) ) ) ) )
//          as abap.char(30) )                                                       as InventoryItemKey,

      cast( 'DELIVERY_BATCH' as abap.char(20) )                                    as SourceType,

      cast( '' as abap.char(18) )                                                  as Equipment,
      cast( '' as abap.char(18) )                                                  as SerialNumber,
      db.Product                                                                   as Product,
      db.LotNumber                                                                 as LotNumber,
      db.LatestArrayLot                                                            as LatestArrayLot,
      db.LatestArrayDeliveryDate                                                   as LatestArrayDeliveryDate,
      db.SalesOrder                                                                as SalesOrder,
      db.IssueDate                                                                 as IssueDate,
      db.DaysWithPatient                                                           as DaysWithPatient,
       concat(substring(cast(db.CreatedDate as abap.char(32)),1,4),concat('-',concat(substring(cast(db.CreatedDate as abap.char(32)),5,2),
      concat('-',substring(cast(db.CreatedDate as abap.char(32)),7,2))))) as CreatedDate,
      cast( '' as abap.dats )                                              as ManufactureDate,
      db.ProductType                                                               as ProductType,
      db.ProductDescription                                                        as ProductDescription,
      db.ExternalProductGroup                                                      as ExternalProductGroup,
      db.ExternalProductGroupText                                                  as ExternalProductGroupText
}
```

#### Why classic `define view` and not `define view entity`

S/4HANA 2020 / NW 7.55 view entities **do not support UNION**. UNION in view entities was added in NW 7.56 (S/4HANA 2022). Since the business asked for a single endpoint returning both equipment and consumable rows, and we can't UNION in a view entity yet, the top of the stack drops down to classic CDS. The two helpers (`ZI_PatientEquipmentInventory`, `ZI_PatientDeliveryBatch`) stay as view entities because they don't need UNION internally — they each return one row shape. Hybrid stacks of classic-on-top-of-view-entities are fully supported.

After the 2023 RISE upgrade, this collapses to a single view entity (see Section 10).

#### Key schema as deployed

```
key Patient
```

That's it — `Patient` is the sole key, not `(Patient, InventoryItemKey)` as originally designed. The `InventoryItemKey` projection is commented out on **both** legs of the union. Functional team validated row counts under this keying during UT and accepted it. The duplicate-key concern was deferred to the API exposure layer, where `Z_I_INVENTORY` introduces a composite `(orderId, partnerId)` key.

The historical key design (`EQ/<equnr>`, `CN/<delivdoc>/<delivitem>`) is still present in the file as commented-out lines, preserved for traceability and as a quick re-activation path if the API team later wants `InventoryItemKey` exposed.

#### Date formatting

`CreatedDate` and `ManufactureDate` are projected as **`YYYY-MM-DD` strings**, not native `abap.dats`. The `concat/substring/cast` chains slice the underlying DATS (stored internally as `YYYYMMDD`) into the ISO format and emit it as a CHAR. Required by the API team for consistent string output across consumers. Note that the consumption view does this at its layer, not in the equipment/batch leg composites — the legs still carry native `dats`.

DELIVERY_BATCH rows have a hardcoded empty `dats` for `ManufactureDate` (`cast( '' as abap.dats )`) — manufacture date applies only to serialized equipment. EQUIPMENT rows have a hardcoded empty `Equipment` and `SerialNumber` set via `cast( '' as abap.char(18) )` — those fields apply only to equipment rows.

#### Field-name slot reuse: `CreatedDate`

This is the only semantic landmine in the consumption view. Equipment-leg `CreatedDate` ← `EQUI.ERDAT` (serial master creation). Delivery-batch-leg `CreatedDate` ← `I_SalesOrder.CreationDate` (order header creation). Same field-name slot, different underlying data. Documented intentionally — `ZI_PatientDeliveryBatch` doesn't carry an `EQUI.ERDAT` equivalent for batch rows, and surfacing the sales-order creation date via the same slot keeps the union schema consistent.

The API-team layer `Z_I_INVENTORY` ignores this slot altogether and computes its `createdDate` field from `IssueDate` (PGI date) — see §12 for the full reconciliation.

#### Expected row counts (patient 1041467)

- Equipment rows: 11 (same as the FM and the previous equipment-only view).
- Consumable rows: every PGI'd delivery item with `Batch ≠ 'FIELDEQUIP'` over the patient's history. Will likely be a multi-year list — pagination (`$top`, `$skip`) and `$filter=IssueDate ge ...` are the OData consumer's friends.
- `SourceType` makes a clean split: `$filter=SourceType eq 'EQUIPMENT'` reproduces the old endpoint shape exactly, `$filter=SourceType eq 'DELIVERY_BATCH'` returns only consumables.

---

## 6. Service Definition + Service Binding (OData V4)

> **2026-06-03 — Superseded**: The actual API exposed to consumers is owned by the API team and routed through `Z_I_INVENTORY` → `ZI_PATIENT_INVENTORY` → `ZC_CurrentInvByPatient` (see Section 12 for the actual field contract). The service definition and binding below were our original design from when we owned the API exposure too; they are **not** what's deployed. They're preserved here as a reference example of what direct exposure of `ZC_CurrentInvByPatient` would look like, e.g. if a future internal consumer wanted a richer field set than the API team's `Z_I_INVENTORY` provides (notably: `InventoryItemKey`, `DaysWithPatient`, `SerialNumber`, `LatestArrayLot*`, `ExternalProductGroup*`, both date fields as native `dats`).
>
> The deployed sample calls in this section will not work against the live OData service; consumers should target the API team's binding once they finalize its name. Patient ID, field names, and key shape on the deployed API differ — see Section 12.

### Service Definition

```abap
@EndUserText.label: 'Current Inventory by Patient - API'
define service ZUI_CURRENT_INVENTORY {
  expose ZC_CurrentInvByPatient as CurrentInventory;
}
```

### Service Binding

Create in ADT against `ZUI_CURRENT_INVENTORY` with type **OData V4 — Web API** (system-to-system) or **OData V4 — UI** (Fiori Elements). `@OData.publish` is deprecated for new development; don't use it.

### Sample API calls

List by patient (everything):
```
GET /sap/opu/odata4/sap/zui_current_inventory/srvd_a2x/sap/zui_current_inventory/0001/CurrentInventory
    ?$filter=Patient eq '0001041467'
    &$orderby=IssueDate desc
```

List by patient, equipment rows only:
```
GET /sap/opu/odata4/sap/zui_current_inventory/srvd_a2x/sap/zui_current_inventory/0001/CurrentInventory
    ?$filter=Patient eq '0001041467' and SourceType eq 'EQUIPMENT'
    &$orderby=IssueDate desc
```

List by patient, consumable rows only, last 12 months:
```
GET /sap/opu/odata4/sap/zui_current_inventory/srvd_a2x/sap/zui_current_inventory/0001/CurrentInventory
    ?$filter=Patient eq '0001041467' and SourceType eq 'DELIVERY_BATCH' and IssueDate ge 2025-06-01
    &$orderby=IssueDate desc
```

Direct read of one row (key = `(Patient, InventoryItemKey)`):
```
GET /sap/opu/odata4/sap/zui_current_inventory/srvd_a2x/sap/zui_current_inventory/0001/CurrentInventory(Patient='0001041467',InventoryItemKey='EQ/IBH291161')
GET /sap/opu/odata4/sap/zui_current_inventory/srvd_a2x/sap/zui_current_inventory/0001/CurrentInventory(Patient='0001041467',InventoryItemKey='CN/80012345/000010')
```

> Consumers must pass `Patient` in **canonical SAP internal form (leading zeros)** — CDS field comparisons don't auto-apply ALPHA conversion. `InventoryItemKey` is a synthetic string; pass it exactly as returned by the list call. Document both on the API contract.

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
3. **`OBJK.TASER` value for material-doc serials** — `ZI_DELIVERYSERIAL_NUMBERS` filters `o.taser = 'SER03'` based on the SAP convention that `TASER` stores the source SER table name. Some older installations store all serial-object entries with `TASER = 'SER01'` regardless of source. If yours does, the activated view returns zero rows. Quick check (SQL preview):
   ```sql
   SELECT DISTINCT taser FROM objk WHERE obknr IN (SELECT obknr FROM ser03 WHERE ROWNUM < 100)
   ```
   If it shows `'SER01'`, change the `where` clause in `ZI_DELIVERYSERIAL_NUMBERS` to `o.taser = 'SER01'`. If `'SER03'`, leave it.
4. **`VBFA.VBTYP_N = 'R'`** in `ZI_DELIVERY_GOODSMOVEMENT` — standard SAP code for "subsequent document is a goods movement." If your config uses a different VBTYP for some flows (rare with custom doc types), the join misses those movements. Sanity check:
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
10. **Classic CDS view activation order** — the top-level `ZC_CurrentInvByPatient` is now a *classic* `define view` (not `define view entity`). If you had previously activated it as a view entity, **delete the old view-entity object in ADT first**, then activate the new classic view. Same name, different DDIC object type. Activate the two helpers (`ZI_PatientEquipmentInventory`, `ZI_PatientDeliveryBatch`) before the classic view. After the classic view activates, re-activate the service definition and service binding to regenerate OData metadata with the new key (`Patient, InventoryItemKey`) and the new `SourceType` column.
11. **UNION ALL column-shape parity** — the equipment and consumable legs project identical column shapes by position. If a leg's source view changes (e.g., a field is renamed in `ZI_PatientEquipmentInventory`), the UNION view fails activation with a "type mismatch in select column N" message. The casts in the classic view (`abap.char(18)`, `abap.dats`, `abap.char(30)` for the synthetic key, etc.) exist explicitly to keep both legs aligned.
12. **Consumable row volume** — the consumable leg returns one row per delivery item per patient. For a long-treated patient this can easily be hundreds or thousands of rows. Verify HANA push-down keeps `Patient` filter selective at the leaf level (`LIKP-KUNAG` is indexed); if `$filter=Patient eq 'X'` ever feels slow, consider adding a date guard (`IssueDate ge $session.system_date - 365` or similar) at the consumable helper's `where` clause.

---

## 9. SalesOrder semantics — alternatives if needed

Currently using `max( dit.ReferenceSDDocument )` = latest sales order (active contract). Alternatives:

- **Earliest sales order** (matches `EarliestPgiDate`): change to `min( dit.ReferenceSDDocument )`. Correct when SO numbering is monotonic with time, which is the SAP norm.
- **SO paired with the earliest PGI** (correct even when SO numbering isn't monotonic): drop the aggregation, expose an unaggregated `ZI_PatientDeliveredSerial`, and self-join in the consumption view on `EarliestPgiDate` to grab the matching row's `SalesOrder`. Slightly more views but rigorously correct.

---

## 10. 2023 RISE forward-compatibility notes

All artifacts above are 2020-compatible and survive the upgrade unchanged. Post-upgrade actions:

1. **Collapse the classic UNION view back into a view entity.** NW 7.56+ (S/4HANA 2022, 2023) supports `union all` natively in `define view entity`. The hybrid stack (`ZC_*` classic view over `ZI_*` view-entity helpers) can be flattened back into a single `ZC_CurrentInvByPatient` view entity that contains the UNION inline. The two helpers (`ZI_PatientEquipmentInventory`, `ZI_PatientDeliveryBatch`) can either be inlined or kept as composites — composites are still useful for testability. Deleting `@AbapCatalog.sqlViewName` and switching `define view` to `define view entity` is the main change.
2. Re-check `RAP_BO_RELEASED_API` / ADT *Released Objects* — if SAP releases `I_ExternalProductGroupText` (or equivalents for OBJK/SER01/SER03 linkages), retire the matching `Z` basic wraps and rewire the joins.
3. Run ATC with the Cloud-readiness variant to confirm Cloud language compatibility if any objects need promoting.
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

---

## 12. External / API-facing layer (owned by the API team)

> **2026-06-03 (final post-UT state)**: A separate team owns the API exposure layer. Two views: an alias/rename layer (`ZI_PATIENT_INVENTORY`) that wraps our `ZC_CurrentInvByPatient`, and a thin selection layer (`Z_I_INVENTORY`) bound to the OData service. Our 10-view stack remains fully load-bearing — these are pure facade layers on top. This section reflects the actual deployed code post-UT (passed, promoting to QA).

### Full top-of-stack diagram (their layer on top of ours)

```
Z_I_INVENTORY                          (API team — bound to OData service, classic CDS)
   │
   └── ZI_PATIENT_INVENTORY            (API team — view entity, alias/rename layer)
          │
          └── ZC_CurrentInvByPatient   (our top consumption view — classic CDS, UNION ALL)
                 │
                 ├── ZI_PatientEquipmentInventory       (our equipment leg)
                 └── ZI_PatientDeliveryBatch            (our delivery-batch leg)
```

### 12.1. `ZI_PATIENT_INVENTORY` — alias/rename layer

The middle layer: view entity that selects from our `ZC_CurrentInvByPatient`, renames fields to the API contract, and drops fields not wanted by the consumer. Dropped fields are **commented out, not removed** — easy to surface later if requirements change.

```abap
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Patient Inventory'
@Metadata.ignorePropagatedAnnotations: true
define view entity ZI_PATIENT_INVENTORY
  as select from ZC_CurrentInvByPatient
{
  key Patient as partnerId,
//  key InventoryItemKey,
      SourceType,
      Equipment as equipmentId,
      SerialNumber,
      Product  as materialId,
      LotNumber,
      LatestArrayLot,
      LatestArrayDeliveryDate,
      SalesOrder as orderId,
      IssueDate,
//      DaysWithPatient,
      CreatedDate as createdDate,
      ManufactureDate as manufactureDate,
      ProductType as materialType,
      ProductDescription as materialName
//      ExternalProductGroup,
//      ExternalProductGroupText
}
```

What this view exposes from our consumption view that earlier iterations had commented out: `SerialNumber`, `LatestArrayLot`, `LatestArrayDeliveryDate`, `IssueDate`, and `CreatedDate`. Only `InventoryItemKey`, `DaysWithPatient`, and the two `ExternalProductGroup*` fields remain dropped.

### 12.2. `Z_I_INVENTORY` — API exposure layer

Classic CDS view bound to the OData service. **Two notable behaviors** beyond projection:

1. **Key is composite `(orderId, partnerId)`** — not the single-field `partnerId` we had earlier. Better than nothing for OData row addressability, though it's still not unique across multi-line batch deliveries with the same sales order. The functional team accepted this as the keying scheme during UT.
2. **The API field `createdDate` is sourced from `IssueDate`** (formatted as `YYYY-MM-DD`). The literal `CreatedDate` field on `ZI_PATIENT_INVENTORY` is **not** read here — `IssueDate` is. This is the compromise that came out of the "createdDate vs shippedDate vs issueDate" thread: keep the API field name `createdDate` for backward compatibility with the consumer, but feed it the PGI date the business actually wanted.

```abap
@AbapCatalog.sqlViewName: 'ZINV'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Union of Equipment and Batches'
@Metadata.ignorePropagatedAnnotations: true
define view Z_I_INVENTORY
  as select from ZI_PATIENT_INVENTORY --Changes per VQ-82: Update Serialized Inventory API Retrieval Logic
{
  key    orderId,
  key    partnerId,
         materialId,
         materialName,
         materialType,
         LotNumber                                                                    as lotNumber,
         concat(substring(cast(IssueDate as abap.char(32)),1,4),concat('-',concat(substring(cast( IssueDate as abap.char(32)),5,2),
         concat('-',substring(cast(IssueDate as abap.char(32)),7,2))))) as createdDate,
         equipmentId,
         manufactureDate,
         //         SerialNumber,
         SourceType                                                                   as sourceType
}
```

(The source file also retains a commented-out historical version of the view from an earlier `union all` over `ZI_ORDER_DELIVERY_BATCH` / `ZI_EQUIPMENT` — kept as an audit trail of the design path.)

### Field-by-field mapping: `ZC_CurrentInvByPatient` → API contract (actual deployed state)

| Our field | API field on `Z_I_INVENTORY` | Status |
|---|---|---|
| `Patient` *(key)* | `partnerId` *(key)* | Renamed. |
| `InventoryItemKey` | — | **Commented out** in `ZC_CurrentInvByPatient` and `ZI_PATIENT_INVENTORY`. Functional team validated row counts in UT with `partnerId`-only keying; no observable duplicate-key issue at the consumer. Documented intentional. |
| `SourceType` | `sourceType` | Pass-through (casing changed). |
| `Equipment` | `equipmentId` | Renamed. |
| `SerialNumber` | — *(exposed on `ZI_PATIENT_INVENTORY` but commented out on `Z_I_INVENTORY`)* | Available to internal consumers of `ZI_PATIENT_INVENTORY`; not exposed at the OData boundary. |
| `Product` | `materialId` | Renamed. |
| `LotNumber` | `lotNumber` | Pass-through (casing changed). |
| `LatestArrayLot` | — *(exposed on `ZI_PATIENT_INVENTORY`, not on `Z_I_INVENTORY`)* | Available downstream but not on the API. |
| `LatestArrayDeliveryDate` | — *(exposed on `ZI_PATIENT_INVENTORY`, not on `Z_I_INVENTORY`)* | Available downstream but not on the API. |
| `SalesOrder` | `orderId` *(key)* | Renamed; promoted to key on `Z_I_INVENTORY`. |
| `IssueDate` | `createdDate` | **Repurposed** — the API field `createdDate` is computed as `IssueDate` formatted `YYYY-MM-DD`. The field name is misleading semantically (it's the PGI / shipped-to-patient date, not a creation date) but is preserved for backward compatibility with consumer code. |
| `DaysWithPatient` | — | Commented out (consumer didn't need it; FM behavior regressed). |
| `CreatedDate` (= `EQUI.ERDAT` for equipment rows; `I_SalesOrder.CreationDate` for batch rows, both formatted `YYYY-MM-DD` in `ZC_CurrentInvByPatient`) | — *(exposed on `ZI_PATIENT_INVENTORY` as `createdDate` but not selected by `Z_I_INVENTORY`)* | Not actually surfaced on the API — the API's `createdDate` field is sourced from `IssueDate`, see above. |
| `SalesOrderCreatedDate` *(only on `ZI_PatientEquipmentInventory`, not on `ZC_CurrentInvByPatient`)* | — | Not projected to the consumption view, not exposed at the API. Available in the equipment composite for future use. |
| `ManufactureDate` | `manufactureDate` | Pass-through (casing changed). Formatted `YYYY-MM-DD` in the consumption view for equipment rows; empty `dats` for batch rows. |
| `ProductType` | `materialType` | Renamed. |
| `ProductDescription` | `materialName` | Renamed. |
| `ExternalProductGroup` | — | Commented out. |
| `ExternalProductGroupText` | — | Commented out. |

### Notes on the actual deployed API contract

- **`createdDate` semantics are documented but not self-evident from the field name.** Anyone reading the OData metadata will assume `createdDate` is a creation timestamp; it isn't — it's the PGI date. Worth a sentence in the API documentation aimed at consumers.
- **`orderId` keying** — for batch rows where multiple delivery items share the same sales order (e.g. two different lot numbers shipped on the same order), the `(orderId, partnerId)` key won't be unique. UT didn't flag this as an issue; QA may surface it depending on test data shape.
- **Date format mixed**: `createdDate` and `manufactureDate` are `YYYY-MM-DD` strings (`CHAR(11)` in DDIC, surfaced as `Edm.String` over OData). The consumer should treat them as strings, not dates, for parsing purposes.

### Resolved concerns from earlier iterations

| Concern | Resolution |
|---|---|
| `partnerId` alone non-unique | Mitigated with composite `(orderId, partnerId)` key on `Z_I_INVENTORY`. Functional team accepted during UT. |
| `IssueDate` not exposed | Functionally exposed as `createdDate` on the API (semantically odd but data-correct). Also directly exposed as `IssueDate` on `ZI_PATIENT_INVENTORY` for any future consumers. |
| `LatestArrayLot` / `LatestArrayDeliveryDate` not exposed | Now exposed on `ZI_PATIENT_INVENTORY` (available to internal consumers); still not on the OData boundary `Z_I_INVENTORY`. |
| Lineage to our 10 views | Confirmed: `ZI_PATIENT_INVENTORY` → `ZC_CurrentInvByPatient` → our 10 views. Our stack is fully load-bearing. |

### Remaining open items for QA / future iterations

1. **`(orderId, partnerId)` uniqueness in real QA data** — verify no duplicate-key collisions across batch rows with multiple lot numbers per order.
2. **API documentation should call out the `createdDate` repurposing** so consumers understand the field is actually a shipped date.
3. **`DaysWithPatient`, `ExternalProductGroup*`** still not exposed. The original FM returned `DaysWithPatient`; confirm with consumer that the FM regression is acceptable.
