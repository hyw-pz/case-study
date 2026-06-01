# Destatis Building Permits — Data Pipeline

## Data Cleaning

**Source**
Destatis Table 31111-0120, downloaded from GENESIS-Online. The raw file contains 177,408 rows covering January–December 2026, with published data for January–March 2026 and future placeholders for the remaining months.

**Filtering**
Three conditions are applied to isolate the target indicator — new residential dwelling permits by Bundesland:
- `4_variable_attribute_code == BTK-GEB-NEU` — new construction only, excludes renovations and extensions
- `5_variable_attribute_code starts with GBD-W` — residential building types only
- `value_variable_code == WOHN01` — dwelling count metric only

After filtering: 1,536 rows.

**Column mapping**
The raw file uses a generic numbered-column schema. The following mappings are applied:

| Raw column | Output column | Notes |
|---|---|---|
| `time` + `1_variable_attribute_code` | `time_period` | Combined into ISO 8601 format, e.g. `2026` + `MONAT03` → `2026-03` |
| `2_variable_attribute_code` | `region_code` | Zero-padded to 2 digits, e.g. `9` → `09` |
| `2_variable_attribute_label` | `region_name` | e.g. `Bayern` |
| `5_variable_attribute_code` | `building_type_code` | e.g. `GBD-W` |
| `5_variable_attribute_label` | `building_type_label` | e.g. `Residential buildings` |
| `value_variable_code` | `source_variable` | e.g. `WOHN01` |
| `value_variable_label` | `source_variable_label` | e.g. `Dwellings` |
| `value` | `value_numeric` | Converted to float; non-numeric sentinels set to NULL |
| `value_unit` | `unit` | e.g. `number` |

Columns containing only fixed repeated values (`N_variable_code`, `N_variable_label`) are dropped.

**Quality flags**
The raw `value` column contains non-numeric sentinel values. These are handled as follows:

| Raw value | Meaning | quality_flag | value_numeric |
|---|---|---|---|
| numeric string | valid data | `ok` | parsed float |
| `-` or `x` or `.` | suppressed for confidentiality | `suppressed` | NULL |
| `...` | future data not yet published | `future` | NULL |

Of the 1,536 filtered rows: 363 are `ok`, 21 are `suppressed`, 1,152 are `future`.

**Building type hierarchy**
The dataset contains 8 residential building type codes. `GBD-W` is the aggregate for all purely residential buildings. Its sub-types (`GBD-W-WO1`, `GBD-W-WO2`, `GBD-W-WO3UM`, `GBD-W-WH`) are preserved in the normalized layer for drill-down analysis. `GBD-W-NW` (mixed residential and non-residential) and `GBD-W-ETW` (owner-occupied) are orthogonal categories. The canonical aggregate uses `GBD-W` only to avoid double-counting.

**Provenance columns**
Every row in the normalized table carries the following traceability fields added at ingestion:

| Column | Value |
|---|---|
| `provider` | `destatis` |
| `source_table` | `31111-0120` |
| `ingestion_timestamp` | UTC timestamp of processing |
| `file_hash` | MD5 hash of the source CSV file |

---

## Validation Checks

The following checks are run on the normalized data before it is promoted to the canonical layer.

**Completeness**
All 16 Bundesländer must be present for each published month. The check groups rows by `time_period` and counts distinct `region_code` values. Any month with fewer than 16 states is flagged as a likely parse error.
Result: 0 months with missing states across all 3 published months.

**No negative values**
Permit counts cannot be negative. Any negative `value_numeric` is treated as a parsing error.
Result: 0 negative values.

**Quality flag distribution**
The breakdown of `ok`, `suppressed`, and `future` rows is logged at each run to detect unexpected shifts in data availability.
Result: ok 363 / suppressed 21 / future 1,152.

**Month-on-month spike detection**
For each state, the percentage change between consecutive months is computed. Changes exceeding ±80% trigger a warning flag for manual review. The threshold is set to catch parsing errors while allowing genuine large swings.

**Revision control**
When a new vintage of the file is ingested, values for the same `(region_code, time_period, source_variable)` are compared against the previously stored value. Revisions exceeding 20% are flagged for analyst review. The `ingestion_timestamp` column records which vintage each row belongs to.

**Region code validity**
All `region_code` values must exist in the AGS (Amtlicher Gemeindeschlüssel) reference table. Unknown codes are rejected rather than passed through silently.

**Duplicate check**
No duplicate `(region_code, time_period, source_variable, provider)` combinations are permitted in the normalized table. The file hash is checked before ingestion to skip re-processing of an identical file.

**YoY cross-validation**
The computed `regional_building_permits_yoy_change` is cross-validated against the official Destatis press release figure for the most recent available month. The computed value must match within rounding tolerance.
Note: YoY requires at least 13 months of history. With the current 3-month dataset, this check will become active once data from the prior year is ingested.
