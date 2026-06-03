# Converting `ZFM_FIORI_CURRENTINV` to a Clean-Core CDS-based API

Working notes from the design discussion. Target system: **S/4HANA 2020 on-premise** today, **S/4HANA 2023 on RISE (Private Cloud Edition assumed)** later this year. The CDS views will be exposed as an OData API; the original Function Module will be retired.

---

## 1. Source Function Module (for reference)

```abap
FUNCTION zfm_fiori_currentinv
  IMPORTING
    VALUE(iv_patientid) TYPE bu_partner
  EXPORTING
    VALUE(et_currentinv) TYPE zcurrentinvtt
    VALUE(es_return) TYPE bapiret2.

  TYPES: BEGIN OF ty_maktx,
           maktx TYPE maktx,
         END OF ty_maktx.
  TYPES: BEGIN OF ty_joininv,
           equnr TYPE equnr,
           datab TYPE datab,
           maktx TYPE maktx,
         END OF ty_joininv.
  TYPES: BEGIN OF ty_currentinv1,
           equnr  TYPE equnr,
           datab  TYPE datab,
           maktx  TYPE maktx,
           ewbez  TYPE ewbez,
           extwg  TYPE extwg,
           rmaind TYPE char1,
           matnr  TYPE matnr,
           charge TYPE charg,
         END OF ty_currentinv1.

  TYPES: BEGIN OF ty_rma,
           wbsta TYPE wbsta,
           vbeln TYPE vbeln,
         END OF ty_rma.

  DATA:
    lt_maktx       TYPE TABLE OF ty_maktx,
    ls_joininv     TYPE ty_joininv,
    lt_joininv     TYPE TABLE OF ty_joininv,
    ls_currentinv1 TYPE ty_currentinv1,
    lt_currentinv1 TYPE TABLE OF ty_currentinv1,
    ls_currentinv  TYPE zcurrentinv,
    lt_currentinv  TYPE TABLE OF zcurrentinv,
    lt_salesorder  TYPE TABLE OF bapiorders,
    lt_salesorders TYPE TABLE OF bapiorders,
    lv_kunnr       TYPE kunnr,
    lt_sofinal     TYPE TABLE OF bapiorders,
    lv_nodays      TYPE i,
    lt_rma         TYPE TABLE OF ty_rma,
    ls_rma         TYPE ty_rma,
    lv_rmaind      TYPE char1.

  CONSTANTS c_messageclass TYPE sy-msgid VALUE 'ZMSG_FIORI'.

  CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
    EXPORTING
      input  = iv_patientid
    IMPORTING
      output = iv_patientid.

  SELECT equnr, datab, matnr,charge FROM v_equi INTO TABLE @DATA(lt_curinv)
    WHERE kund2 = @iv_patientid.
  SORT lt_curinv BY matnr.
  IF sy-subrc <> 0.
    CALL FUNCTION 'BALW_BAPIRETURN_GET2'
      EXPORTING
        type   = 'E'
        cl     = c_messageclass
        number = '000'
      IMPORTING
        return = es_return.
    RETURN.
  ENDIF.

  " 175365 - QC: RMA Indicator Incorrect - Start
  IF lt_curinv IS NOT INITIAL.
    SELECT objk~equnr,
           s~kunde,
           s~vbtyp
      FROM objk
      INNER JOIN ser01 AS s ON ( objk~obknr EQ s~obknr )
      INTO TABLE @DATA(lt_rma_data)
      FOR ALL ENTRIES IN @lt_curinv
      WHERE objk~equnr EQ @lt_curinv-equnr
        AND objk~matnr EQ @lt_curinv-matnr
        AND objk~taser = 'SER01'
        AND s~vbtyp = 'T'
        AND s~kunde EQ @iv_patientid.
  ENDIF.

  lv_kunnr = iv_patientid.
  CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
    EXPORTING
      input  = lv_kunnr
    IMPORTING
      output = lv_kunnr.

  LOOP AT lt_curinv INTO DATA(ls_curinv).
    SELECT vbak~vbeln, vbak~kunnr, vbap~matnr
      INTO TABLE @DATA(lt_sorder)
      FROM vbak AS vbak
      INNER JOIN vbap AS vbap ON vbak~vbeln = vbap~vbeln
      INNER JOIN lips AS lips ON vbak~vbeln EQ lips~vgbel
      WHERE vbak~kunnr EQ @lv_kunnr
        AND vbap~matnr EQ @ls_curinv-matnr
        AND vbak~auart IN ('ZRE','ZRA','ZCPU').
  ENDLOOP.

  SORT lt_sorder BY vbeln.
  DELETE ADJACENT DUPLICATES FROM lt_sorder COMPARING vbeln.

  IF lt_sorder IS NOT INITIAL.
    SELECT lips~wbsta, vbak~vbeln
      INTO TABLE @DATA(lt_pgi)
      FROM vbak AS vbak
      INNER JOIN lips AS lips ON vbak~vbeln EQ lips~vgbel
      WHERE lips~wbsta IN ('A', 'B').
    SORT lt_pgi BY vbeln.
    DELETE ADJACENT DUPLICATES FROM lt_pgi COMPARING vbeln.
    LOOP AT lt_sorder INTO DATA(ls_sorder).
      READ TABLE lt_pgi INTO DATA(ls_pgi) WITH KEY vbeln = ls_sorder-vbeln BINARY SEARCH.
      INSERT LINES OF lt_pgi INTO TABLE lt_rma.
    ENDLOOP.
  ENDIF.

  CLEAR: ls_curinv.
  LOOP AT lt_curinv INTO ls_curinv.
    SELECT SINGLE maktx FROM makt INTO @DATA(lv_maktx)
      WHERE matnr = @ls_curinv-matnr AND spras = 'E'.
    SELECT SINGLE extwg FROM mara INTO @DATA(lv_extwg)
      WHERE matnr = @ls_curinv-matnr.
    SELECT SINGLE ewbez FROM twewt INTO @DATA(lv_ewbez)
      WHERE extwg = @lv_extwg.
    ls_currentinv1-equnr  = ls_curinv-equnr.
    ls_currentinv1-charge = ls_curinv-charge.
    ls_currentinv1-datab  = ls_curinv-datab.
    ls_currentinv1-maktx  = lv_maktx.
    ls_currentinv1-ewbez  = lv_ewbez.
    ls_currentinv1-extwg  = lv_extwg.
    ls_currentinv1-matnr  = ls_curinv-matnr.
    SHIFT ls_currentinv1-matnr LEFT DELETING LEADING '0'.
    IF lt_sorder IS INITIAL.
      ls_currentinv1-rmaind = ''.
    ELSEIF lt_sorder IS NOT INITIAL AND lt_rma IS NOT INITIAL.
      ls_currentinv1-rmaind = abap_true.
    ENDIF.
    APPEND ls_currentinv1 TO lt_currentinv1.
  ENDLOOP.

  IF sy-subrc = 0.
    SORT lt_currentinv1 DESCENDING BY datab.
  ENDIF.
  LOOP AT lt_currentinv1 INTO ls_currentinv1.
    lv_nodays = sy-datum - ls_currentinv1-datab.
    ls_currentinv-equnr     = ls_currentinv1-equnr.
    ls_currentinv-charg     = ls_currentinv1-charge.
    ls_currentinv-dateissue = ls_currentinv1-datab.
    ls_currentinv-nodaypat  = lv_nodays.
    ls_currentinv-maktx     = ls_currentinv1-maktx.
    ls_currentinv-ewbez     = ls_currentinv1-ewbez.
    ls_currentinv-bp        = iv_patientid.
    ls_currentinv-matnr     = ls_currentinv1-matnr.
    ls_currentinv-extwg     = ls_currentinv1-extwg.

    " 175365 - QC: RMA Indicator Incorrect
    READ TABLE lt_rma_data INTO DATA(ls_rma_data)
      WITH KEY equnr = ls_currentinv1-equnr
               kunde = iv_patientid
               vbtyp = 'T'.
    IF sy-subrc = 0.
      ls_currentinv-rmaind = 'X'.
    ELSE.
      CLEAR ls_currentinv-rmaind.
    ENDIF.
    APPEND ls_currentinv TO lt_currentinv.
  ENDLOOP.

  IF sy-subrc = 0.
    et_currentinv = lt_currentinv.
  ENDIF.

  CLEAR: lt_currentinv[], lt_currentinv[], ls_currentinv, ls_currentinv1,
         lt_currentinv1[], lt_rma[], lt_pgi[], lt_sorder[], lt_curinv[].
ENDFUNCTION.
```

