# fgl-northwind

Genero BDL program to build and populate the **Northwind** sample database.

A single loader, [db_create_northwind.4gl](db_create_northwind.4gl), creates the
schema, loads the seed data, and sets up the auto-increment sequences. It is
portable across the Genero-supported backends and isolates the per-database
differences behind branching keyed off `fgl_db_driver_type()`.

## Supported databases

PostgreSQL (`pgs`), Informix (`ifx`), Oracle (`ora`), SQL Server (`snc`),
MariaDB (`mdb`), SQLite (`sqt`).

Portability is handled in the program:

- **Auto-increment** — on the sequence-capable engines, plain `INTEGER`
  primary keys plus one portable `CREATE SEQUENCE … START WITH <max+1>` per
  table; the application obtains new keys via the driver's `nextval`
  expression. SQLite has no sequences, so its keys use
  `INTEGER PRIMARY KEY AUTOINCREMENT` instead.
- **Binary columns** (`picture`, `photo`) — mapped per driver: `BYTEA` (pgs),
  `BYTE` (ifx), `BLOB` (ora), `VARBINARY(MAX)` (snc), `LONGBLOB` (mdb),
  `BLOB` (sqt).
- **Long text** — columns wider than 255 use `LVARCHAR` on Informix and
  `VARCHAR` elsewhere.
- **Money** — `DECIMAL(p,s)` (portable) rather than `SMALLFLOAT`/`FLOAT`.

## Prerequisites

- Genero BDL (`fglcomp`, `fglrun`).
- A reachable database of one of the supported types, with an `FGLPROFILE`
  entry for the connection name **`northwind`** (driver + source + credentials).
  The account needs rights to create tables and sequences.

## Build and run

```bash
fglcomp -M -Wall db_create_northwind.4gl     # produces db_create_northwind.42m
export FGLPROFILE=/path/to/your/fglprofile   # defines dbi.database.northwind.*
fglrun db_create_northwind.42m
```

The program connects (`DATABASE northwind`), drops any existing objects,
recreates the tables, inserts the seed data, and creates the sequences.

## Oracle notes

Oracle needs three environment settings the other backends do not (UTF-8 /
length-semantics alignment), set **before** running:

```bash
export FGL_LENGTH_SEMANTICS=CHAR              # VARCHAR(n) = n characters, not bytes
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8     # OCI client charset matches the UTF-8 locale
export NLS_DATE_FORMAT='YYYY-MM-DD'           # accept the ISO date literals in the seed data
```

The `FGLPROFILE` entry should also enable the Informix→Oracle type translation:

```ini
dbi.database.northwind.ifxemul.datatype.char    = true
dbi.database.northwind.ifxemul.datatype.varchar = true
```

Without these you will hit `ORA-12899` (multibyte data overflowing byte-sized
columns) and `ORA-01861` (date literal format).

## SQLite notes

The `FGLPROFILE` `source` is the database file path, e.g.:

```ini
dbi.database.northwind.source = "/path/to/northwind.db"
dbi.database.northwind.driver = "dbmsqt"
```

The `dbmsqt` driver does **not** create the file, so the loader creates an
empty one (a valid empty SQLite database) before connecting if it is missing.
SQLite is always UTF-8 and uses character length semantics; run with
`FGL_LENGTH_SEMANTICS=CHAR` to match.
