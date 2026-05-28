---
description: "Scan subtask: find all references to a SQL Server 2016 column in the <TABLE> table and write a structured findings file"
subtask: true
---

# Column Scan Subtask

You are a code archaeologist. Your only job is to find evidence — no interpretation, no business logic, no prose.
Produce a structured findings file that the writing subtask will reason over.

## Input

Column to scan: **`$ARGUMENTS`**

## Hardcoded context

- **Table**: `<TABLE>` (do not derive this from project_index.md — it is fixed)

---

## Phase 0 — Read project context

@project_index.md

From this, note:
- Schema file locations and tooling (SSDT `.sqlproj`, Flyway, Liquibase, raw `.sql` with `GO` separators, ORM)
- Languages and file extensions in the application layer
- Documentation source: extended properties, dbt YAML, inline T-SQL comments, or none

---

## Phase 1 — Column definition

Find the canonical DDL for `$ARGUMENTS` in the schema files identified in Phase 0.

Extract verbatim:
- Full column declaration from `CREATE TABLE <TABLE>`
- Data type — note `BIT` (boolean), `NVARCHAR` vs `VARCHAR` (unicode), `DECIMAL`/`NUMERIC` precision, `DATETIME2` vs `DATETIME`, `ROWVERSION`, `UNIQUEIDENTIFIER`
- `NULL` / `NOT NULL`
- `DEFAULT` constraint (may be named: `CONSTRAINT DF_... DEFAULT ...`)
- `IDENTITY(seed, increment)` if present
- `AS <expression> [PERSISTED]` for computed columns
- `CHECK` constraint if present (inline or named)
- Any `CONSTRAINT` definitions in the `CREATE TABLE` block that reference this column

**Extended properties** — the SQL Server equivalent of MySQL `COMMENT`:

!`grep -rn "sp_addextendedproperty\|sp_updateextendedproperty\|extended_properties" --include="*.sql" . 2>/dev/null | grep -i "\b$ARGUMENTS\b" | head -30`

Read every match in full — the `@value` parameter is the human-written description for this column.

**Migration history** — every T-SQL statement that ever altered this column:

!`grep -rn "\b$ARGUMENTS\b" --include="*.sql" . 2>/dev/null | grep -iE "(ALTER TABLE|ALTER COLUMN|ADD COLUMN|sp_rename|EXEC sp_rename)" | grep -v "^\s*--" | sort | head -40`

**Foreign key constraints** referencing this column:

!`grep -rn "FOREIGN KEY\|REFERENCES" --include="*.sql" . 2>/dev/null | grep -i "\b$ARGUMENTS\b" | head -20`

Also check for FK constraints defined separately from the `CREATE TABLE`:

!`grep -rn "ALTER TABLE.*ADD.*CONSTRAINT.*FOREIGN KEY" --include="*.sql" . 2>/dev/null | head -30`

Read each result to check if `$ARGUMENTS` is the constrained column.

---

## Phase 2 — Triggers and computed columns

SQL Server `INSTEAD OF` triggers are particularly important — they silently intercept writes:

!`grep -rn "CREATE TRIGGER\|ALTER TRIGGER\|INSTEAD OF\|AFTER INSERT\|AFTER UPDATE\|AFTER DELETE\|FOR INSERT\|FOR UPDATE" --include="*.sql" . 2>/dev/null | head -40`

For each trigger file or block found, read the full body and check whether it reads or writes `$ARGUMENTS`.

Also check for computed column expressions referencing `$ARGUMENTS` from other columns:

!`grep -rn "\bAS\b.*\b$ARGUMENTS\b\|PERSISTED" --include="*.sql" . 2>/dev/null | grep -v "^\s*--" | head -20`

---

## Phase 3 — All writes in T-SQL and stored procedures

Stored procedures and functions are first-class write sources in SQL Server repos — treat them with equal weight to application code.

**T-SQL writes:**

!`grep -rn "\b$ARGUMENTS\b" --include="*.sql" . 2>/dev/null | grep -iE "(INSERT|UPDATE|SET|MERGE|OUTPUT)" | grep -v "^\s*--" | head -80`

Pay special attention to `MERGE` statements — SQL Server's `MERGE ... WHEN MATCHED THEN UPDATE` and `WHEN NOT MATCHED THEN INSERT` are a common source of hidden conditional writes. For every `MERGE` hit, read the full statement.

**Application code** — use the extensions from `project_index.md`:

!`grep -rn "\b$ARGUMENTS\b" --include="*.cs" --include="*.vb" --include="*.php" --include="*.py" --include="*.ts" --include="*.js" --include="*.java" . 2>/dev/null | grep -iEv "^\s*(//|#|/\*|\*)" | grep -iE "(insert|update|set|save|merge|upsert|persist|create)" | head -80`

For each file that appears, open and read the surrounding method or class to understand:
- What triggers the write (user action, scheduled job, event, ETL step, API call)
- Whether the value is hardcoded, computed, passed in from outside, or set by an ORM
- Any conditional logic that changes the value written
- ORM lifecycle hooks or interceptors that silently set this column

---

## Phase 4 — All reads and downstream consumers

**T-SQL reads:**

