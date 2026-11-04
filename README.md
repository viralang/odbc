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

---

## 2. Installing in Any Vira Project

Because `odbc` is a standalone external Vira package repository, you can install it directly from GitHub without bundling it into the compiler binary:

```bash
vira add github.com/viralang/odbc
```

Or add it to your project's `vira.mod`:

```vira
module my_app
vira 0.7.1

require (
	github.com/viralang/odbc v0.1.0
)
```

---

## 3. Quick Start

### Discover Installed Drivers & DSNs
```vira
module main

import odbc

fn main() {
	drivers := odbc.list_drivers() or {
		panic("failed to list ODBC drivers: ${err}")
	}
	for mut i := 0; i < drivers.len; i += 1 {
		drv := drivers[i]
		println("Driver: ${drv.name} (${drv.attributes})")
	}
}
```

### Connect, Execute DDL/DML, and Query Rows
```vira
module main

import odbc

fn main() {
	// Build a connection string for PostgreSQL, MySQL, SQL Server, SQLite3, or DuckDB:
	dsn := odbc.postgres_dsn("", "127.0.0.1", 5432, "analytics", "postgres", "secret")

	mut db := odbc.connect(dsn) or {
		println("Connection error: ${err}")
		return
	}
	defer db.close()

	// Execute DDL / DML
	_ := db.exec("CREATE TABLE IF NOT EXISTS metrics (id INT, service VARCHAR(64), latency_ms FLOAT)") or {
		panic(err)
	}
	_ := db.exec("INSERT INTO metrics VALUES (1, 'vira-gateway', 1.42)") or {
		panic(err)
	}

	// Execute SELECT query and iterate typed rows
	rs := db.query("SELECT id, service, latency_ms FROM metrics") or {
		panic(err)
	}

	for mut i := 0; i < rs.row_count(); i += 1 {
		row := rs.rows[i]
		id := row.get_int("id")
		svc := row.get("service")
		lat := row.get_f64("latency_ms")
		println("[${id}] service=${svc} latency=${lat}ms")
	}
}
```

### ACID Transactions (`begin_tx`, `commit`, `rollback`)
```vira
mut db := odbc.connect(dsn)!
defer db.close()

db.begin_tx()!
_ := db.exec("UPDATE accounts SET balance = balance - 100 WHERE id = 1") or {
	db.rollback()!
	return
}
_ := db.exec("UPDATE accounts SET balance = balance + 100 WHERE id = 2") or {
	db.rollback()!
	return
}
db.commit()!
```

### Prepared Statements (`prepare`)
```vira
mut stmt := db.prepare("SELECT id, username FROM users WHERE is_active = 1")!
defer stmt.close()

rs := stmt.query()!
println("Fetched ${rs.row_count()} active users")
```

---

## 4. API Overview

| Symbol | Description |
| :--- | :--- |
| `odbc.Env.new() !Env` | Allocates an `SQL_OV_ODBC3` environment handle (`SQLHENV`). |
| `odbc.list_drivers() ![]DriverInfo` | Enumerates all installed ODBC drivers on the host via `SQLDrivers`. |
| `odbc.list_data_sources() ![]DataSourceInfo` | Enumerates all User & System DSNs via `SQLDataSources`. |
| `odbc.connect(conn_str string) !Connection` | Connects to any database via `SQLDriverConnect` (`SQL_DRIVER_NOPROMPT`). |
| `odbc.connect_with_timeout(conn_str string, sec int) !Connection` | Connects with explicit `SQL_ATTR_LOGIN_TIMEOUT` and `SQL_ATTR_CONNECTION_TIMEOUT`. |
| `odbc.connect_dsn(dsn string, user string, pwd string) !Connection` | Connects to a named DSN via `SQLConnect`. |
| `conn.exec(sql string) !int` | Executes DDL/DML via `SQLExecDirect` and returns affected row count (`SQLRowCount`). |
| `conn.query(sql string) !ResultSet` | Executes a `SELECT` query and materializes column metadata (`SQLDescribeCol`) and rows (`SQLFetch`/`SQLGetData`). |
| `conn.query_scalar(sql string) !string` | Returns the first column of the first row as a `string`. |
| `conn.prepare(sql string) !Statement` | Compiles a reusable statement on the server via `SQLPrepare`. |
| `conn.begin_tx() !bool` / `commit() !bool` / `rollback() !bool` | Controls manual ACID transactions via `SQL_ATTR_AUTOCOMMIT` and `SQLEndTran`. |
| `odbc.postgres_dsn` / `mysql_dsn` / `sqlserver_dsn` / `sqlite_dsn` / `duckdb_dsn` | Helper builders for standard driver connection strings. |

---

## 5. Verification & Tests

Run the unit test suite and demo with Vira's leak detector:

```bash
vira test
vira run examples/universal_db_demo.vir
```

