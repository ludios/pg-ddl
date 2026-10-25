<!-- Model-output: Claude Mythos 5.1 -->
<!-- Model-output: ChatGPT 6 Astra -->

# House PostgreSQL DDL conventions

## Shape of a table

```sql
-- The scions with quantities included in an order. A refunded item may be deleted.
CREATE TABLE order_items_scions (
	order_id               int      NOT NULL REFERENCES orders (id),
	accession_id           int      NOT NULL REFERENCES accessions (id),
	-- Quantity requested when creating the order; a request for 0 has no row
	requested_quantity     int      NOT NULL CHECK (requested_quantity >= 1 AND requested_quantity < 1000000),
	-- Quantity actually added to the order
	quantity               int      NOT NULL CHECK (quantity >= 0 AND quantity <= requested_quantity),
	-- Price for one scion at the time of order
	unit_price_usd         decimal  NOT NULL CHECK (unit_price_usd >= 0),
	-- Whether this item went back to inventory, e.g. because the order was refunded
	returned_to_inventory  boolean  NOT NULL DEFAULT FALSE,

	PRIMARY KEY (order_id, accession_id)
);
SELECT periods.add_system_time_period('order_items_scions', 'row_start', 'row_end');
SELECT periods.add_system_versioning('order_items_scions');

CREATE TRIGGER order_items_scions_check_update
	BEFORE UPDATE ON order_items_scions
	FOR EACH ROW
	WHEN (
		OLD.order_id           != NEW.order_id           OR
		OLD.accession_id       != NEW.accession_id       OR
		OLD.requested_quantity != NEW.requested_quantity OR
		OLD.unit_price_usd     != NEW.unit_price_usd
	)
	EXECUTE FUNCTION raise_exception('cannot change order_id, accession_id, requested_quantity, or unit_price_usd');

CREATE TRIGGER order_items_scions_forbid_truncate
	BEFORE TRUNCATE ON order_items_scions
	EXECUTE FUNCTION raise_exception('truncate is forbidden');
```

Align name, type, and constraints in columns. Uppercase keywords, `TRUE`, `FALSE`, `NULL`. A blank line between groups of related columns. Table-level CHECKs, then PRIMARY KEY, last inside the parentheses. After the table: periods, indexes, triggers, `CLUSTER ON`, views. Three blank lines between tables. Plural table names. Quote reserved words rather than renaming (`"user"`, `"full"`).

## Files and workflow

- One schema per project, named for the project, never `public`. `schema.sql`: `\set ON_ERROR_STOP on`, `CREATE SCHEMA`, `SET search_path`, `raise_exception()` (a trigger function which raises `TG_ARGV[0]`), then `\ir` each file in dependency order. A superuser applies `extensions.sql` first; extensions install into `public`.
- One file per concept: the table, its companions, triggers, views, immvs. Views joining many files load last from their own file.
- Fixtures which are part of the model live in the schema file: property profiles, the root directory row, version lists, `\copy` of reference data. Known configuration goes in a fixture row, not a DEFAULT. A DEFAULT is the justified ordinary case, never an observation nobody made.
- No migration files. The schema files describe a fresh install and are the truth; production is ALTERed by hand to match, then the CREATE TABLE is edited. Write statements which paste into production psql unchanged: `CREATE INDEX CONCURRENTLY` on big tables, `CREATE OR REPLACE` for views and functions.
- Tests load the schema into a fresh pg_tmp database, assert the exact trigger messages, and assert mutable columns do change.
- Schema-qualify tables inside function bodies and immv definitions; the caller's search_path is not guaranteed. REFERENCES may cross project schemas within one database.

## Keys

- An external identifier can be the key as it is: a Drive file id, a service account email. A 1:1 companion reuses the parent's key: `accession_id int PRIMARY KEY REFERENCES accessions (id)`.
- `GENERATED ALWAYS AS IDENTITY` so the application cannot pick ids; `BY DEFAULT` only when fixtures or imports supply them. Never `serial`. Ids are local to their table, not slices of a shared space. ALWAYS still permits `SET id = DEFAULT`; the update trigger makes an id immutable.
- Type and range are separate decisions. Every identity column: `CHECK (id >= 1 AND id < N)` under `-- Limit of N can be raised if needed`. N documents the scale and picks smallint, int, or bigint.
- Composite keys get composite FKs: `FOREIGN KEY (location_id, position) REFERENCES positions (location_id, name)`.
- Point a companion's FK at the companion it depends on, not the base entity.
- For every FK, index the referencing column or forbid DELETE on the referenced table; otherwise each parent delete scans the child.
- `ON DELETE CASCADE` only where the child is meaningless alone and deleting the parent is a real operation. Deletion policy is separate from immutability: refunded orders may be deleted; compression definitions never.
- Uniqueness by a UNIQUE constraint or UNIQUE expression index such as `(LOWER(username))`, not a generated column.
- Mutually referencing tables use `DEFERRABLE INITIALLY DEFERRED`; the root row references itself.

