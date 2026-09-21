---
name: sql-requirement-ddl
description: Convert business requirements into review-ready database DDL for creating or altering tables, columns, indexes, constraints, and comments. Use when the user asks to design a schema or generate requirement-driven DDL; do not use for ordinary SELECT queries or data-only UPDATE/DELETE statements.
---

# SQL Requirement DDL

Create DDL that fits the real database and project conventions while keeping execution authority separate from SQL generation.

## Rule priority

Apply rules in this order: the user's current explicit requirement, the custom rules in this skill, then compatible verified project conventions.

For every new `CREATE TABLE`, omitting the storage engine, character set, and collation is a mandatory custom rule. Do not add `ENGINE`, `DEFAULT CHARSET`, `CHARACTER SET`, or `COLLATE` merely to satisfy an external checklist.

## Establish the target

- Identify the database engine and version when syntax or online-DDL behavior depends on them. Use the confirmed project database when context already establishes it; otherwise state the assumption or ask only when the choice materially changes the result.
- Extract the business entities, field meanings, required/optional status, defaults, uniqueness, relationships, lifecycle, and expected query patterns from the requirement.
- Do not invent business enums, default values, foreign-key behavior, or naming conventions. Mark unresolved decisions explicitly.

## Inspect before designing

For changes to an existing database, inspect the current structure with read-only tools when available:

- Check the exact database, table, columns, types, defaults, comments, indexes, constraints, charset, collation, and engine.
- Inspect related tables for established naming, primary-key, timestamp, soft-delete, and index conventions.
- Preview data conditions that can make the migration fail, such as NULL values, duplicates, incompatible values, overlength text, or orphaned relations.
- Treat remembered or supplied schemas as potentially stale. If live inspection is unavailable, label the DDL as unverified against the current schema.
- Reuse one database connection during the task and disconnect it after the final database check.

## Design rules

- Prefer the smallest schema change that satisfies the stated requirement.
- Match existing conventions rather than introducing a parallel convention.
- Choose the narrowest correct data type without sacrificing known valid values; specify precision, length, signedness, nullability, and defaults deliberately.
- Add column and table comments that express business meaning, not implementation history.
- Create indexes from actual lookup, join, sorting, and uniqueness requirements. Avoid redundant indexes and remember that a composite index is ordered.
- Use foreign keys only when the existing system uses them and their delete/update behavior is confirmed.
- For `NOT NULL`, unique constraints, type narrowing, or new required columns on populated tables, separate data cleanup/backfill from constraint enforcement when needed.
- Call out table-locking, rebuild, long-running, replication, or compatibility risk for large or production tables. Do not claim an operation is online without support from the confirmed engine/version.
- Never include destructive drops, truncation, or irreversible conversion unless explicitly requested. When requested, provide a precheck and a recoverable migration path where practical.

## Required conventions for new tables

Apply these rules whenever generating `CREATE TABLE` DDL:

- Do not specify a storage engine. Omit clauses such as `ENGINE=InnoDB` and let the database or project default apply.
- Do not specify a character set or collation. Omit table-level and column-level `CHARSET`, `CHARACTER SET`, and `COLLATE` clauses unless the user explicitly overrides this rule for a confirmed compatibility need.
- Prefer `NOT NULL` for columns and choose an explicit default from confirmed business semantics. Do not use an arbitrary placeholder merely to satisfy the default requirement.
- Every `NOT NULL` column must declare an explicit `DEFAULT`. When the database syntax does not permit a default, such as an auto-increment or generated column, keep the required system syntax and call out the exception rather than inventing an invalid default.
- Use `TINYINT` for status-like columns when its range is sufficient; use `SMALLINT` when more status values are required. Do not use `INT`, `VARCHAR`, or `ENUM` for status-like columns. Document the confirmed value-to-meaning mapping in the column comment, but do not invent unspecified status values.
- Prefer `VARCHAR` for textual columns and choose a length that matches the known business limit. Keep the declared `VARCHAR` length at or below `2700` whenever practical. Use `TEXT` only when the content must exceed a reasonable `VARCHAR` limit or is genuinely unbounded, and explain why it is needed.
- Keep column names short, clear, and directly tied to their business meaning. Avoid redundant prefixes, vague abbreviations, and implementation-specific wording.
- Check names against the reserved words of the target database/version. Do not use a database keyword as a column name and do not rely on identifier quoting to make a keyword acceptable.

### Field design