!`grep -rn "\b$ARGUMENTS\b" --include="*.sql" . 2>/dev/null | grep -iE "(SELECT|WHERE|GROUP BY|ORDER BY|HAVING|JOIN|CASE WHEN)" | grep -v "^\s*--" | head -60`

**Application reads:**

!`grep -rn "\b$ARGUMENTS\b" --include="*.cs" --include="*.vb" --include="*.php" --include="*.py" --include="*.ts" --include="*.js" --include="*.java" . 2>/dev/null | grep -iEv "^\s*(//|#|/\*|\*)" | grep -iEv "(insert|update|set|save|merge|upsert|persist|create)" | head -60`

Note: API response serializers, report queries, views (`CREATE VIEW`), and any `CASE WHEN $ARGUMENTS =` branching logic that drives business behaviour.

**Views referencing this column:**

!`grep -rn "CREATE VIEW\|ALTER VIEW" --include="*.sql" . 2>/dev/null | head -20`

For each view file found, check whether it selects or filters on `$ARGUMENTS`.

---

## Phase 5 — Existing documentation

**Extended properties** already captured in Phase 1 — re-use those results.

**Inline T-SQL comments adjacent to the column:**

!`grep -rn -A3 -B3 "\b$ARGUMENTS\b" --include="*.sql" . 2>/dev/null | grep -E "(--|/\*|\*/)" | head -30`

**Markdown / RST docs:**

!`grep -rn "\b$ARGUMENTS\b" --include="*.md" --include="*.mdx" --include="*.rst" . 2>/dev/null | grep -v "document-column" | head -30`

**dbt / catalog YAML:**

!`grep -rn "\b$ARGUMENTS\b" --include="*.yml" --include="*.yaml" . 2>/dev/null | head -30`

**Inline application code comments near writes:**

!`grep -rn -A2 -B2 "\b$ARGUMENTS\b" --include="*.cs" --include="*.vb" --include="*.php" --include="*.py" --include="*.ts" --include="*.js" . 2>/dev/null | grep -E "(//|/\*|\*|#)" | head -30`

---

## Phase 6 — Example values

Search only in test fixtures, seed scripts, and factory files — never production data dumps:

!`find . -type f \( -name "*seed*" -o -name "*fixture*" -o -name "*factory*" -o -name "*mock*" -o -name "*fake*" -o -name "*testdata*" \) \( -name "*.sql" -o -name "*.cs" -o -name "*.json" -o -name "*.xml" \) 2>/dev/null | head -20`

!`grep -rn "\b$ARGUMENTS\b" --include="*.sql" . 2>/dev/null | grep -iE "(seed|fixture|factory|test|sample|mock|fake|dummy)" | head -20`

From matching files only, extract representative values.

---

## Output

Write the findings to:

**`docs/columns/<TABLE>/$ARGUMENTS.findings.md`**

Create the directory if needed. Use this exact structure:

```markdown
# Findings: `<TABLE>.$ARGUMENTS`

_Raw scan output — uninterpreted. Do not edit manually._
_Scanned on:_ !`date +%Y-%m-%d`

---

## 1. Column DDL

```sql
-- verbatim column declaration from CREATE TABLE
```

| Property            | Value |
|---------------------|-------|
| Data type           |       |
| Nullable            |       |
| Default             |       |
| Identity            |       |
| Computed expression |       |
| Constraints         |       |

**Extended property (MS_Description):**

_Quote the full @value text from sp_addextendedproperty, or "None found."_

**Migration history** (oldest → newest):

| Migration file | Change |
|----------------|--------|
|                |        |

**Foreign keys:**

| Constraint name | References | On delete / update |
|-----------------|------------|--------------------|
|                 |            |                    |

---

## 2. Triggers & Computed Columns

_List any trigger (especially INSTEAD OF) or computed expression that reads or writes this column._
_Quote the relevant T-SQL block. If none: "None found."_

---

## 3. Write Locations

| File : line | Operation | Trigger / condition | Value written |
|-------------|-----------|---------------------|---------------|
|             |           |                     |               |

⚠️ _Flag any conflicts: multiple sources writing different values under overlapping conditions._
⚠️ _Flag any MERGE statements — list both the WHEN MATCHED and WHEN NOT MATCHED branches separately._

---

## 4. Read Locations

| File : line | Usage type | Context |
|-------------|------------|---------|
|             |            |         |

---

## 5. Existing Documentation Found

_Quote verbatim any extended property value, T-SQL comment, dbt description, or Markdown paragraph that describes this column._
_Cite source file and line for each._

- `path/to/file:line` — "exact text found"

_If nothing found: "No documentation found."_

---

## 6. Example Values

_From seed / fixture / factory / test files only._

| Source file | Value |
|-------------|-------|
|             |       |

_If nothing found: "No example data found."_

---

## 7. Open Questions

_Anything ambiguous, contradictory, or missing that the writing subtask should flag._

-
```

---

## Hard rules

- Record only what you actually found. Empty cells beat invented ones.
- Quote file paths and line numbers for every finding.
- Flag every `MERGE` statement with ⚠️ — they are the most common source of undocumented conditional writes in SQL Server repos.
- Flag `INSTEAD OF` triggers with ⚠️ — they replace the write entirely and are easy to miss.
- Do not write the final column doc — that is the writing subtask's job.
- Do not modify any source file.