## Columns

- `text`, never `varchar(n)`; shape it with a CHECK (regex anchored with `\A` and `\Z`; `octet_length` when the limit is bytes) or a DOMAIN when the concept recurs. A DOMAIN may carry a COLLATE; drop one which fights the driver. Digits which aren't numbers (ZIP codes) are text.
- Small closed sets: `text CHECK (x IN (...))`. When rows are many, `smallint` codes with a `-- 1 = ..., 2 = ...` comment and sentinels outside the source's plausible range. ENUM only where psql readability outweighs compactness.
- `timestamptz`; `timestamp` only for zoneless wall-clock source data; `date` for dates.
- `decimal` for money and measurements, unit in the name: `_usd`, `_grams`, `_ounces`, `_inches`.
- `boolean NOT NULL`; nullable only when unknown is a real state, such as a fact observed at capture time.
- `bytea` with an exact `octet_length` CHECK for hashes and secrets. `uuid` as a 128-bit field and `int` as a 32-bit pattern each save a byte over bytea; forgo the byte when it complicates the application.
- `jsonb` for payloads whose schema someone else owns, stored as received.
- Arrays only when short and rarely queried; on big tables prefer newline-terminated text or an index inside the jsonb.
- In a key, `''` means none. A count the source didn't give stays NULL, not 0.
- A CHECK on every numeric column: sign, range, sanity bound. Cross-column: `num_nonnulls(a, b) = 1`, `(a IS NULL) = (b IS NULL)`, `quantity <= requested_quantity`.
- Columns which are all NULL or all NOT NULL together: two or three stay with a named both-or-neither CHECK; a larger group, or one which is secret, big, or changes on another schedule, becomes a 1:1 table.
- Naming: `_time` for instants, `_date` for dates, `_id` for references, `_provenance` for source codes, `_from_<source>` for the same fact from another source, `had_` and `_fixed` for applied cleanup, `<verb>ed_time` for state transitions (`paid_time`, `shipped_time`) instead of a status column.

## Comments

A docstring above each column unless the name says everything: what the value is, where it comes from, what NULL means, oddities with example ids. Above each table: what a row is and when it may be deleted. List what you deliberately don't store and why. Don't restate a constraint, narrate the design, or explain a view's purpose. Length is fine when the comment answers a real adjacent alternative.

## Mutability

Every table gets the two triggers above. Immutable: the key, the birth facts, anything whose change would be a different fact. A fact table drops the WHEN and says `'cannot change row'`. `IS DISTINCT FROM` for nullable columns. `REVOKE TRUNCATE` doesn't bind the superuser; a trigger does. `forbid_delete` where a delete would scan an unindexed FK.

Invariants SQL can't state go in triggers: one directory-entry change per transaction via `set_config(..., true)`, an array-element FK with `FOR KEY SHARE`, a derived table refreshed from its source. Leave an escape hatch as a custom setting for trusted bulk operations.

## History

The two `periods` calls above go right after CREATE TABLE on tables people edit, once the shape has settled. Skip them where write volume or size makes history too costly (a hot storage table lost them and gained an explicit `put_time`) and on huge tables of immutable observations. Versioning replaces `updated_at` and audit tables; query the past with `t__as_of(ts)`. Compare OLD and NEW before updating through a view; a no-op UPDATE still writes a history row. Decide per view whether `row_start` and `row_end` belong in it.

## Derived data

- `t_view` LEFT JOINs the table to everything a reader wants in one round trip, joined columns prefixed by role (`plant_`, `accession_`). An INSTEAD OF UPDATE trigger lets an admin UI edit through it.
- `pgivm.create_immv` for aggregates which must be current and fast. When ivm can't (window functions), a trigger-maintained table, commented as updated only by that trigger.
- Reporting views for the business owner and integrity views for the operator live in the schema. Ad-hoc investigations are saved as `.sql` files beside it.

## Indexes and storage

- An index exists for a known query; say which when not obvious. Partial indexes for rare flags (`WHERE in_hold_window = TRUE`). Expression indexes for `LOWER()`, jsonb paths, tsvector. Drop indexes which stop earning their cost.
- `ALTER TABLE t CLUSTER ON t_pkey` declares the physical order for a future CLUSTER.
- Big tables only: order columns 8-byte, 4, 2, 1, then variable-length, and say so; `COMPRESSION lz4`; `SET STORAGE MAIN` for compressible text kept inline, `EXTERNAL` for already-compressed bytea; large text in a 1:1 table.
