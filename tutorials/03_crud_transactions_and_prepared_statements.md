# Tutorial 3: Full Database Operations (CRUD, Batch Transactions & Prepared Statements)

> **Package:** `odbc` (`v0.1.0`)  
> **Source Files:** [`src/odbc.vir`](../src/odbc.vir), [`src/operations.vir`](../src/operations.vir)

---

## 1. Executing Schema DDL (`CREATE`, `ALTER`, `DROP`)

Use `conn.exec(sql)` to execute DDL statements or raw DML queries:

```v
_ := conn.exec("
	CREATE TABLE IF NOT EXISTS orders (
		id INT PRIMARY KEY,
		customer VARCHAR(128) NOT NULL,
		total_usd DECIMAL(10, 2) NOT NULL,
		status VARCHAR(32) NOT NULL,
		notes VARCHAR(255)
	)
")!
```

---

## 2. Safe Literal Escaping & CRUD Helpers

The `odbc` package provides SQL-injection-safe literal escaping (`odbc.escape_literal`) and dialect-aware identifier quoting (`odbc.quote_ident`), along with high-level CRUD builders and execution methods.

### Inserting Rows (`insert_row` & `build_insert_sql`)
Pass `"NULL"` as a value string whenever you want to insert a SQL `NULL`:

```v
mut cols := []string{}
cols.push("id")
cols.push("customer")
cols.push("total_usd")
cols.push("status")
cols.push("notes")

mut vals := []string{}
vals.push("1001")
vals.push("O'Reilly Media") // Automatically escaped to 'O''Reilly Media'
vals.push("249.99")
vals.push("PAID")
vals.push("NULL")           // Emitted as SQL NULL

// Execute directly on a connection:
affected := conn.insert_row("orders", cols, vals)!
println("Inserted ${affected} row(s)")
```

### Upserting Rows (`build_upsert_postgres_sql`)
For PostgreSQL, SQLite, and DuckDB `INSERT ... ON CONFLICT DO UPDATE`:

```v
mut update_cols := []string{}
update_cols.push("total_usd")
update_cols.push("status")

upsert_sql := odbc.build_upsert_postgres_sql("orders", cols, vals, "id", update_cols)
_ := conn.exec(upsert_sql)!
```

### Updating Rows (`update_rows` & `build_update_sql`)
```v
mut set_cols := []string{}
set_cols.push("status")
set_cols.push("notes")

mut set_vals := []string{}
set_vals.push("SHIPPED")
set_vals.push("Dispatched via express courier")

updated := conn.update_rows("orders", set_cols, set_vals, "id = 1001")!
println("Updated ${updated} row(s)")
```

### Deleting Rows (`delete_rows` & `build_delete_sql`)
```v
deleted := conn.delete_rows("orders", "status = 'CANCELLED'")!
println("Deleted ${deleted} cancelled order(s)")
```

---

## 3. ACID Transactions & Atomic Batch Execution

### Manual Transaction Control (`begin_tx`, `commit`, `rollback`)
```v
conn.begin_tx()!

_ := conn.exec("UPDATE accounts SET balance = balance - 500 WHERE id = 1") or {
	_ := conn.rollback()!
	return
}

_ := conn.exec("UPDATE accounts SET balance = balance + 500 WHERE id = 2") or {
	_ := conn.rollback()!
	return
}

conn.commit()!
```

### Atomic Batch Execution (`exec_batch_tx`)
Use `conn.exec_batch_tx(statements)` to run a slice of SQL statements inside a single transaction. If any statement in the slice fails, `exec_batch_tx` automatically rolls back the entire batch and returns the error with the failing statement index:

```v
mut batch := []string{}
batch.push("INSERT INTO audit_log (event, actor) VALUES ('TX_START', 'system');")
batch.push("UPDATE inventory SET qty = qty - 1 WHERE sku = 'VIRA-GPU-01';")
batch.push("INSERT INTO audit_log (event, actor) VALUES ('TX_COMPLETE', 'system');")

total_affected := conn.exec_batch_tx(batch) or {
	eprintln("Batch transaction rolled back: ${err}")
	return
}
println("Batch committed! Total affected rows: ${total_affected}")
```

---

## 4. Server-Side Prepared Statements (`prepare`)

When executing a query repeatedly, compile it once on the database server using `conn.prepare(sql)`:

```v
mut stmt := conn.prepare("SELECT id, customer, total_usd FROM orders WHERE status = 'PAID'")!
defer stmt.close()

rs := stmt.query()!
println("Retrieved ${rs.row_count()} paid orders")
```