- Core tables must include `create_time` and `update_time`. Match the project's established time type and automatic-update convention instead of inventing a new one.
- Prefer vertically splitting large `BLOB` or `TEXT` content into a secondary table so common queries do not load it. Use `ENUM`, `SET`, `BLOB`, and `TEXT` only after confirming that simpler scalar types or a separate table are unsuitable.
- Fields used by high-frequency joins may be deliberately duplicated to reduce joins only when the read benefit and ownership are clear. Document the source of truth and the application-level synchronization strategy to prevent stale copies.
- Store IPv4 addresses as an integer, normally `INT UNSIGNED`, when only IPv4 is required. If IPv6 may be stored, do not force it into `INT`; confirm an IPv6-capable representation such as `VARBINARY(16)`.
- Store monetary amounts as integers in the smallest required currency unit rather than floating-point types. Choose `INT` or `BIGINT` from the confirmed amount range, and state the currency unit in the column comment.

### Primary keys and indexes

- For InnoDB tables, use a single auto-increment primary-key column named `id` with type `INT UNSIGNED` or `BIGINT UNSIGNED`. Prefer `UNSIGNED` for auto-increment integer columns because generated identifiers are nonnegative; choose `BIGINT` only when the expected row count or identifier lifecycle requires it.
- Do not use a business-entity identifier such as `user_id` or `order_id` directly as the primary key. Keep `id` as the surrogate primary key and add a `uniq_` unique index to the business identifier only when business uniqueness is confirmed.
- For MySQL, declare the primary key directly as `PRIMARY KEY (...)` and do not emit a named `CONSTRAINT pk_...` clause because the actual primary-key index name is always `PRIMARY`. Apply a `pk_` constraint name only for a different target database that genuinely preserves named primary-key constraints and only when the user requests it.
- Name unique indexes or unique constraints with the `uniq_` prefix. Do not use `uk_` or `uq_`.
- Name ordinary indexes with the `idx_` prefix. Keep index names concise and descriptive of their columns.
- Use `BTREE` indexes for InnoDB and MyISAM tables, emitting `USING BTREE` where supported by the target syntax.
- Keep the number of indexes on a table at seven or fewer whenever practical. Count the proposed indexes before finalizing the DDL; if more than seven are justified, call out the count and reason.
- For a composite index, prefer the column with higher selectivity as the leading column while also respecting the actual equality, range, sorting, and leftmost-prefix query patterns.
- Ensure the join column on the driven or lookup side of a join has an applicable index; prefer extending a useful composite index over adding a redundant single-column index.
- Avoid redundant indexes. For example, when an index on `(a, b)` already satisfies the relevant leftmost-prefix queries, do not also create an index on `(a)` unless a verified query or covering-index need justifies it.

## Post-generation review

After producing the final DDL draft and before saving it, compare the DDL against the custom rules captured in this skill. Do not fetch or reread an external standards page unless the user explicitly asks for a fresh external comparison.

- Review every generated table and altered column or index, not just the fields mentioned explicitly in the requirement.
- Check at least primary-key design, nullability and defaults, time fields, status types, text and large-object types, IP and monetary storage, index naming/type/count/order, join-column coverage, and redundant indexes when applicable.
- Correct a non-compliant draft before saving when the correction does not change an unresolved business decision.
- The explicit rules in this skill and the user's current request take precedence over other conventions. In particular, continue to omit table engine, character set, and collation clauses unless the user explicitly changes those custom rules.
- Return a concise review result with three categories: passed checks, corrected issues, and remaining deviations. Do not claim full compliance when a required business decision or live-schema check is unresolved.
- Add a short SQL comment header to the saved file recording the review date and whether the local DDL review passed. List only remaining deviations so the SQL stays readable.

## Deliverable

Return executable SQL in the order it should be reviewed or run:

1. Read-only prechecks that expose conflicts or affected rows, when applicable.
2. The DDL, split into clear statements when independent execution is safer.
3. Post-change verification SQL.
4. Rollback SQL when it is technically meaningful; explain when rollback would lose data or requires a backup.

## SQL file output

- Save each completed requirement DDL as a `.sql` file under `/Users/a0000/tools/SQL/`.
- Name the file with the requirement name followed by the creation date: `<需求名称>_YYYYMMDD.sql`, for example `新增商品扩展表_20260918.sql`.
- Derive a short, recognizable requirement name from the user's wording. Remove path separators and other characters that are unsafe in filenames without changing the business meaning.
- Check whether the target filename already exists before writing. Do not overwrite an existing SQL file unless the user explicitly authorizes it; if there is a collision, report the exact path and ask how to proceed.
- The saved file must contain the final reviewed SQL, including applicable prechecks, DDL, verification, and rollback notes or statements. Do not save discarded drafts.
- Return the absolute saved file path to the user after writing it.

Keep the answer concise when the user asks for only a statement, but still surface any assumption that can change correctness. Preserve exact requested table, field, and business-key names.

Generating DDL does not authorize executing it. Execute schema changes only after the user explicitly asks, and re-confirm the resolved database and affected objects immediately beforehand.
