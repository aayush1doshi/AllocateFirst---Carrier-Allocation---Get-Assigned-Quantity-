# Acceptance Criteria — `Z_LOG_GET_CARRIER_QTY`

| Field | Value |
|---|---|
| **Function module** | `Z_LOG_GET_CARRIER_QTY` |
| **Function group** | `ZLOG_CARR_QTY` |
| **Technical spec** | [Z_LOG_GET_CARRIER_QTY — Technical Specification (Phase 2).md](Z_LOG_GET_CARRIER_QTY%20%E2%80%94%20Technical%20Specification%20(Phase%202).md) |
| **Sibling FM** | `ZSCM_GET_SHIPMENT_UNASSIGNED` — same selection import pattern |
| **DOI** | [DOI # Document of Intent — Carrier Quantity Function Module.txt](DOI%20%23%20Document%20of%20Intent%20%E2%80%94%20Carrier%20Quantity%20Function%20Module.txt) |
| **Document status** | Ready for QA sign-off (interface revised 2025-06-19) |
| **Date** | 2025-06-19 |

§6 open decisions are closed in the Phase 2 technical spec. Matrices **AC-D02** and **AC-O02** are locked there and referenced below.

**Import parameters (v1):** `IV_LDDAT`, `ITR_TPLST`, `ITR_SHTYP` — parity with `ZSCM_GET_SHIPMENT_UNASSIGNED`, using LE field names (`TPLST` / `SHTYP`).

**Verified column:** ☐ = not yet tested | ☑ = passed

---

## 1. Deliverable and transport completeness (AC-T)

### AC-T01 — Activation without errors

**Given** the transport is imported into the target system  
**When** the following objects are activated in SE80/SE37  
**Then:**
- Function group `ZLOG_CARR_QTY` activates with zero errors
- Function module `Z_LOG_GET_CARRIER_QTY` activates with zero errors
- Custom type `QUAN_13_3` (if created) activates with zero errors
- Range types for `ITR_TPLST` / `ITR_SHTYP` (`TPLST_RANGE`, `SHTYP_RANGE` or `RANGE OF tplst/shtyp`) are active

| Verified |
|---|
| ☐ |

---

### AC-T02 — Interface data elements aligned to source fields

**Given** the FM signature is displayed in SE37  
**When** import/export parameters are compared to source SAP fields  
**Then:**

| Parameter | Expected data element / type | Maps to | Verified |
|---|---|---|---|
| `IV_LDDAT` | `LDDAT` | TVARVC date window → `VTTK-DPLBG` | ☐ |
| `ITR_TPLST` | `TPLST_RANGE` (or `RANGE OF tplst`) | `VTTK-TPLST` | ☐ |
| `ITR_SHTYP` | `SHTYP_RANGE` (or `RANGE OF shtyp`) | `VTTK-SHTYP` | ☐ |
| `EV_TOTAL_QTY` | `QUAN_13_3` (3-decimal MT) | — | ☐ |
| `EV_RETURN` | `BAPIRET2` | — | ☐ |

No ad-hoc types truncate planning-point codes or quantities.

---

### AC-T03 — Single transport completeness

**Given** the change is ready for promotion  
**When** the transport request is reviewed  
**Then:**
- Function group, FM, and all dependent types are in **one** transport request
- No split transport leaves FM callable against outdated signature/types in QAS/PRD

| Verified |
|---|
| ☐ |

---

### AC-T04 — SE37 smoke test

**Given** a known smoke-test setup exists in DEV (`IV_LDDAT`, non-empty `ITR_TPLST` / `ITR_SHTYP`, at least one assigned shipment in the TVARVC date window)  
**When** the FM is executed in SE37 with documented parameters  
**Then:**
- No short dump occurs
- `EV_TOTAL_QTY` and `EV_RETURN` are returned
- Optional exports may be omitted without error

| Verified |
|---|
| ☐ |

---

### AC-T05 — Remote-enabled RFC parity

**Given** the FM is marked remote-enabled and a valid RFC destination exists  
**When** the same smoke-test inputs are executed via RFC test (SE37 remote or consuming system)  
**Then:**
- RFC call completes without authorization or communication failure (for authorized user)
- `EV_TOTAL_QTY`, counts, and `EV_RETURN` match the local SE37 result exactly

| Verified |
|---|
| ☐ |

---

## 2. Input parameter boundary coverage (AC-I)

### AC-I01 — IV_LDDAT default

**Given** the caller passes initial/blank `IV_LDDAT`  
**When** the FM is executed  
**Then:**
- Default `SY-DATUM` is applied (set at FM signature level, not only assumed in code)
- Date window is built from `SY-DATUM` ± `TVARVC-LOW`

| Verified |
|---|
| ☐ |

---

### AC-I02 — Empty range tables

**Given** all other inputs would be valid  
**When** the FM is called with `ITR_TPLST[]` initial **or** `ITR_SHTYP[]` initial  
**Then:**
- `NO_DATA_FOUND` is raised before any `VTTK` read
- `EV_TOTAL_QTY` remains 0

| Verified |
|---|
| ☐ |

---

### AC-I03 — Multi-value ranges (TPLST / SHTYP)

**Given** `ITR_TPLST` with multiple `EQ` entries and at least one `BT` entry, and `ITR_SHTYP` with multiple values  
**When** the FM is called  
**Then:**
- Union of matching shipments is returned — no rows dropped due to range handling
- `VTTK-TPLST IN itr_tplst` and `VTTK-SHTYP IN itr_shtyp` both apply

| Verified |
|---|
| ☐ |

---

### AC-I04 — Non-matching TPLST / SHTYP

**Given** valid `IV_LDDAT` and TVARVC config, but `ITR_TPLST` / `ITR_SHTYP` values match zero `VTTK` rows  
**When** the FM is called  
**Then:**
- `NO_DATA_FOUND` is raised after VTTK select (not success with zero qty)

| Verified |
|---|
| ☐ |

---

### AC-I05 — TPLST / SHTYP exact match

**Given** assigned shipments with `TPLST = 1000`, `SHTYP = 01` and others with `TPLST = 2000`, `SHTYP = 02`  
**When** the FM is called with `ITR_TPLST = EQ 1000` and `ITR_SHTYP = EQ 01`  
**Then:**
- Only shipments matching both range filters contribute
- Partial or fuzzy matches are excluded

| Verified |
|---|
| ☐ |

---

### AC-I06 — TVARVC date parameter missing

**Given** `ITR_TPLST` and `ITR_SHTYP` are populated  
**And** `ZSCM_GET_SHIPMENT_DATE` does not exist in `TVARVC`  
**When** the FM is called  
**Then:**
- `NO_DATE_PARAM_FOUND` is raised before any `VTTK` read
- `EV_TOTAL_QTY` remains 0

| Verified |
|---|
| ☐ |

---

## 3. Date window and master-data interaction (AC-D)

### AC-D01 — TVARVC offset at month/year boundary

**Given** `IV_LDDAT = 20260301` and `TVARVC-LOW = 3` for `ZSCM_GET_SHIPMENT_DATE`  
**When** the FM is called  
**Then:**
- Date window is `20260226` to `20260304` with no date arithmetic dump
- Only shipments with `DPLBG` in that inclusive range are selected

| Verified |
|---|
| ☐ |

---

### AC-D02 — TVARVC window / DTAM intersection matrix

**Given** test data per locked matrix in Phase 2 spec §5 (cases D2-1 through D2-6)  
**When** the FM is executed for each case with `IV_LDDAT = 20250615`, `TVARVC-LOW = 3`, and matching DTAM setup  
**Then:**

| Case | Expected included? | Verified |
|---|---|---|
| D2-1 — in TVARVC window and DTAM validity | Yes | ☐ |
| D2-2 — before DTAM `DATE_FROM` | No | ☐ |
| D2-3 — after TVARVC window end | No | ☐ |
| D2-4 — in intersection only | Yes | ☐ |
| D2-5 — outside TVARVC window | No | ☐ |
| D2-6 — no DTAM row | No | ☐ |

---

### AC-D03 — No DTAM row

**Given** assigned shipments exist for belt + MFRGR with **no** domestic belt-level `ZLOG_DTAM` row (`WERKS` initial)  
**When** the FM is called with valid `IV_LDDAT`, `ITR_TPLST`, `ITR_SHTYP` covering those shipments  
**Then:**
- Lines are excluded from the sum
- If no other qualifying lines remain: `EV_TOTAL_QTY = 0` with `EV_RETURN` TYPE `W` (not a hard error)

| Verified |
|---|
| ☐ |

---

### AC-D04 — Inactive/expired DTAM row

**Given** a `ZLOG_DTAM` row where `DATE_TO < DPLBG` (expired) or `DATE_FROM > DPLBG` (not yet active)  
**And** `DPLBG` is within the TVARVC date window  
**When** the FM is called  
**Then:**
- Those lines are excluded from the sum per DTAM validity check

| Verified |
|---|
| ☐ |

---

### AC-D05 — TVARVC-LOW = 0 (single-day window)

**Given** `TVARVC-LOW = 0` for `ZSCM_GET_SHIPMENT_DATE`  
**When** the FM is called with `IV_LDDAT = 20250615`  
**Then:**
- Window collapses to single day (`LV_DATE_FROM = LV_DATE_TO = IV_LDDAT`)
- Only shipments with `DPLBG = 20250615` are selected at VTTK level

| Verified |
|---|
| ☐ |

---

## 4. Aggregation, join integrity, and cardinality (AC-J)

### AC-J01 — One shipment, multiple deliveries

**Given** a single assigned `TKNUM` linked to N deliveries via `VTTP`, each with qualifying `LIPS` lines  
**When** the FM is called with matching `IV_LDDAT`, `ITR_TPLST`, `ITR_SHTYP`  
**Then:**
- MT from all N deliveries is summed into `EV_TOTAL_QTY`
- No line is dropped due to join logic

| Verified |
|---|
| ☐ |

---

### AC-J02 — One delivery, mixed MFRGR

**Given** one delivery with lines for `M1` and `M2` (per `MARC-MFRGR`), each with a matching domestic belt-level `ZLOG_DTAM` row  
**When** the FM is called  
**Then:**
- Both MFRGR lines contribute (MFRGR is not a caller filter)
- Total equals manual sum of all DTAM-qualified lines

| Verified |
|---|
| ☐ |

---

### AC-J03 — Multiple carriers in scope

**Given** assigned shipments for carrier `C100` and `C200` both matching `ITR_TPLST` / `ITR_SHTYP` and date window  
**When** the FM is called  
**Then:**
- MT from **all** assigned carriers in scope is summed into `EV_TOTAL_QTY`
- Carrier is not a caller import — no single-carrier filter applied

| Verified |
|---|
| ☐ |

---

### AC-J04 — No double-count

**Given** a qualifying line identified by (`TKNUM`, `VBELN`, `POSNR`)  
**When** the FM is executed  
**Then:**
- That line contributes at most once to `EV_TOTAL_QTY` and `EV_LINE_COUNT`

| Verified |
|---|
| ☐ |

---

### AC-J05 — Orphan VTTP

**Given** an assigned shipment passes all `VTTK` filters but has no `VTTP` rows  
**When** the FM is called  
**Then:**
- Shipment contributes 0 MT
- No exception is raised (unless all VTTK rows are orphan → see AC-I04)

| Verified |
|---|
| ☐ |

---

### AC-J06 — Orphan LIPS

**Given** `VTTP` exists for a shipment but no `LIPS` rows for that `VBELN`  
**When** the FM is called  
**Then:**
- Delivery contributes 0 MT
- No exception is raised

| Verified |
|---|
| ☐ |

---

### AC-J07 — Distinct shipment count

**Given** multiple qualifying lines across one or more shipments  
**When** the FM is called with `EV_SHIPMENT_COUNT` exported  
**Then:**
- `EV_SHIPMENT_COUNT` equals the count of distinct `TKNUM` that contributed at least one qualifying line to `EV_TOTAL_QTY`

| Verified |
|---|
| ☐ |

---

### AC-J08 — Line count semantics

**Given** qualifying `LIPS` rows exist after all filters  
**When** the FM is called with `EV_LINE_COUNT` exported  
**Then:**
- `EV_LINE_COUNT` equals the count of qualifying `LIPS` rows **after filters, before conversion**
- Semantics are consistent with technical spec Step 7

| Verified |
|---|
| ☐ |

---

## 5. MFRGR authority and master-data gaps (AC-M)

### AC-M01 — MARC wins over LIPS

**Given** test data where `MARC-MFRGR ≠ LIPS-MFRGR` for the same material/plant  
**When** the FM is called  
**Then:**
- DTAM matching and line qualification use `MARC-MFRGR`, not `LIPS-MFRGR`

| Verified |
|---|
| ☐ |

---

### AC-M02 — Missing MARC row

**Given** a delivery line where no `MARC` row exists for `MATNR + WERKS`  
**When** the FM is called  
**Then:**
- Line is excluded from sum (no silent fallback to `LIPS-MFRGR`)

| Verified |
|---|
| ☐ |

---

### AC-M03 — Blank MARC-MFRGR

**Given** a delivery line where `MARC-MFRGR` is initial  
**When** the FM is called  
**Then:**
- Line is excluded from sum (no matching DTAM row possible)

| Verified |
|---|
| ☐ |

---

### AC-M04 — Multi-plant delivery

**Given** one delivery with lines in plants `1000` and `2000` with different `MARC-MFRGR` per plant  
**When** the FM is called  
**Then:**
- MFRGR is resolved per `LIPS-WERKS`; each line qualified independently against `ZLOG_DTAM`

| Verified |
|---|
| ☐ |

---

## 6. Unit conversion, rounding, and quantity edge cases (AC-U)

### AC-U01 — VRKME = MT passthrough

**Given** a qualifying line with `LIPS-VRKME = 'MT'`  
**When** the FM is called  
**Then:**
- Line quantity equals face `LFIMG` without calling `MATERIAL_UNIT_CONVERSION`

| Verified |
|---|
| ☐ |

---

### AC-U02 — Non-MT sales unit conversion

**Given** a qualifying line with `VRKME ≠ 'MT'` and a known material with documented conversion  
**When** the FM is called  
**Then:**
- Converted MT matches manual `MATERIAL_UNIT_CONVERSION` result for that material

| Verified |
|---|
| ☐ |

---

### AC-U03 — Three-decimal rounding on final sum

**Given** three qualifying lines each converting to `0.3333` MT (unrounded)  
**When** the FM is called  
**Then:**
- `EV_TOTAL_QTY` = `0.999` MT (sum then round to 3 decimals per Phase 2 §2.6)
- Edge-case rounding rule is documented in test evidence

| Verified |
|---|
| ☐ |

---

### AC-U04 — Zero-quantity lines

**Given** a line that would pass VTTK/TPLST/SHTYP filters but `LFIMG = 0`  
**When** the FM is called  
**Then:**
- Line does not increment `EV_TOTAL_QTY`, `EV_LINE_COUNT`, or `EV_SHIPMENT_COUNT`

| Verified |
|---|
| ☐ |

---

### AC-U05 — Negative quantities

**Given** a returns/credit line with negative `LFIMG` that otherwise qualifies  
**When** the FM is called  
**Then:**
- Negative quantity reduces `EV_TOTAL_QTY` (matches `LE_CAR_ASSI` behavior)

| Verified |
|---|
| ☐ |

---

### AC-U06 — Partial conversion failure

**Given** two qualifying lines where the second fails `MATERIAL_UNIT_CONVERSION`  
**When** the FM is called  
**Then:**
- `CONVERSION_ERROR` is raised
- `EV_TOTAL_QTY` is not returned as an authoritative partial sum

| Verified |
|---|
| ☐ |

---

### AC-U07 — Missing material master for conversion

**Given** a non-MT line where material master data required for conversion is missing  
**When** the FM is called  
**Then:**
- Treated as conversion failure (`CONVERSION_ERROR`), not silent zero

| Verified |
|---|
| ☐ |

---

## 7. Shipment selection filters (AC-F)

### AC-F01 — STTRG boundary (minimum included)

**Given** one assigned shipment with `STTRG = '1'` and one with `STTRG = '0'` (same `TPLST`/`SHTYP`/date)  
**When** the FM is called  
**Then:**
- `STTRG = '1'` shipment is included
- `STTRG = '0'` shipment is excluded

| Verified |
|---|
| ☐ |

---

### AC-F02 — STTRG above minimum

**Given** assigned shipments with `STTRG > '1'` matching `ITR_TPLST` / `ITR_SHTYP`  
**When** the FM is called  
**Then:**
- Higher-status shipments are included

| Verified |
|---|
| ☐ |

---

### AC-F03 — TDNLR edge cases (assigned only)

**Given** shipments where `TDNLR` is spaces-only or blank (unassigned)  
**When** the FM is called  
**Then:**
- Unassigned shipments are excluded
- Only `TDNLR ≠ SPACE` (assigned) shipments contribute

| Verified |
|---|
| ☐ |

---

### AC-F04 — ITR_TPLST filter

**Given** a shipment with `TPLST` outside the values in `ITR_TPLST`  
**When** the FM is called  
**Then:**
- Shipment is excluded at VTTK select

| Verified |
|---|
| ☐ |

---

### AC-F05 — ITR_SHTYP filter

**Given** a shipment with `SHTYP` outside the values in `ITR_SHTYP`  
**When** the FM is called  
**Then:**
- Shipment is excluded at VTTK select

| Verified |
|---|
| ☐ |

---

### AC-F06 — TVARVC date param missing

**Given** `ZSCM_GET_SHIPMENT_DATE` is missing from `TVARVC` in QAS  
**When** the FM is called with otherwise valid `ITR_TPLST` / `ITR_SHTYP`  
**Then:**
- `NO_DATE_PARAM_FOUND` is raised before any `VTTK` read
- FM does not proceed with an undefined date window

| Verified |
|---|
| ☐ |

---

### AC-F07 — No TVARVC TPLST/SHTYP allowlists in v1

**Given** v1 uses caller-supplied `ITR_TPLST` / `ITR_SHTYP` only (per Phase 2 §2.4)  
**When** code review is performed  
**Then:**
- FM does **not** read `Z_CARR_LE_DOM_TPP` or `Z_CARR_LE_DOM_SHIP`
- AC-F07 marked **N/A for v1** — filters come from caller ranges

| Verified |
|---|
| ☐ N/A (v1) |

---

## 8. Assigned-shipment scope and DTAM alignment (AC-V)

### AC-V01 — Assigned shipments only

**Given** shipments with `TDNLR` filled and blank in the same date/TPLST/SHTYP scope  
**When** the FM is called  
**Then:**
- Only assigned shipments (`TDNLR ≠ SPACE`) contribute to `EV_TOTAL_QTY`
- Unassigned shipments are never counted (distinct from sibling FM which selects unassigned)

| Verified |
|---|
| ☐ |

---

### AC-V02 — VTTK exists, no qualifying lines

**Given** assigned shipments match VTTK filters but no lines pass DTAM/MFRGR/belt gates  
**When** the FM is called  
**Then:**
- `EV_TOTAL_QTY = 0`
- `EV_RETURN` TYPE `W` (no exception)

| Verified |
|---|
| ☐ |

---

### AC-V03 — No VTTK in date window

**Given** valid `IV_LDDAT`, `ITR_TPLST`, `ITR_SHTYP`, and TVARVC config  
**And** no assigned shipments match the VTTK select  
**When** the FM is called  
**Then:**
- `NO_DATA_FOUND` is raised
- `EV_TOTAL_QTY = 0`

| Verified |
|---|
| ☐ |

---

## 9. Output semantics and exception matrix (AC-O)

### AC-O01 — EV_RETURN populated on all paths

**Given** success, warning, and error scenarios  
**When** the FM completes or raises  
**Then:**
- `EV_RETURN` contains `TYPE`, `ID`, `NUMBER`, and message text suitable for caller logging on every path

| Verified |
|---|
| ☐ |

---

### AC-O02 — Exception vs return-code matrix

**Given** test data for each scenario in Phase 2 spec §6  
**When** the FM is executed per scenario  
**Then:**

| Scenario | Expected behavior | Verified |
|---|---|---|
| `ITR_TPLST` or `ITR_SHTYP` initial | `NO_DATA_FOUND`; `EV_TOTAL_QTY = 0`; TYPE `E`; no DB reads | ☐ |
| `ZSCM_GET_SHIPMENT_DATE` missing | `NO_DATE_PARAM_FOUND`; TYPE `E`; no VTTK read | ☐ |
| Valid inputs, no VTTK in window | `NO_DATA_FOUND`; TYPE `E` | ☐ |
| VTTK exists, no qualifying lines | `EV_TOTAL_QTY = 0`; TYPE `W`; no exception | ☐ |
| Conversion failure | `CONVERSION_ERROR`; `EV_TOTAL_QTY = 0`; TYPE `E` | ☐ |
| Success with data | `EV_TOTAL_QTY > 0`; TYPE `S`; counts populated | ☐ |

---

### AC-O03 — Optional exports omitted

**Given** a successful FM call  
**When** the caller does not pass `EV_SHIPMENT_COUNT` or `EV_LINE_COUNT` (optional exports omitted)  
**Then:**
- No short dump occurs

| Verified |
|---|
| ☐ |

---

### AC-O04 — Initial output values on error paths

**Given** an error path (`NO_DATA_FOUND`, `NO_DATE_PARAM_FOUND`, `CONVERSION_ERROR`)  
**When** the exception is raised  
**Then:**
- `EV_TOTAL_QTY` and counts are initial/zero
- Two consecutive error calls do not return stale values from a prior success in the same session

| Verified |
|---|
| ☐ |

---

## 10. Code safety, performance, and read-only guarantee (AC-S)

### AC-S01 — FOR ALL ENTRIES guards

**Given** the FM source code is reviewed  
**When** each `FOR ALL ENTRIES` select is inspected  
**Then:**
- `LT_VTTK` checked before `VTTP`
- `LT_VTTP` checked before `LIPS`
- Material keys checked before `MARC`
- No FAE on initial driver table

| Verified |
|---|
| ☐ |

---

### AC-S02 — No SELECT *

**Given** the FM source code is reviewed  
**When** all SELECT statements are inspected  
**Then:**
- Every select uses an explicit field list
- No `SELECT *` appears

| Verified |
|---|
| ☐ |

---

### AC-S03 — No side effects

**Given** the FM is executed for a valid call  
**When** database changes are monitored (ST05 or table logging)  
**Then:**
- No update/insert/delete on `VTTK`, `ZLOG_TAA`, `ZLOG_DTAM`, or related globals

| Verified |
|---|
| ☐ |

---

### AC-S04 — No nested selects in loop

**Given** the FM source code is reviewed (or SAT trace on representative volume)  
**When** the main processing loop is inspected  
**Then:**
- No nested SELECT inside loop over result set

| Verified |
|---|
| ☐ |

---

### AC-S05 — Runtime within threshold

**Given** representative volume for one `ITR_TPLST` / `ITR_SHTYP` / date-window call (same order of magnitude as `LE_CAR_ASSI` read)  
**When** the FM is executed in SE37 or from batch caller  
**Then:**
- Execution completes without timeout within agreed threshold (document runtime in test evidence)

| Verified |
|---|
| ☐ |

---

### AC-S06 — Idempotency

**Given** identical import parameters (`IV_LDDAT`, `ITR_TPLST`, `ITR_SHTYP`)  
**When** the FM is called twice consecutively in the same session  
**Then:**
- Both calls return identical `EV_TOTAL_QTY`, counts, and `EV_RETURN-TYPE`

| Verified |
|---|
| ☐ |

---

## 11. Reference reconciliation with ZLOG_CARR_ASSIGN_NEW (AC-R)

### AC-R01 — Fixed reconciliation suite (≥5 scenarios)

**Given** a fixed suite of at least 5 scenarios with manual baselines from `LE_CAR_ASSI`  
**When** the FM is called with `IV_LDDAT`, `ITR_TPLST`, `ITR_SHTYP` matching the report selection  
**Then:**
- `EV_TOTAL_QTY` matches manual sum of assigned MT for the same TPLST/SHTYP/date window

| Scenario # | Description | FM total | Manual total | Verified |
|---|---|---|---|---|
| R-1 | Non-zero, single shipment | | | ☐ |
| R-2 | Non-zero, multi-shipment / multi-carrier | | | ☐ |
| R-3 | Zero movement (no qualifying lines) | | | ☐ |
| R-4 | Cross-month TVARVC window | | | ☐ |
| R-5 | Multiple TPLST values in range | | | ☐ |

---

### AC-R02 — Aligned selection dimensions

**Given** report assigned-history read uses `TPLST`, `SHTYP`, and date window  
**When** reconciliation is performed using `ITR_TPLST`, `ITR_SHTYP`, and `IV_LDDAT` + same TVARVC offset  
**Then:**
- FM grand total matches report total for the same scope
- Reconciliation evidence documents TPLST/SHTYP values and date window used

| Verified |
|---|
| ☐ |

---

### AC-R03 — Same read chain

**Given** one reconciliation scenario  
**When** MT total is traced manually through VTTK → VTTP → LIPS → MARC → conversion  
**Then:**
- FM total matches the manual chain sum (no alternate shortcut query)

| Verified |
|---|
| ☐ |

---

### AC-R04 — Regression after interface revision

**Given** interface revision (`IV_LDDAT`, `ITR_TPLST`, `ITR_SHTYP`) implemented per Phase 2 spec  
**When** the full reconciliation suite (AC-R01) is re-run  
**Then:**
- All scenarios pass before final sign-off

| Verified |
|---|
| ☐ |

---

### AC-R05 — STTRG and assigned-carrier parity

**Given** the same `ITR_TPLST`, `ITR_SHTYP`, date window, `STTRG ≥ '1'`, and assigned-only rule as Create path  
**When** shipments excluded in `LE_CAR_ASSI` assigned-history read are compared to FM output  
**Then:**
- Excluded shipments in the report are also excluded by the FM

| Verified |
|---|
| ☐ |

---

## 12. Operational and consumption readiness (AC-P)

### AC-P01 — SE37 documentation

**Given** the FM is displayed in SE37  
**When** short text and parameter help are reviewed by a developer without ABAP access  
**Then:**
- Purpose, each import/export, exceptions, and parity with `ZSCM_GET_SHIPMENT_UNASSIGNED` are sufficient to integrate the FM
- Help text documents `ITR_TPLST` → `VTTK-TPLST`, `ITR_SHTYP` → `VTTK-SHTYP`, `IV_LDDAT` → TVARVC window on `DPLBG`

| Verified |
|---|
| ☐ |

---

### AC-P02 — Integrated caller test

**Given** at least one consuming program or interface is wired to the FM  
**When** an end-to-end test is executed (not SE37-only)  
**Then:**
- Caller invokes FM with `IV_LDDAT`, `ITR_TPLST`, `ITR_SHTYP` and uses `EV_TOTAL_QTY` in its decision path

| Verified |
|---|
| ☐ |

---

### AC-P03 — Remote authorization

**Given** the FM is remote-enabled  
**When** an unauthorized RFC user attempts to call the FM  
**Then:**
- Standard SAP authorization failure is returned
- Behavior is documented for support

| Verified |
|---|
| ☐ |

---

### AC-P04 — Support runbook

**Given** unexpectedly zero result or exception in production support  
**When** the support runbook is consulted  
**Then:**
- Runbook lists checks for: `IV_LDDAT`, `ITR_TPLST`, `ITR_SHTYP` values, `ZSCM_GET_SHIPMENT_DATE` in TVARVC, DTAM validity, belt route mapping, and assigned-carrier filter

| Verified |
|---|
| ☐ |

---

### AC-P05 — Landscape readiness

**Given** DEV, QAS, and PRD landscapes  
**When** readiness is checked before production consumption  
**Then:**
- TVARVC entry `ZSCM_GET_SHIPMENT_DATE` exists with agreed offset per landscape
- `ZLOG_BELT_ROUTE` and domestic belt-level `ZLOG_DTAM` data exist for reconciliation scenarios

| Landscape | TVARVC date param ready | Mapping master ready | Verified |
|---|---|---|---|
| DEV | | | ☐ |
| QAS | | | ☐ |
| PRD | | | ☐ |

---

*End of document — 65 acceptance criteria (AC-T01 through AC-P05, incl. AC-D05)*
