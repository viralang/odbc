# `odbc` — Universal ODBC 3.x Database Driver for Vira

A standalone, zero-allocation-leak **Universal ODBC 3.x Database Driver** for the [Vira Programming Language](https://github.com/viralang/vira), built directly on Vira's native C99 FFI (`extern "C"`, `#flag -lodbc`, and deterministic `autofree`).

With a single package, Vira applications can connect to **any relational, analytical, or cloud data warehouse** that provides a standard ODBC driver:

- **PostgreSQL** (`psqlodbc` / `PostgreSQL Unicode`)
- **MySQL / MariaDB** (`MySQL ODBC 8.0 Unicode Driver` / `maodbc`)
- **Microsoft SQL Server & Azure SQL** (`ODBC Driver 17/18 for SQL Server`)
- **SQLite3** (`libsqlite3odbc`)
- **DuckDB** (`DuckDB Driver`)
- **Snowflake, Databricks, ClickHouse, BigQuery, Oracle, IBM Db2, SAP HANA**

---

## Tutorials & Guides

- **[Tutorial 1: Universal Database Connections, DSN Builders & Diagnostics](tutorials/01_connections_and_dsn_discovery.md)**
- **[Tutorial 2: OS Driver Manager Pooling & Managed `ConnectionPool`](tutorials/02_connection_pooling.md)**
- **[Tutorial 3: Full Database Operations (CRUD, Batch Transactions & Prepared Statements)](tutorials/03_crud_transactions_and_prepared_statements.md)**
- **[Tutorial 4: Retrieving Results, Export Formats (ASCII/CSV/JSON) & SQL Pagination](tutorials/04_retrieving_results_and_pagination.md)**

---

## 1. Host Prerequisites

### Linux (Ubuntu / Debian)
```bash
sudo apt-get update
sudo apt-get install -y unixodbc unixodbc-dev

# Optional: Install specific database ODBC drivers
sudo apt-get install -y odbc-postgresql libsqliteodbc
```

### macOS (Homebrew)
```bash
brew install unixodbc
brew install psqlodbc sqliteodbc
```

### Windows (Native & Cross-Compilation from Linux/macOS)
Windows includes the native Microsoft ODBC Driver Manager (`odbc32.dll` / `libodbc32.a`) out of the box. When compiling or cross-compiling for Windows (`-target windows`), Vira automatically translates `#flag -lodbc` to `-lodbc32`.

To cross-compile Windows `.exe` binaries from Linux or macOS:
```bash
# Ubuntu / Debian
sudo apt-get install -y gcc-mingw-w64-x86-64

# macOS
brew install mingw-w64
```

---

## 2. Installing in Any Vira Project

Because `odbc` is a standalone external Vira package repository, you can install it directly from GitHub without bundling it into the compiler binary:

```bash
vira add github.com/viralang/odbc
```

Or add it to your project's `vira.mod`:

```v
module my_app
vira 0.7.1

require (
	github.com/viralang/odbc v0.1.0
)
```

---

## 3. Feature Summary & API Reference

| Category | Symbols | Description |
| :--- | :--- | :--- |
| **Discovery & DSNs** | `list_drivers()`, `list_data_sources()`, `postgres_dsn`, `mysql_dsn`, `sqlserver_dsn`, `sqlite_dsn`, `duckdb_dsn` | Enumerates OS-registered ODBC drivers/DSNs and builds connection strings. |
| **Connections** | `connect()`, `connect_with_timeout()`, `connect_dsn()`, `conn.ping()`, `conn.close()` | Establishes connections (`SQLDriverConnect` / `SQLConnect`), checks liveness (`SELECT 1`), and frees handles. |
| **Connection Pooling** | `enable_driver_pooling()`, `ConnectionPool.new()`, `pool.warmup()`, `pool.acquire()`, `pool.release()`, `pool.stats()`, `pool.close_all()` | OS-level Driver Manager pooling (`SQL_ATTR_CONNECTION_POOLING`) and managed `ConnectionPool` with health checks and telemetry. |
| **CRUD & Batching** | `conn.exec()`, `conn.insert_row()`, `conn.update_rows()`, `conn.delete_rows()`, `conn.exec_batch_tx()`, `escape_literal()`, `quote_ident()`, `build_insert_sql()`, `build_upsert_postgres_sql()`, `build_update_sql()`, `build_delete_sql()` | Full SQL CRUD builders, safe literal escaping, ACID transactions (`begin_tx`, `commit`, `rollback`), and atomic batch execution. |
| **Result Retrieval & Export** | `conn.query()`, `conn.query_row()`, `conn.query_scalar()`, `conn.query_int()`, `rs.to_ascii_table()`, `rs.to_csv()`, `rs.to_json()`, `rs.filter_eq()`, `rs.column_values()` | Materializes `ResultSet` & `Row` values with typed getters (`get_int`, `get_f64`, `get_bool`, `is_null`) and exports to ASCII tables, CSV, and JSON. |
| **Pagination** | `new_page_request()`, `conn.query_page()`, `rs.paginate()`, `build_paginated_sql()`, `build_keyset_sql()` | Dialect-aware `LIMIT/OFFSET`, SQL Server/Oracle `OFFSET/FETCH NEXT`, cursor/keyset pagination, and in-memory `ResultSet` pagination. |

---

## 4. Verification, Cross-Compilation & Bundling

### Run Unit Tests & Live Demo
Run the unit test suite and full operations demo with Vira's zero-leak verifier:

```bash
vira test
vira run examples/universal_db_demo.vir
```

### Building & Bundling (CLI vs. Native GUI Applications)

You can compile and package ODBC-powered applications for Linux, macOS, and Windows:

```bash
# 1. Native Console / CLI build (Linux / macOS / Windows)
vira build -prod -o db_demo examples/universal_db_demo.vir

# 2. Cross-compile a Windows Console (.exe) binary from Linux/macOS
vira build -prod -target windows -o db_demo.exe examples/universal_db_demo.vir

# 3. Bundle a portable distribution archive (auto-detects Console CLI vs Native GUI)
vira bundle -target windows -name "UniversalDBDemo" examples/universal_db_demo.vir
```

> **Note on GUI vs. CLI Bundling (`-gui` vs `-cli`):**  
> - **Console / CLI tools** (like `examples/universal_db_demo.vir`) do not call `gui_init(...)`. `vira bundle` automatically detects this and builds them with the **Console Subsystem** (`IMAGE_SUBSYSTEM_WINDOWS_CUI` on Windows + `run.bat`, `.command` launcher on macOS, `Terminal=true` on Linux) so terminal output remains visible and the app never runs invisibly in the background.
> - **Native GUI Database Dashboards** (apps that call `gui_init(...)` or `widget.App.init()`) are automatically compiled with the **GUI Subsystem** (`-mwindows` on Windows, `.app` bundle on macOS, `Terminal=false` on Linux) so no black `cmd.exe` console window appears behind your GUI window. You can also explicitly force either mode via `vira bundle -gui` or `vira bundle -cli`.