---

## 2. Review of the FM

### What it does
- Input: patient (`bu_partner`).
- Output: rows of equipment currently issued to that patient, plus an RMA flag indicating an active return.

### Output fields (`zcurrentinv`)
- `equnr`, `charg`, `dateissue` (= `datab`), `nodaypat` (days at patient = `sy-datum - datab`)
- `maktx`, `ewbez`, `bp`, `matnr` (alpha-stripped), `extwg`
- `rmaind` — derived from `OBJK` + `SER01` (note #175365)

### Issues identified
1. **N+1 selects** in the loop against `MAKT`, `MARA`, `TWEWT` — prime candidate for code-pushdown.
2. **Dead code**: the `VBAK/VBAP/LIPS` → `lt_sorder`/`lt_pgi`/`lt_rma` block populates `ls_currentinv1-rmaind`, but the second `LOOP` overwrites it with the `OBJK`/`SER01` lookup. The VBAK chain is effectively orphaned.
3. `TWEWT-ewbez` select has **no `SPRAS` filter** — language-ambiguous.
4. `SY-SUBRC` check after `CONVERSION_EXIT_ALPHA_INPUT` is meaningless (alpha never fails).
5. `CLEAR: lt_currentinv[], lt_currentinv[], ...` — `lt_currentinv` cleared twice; harmless but sloppy.

---

## 3. Architectural decisions

- The CDS views will back an **OData API** — the FM is to be retired, not wrapped.
- We adopt **Clean Core**: build on **C1-released `I_*` views** where they exist, fall back to thin Z basic views for the gaps.
- Tier the views: **Z basic (wrap of classic) → consumption view (parameterized) → service definition → service binding**.
- A CDS **table function (AMDP)** is **not** required — all logic expresses in standard CDS SQL (joins, `dats_days_between`, `ltrim`, `CASE`, `$session.system_date/language`, `$parameters`).
- `@OData.publish` is deprecated for new dev — use Service Definition + Service Binding (OData V4).

### Mapping classical → released views, on **S/4HANA 2020**

| Classical | Released view (2020) | Action |
|---|---|---|
| `MARA` | `I_Product` | Use. |
| `MAKT` | `I_ProductDescription` | Use. |
| `TWEWT` | *not released in 2020* | Wrap as `ZI_ExternalProductGroupText`. |
| `EQUI` | `I_Equipment` (released but minimal field set; no `Customer`/time-dep fields) | Don't rely on it. Wrap `EQUI`+`EQUZ` as `ZI_EquipmentInstallation`. |
| `EQUZ` | no released view | Wrapped together with `EQUI`. |
| `OBJK` + `SER01` | no released view | Wrap as `ZI_EquipmentReturn`. |
| `V_EQUI` | — | Don't use from consumption views. |

---

## 4. Final CDS solution (works on 2020, future-proof for 2023 RISE)

### 4.1. Z basic view — equipment + current installation time-slice (replaces `V_EQUI`)

```abap
@AbapCatalog.sqlViewName: 'ZIEQPINST'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'EQUI+EQUZ wrap (classic-table extension)'
@VDM.viewType: #BASIC
@Metadata.allowExtensions: true

define view entity ZI_EquipmentInstallation
  as select from equi as e
    inner join equz as z
      on e.equnr = z.equnr
{
  key e.equnr           as Equipment,
      e.matnr           as Product,
      e.charge          as Batch,
      z.datab           as InstallationDate,
      z.datbi           as InstallationEndDate,
      z.kund2           as InstalledAtCustomer
}
where z.datbi = '99991231'                -- only the currently-valid time slice
```

### 4.2. Z basic view — active return (RMA) flag source

```abap
@AbapCatalog.sqlViewName: 'ZIEQPRET'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Equipment in Active SD Return'
@VDM.viewType: #BASIC

define view entity ZI_EquipmentReturn
  as select distinct from objk as o
    inner join ser01 as s
      on o.obknr = s.obknr
{
  key o.equnr    as Equipment,
  key o.matnr    as Product,
  key s.kunde    as Customer,
      s.vbtyp    as SDDocumentCategory
}
where o.taser = 'SER01'
  and s.vbtyp = 'T'
```

### 4.3. Z basic view — external product group text (2020 gap; retire post-2023 if released)

```abap
@AbapCatalog.sqlViewName: 'ZIEXTPGTXT'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
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

### 4.4. Consumption view — composes released + Z basics, parameterized by patient

```abap
@AbapCatalog.sqlViewName: 'ZCCURRENTINV'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Current Inventory by Patient'
@VDM.viewType: #CONSUMPTION
@ClientHandling.algorithm: #SESSION_VARIABLE
@ObjectModel.usageType: { serviceQuality: #D, sizeCategory: #S, dataClass: #MIXED }
@Search.searchable: true

define view entity ZC_CurrentInventoryByPatient
  with parameters
    P_PatientID : bu_partner

  as select from ZI_EquipmentInstallation as eq

    left outer join I_Product                    as prd
      on  prd.Product = eq.Product

    left outer join I_ProductDescription         as pdt
      on  pdt.Product  = eq.Product
      and pdt.Language = $session.system_language

    left outer join ZI_ExternalProductGroupText  as egt
      on  egt.ExternalProductGroup = prd.ExternalProductGroup
      and egt.Language             = $session.system_language

    left outer join ZI_EquipmentReturn           as rma
      on  rma.Equipment = eq.Equipment
      and rma.Product   = eq.Product
      and rma.Customer  = $parameters.P_PatientID

{
  @UI.lineItem: [{ position: 10, label: 'Equipment' }]
  key eq.Equipment                                                          as Equipment,

  @Semantics.batch.batchNumber: true
  eq.Batch                                                                  as Batch,

  @Semantics.businessDate.from: true
  eq.InstallationDate                                                       as IssueDate,

  @Semantics.quantity.unitOfMeasure: 'DAY'
  cast( dats_days_between( eq.InstallationDate, $session.system_date )
        as abap.int4 preserving type )                                      as DaysWithPatient,

  pdt.ProductDescription                                                    as ProductDescription,
  egt.ExternalProductGroupText                                              as ExternalProductGroupText,

  @Semantics.businessPartner.id: true
  cast( $parameters.P_PatientID as bu_partner preserving type )             as Patient,

  ltrim( eq.Product, '0' )                                                  as Product,
  prd.ExternalProductGroup                                                  as ExternalProductGroup,

  case when rma.Equipment is not null then cast( 'X' as char1 )
       else                                cast( ''  as char1 )
  end                                                                       as ReturnIndicator
}
where eq.InstalledAtCustomer = $parameters.P_PatientID
```

### 4.5. Service Definition

```abap
@EndUserText.label: 'Current Inventory by Patient - API'
define service ZUI_CURRENT_INVENTORY {
  expose ZC_CurrentInventoryByPatient as CurrentInventory;
}
```

### 4.6. Service Binding

Create in ADT against `ZUI_CURRENT_INVENTORY`. Binding type:
- **OData V4 — Web API** for system-to-system consumers.
- **OData V4 — UI** if a Fiori Elements list/object page consumes it.

Sample call:
```
GET /sap/opu/odata4/sap/zui_current_inventory/srvd_a2x/sap/zui_current_inventory/0001/
    CurrentInventory(P_PatientID='0000012345')/Set?$orderby=IssueDate desc
```

---

## 5. Decommission plan

- Switch all FM callers to the OData service.
- Delete `ZFM_FIORI_CURRENTINV`.
- Delete `ZCURRENTINV` / `ZCURRENTINVTT` once a grep confirms no remaining references — clients should bind to the CDS shape, not the DDIC structure.

---

## 6. 2023 RISE migration considerations (Private Cloud Edition assumed)

Confirmed-safe-to-use today on 2020 and forward-compatible:
- `define view entity` syntax (7.55+).
- Parameterized views, `$session.*`, `$parameters.*`.
- `dats_days_between`, `ltrim`, `cast(... preserving type)`.
- Service Definition + OData V4 Service Binding (Web API and UI variants).

### Pre-emptive work that pays off for the upgrade
1. **Develop in ADT (Eclipse)** — not SE80/SE11.
2. **ABAP language version = Standard ABAP** for now; revisit per-object on 2023.
3. **Put all new objects in one package** (e.g. `Z_CURRENT_INVENTORY_API`) for scope-checking during the upgrade.
4. **Run ATC with Cloud-readiness checks** periodically; fix flagged patterns.
5. **Tag each Z basic view header** with `'classic-table extension, replace with released view when available'` so the upgrade review has an explicit retirement candidate list.
6. **Pre-upgrade `RAP_BO_RELEASED_API`** check against 2023 will show which gap views (`ZI_ExternalProductGroupText`, possibly equipment-customer) finally have released equivalents → swap them out post-upgrade.
7. **Verify edition with basis team.** Private Cloud (typical RISE) preserves classic ABAP + `Z*`. Public Cloud blocks direct access to `EQUI`/`EQUZ`/`OBJK`/`SER01`/`TWEWT` — would force a different sourcing strategy.

### Items to verify in your 2020 system before activation
1. `I_Product` and `I_ProductDescription` field names in your stack (`Product` vs. `Material`) via ADT *Released Objects*.
2. `EQUZ.datbi = '99991231'` semantic for "current slice" matches the master-data convention in your system (it should — that's the standard valid-to placeholder).
3. Auth check behavior of `@AccessControl.authorizationCheck: #CHECK` against your `bu_partner` auth object — adjust to `#NOT_REQUIRED` only if explicitly intentional for the API.

---

## 7. Open questions / pending decisions

- **RISE edition**: Private Cloud (assumed) vs. Public Cloud — confirm with basis. Outcome changes whether `Z*` wraps of `EQUI/EQUZ/OBJK/SER01/TWEWT` survive the upgrade.
- **Consumer**: external system vs. Fiori Elements UI → drives Service Binding variant.
- **Patient ID alpha formatting at the caller**: the OData URL must pass `BU_PARTNER` with leading zeros (`'0000012345'`), since CDS parameters do not auto-apply ALPHA conversion. Document this on the API contract.

---

> **Note**: This file documents the **initial** equipment-master based design. It was later superseded by `updated_design.md`, which uses a delivery-trace algorithm per the functional team's revised spec (Sold-To deliveries + PGI + SER01 ship-to check). The CDS views shown above were never activated; the active design lives in `updated_design.md`.
