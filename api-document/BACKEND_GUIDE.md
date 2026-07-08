
# ============================================================
# 7. BACKEND_GUIDE.md
# ============================================================

backend_guide_md = r'''# Nooht Framework — Backend Configuration Guide

**Version:** 0.1.0-alpha  
**Scope:** Storage backend selection, configuration, and migration

---

## Overview

Nooht supports multiple storage backends through the `MemoryBackend` abstraction. This guide helps you choose and configure the right backend for your use case.

| Backend | Best For | Persistence | Scale | Latency |
|---------|----------|-------------|-------|---------|
| `InMemoryBackend` | Development, testing, small projects | No | < 100K entities | < 1ms |
| `DuckDBBackend` | Production, large repositories | Yes | Unlimited | < 10ms @ 1M |

---

## InMemoryBackend

### When to Use

- Prototyping and unit tests
- Single-session analysis (< 100K symbols)
- No persistence requirement
- Single-threaded environments

### Configuration

```python
from nooht import SymbolMemory
from nooht.memory.backend import InMemoryBackend

memory = SymbolMemory(backend=InMemoryBackend())
```

### Characteristics

```
Capacity:     ~100,000 entities
Memory:       ~500MB @ 100K entities
Persistence:  None (lost on exit)
Threading:    Single-threaded only
Query:        Python set intersection
```

### Limitations

- ❌ Data lost when process exits
- ❌ Not thread-safe
- ❌ OOM risk at > 100K entities
- ❌ No SQL-powered analytics

---

## DuckDBBackend

### When to Use

- Production deployments
- Multi-repo codebases (> 100K symbols)
- Persistent memory across sessions
- Analytical queries (compression, statistics)
- Multi-threaded environments

### Configuration

```python
from nooht import SymbolMemory
from nooht.memory.backend import DuckDBBackend

# Persistent file
memory = SymbolMemory(backend=DuckDBBackend("nooht.db"))

# In-memory (for testing)
memory = SymbolMemory(backend=DuckDBBackend(":memory:"))
```

### Characteristics

```
Capacity:     Unlimited (columnar storage)
Memory:       ~100MB @ 1M entities (working set)
Persistence:  File-based (.db file)
Threading:    Thread-local connections
Query:        SQL with SIMD vectorized scan
```

### Schema

```sql
CREATE TABLE symbols (
    id VARCHAR PRIMARY KEY,
    name VARCHAR,
    type VARCHAR,
    source TEXT,
    file_path VARCHAR,
    line_start INTEGER,
    line_end INTEGER,
    signature VARCHAR,
    docstring TEXT,
    parameters JSON,
    return_type VARCHAR,
    dependencies VARCHAR[],    -- Native LIST type
    callers VARCHAR[],         -- Native LIST type
    parent_id VARCHAR,
    summary TEXT,
    semantic_hash VARCHAR,
    status VARCHAR,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    access_count INTEGER DEFAULT 0,
    last_accessed TIMESTAMP,
    metadata JSON,
    tags VARCHAR[],            -- Native LIST type
    embedding BLOB,
    tombstone BOOLEAN DEFAULT FALSE
);
```

### Indexes

```sql
CREATE INDEX idx_name ON symbols(name);
CREATE INDEX idx_type ON symbols(type);
CREATE INDEX idx_status ON symbols(status);
CREATE INDEX idx_tombstone ON symbols(tombstone);
```

> **Note:** DuckDB does not support GIN indexes. `list_contains()` queries leverage DuckDB's native SIMD vectorized scan, which is already extremely fast for analytical workloads.

### Thread Safety

DuckDB connections are **not thread-safe** by default. Nooht handles this via `threading.local()`:

```python
# Each thread gets its own connection
# Safe to use in multi-threaded environments

from concurrent.futures import ThreadPoolExecutor

def worker(memory, entity_id):
    return memory.get(entity_id)

with ThreadPoolExecutor(max_workers=8) as executor:
    futures = [executor.submit(worker, memory, eid) for eid in ids]
```

### Tombstone Mechanism

Soft-delete is the default:

```python
memory.remove(entity_id)  # Sets tombstone = TRUE
memory.get(entity_id)     # Returns None (filtered)
memory.stats()            # Excludes tombstoned entities
```

Physical cleanup:

```python
backend = memory.backend  # DuckDBBackend
removed_count = backend.vacuum()  # DELETE WHERE tombstone = TRUE
```

**Recommendation:** Schedule `vacuum()` during maintenance windows. Do not call it during high-traffic periods.

---

## Backend Selection Decision Tree

```
Do you need persistence?
├── No → InMemoryBackend
└── Yes → Do you have > 100K entities?
    ├── No → InMemoryBackend (with periodic JSON export)
    └── Yes → DuckDBBackend
        └── Do you need multi-threading?
            ├── No → DuckDBBackend (default)
            └── Yes → DuckDBBackend (thread-local, already handled)
```

---

## Migration: InMemory → DuckDB

```python
from nooht import SymbolMemory
from nooht.memory.backend import InMemoryBackend, DuckDBBackend

# Source: InMemory
old_memory = SymbolMemory(backend=InMemoryBackend())
# ... populate with data ...

# Target: DuckDB
new_memory = SymbolMemory(backend=DuckDBBackend("migrated.db"))

# Migrate
for batch in old_memory.iter_all(batch_size=1000):
    for entity in batch:
        new_memory.add(entity)

print(f"Migrated {new_memory.count()} entities")
old_memory.close()
new_memory.close()
```

---

## Performance Tuning

### DuckDB Optimization

```python
# 1. Use file-based storage (not :memory:) for large datasets
backend = DuckDBBackend("production.db")

# 2. Batch inserts
entities = [entity_1, entity_2, ..., entity_1000]
for entity in entities:
    memory.add(entity)  # Each add is a transaction

# 3. Periodic vacuum (weekly)
backend.vacuum()

# 4. Close connections when done
memory.close()
```

### InMemory Optimization

```python
# 1. Pre-allocate if known capacity (not required, but helps)
backend = InMemoryBackend()

# 2. Batch queries instead of individual lookups
results = memory.query(tag="auth", limit=100)  # One query
# vs
for eid in entity_ids:
    memory.get(eid)  # N queries

# 3. Export before exit
import json
with open("backup.json", "w") as f:
    json.dump([e.to_dict() for e in memory.backend._entities.values()], f)
```

---

## Troubleshooting

### DuckDB: "Connection is busy"

**Cause:** Multiple threads sharing a single connection.

**Fix:** Ensure you're using `DuckDBBackend` (handles thread-local) and not raw `duckdb.connect()`.

### DuckDB: "Database is locked"

**Cause:** Multiple processes accessing the same .db file.

**Fix:** Use one DuckDB file per process, or implement file locking.

### InMemory: MemoryError

**Cause:** Too many entities loaded.

**Fix:** Switch to `DuckDBBackend` or increase system RAM.

### Slow `search_by_source()`

**Cause:** `LIKE '%pattern%'` requires full table scan.

**Fix:** This is expected behavior. For faster text search, consider:
- Pre-indexing with a full-text search engine (future plugin)
- Using `query(tag=...)` instead when possible
- Limiting `limit` parameter

---

## Comparison Table

| Feature | InMemoryBackend | DuckDBBackend |
|---------|-----------------|---------------|
| `insert()` | < 0.1ms | ~2ms |
| `get()` | < 0.1ms | ~1ms |
| `query()` | < 1ms | ~5ms |
| `count()` | < 0.1ms | ~1ms |
| `iter_all(1000)` | ~10ms | ~20ms |
| `search_by_source()` | ~50ms | ~50-200ms |
| Persistence | ❌ | ✅ |
| Thread-safe | ❌ | ✅ |
| SQL analytics | ❌ | ✅ |
| Vacuum | N/A | ✅ |
| Tombstone | Hard delete | Soft delete |

---

<p align="center"><i>For storage extension questions, see <a href="ARCHITECTURE.md">Architecture Overview</a></i></p>
'''
