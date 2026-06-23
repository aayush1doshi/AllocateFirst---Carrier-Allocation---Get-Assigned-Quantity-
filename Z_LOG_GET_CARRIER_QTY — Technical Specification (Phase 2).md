# Technical Specification — Phase 2

| Field | Value |
|---|---|
| **Object** | `Z_LOG_GET_CARRIER_QTY` |
| **Function group** | `ZLOG_CARR_QTY` |
| **Reference DOI** | [DOI # Document of Intent — Carrier Quantity Function Module.txt](DOI%20%23%20Document%20of%20Intent%20%E2%80%94%20Carrier%20Quantity%20Function%20Module.txt) |
| **Reference logic** | `ZLOG_CARR_ASSIGN_NEW` method `LE_CAR_ASSI` (LE Domestic Create path) |
| **Reference docs** | `ZLOG_CARR_ASSIGN_NEW_Carrier_Tonnage.md`, `ZLOG_CARR_ASSIGN_NEW_LE_Domestic.md` |
| **Status** | Phase 2 — §6 decisions closed (interface revised 2025-06-19) |
| **Date** | 2025-06-19 |
| **Sibling FM** | `ZSCM_GET_SHIPMENT_UNASSIGNED` — same selection import pattern |

---

## 1. Purpose

This document closes DOI §6 open decisions and provides the build-ready technical design for `Z_LOG_GET_CARRIER_QTY`. It locks the **AC-D02** (date/DTAM intersection) and **AC-O02** (exception/return matrix) test matrices referenced in the acceptance criteria document.

---

## 2. Closed decisions — DOI §6

### §6.1 Source and destination field mapping — **CLOSED** (revised)

**Decision:** Source/destination are **not caller imports**. They are resolved internally for DTAM line gating and reconciliation tracing:

| Dimension | Resolved from | Table / join |
|---|---|---|
| Source (internal) | Shipping point on delivery | `LIKP-VSTEL` via `VTTP-VBELN` |
| Destination (internal) | Belt on shipment route | `ZLOG_BELT_ROUTE-BELT` via `VTTK-ROUTE` |

Caller supplies only **`IV_LDDAT`**, **`ITR_TPLST`**, **`ITR_SHTYP`**. All assigned shipments matching those filters contribute to `EV_TOTAL_QTY` (subject to DTAM and MFRGR rules).

**Reconciliation with `ZLOG_CARR_ASSIGN_NEW` (AC-R02):** Filter the report to the same `TPLST`/`SHTYP` values and date window as the FM call; compare grand total or per belt/MFRGR/carrier breakdown from manual trace.

---

### §6.2 Relationship to `ZLOG_DTAM` — **CLOSED** (revised)

| Question | Decision |
|---|---|
| Must an active `ZLOG_DTAM` row exist for the FM to execute? | **No** — FM executes when inputs and TVARVC date param are valid |
| Date rule: caller vs master | **Intersection at line level** — TVARVC window ∩ DTAM validity |
| Carrier validation via `Z_SCM_VENDOR_COMPANY_VALIDATE`? | **No** — carrier not a caller import; all assigned carriers in scope |

**Date and DTAM rules:**

1. **VTTK selection** uses **TVARVC window** from `IV_LDDAT`:
   ```
   LV_DATE_FROM ≤ VTTK-DPLBG ≤ LV_DATE_TO
   ```
2. **Line inclusion** additionally requires a matching **domestic belt-level** `ZLOG_DTAM` row where:
   - `BELT =` resolved belt from route
   - `MFRGR = MARC-MFRGR`
   - `WERKS` is initial (belt-level domestic row)
   - `DATE_FROM ≤ VTTK-DPLBG ≤ DATE_TO`
3. **Effective inclusion window** = intersection of TVARVC window and DTAM validity for the matching row.
4. **No `ZLOG_DTAM` row** for belt + MFRGR (domestic belt-level): line **excluded** (not a hard FM error).
5. **Inactive/expired DTAM**: lines **excluded** per rule 2.

---

### §6.3 Shipment status and carrier field — **CLOSED**

| Rule | Value |
|---|---|
| Minimum `VTTK-STTRG` | **`≥ '1'`** (LE Create assigned-history path) |
| Carrier field | `VTTK-TDNLR` |
| Assigned only | `TDNLR ≠ SPACE` (trimmed; spaces-only treated as unassigned) |
| Boundary | `STTRG = '1'` **included**; `STTRG = '0'` **excluded** |

---

### §6.4 Planning point and shipment type filters — **CLOSED** (revised)

| Rule | Value |
|---|---|
| Filter source | **Caller import range tables** — same pattern as `ZSCM_GET_SHIPMENT_UNASSIGNED` |
| Planning point | `ITR_TPLST` → `VTTK-TPLST IN itr_tplst` |
| Shipment type | `ITR_SHTYP` → `VTTK-SHTYP IN itr_shtyp` |
| TVARVC `Z_CARR_LE_DOM_TPP` / `Z_CARR_LE_DOM_SHIP` | **Not read in v1** — replaced by caller ranges |
| Empty `ITR_TPLST` or `ITR_SHTYP` | Raise **`NO_DATA_FOUND`** before any `VTTK` read (same as sibling FM) |

---

### §6.5 MFRGR authority — **CLOSED** (unchanged from DOI)

Use `MARC-MFRGR` for `MATNR + LIPS-WERKS`. Do not fall back to `LIPS-MFRGR`.

| Condition | Behavior |
|---|---|
| No `MARC` row | Line **excluded** from sum (no FM-level error) |
| Blank `MARC-MFRGR` | Line **excluded** (no DTAM match possible) |

---

### §6.6 Rounding and zero result — **CLOSED**

| Rule | Value |
|---|---|
| Rounding | Final `EV_TOTAL_QTY` rounded to **3 decimal places** (same as ZLOG tonnage reports); round **after** summing unrounded line MT |
| Zero-quantity lines (`LFIMG = 0`) | **Excluded** from sum and from `EV_LINE_COUNT` / shipment count |
| Negative quantities | **Included** — reduce `EV_TOTAL_QTY` (matches `LE_CAR_ASSI` COLLECT behavior) |
| No qualifying lines after VTTK (DTAM/MFRGR/belt gates) | `EV_TOTAL_QTY = 0`, `EV_RETURN` TYPE **`W`**, **no exception** |
| Empty `ITR_TPLST`/`ITR_SHTYP`, no VTTK in window | Raise **`NO_DATA_FOUND`** (same as sibling FM) |

---

## 3. Interface (final)

Selection imports match **`ZSCM_GET_SHIPMENT_UNASSIGNED`**, using LE field names (`TPLST` / `SHTYP`) instead of `VSTEL` / `VSART`.

```
FUNCTION z_log_get_carrier_qty.
*"----------------------------------------------------------------------
*"*"Local Interface:
*"  IMPORTING
*"     VALUE(IV_LDDAT)    TYPE LDDAT DEFAULT SY-DATUM
*"     VALUE(ITR_TPLST)   TYPE TPLST_RANGE OPTIONAL
*"     VALUE(ITR_SHTYP)   TYPE SHTYP_RANGE OPTIONAL
*"  EXPORTING
*"     VALUE(EV_TOTAL_QTY)       TYPE QUAN_13_3
*"     VALUE(EV_SHIPMENT_COUNT)  TYPE I OPTIONAL
*"     VALUE(EV_LINE_COUNT)      TYPE I OPTIONAL
*"     VALUE(EV_RETURN)          TYPE BAPIRET2
*"  EXCEPTIONS
*"     NO_DATE_PARAM_FOUND
*"     NO_DATA_FOUND
*"     CONVERSION_ERROR
*"----------------------------------------------------------------------
```

### 3.1 Import parameters

| Parameter | Type | Description | Maps to `VTTK` |
|---|---|---|---|
| `IV_LDDAT` | `LDDAT` | Planned load date anchor (defaults `SY-DATUM`) | Date window via TVARVC → `DPLBG` |
| `ITR_TPLST` | `TPLST_RANGE` | Planning point — multiple / range | `TPLST` |
| `ITR_SHTYP` | `SHTYP_RANGE` | Shipment type — multiple / range | `SHTYP` |

> Use `TYPE RANGE OF tplst` / `TYPE RANGE OF shtyp` if named range types are not in DDIC.

### 3.2 Parity with `ZSCM_GET_SHIPMENT_UNASSIGNED`

| `ZSCM_GET_SHIPMENT_UNASSIGNED` | `Z_LOG_GET_CARRIER_QTY` | Notes |
|---|---|---|
| `IV_LDDAT` | `IV_LDDAT` | Same — anchor date for TVARVC window |
| `IT_VSTEL` | `ITR_TPLST` | LE domestic uses **`TPLST`**, not `VSTEL` |
| `IT_VSART` | `ITR_SHTYP` | LE domestic uses **`SHTYP`**, not `VSART` |

| Object | Notes |
|---|---|
| `QUAN_13_3` | Domain `QUAN`, decimals 3 — MT total |
| Remote-enabled | **Yes** |

---

## 4. Processing logic

### Step 0 — Initialize outputs

Set `EV_TOTAL_QTY`, `EV_SHIPMENT_COUNT`, `EV_LINE_COUNT` to initial; clear `EV_RETURN`.

### Step 1 — Validate inputs

Raise **`NO_DATA_FOUND`** if:

- `ITR_TPLST[]` is initial, **or**
- `ITR_SHTYP[]` is initial

(Same guard as `ZSCM_GET_SHIPMENT_UNASSIGNED`.)

### Step 2 — Build date window (TVARVC)

Read `TVARVC` where `NAME = 'ZSCM_GET_SHIPMENT_DATE'` and `TYPE = 'P'`.

- `LV_DATE_FROM = IV_LDDAT - TVARVC-LOW`
- `LV_DATE_TO   = IV_LDDAT + TVARVC-LOW`

If no entry found → **`NO_DATE_PARAM_FOUND`** (no `VTTK` read).

> **Example:** `IV_LDDAT = 20260618`, `TVARVC-LOW = 3` → window `20260615`–`20260621`.

### Step 3 — Select assigned shipments (`VTTK`)

| Filter | Value |
|---|---|
| `TPLST` | IN `ITR_TPLST` |
| `SHTYP` | IN `ITR_SHTYP` |
| `DPLBG` | BETWEEN `LV_DATE_FROM` AND `LV_DATE_TO` |
| `STTRG` | `≥ '1'` |
| `TDNLR` | `≠ SPACE` (assigned only) |

Fields: `TKNUM`, `ROUTE`, `TDNLR`, `TPLST`, `SHTYP`, `DPLBG`.

If `LT_VTTK` is initial → **`NO_DATA_FOUND`**.

### Step 4 — Resolve belt (`ZLOG_BELT_ROUTE`)

FOR ALL ENTRIES on `LT_VTTK` where table is not initial. Drop shipments with no belt mapping.

### Step 5 — Deliveries and lines

`VTTP` on `TKNUM` → `LIPS` on `VBELN` → `MARC` on `MATNR`/`WERKS`.

**Line filters:**

1. `LFIMG ≠ 0`
2. Matching domestic belt-level `ZLOG_DTAM` row exists for resolved belt + `MARC-MFRGR` and `DATE_FROM ≤ DPLBG ≤ DATE_TO`

### Step 6 — Convert to MT

If `LIPS-VRKME = 'MT'`, use `LFIMG`; else `MATERIAL_UNIT_CONVERSION` to MT. Any failure → **`CONVERSION_ERROR`** (no partial sum).

### Step 7 — Aggregate

- Sum line MT across **all** qualifying assigned shipments in scope → round to 3 decimals → `EV_TOTAL_QTY`
- `EV_LINE_COUNT` = count of qualifying `LIPS` rows (after filters, before conversion)
- `EV_SHIPMENT_COUNT` = distinct `TKNUM` with ≥1 qualifying line

### Step 8 — Return

| Outcome | `EV_RETURN` | Exception |
|---|---|---|
| Success, qty > 0 | TYPE `S` | None |
| Success, qty = 0 | TYPE `W` — "No qualifying movement in period" | None |
| Empty range tables | TYPE `E` | `NO_DATA_FOUND` |
| TVARVC date param missing | TYPE `E` | `NO_DATE_PARAM_FOUND` |
| No VTTK in window | TYPE `E` | `NO_DATA_FOUND` |
| Conversion error | TYPE `E` | `CONVERSION_ERROR` |

---

## 5. Locked test matrix — AC-D02 (date / DTAM intersection)

Assumes belt `B01`, MFRGR `M1`, matching domestic `ZLOG_DTAM` row as stated per case.  
Date window from `IV_LDDAT = 20250615`, `TVARVC-LOW = 3` → **Jun 12 – Jun 18**.

| ID | TVARVC window | DTAM validity | `DPLBG` | Included? | Rationale |
|---|---|---|---|---|---|
| D2-1 | Jun 12 – Jun 18 | Jun 1 – Jun 30 | Jun 15 | **Yes** | In window and DTAM validity |
| D2-2 | Jun 12 – Jun 18 | Jun 15 – Jun 30 | Jun 13 | **No** | Before DTAM `DATE_FROM` |
| D2-3 | Jun 12 – Jun 18 | Jun 1 – Jun 30 | Jun 20 | **No** | After TVARVC window end |
| D2-4 | Jun 12 – Jun 18 | Jun 14 – Jun 16 | Jun 15 | **Yes** | In intersection (Jun 14 – Jun 16) |
| D2-5 | Jun 12 – Jun 18 | Jun 20 – Jun 30 | Jun 22 | **No** | Outside TVARVC window |
| D2-6 | Jun 12 – Jun 18 | *(no DTAM row)* | Jun 15 | **No** | No domestic belt-level DTAM row |

---

## 6. Locked test matrix — AC-O02 (exception / return)

| Scenario | `EV_TOTAL_QTY` | `EV_RETURN` | Exception | DB reads after failure point |
|---|---|---|---|---|
| `ITR_TPLST` or `ITR_SHTYP` initial | 0 | TYPE `E` | `NO_DATA_FOUND` | None |
| `ZSCM_GET_SHIPMENT_DATE` missing in TVARVC | 0 | TYPE `E` | `NO_DATE_PARAM_FOUND` | None |
| Valid inputs, no VTTK in window | 0 | TYPE `E` | `NO_DATA_FOUND` | After VTTK only |
| VTTK exists, no qualifying lines | 0 | TYPE `W` | None | Completed |
| Conversion failure on any line | 0 | TYPE `E` | `CONVERSION_ERROR` | Stopped at conversion |
| Success with qualifying lines | > 0 | TYPE `S` | None | Completed |

---

## 7. Tables read (explicit field lists — no `SELECT *`)

| Table | Purpose | Key fields |
|---|---|---|
| `TVARVC` | Date window offset (`ZSCM_GET_SHIPMENT_DATE`) | `NAME`, `TYPE`, `LOW` |
| `VTTK` | Assigned shipments | `TKNUM`, `ROUTE`, `TDNLR`, `TPLST`, `SHTYP`, `DPLBG`, `STTRG` |
| `VTTP` | Shipment–delivery link | `TKNUM`, `VBELN` |
| `LIPS` | Delivery quantities | `VBELN`, `POSNR`, `MATNR`, `WERKS`, `LFIMG`, `MEINS`, `VRKME` |
| `MARC` | MFRGR authority | `MATNR`, `WERKS`, `MFRGR` |
| `ZLOG_BELT_ROUTE` | Route → belt | `ROUTE`, `BELT` |
| `ZLOG_DTAM` | Line validity gate | `BELT`, `MFRGR`, `WERKS`, `DATE_FROM`, `DATE_TO` |

**No updates** to any table.

---

## 8. Sign-off gates satisfied

| DOI §6 item | Closed in | AC rows unlocked |
|---|---|---|
| 6.1 Source/dest mapping | §2.1 | AC-I05, AC-J03, AC-R02 |
| 6.2 DTAM relationship | §2.2, §5 | AC-D02–D04, AC-V01–V03 |
| 6.3 STTRG / TDNLR | §2.3 | AC-F01–F03, AC-R05 |
| 6.4 Planning point / shipment type filters | §2.4 | AC-I02–I05, AC-F04–F07 |
| 6.6 Rounding / zero result | §2.6, §6 | AC-U03, AC-O02, AC-V02–V03 |

---

## 9. Next phase

Phase 3 (build): function group `ZLOG_CARR_QTY`, FM implementation, SE37 unit tests per [AC_Z_LOG_GET_CARRIER_QTY.md](AC_Z_LOG_GET_CARRIER_QTY.md).
