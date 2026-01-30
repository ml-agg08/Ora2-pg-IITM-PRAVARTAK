# Ora2Pg – Extended Feature Enhancements

This repository contains a **modified Ora2Pg codebase** with four major enhancements aimed at making Oracle → PostgreSQL migrations **safer, more correct, and production‑ready**. These changes address real-world migration gaps observed in package handling, incremental data sync, PL/SQL translation correctness, and type-mapping maintainability.

All features are implemented directly inside the Ora2Pg source and are designed to be **backward-compatible** with existing configurations unless explicitly enabled.

---

## Table of Contents

1. Package Function Visibility Preservation (Public vs Private)
2. Hybrid SCN‑Based Incremental Data Sync with Safe Upsert
3. Explicit Cursor Attribute Translation (%ISOPEN / %ROWCOUNT)
4. External Type Mapping with Selective Overrides (JSON‑based)

---

## 1. Package Function Visibility Preservation (Public vs Private)

### Problem

Oracle packages distinguish between **public routines** (declared in the PACKAGE SPEC) and **private helper routines** (defined only in the PACKAGE BODY).

Before this fix, Ora2Pg:

* Could not distinguish public vs private package routines
* Generated standalone PostgreSQL functions for *all* routines
* Injected `REVOKE ALL` statements but commented them out
* Forced users to manually uncomment revokes for private routines

This resulted in:

* Broken package encapsulation
* Private helper functions being exposed publicly
* Manual, error‑prone post‑migration fixes
* Potential security risks

---

### Solution: Two‑Pass Visibility Resolution

This feature introduces a **two‑pass approach** to correctly preserve Oracle package visibility semantics.

#### Pass 1 – Collect Public APIs (PACKAGE SPEC)

* While parsing the PACKAGE SPEC, Ora2Pg now extracts all declared function and procedure names.
* These names are stored in an internal lookup structure:

  * `self->{PACKAGE_PUBLIC}{package_name}{routine_name}`
* This acts as a *whitelist* of routines allowed to remain public.

#### Pass 2 – Enforce Visibility During Function Conversion

* During function generation:

  * If the routine exists in the public list → no action taken
  * If the routine is **not** in the public list → it is treated as private
* For private routines, Ora2Pg automatically injects:

  ```sql
  REVOKE ALL ON FUNCTION ... FROM PUBLIC;
  ```
* REVOKE statements are **no longer commented out**

---

### Result

* Public package APIs remain accessible
* Private helper routines are callable internally but hidden externally
* Package encapsulation is preserved in PostgreSQL
* No manual intervention required after migration

This makes PostgreSQL behavior closely mirror Oracle package semantics.

---

## 2. Hybrid SCN‑Based Incremental Data Sync with Safe Upsert

### Problem

In real‑world systems, data changes **during migration**. Previously, Ora2Pg:

* Re-exported *all* table data on every run
* Re-generated COPY / INSERT statements for unchanged rows
* Caused **duplicate primary key violations** on re‑import
* Was inefficient and unsafe for large databases

Oracle’s `ORA_ROWSCN` complicates this further because:

* SCNs are tracked at the **block level**, not per row
* Unchanged rows inside a modified block appear as “changed”

---

### Solution: Hybrid SCN + Upsert Strategy

This enhancement introduces a **reliable incremental sync mechanism**.

#### Step 1 – Smart SCN Filtering

* Ora2Pg is patched to filter rows using:

  ```sql
  WHERE ORA_ROWSCN > last_scn
  ```
* The highest processed SCN is stored in a log file after each run
* On subsequent runs, only rows from newly modified blocks are fetched

#### Step 2 – Safe Upsert Handling

* Because block-level SCN may include unchanged rows:

  * Pure INSERTs could fail due to duplicate keys
* Generated SQL now uses:

  ```sql
  INSERT ... ON CONFLICT DO UPDATE
  ```

This guarantees:

* New rows are inserted
* Existing rows are safely updated
* No primary key constraint errors

---

### Enabling the Feature

Set the following in `ora2pg.conf`:

```ini
ENABLE_INCREMENTAL_SNAPSHOT = 1
ENABLE_SCN_UPSERT = 1
```

---

### Result

* Only new or modified rows are migrated
* Migration becomes **efficient and repeatable**
* Suitable for production‑scale databases
* Enables true incremental Oracle → PostgreSQL sync

---

## 3. Explicit Cursor Attribute Translation (%ISOPEN / %ROWCOUNT)

### Problem

Oracle PL/SQL supports explicit cursor attributes such as:

* `cursor_name%ISOPEN`
* `cursor_name%ROWCOUNT`

Before this fix, Ora2Pg:

* Left these attributes untranslated
* Generated invalid PL/pgSQL
* Caused runtime errors like:

  ```text
  ERROR: column "isopen" does not exist
  ```

---

### Solution: Context‑Aware Cursor Attribute Translation

A new subroutine `handle_explicit_cursor_attributes` was introduced and integrated into the PL/SQL translation pipeline.

#### What It Does

1. **Detects explicit cursor declarations** in the DECLARE block
2. **Injects tracking variables**:

   * `<cursor>_is_open BOOLEAN`
   * `v_ora2pg_row_count INTEGER`
3. **Tracks cursor state**:

   * Sets `<cursor>_is_open := TRUE` after `OPEN`
   * Sets `<cursor>_is_open := FALSE` after `CLOSE`
4. **Translates attributes**:

   * `%ISOPEN` → `<cursor>_is_open`
   * `%ROWCOUNT` → `GET DIAGNOSTICS v_ora2pg_row_count = ROW_COUNT`

---

### Result

* Generated PL/pgSQL is valid and executable
* Cursor state and row counts behave correctly
* Runtime failures are eliminated
* Improves correctness of complex procedural migrations

---

## 4. External Type Mapping with Selective Overrides (JSON‑Based)

### Problem

Earlier Ora2Pg type mapping relied on inline config entries:

```ini
DATA_TYPE NUMBER(*,0):bigint
MODIFY_TYPE USERS:IS_ACTIVE:boolean
```

This caused:

* Poor readability
* Hard-to-maintain configurations
* No version control friendliness
* Limited flexibility for large schemas

---

### Solution: External JSON‑Based Type Mapping

A new configuration directive was introduced:

```ini
EXTERNAL_TYPE_MAP = /path/to/type_mapping.json
```

#### JSON Structure

```json
{
  "DATA_TYPE": {
    "NUMBER(*,0)": "bigint",
    "CLOB": "text"
  },
  "MODIFY_TYPE": {
    "USERS": {
      "IS_ACTIVE": "boolean"
    },
    "TASKS": {
      "PRIORITY_LEVEL": "smallint"
    }
  }
}
```

---

### Selective Override Logic

Mapping priority (highest wins):

1. Built‑in defaults
2. DATA_TYPE (config)
3. DATA_TYPE (JSON)
4. MODIFY_TYPE (config)
5. MODIFY_TYPE (JSON)

Column‑level overrides always take precedence.

---

### Additional Benefits

* Clean separation of concerns
* Easy auditing and debugging via logs
* Environment‑specific mappings supported
* Fully backward compatible

---

## Summary

These enhancements collectively make Ora2Pg:

* **More correct** (package semantics, cursors)
* **More efficient** (incremental sync)
* **Safer** (visibility control, upserts)
* **More maintainable** (external type mapping)

This codebase is well‑suited for **enterprise‑grade Oracle → PostgreSQL migrations** where correctness, repeatability, and maintainability are critical.

---

## Author

**Anand Gopan G**

Feature implementations, fixes, documentation, and testing
