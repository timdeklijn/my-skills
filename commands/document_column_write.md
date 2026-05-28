---
description: "Write subtask: interpret scan findings and produce the final standardized SQL Server column documentation file"
subtask: true
---

# Column Write Subtask

You are a senior data engineer and technical writer.
You have been handed a structured findings file produced by a SQL Server 2016 codebase scan.
Your job is to interpret the evidence and write clear, accurate, business-facing documentation.

You do not run searches. You do not look at source files directly.
You reason only from what is in the findings file and `project_index.md`.

## Input

Column to document: **`$ARGUMENTS`**

---

## Step 1 — Read context

@project_index.md

Derive the table name, then read the findings file:

@docs/columns/$ARGUMENTS.findings.md

_(If the findings file path needs the table name prefix, derive it from `project_index.md` first.)_

---

## Step 2 — Reason over the findings

Work through these questions before writing a single word of output:

**Definition**
- What does the SQL Server data type imply?
  - `BIT` → boolean; document the meaning of `0`, `1`, and `NULL` separately
  - `NVARCHAR` vs `VARCHAR` → unicode matters; note if non-ASCII values are expected
  - `DATETIME2` vs `DATETIME` → precision and range difference; note which is used and why
  - `UNIQUEIDENTIFIER` → is this a natural or surrogate key? client-generated or `NEWID()`/`NEWSEQUENTIALID()`?
  - `DECIMAL`/`NUMERIC(p,s)` → what do the precision and scale tell you about the domain? (e.g. currency, percentage)
  - `ROWVERSION` / `TIMESTAMP` → optimistic concurrency marker, not a real timestamp
- Does an extended property (`MS_Description`) exist? If yes, it is the authoritative definition. Your prose must be consistent with it. If application behaviour contradicts it, flag with ⚠️.
- Did the column change type or constraints across migrations? What does that history say about how its meaning evolved?
- Is it a computed column (`AS expression PERSISTED`)? If so, document the expression and what it means.
- Is it an `IDENTITY` column? Document seed and increment, and whether application code ever overrides it with `SET IDENTITY_INSERT ON`.

**Business logic**
- Who sets this value, when, and under what conditions?
- Are there multiple writers? Do they agree, or is there a conflict (⚠️)?
- Were any `MERGE` statements found? Document the `WHEN MATCHED` and `WHEN NOT MATCHED` branches as separate write conditions.
- Were any `INSTEAD OF` triggers found? These replace the write entirely — document what they actually do to this column as the true write logic.
- Were any `AFTER` triggers found? Document any side-effects on this column.
- Is the value derived from other columns, a trigger, an external input, or a default constraint?
- If it looks like an enum or status field: what are all observed values and what do they mean in the domain?
- What invariants hold? (always set, only set once, never null in practice even if nullable in schema, etc.)

**Downstream impact**
- What breaks if this column is null, wrong, or missing?
- Who depends on it — APIs, reports, views, other stored procedures?
- Are there views that expose this column under a different alias? Note it.

**Gaps**
- What could not be determined from the findings? List these honestly in the Open Questions section.

---

## Step 3 — Write the output file

Create: **`docs/columns/<TABLE>/$ARGUMENTS.md`**

Use exactly this structure. Do not add, remove, or rename sections.

```markdown
# `<table>.$ARGUMENTS`

> _One sentence in plain business English: what does this column store and why does it exist?_

---

## Column Definition

| Property            | Value |
|---------------------|-------|
| Data type           |       |
| Nullable            |       |
| Default             |       |
| Identity            |       |
| Computed expression |       |
| Constraints         |       |

**Extended property (MS_Description):**

> _Quote the sp_addextendedproperty @value verbatim, or "None found."_

**DDL:**
```sql
-- verbatim column line from CREATE TABLE or most recent ALTER TABLE
```

**Schema history** _(migrations that changed this column, oldest → newest):_

| Migration file | Change |
|----------------|--------|
|                |        |

---

## Business Logic

_Plain-English explanation of what this column means in the domain — not just its type, but its purpose and the rules that govern it._

_State any invariants explicitly:_
- _e.g. "Set once on record creation via IDENTITY, never manually written."_
- _e.g. "NULL means the approval step has not been reached yet; 0 means rejected; 1 means approved."_
- _e.g. "Always NVARCHAR because values may contain accented characters from European customer names."_

**BIT value meanings** _(only if data type is BIT):_

| Value | Business meaning |
|-------|-----------------|
| `0`   |                 |
| `1`   |                 |
| `NULL`|                 |

**Enum / fixed value set** _(omit if not applicable):_

| Value | Business meaning |
|-------|-----------------|
|       |                 |

---

## Data Sources

_Every place in the codebase that writes this column, ordered by importance._

| File : line | Operation | Trigger / condition | Value written |
|-------------|-----------|---------------------|---------------|
|             |           |                     |               |

⚠️ _Document any conflicts — multiple sources writing different values under overlapping conditions._

**MERGE statements** _(if any — list each branch separately):_

| File : line | Branch | Condition | Value written |
|-------------|--------|-----------|---------------|
|             | `WHEN MATCHED` | | |
|             | `WHEN NOT MATCHED` | | |

**Triggers** _(if any):_

| Trigger name | Type | Effect on this column |
|--------------|------|-----------------------|
|              | `INSTEAD OF` / `AFTER` | |

---

## Dependencies

_What must exist or be known before this column can be set._

| Depends on | Relationship | Notes |
|------------|-------------|-------|
|            |             |       |

---

## Downstream Consumers

_Who reads this column and for what purpose._

| File : line | Usage | Purpose |
|-------------|-------|---------|
|             |       |         |

**Views exposing this column** _(if any):_

| View name | Column alias (if different) |
|-----------|-----------------------------|
|           |                             |

---

## Related Documentation

_References to extended properties, T-SQL comments, dbt descriptions, ADRs, runbooks, or Markdown docs found during the scan. Quote the source and cite the file._

- `path/to/file:line` — _"exact text"_

_If nothing found: "No documentation found in repository."_

---

## Example Values

_Representative values from seed data, fixtures, or test factories only. No production data._

| Value | Source file |
|-------|-------------|
|       |             |

_If nothing found: "No example data found in repository."_

---

## Open Questions

_Gaps, ambiguities, or conflicts that could not be resolved from the codebase alone. A human needs to answer these before this doc can be considered complete._

-

---

_Generated by `/document-column` — scan findings:_ `docs/columns/<table>/$ARGUMENTS.findings.md`
_Written on:_ !`date +%Y-%m-%d`
```

---

## Hard rules

- Every claim must be traceable to the findings file. If the findings don't support it, write `Not found in codebase.`
- The extended property `MS_Description` is the authoritative definition. If application behaviour contradicts it, flag with ⚠️ and describe the discrepancy plainly.
- `INSTEAD OF` triggers override the stated write logic — if one exists, the trigger body is the real write logic, not the `INSERT`/`UPDATE` statement that invoked it.
- `MERGE` branches must be documented separately — they represent distinct conditional write paths.
- Conflicts between write sources get a ⚠️ prefix and a plain-English description of the discrepancy.
- The "Open Questions" section must be honest — surfacing a gap is better than papering over it.
- Do not modify any source file.
- Keep the output template structure unchanged — every column doc in this project follows the same format.
