# Tutorial 1: Universal Database Connections, DSN Builders & Diagnostics

> **Package:** `odbc` (`v0.1.0`)  
> **Matching Example:** [`examples/universal_db_demo.vir`](../examples/universal_db_demo.vir)

---

## 1. Overview

The `odbc` package binds Vira directly to the industry-standard C99 ODBC 3.x API (`libodbc` via `unixODBC` on Linux/macOS and `odbc32` on Windows). With a single Vira API, your application can connect to:
- **PostgreSQL** (`psqlodbc`)
- **MySQL / MariaDB** (`MySQL ODBC 8.0 Unicode Driver` / `maodbc`)
- **Microsoft SQL Server & Azure SQL** (`ODBC Driver 17/18 for SQL Server`)
- **SQLite3** (`libsqlite3odbc`)
- **DuckDB** (`DuckDB Driver`)
- **Snowflake, Databricks, ClickHouse, BigQuery, Oracle, IBM Db2, SAP HANA**

---

## 2. Discovering Installed Host Drivers & DSNs

Before opening a database connection, you can inspect all ODBC drivers and pre-configured Data Source Names (DSNs) registered on the host machine:

```vira
module main

import odbc

fn main() {
	// 1. Enumerate all installed ODBC drivers (/etc/odbcinst.ini)
	drivers := odbc.list_drivers() or {
		panic("Failed to list ODBC drivers: ${err}")
	}
	println("Installed ODBC Drivers (${drivers.len}):")
	for mut i := 0; i < drivers.len; i += 1 {
		drv := drivers[i]
		println("  - ${drv.name} (${drv.attributes})")
	}

	// 2. Enumerate configured User & System DSNs (~/.odbc.ini & /etc/odbc.ini)
	dsns := odbc.list_data_sources() or {
		panic("Failed to list DSNs: ${err}")
	}
	for mut i := 0; i < dsns.len; i += 1 {
		dsn := dsns[i]
		println("  - DSN: ${dsn.name} -> Driver: ${dsn.driver}")
	}
}
```

---

## 3. Building Universal Connection Strings

Use the built-in DSN helper functions in [`src/types.vir`](../src/types.vir) to generate standard ODBC connection strings. Passing `""` as the `driver` argument automatically selects the standard driver name for that database engine:

```vira
// PostgreSQL (defaults to "PostgreSQL Unicode")
pg_dsn := odbc.postgres_dsn("", "127.0.0.1", 5432, "analytics", "postgres", "secret")

// MySQL / MariaDB (defaults to "MySQL ODBC 8.0 Unicode Driver")
mysql_dsn := odbc.mysql_dsn("", "127.0.0.1", 3306, "commerce", "root", "secret")

// Microsoft SQL Server (defaults to "ODBC Driver 17 for SQL Server")
mssql_dsn := odbc.sqlserver_dsn("", "127.0.0.1", 1433, "enterprise", "sa", "StrongPass!123")

// SQLite3 (defaults to "SQLite3")
sqlite_dsn := odbc.sqlite_dsn("", "/var/lib/vira/app.db")

// DuckDB (defaults to "DuckDB Driver")
duck_dsn := odbc.duckdb_dsn("", ":memory:")
```

---

## 4. Establishing & Verifying Connections

### Connecting with Default Timeout (10s)
```vira
mut conn := odbc.connect(pg_dsn) or {
	eprintln("Database connection failed: ${err}")
	return
}
defer conn.close()
```

### Connecting with Explicit Login & Connection Timeout
```vira
mut conn := odbc.connect_with_timeout(mssql_dsn, 5) or {
	eprintln("Connection timed out or failed: ${err}")
	return
}
defer conn.close()
```

### Connecting via Named DSN (`SQLConnect`)
If your system administrator has configured a named DSN in `/etc/odbc.ini`:
```vira
mut conn := odbc.connect_dsn("ProdWarehouseDSN", "db_user", "db_password")!
defer conn.close()
```

### Checking Connection Liveness (`ping`)
Use `conn.ping()` to execute a lightweight `SELECT 1` round-trip check:
```vira
if conn.ping() {
	println("Database connection is healthy and responsive!")
}
```

---

## 5. Native `SQLSTATE` Error Diagnostics (`SQLGetDiagRec`)

Every failable operation in `odbc` (`!Env`, `!Connection`, `!ResultSet`, `!int`) automatically queries `SQLGetDiagRec` on the underlying `SQLHENV`, `SQLHDBC`, or `SQLHSTMT` handle and returns a structured error message containing:
1. The 5-character ISO/ANSI **`SQLSTATE`** code (e.g. `08001` connection failure, `28000` invalid authorization, `42S02` table not found, `23505` unique constraint violation).
2. The database vendor's **Native Error Code**.
3. The driver's full diagnostic message.
