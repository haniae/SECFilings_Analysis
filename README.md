# SECFilings_Analysis
# SECDB (XBRL) — Assignment 2, Group #5

This repository contains the reproducible steps we used to build a PostgreSQL database (**SECDB**) for SEC XBRL datasets and to load January data for **SUB, TAG, DIM, NUM, PRE** with pragmatic fixes for real‑world data issues.

> Source of truth for screenshots, SQL blocks, and rationale is the compiled report: `DNSC6305_Assignment2_Group#5.pdf`.

---

## 1) What this does

- Creates a fresh PostgreSQL database named **SECDB**.
- Defines tables for **SUB, TAG, DIM, NUM, PRE** with comments and sensible keys.
- Loads CSVs via `COPY` while handling common schema/data mismatches (length, precision, nullability, duplicates).
- Documents validation checks (row counts vs. CSV, integrity assumptions).

---

## 2) Prerequisites

- **PostgreSQL** client utilities on PATH (`dropdb`, `createdb`).
- **Python** with Jupyter (optional) and packages:
  - `ipython-sql==0.4.1`
  - `psycopg2==2.9.5` (or `psycopg2-binary==2.9.5`)

> These allow running SQL within a notebook and connecting to the local Postgres instance.

---

## 3) Quick start (local, terminal)

```bash
# 0) Adjust user if needed; assumes a local role 'student' without password
dropdb -U student SECDB || true
createdb -U student SECDB

# 1) (optional) Open a Jupyter notebook and run:
# %load_ext sql
# %sql postgresql://student@/SECDB
```

If you prefer plain `psql`, just connect and run the DDL in the order below.

---

## 4) Schema (DDL) overview

### SUB
- **PK:** `adsh`
- Rich filing metadata (CIK, company name, geographies, etc.).
- Notes: Some fields in raw CSVs have occasional NULLs; see _Load tips_.

### TAG
- **PK:** (`tag`, `version`)
- Includes `abstract`, `iord`, `tlabel`, `doc` (some may be NULL).

### DIM
- **PK:** `dimhash`
- `dimhash` can exceed 32 chars → use `VARCHAR(34)`
- `segments` may be NULL in practice.

### NUM
- **PK:** (`adsh`, `tag`, `version`, `ddate`, `qtrs`, `uom`, `dimh`, `iprx`)
- FKs: `adsh → SUB`, (`tag`,`version`) → `TAG`, `dimh → DIM.dimhash`
- Practical adjustments:
  - `dimh` as `VARCHAR(34)`
  - `dcml` may require widening beyond `NUMERIC(2,0)`
  - `value` may require widening beyond `NUMERIC(20,4)` for large magnitudes

### PRE
- **PK:** (`adsh`, `report`, `line`)
- FKs: `adsh → SUB`, (`tag`,`version`) → `TAG`
- Real datasets include NULLs in `stmt`/`plabel` → drop NOT NULL to ingest, then impute `stmt='UN'` where unknown; allow sparse `plabel`.

> Exact column lists and comments are captured in the PDF; use those as canonical definitions when building your tables.

---

## 5) Load tips (CSV → COPY)

**SUB**
- Expect occasional NULLs in address fields (`countryba`, `cityba`, etc.).
- `pubfloatusd` may overflow narrow numeric definitions—widen precision/scale before load.

**DIM**
- If `dimhash` > 32 chars → `ALTER COLUMN dimhash TYPE varchar(34)`.
- `segments` may be NULL—temporarily drop NOT NULL to load.
- When stacking multiple months, deduplicate to a **unique** `dim.csv` before `COPY`.

**NUM**
- If `dimh` > 32 chars → widen to `varchar(34)`.
- `dcml` may contain values like `-32768` → widen precision (e.g., `NUMERIC(5,0)`).
- `value` can exceed `NUMERIC(20,4)` → increase precision/scale.
- PK duplicates may surface when appending months—deduplicate upstream CSVs.

**PRE**
- `stmt` and `plabel` contain NULLs in real data:
  - Drop NOT NULL → load → impute `stmt='UN'` for unknowns.
  - Keep `plabel` NULLs (descriptive only) or backfill if authoritative labels exist.

---

## 6) Minimal execution order

1. Create **SUB**, **TAG**, **DIM**, **NUM**, **PRE** (in that order).
2. `COPY` load **SUB**, **TAG**, **DIM**.
3. `COPY` load **NUM** (after widening types if needed).
4. `COPY` load **PRE** (after handling `stmt`/`plabel` constraints).
5. Run validations (below).

---

## 7) Validation checks

- Compare table counts to CSV line counts minus header.
- Spot checks:
  ```sql
  -- Example row count checks (January examples)
  SELECT COUNT(*) FROM dim;   -- ~43,201
  SELECT COUNT(*) FROM num;   -- ~598,843
  SELECT COUNT(*) FROM pre;   -- ~511,413
  ```
- Integrity assumptions:
  - `TAG (tag,version)` is unique and referenced by `NUM` and `PRE`.
  - `SUB (adsh)` is referenced by `NUM` and `PRE`.

---

## 8) Troubleshooting (common errors → fixes)

| Error | Likely cause | Fix |
|---|---|---|
| `value too long for type character varying(32)` on `DIM.dimhash`/`NUM.dimh` | Keys up to 34 chars | `ALTER COLUMN ... TYPE varchar(34)` |
| `NOT NULL violation` on `DIM.segments` | Data contains NULLs | Drop NOT NULL → load; optionally audit afterward |
| `numeric field overflow` on `NUM.value` | Value exceeds precision/scale | Increase precision (e.g., `NUMERIC(22,4)` or `NUMERIC(22)`) |
| `numeric field overflow` on `NUM.dcml` | Values like `-32768` | Widen to `NUMERIC(5,0)` |
| `NOT NULL violation` on `PRE.stmt`/`plabel` | Real data has NULLs | Drop NOT NULL; impute `stmt='UN'`; keep/repair `plabel` |
| PK duplicate on `NUM` | Appending multi-month data | De‑dupe upstream CSVs (exact PK list above) |
| `SUB.pubfloatusd` overflow | Too narrow numeric type | Increase precision/scale (`NUMERIC(12,2)` or higher) |
| Address field NULLs in `SUB` | CSV contains blanks | Drop NOT NULL during ingest, then clean if needed |

---

## 9) Notes & assumptions

- We prioritize **loadability + provenance**: keep the raw facts, widen columns to fit reality, and preserve citations to sources in analysis. 
- Nullability is tuned to match observed data rather than idealized docs; where we relax constraints, we explicitly state why.

---

## 10) Credits

- Course: DNSC 6305 — Assignment 2 (Group #5)
- Report: `DNSC6305_Assignment2_Group#5.pdf` (includes full SQL blocks, screenshots, and discussion)
