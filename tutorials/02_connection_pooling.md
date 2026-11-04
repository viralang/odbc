# Tutorial 2: OS Driver Manager Pooling & Managed `ConnectionPool`

> **Package:** `odbc` (`v0.1.0`)  
> **Source File:** [`src/pool.vir`](../src/pool.vir)

---

## 1. Overview

Opening a new TCP/TLS database connection and authenticating on every HTTP request or background job adds 10–50ms of latency. The `odbc` package provides **two complementary layers of connection pooling**:

1. **OS-Level ODBC Driver Manager Pooling (`enable_driver_pooling`)**: Configures `unixODBC` / `odbc32` via `SQL_ATTR_CONNECTION_POOLING` to transparently reuse physical connections across `SQLDisconnect` / `SQLDriverConnect` calls.
2. **Application-Level Managed `ConnectionPool`**: Maintains a bounded pool of live `odbc.Connection` handles in Vira memory with automatic health checks (`SELECT 1`), transaction cleanup on release, and real-time pool telemetry (`PoolStats`).

---

## 2. Enabling Native Driver Manager Pooling

Call `odbc.enable_driver_pooling(per_driver, relaxed_match)` once at the start of `main()` before creating any `Env` or `Connection`:

```vira
module main

import odbc

fn main() {
	// per_driver = true  -> SQL_CP_ONE_PER_DRIVER
	// relaxed_match = true -> SQL_CP_RELAXED_MATCH
	odbc.enable_driver_pooling(true, true) or {
		eprintln("Warning: could not enable driver pooling: ${err}")
		return
	}
}
```

---

## 3. Using the Application-Level `ConnectionPool`

### Configuring & Initializing a Pool
Use `odbc.default_pool_config(conn_str, max_conns)` or construct `odbc.PoolConfig` directly:

```vira
dsn := odbc.postgres_dsn("", "127.0.0.1", 5432, "production", "postgres", "secret")

cfg := odbc.PoolConfig{
	conn_str:            dsn,
	min_conns:           2,    // Pre-warm 2 connections immediately on startup
	max_conns:           10,   // Maximum concurrent active connections
	login_timeout_sec:   5,    // Timeout in seconds when opening new connections
	validate_on_acquire: true, // Run `conn.ping()` (`SELECT 1`) before handing out idle connections
}

mut pool := odbc.ConnectionPool.new(cfg) or {
	panic("Failed to initialize connection pool: ${err}")
}
defer pool.close_all()
```

### Acquiring & Releasing Connections
Lease a connection from the pool using `pool.acquire()` and return it when finished using `pool.release(mut conn)`:

```vira
mut conn := pool.acquire() or {
	eprintln("Pool exhausted or database unreachable: ${err}")
	return
}

// Perform queries using the leased connection
rs := conn.query("SELECT id, email FROM users WHERE is_active = true") or {
	pool.release(mut conn)
	return
}
println("Fetched ${rs.row_count()} active users")

// Always release the connection back to the pool
pool.release(mut conn)
```

> **Automatic Transaction Safety:** If a leased connection was left inside an uncommitted transaction (`conn.in_tx == true`), `pool.release(mut conn)` automatically executes `conn.rollback()` before placing the connection back into the idle pool so subsequent callers never inherit dirty transaction state.

---

## 4. Monitoring Pool Telemetry (`PoolStats`)

Inspect live pool utilization at any time via `pool.stats()`:

```vira
st := pool.stats()
println("Pool Telemetry:")
println("  Max Capacity         : ${st.max_conns}")
println("  Idle Connections     : ${st.idle_conns}")
println("  Active Leases        : ${st.active_conns}")
println("  Total Created        : ${st.total_created}")
println("  Total Acquired       : ${st.total_acquired}")
println("  Total Released       : ${st.total_released}")
println("  Health Checks Failed : ${st.health_checks_failed}")
```
