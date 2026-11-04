# Tutorial 4: Retrieving Results, Export Formats (ASCII/CSV/JSON) & SQL Pagination

> **Package:** `odbc` (`v0.1.0`)  
> **Source Files:** [`src/types.vir`](../src/types.vir), [`src/operations.vir`](../src/operations.vir)

---

## 1. Querying & Retrieving Results

The `odbc.Connection` type provides four query methods tailored to different access patterns:

| Method | Return Type | Use Case |
| :--- | :--- | :--- |
| `conn.query(sql)` | `!ResultSet` | Materializes all columns, schema metadata (`ColumnInfo`), and rows (`[]Row`). |
| `conn.query_row(sql)` | `!Row` | Returns the first `Row` of a query (or an error if 0 rows matched). |
| `conn.query_scalar(sql)` | `!string` | Returns column `0` of row `0` as a `string`. |
| `conn.query_int(sql)` | `!int` | Executes a scalar query (e.g. `SELECT COUNT(*)`) and parses the result as `int`. |

### Typed Row Accessors (`Row`)
Every `odbc.Row` supports both column-name and 0-based index lookups with automatic type parsing and `NULL` detection:

```vira
rs := conn.query("SELECT id, service, latency_ms, healthy, last_error FROM cluster_nodes")!

for mut i := 0; i < rs.row_count(); i += 1 {
	row := rs.rows[i]

	id      := row.get_int("id")          // int (or get_i64("id") for 64-bit)
	service := row.get("service")         // string
	latency := row.get_f64("latency_ms")  // f64
	healthy := row.get_bool("healthy")    // bool ("1", "true", "t", "yes", "y")

	if row.is_null("last_error") {
		println("[${id}] ${service}: ${latency}ms (healthy=${healthy}, error=NULL)")
	} else {
		err_txt := row.get("last_error")
		println("[${id}] ${service}: ${latency}ms (healthy=${healthy}, error=${err_txt})")
	}
}
```

---

## 2. Formatting, Filtering & Exporting `ResultSet`

### Pretty-Printing ASCII Tables (`to_ascii_table`)
Render any `ResultSet` into an aligned ASCII box table for terminal output or logs:
```vira
table_str := rs.to_ascii_table()
println(table_str)
```
Output:
```text
+-----+-----------------+--------------+------------+---------+--------------------+
| id  | service         | region       | latency_ms | healthy | last_error         |
+-----+-----------------+--------------+------------+---------+--------------------+
| 101 | vira-gateway    | eu-central-1 | 1.12       | true    | NULL               |
| 104 | vira-storage-io | eu-central-1 | 9.70       | false   | disk queue timeout |
+-----+-----------------+--------------+------------+---------+--------------------+
```

### In-Memory Filtering & Column Extraction (`filter_eq`, `column_values`)
```vira
// Filter a ResultSet by column value without re-querying the database
eu_nodes := rs.filter_eq("region", "eu-central-1")

// Extract a single column across all rows as []string
service_names := eu_nodes.column_values("service")
```

### Exporting to CSV & JSON (`to_csv`, `to_json`)
Export any `ResultSet` directly to RFC-4180 CSV or a JSON array of objects (with SQL `NULL` values preserved as JSON `null`):
```vira
csv_payload  := eu_nodes.to_csv()
json_payload := eu_nodes.to_json()
```

---

## 3. Database Pagination (Offset/Limit, SQL Server `FETCH NEXT`, & Keyset Seek)

Different SQL engines use different syntax for pagination:
- **PostgreSQL, MySQL, MariaDB, SQLite3, DuckDB**: `ORDER BY ... LIMIT <page_size> OFFSET <offset>`
- **Microsoft SQL Server & Oracle 12c+**: `ORDER BY ... OFFSET <offset> ROWS FETCH NEXT <page_size> ROWS ONLY`

### Dialect-Aware Offset Pagination (`query_page` & `build_paginated_sql`)
```vira
// Create a PageRequest for Page 2 (25 rows per page, ordered by id ASC)
req := odbc.new_page_request(2, 25, "id ASC", odbc.DIALECT_POSTGRES)

// Option A: Execute both the total count query and paginated data query on a live Connection
page_res := conn.query_page(
	"SELECT id, service, region FROM cluster_nodes WHERE healthy = true",
	"SELECT COUNT(*) FROM cluster_nodes WHERE healthy = true",
	req
)!

println("Page ${page_res.page} of ${page_res.total_pages} (Total Rows: ${page_res.total_rows})")
println("Has Previous: ${page_res.has_prev} | Has Next: ${page_res.has_next}")

// Option B: Generate the dialect-specific SQL string directly
sql_pg := odbc.build_paginated_sql("SELECT id, service FROM cluster_nodes", req)
// -> SELECT id, service FROM cluster_nodes ORDER BY id ASC LIMIT 25 OFFSET 25;

req_mssql := odbc.new_page_request(2, 25, "id ASC", odbc.DIALECT_SQLSERVER)
sql_ms := odbc.build_paginated_sql("SELECT id, service FROM cluster_nodes", req_mssql)
// -> SELECT id, service FROM cluster_nodes ORDER BY id ASC OFFSET 25 ROWS FETCH NEXT 25 ROWS ONLY;
```

### High-Performance Keyset / Cursor Pagination (`build_keyset_sql`)
For large tables with millions of rows where `OFFSET` scans become slow, use `odbc.build_keyset_sql` to seek directly past the last seen cursor value using an index (`O(1)` lookup):

```vira
// Fetch the next 25 rows after id = '1050' in ascending order
seek_sql := odbc.build_keyset_sql(
	"SELECT id, service, latency_ms FROM cluster_nodes WHERE healthy = true",
	"id",
	"1050",
	25,
	true,
	odbc.DIALECT_POSTGRES
)
// -> SELECT id, service, latency_ms FROM cluster_nodes WHERE healthy = true AND id > '1050' ORDER BY id ASC LIMIT 25;
```

### In-Memory `ResultSet` Pagination (`rs.paginate`)
When caching a `ResultSet` in memory, slice any page on demand with `rs.paginate(page, page_size)`:

```vira
for mut p := 1; p <= 3; p += 1 {
	page := rs.paginate(p, 2)
	println("Page ${page.page}/${page.total_pages} (${page.rows.len} rows)")
}
```
