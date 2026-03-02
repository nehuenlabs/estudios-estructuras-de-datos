# Guía de Ejercicios — Cap.17: Índices: Cómo las Bases de Datos Encuentran tus Datos

> Este capítulo no introduce nada nuevo.
> Este capítulo UNIFICA todo lo que ya sabes.
>
> Cap.04: Hash tables → hash index.
> Cap.06: Bloom filters → skip innecesario en LSM-Trees.
> Cap.09: B+ Tree → el índice estándar de bases de datos.
> Cap.10: LSM-Tree → el índice de Cassandra y RocksDB.
> Cap.16: Columnar → column pruning y predicate pushdown.
>
> Una base de datos es una COLECCIÓN de índices.
> PostgreSQL elige entre B+ Tree, hash, GIN, GiST, BRIN.
> MySQL/InnoDB tiene clustered B+ Tree + secondary indexes.
> Cassandra tiene LSM-Tree + Bloom filters + partition key.
> Elasticsearch tiene inverted index + FST.
>
> La pregunta no es "¿qué índice usar?"
> sino "¿qué COMBINACIÓN de índices resuelve mi workload?"
>
> Este capítulo construye un mini database engine
> que soporta 3 tipos de índice y demuestra
> cuándo cada uno gana.

---

## El modelo mental: un índice es un tradeoff

```
Sin índice:
  SELECT * FROM users WHERE email = 'alice@example.com';
  → Full table scan: leer TODAS las filas. O(n).
  10M filas × 200 bytes = 2 GB de I/O. ~2 segundos.

Con B+ Tree index en email:
  → Navegar el árbol: 2-3 I/Os. O(log n).
  → Leer la fila: 1 I/O.
  Total: 3-4 I/Os × 8 KB = 32 KB. ~0.1 ms.
  → 20,000× más rápido.

Pero:
  Cada INSERT actualiza el índice. Más lento.
  El índice ocupa espacio en disco. Más storage.
  Más índices = más overhead por write.

  ÍNDICE = acelerar reads A COSTA de writes y espacio.

  Tipo            Read          Write         Space    Soporta
  ────            ────          ─────         ─────    ───────
  Hash index      O(1) exact    O(1)          Bajo     Solo =
  B+ Tree         O(log n)      O(log n)      Medio    =, <, >, range, ORDER BY
  LSM-Tree        O(log n)*     O(1) amort.   Medio    =, <, >, range
  Bitmap index    O(1) filter   O(n) rebuild  Bajo     Low cardinality, AND/OR
  Inverted index  O(1) per term O(n) rebuild  Alto     Full-text search

  * Con Bloom filters para evitar lecturas innecesarias.

  No hay un índice universal.
  La elección depende del WORKLOAD.
```

---

## Tabla de contenidos

- [Sección 17.1 — Hash index: lookup exacto en O(1)](#sección-171--hash-index-lookup-exacto-en-o1)
- [Sección 17.2 — B+ Tree index: el estándar relacional](#sección-172--b-tree-index-el-estándar-relacional)
- [Sección 17.3 — LSM-Tree index: write-optimized](#sección-173--lsm-tree-index-write-optimized)
- [Sección 17.4 — Bitmap index: baja cardinalidad](#sección-174--bitmap-index-baja-cardinalidad)
- [Sección 17.5 — Composite indexes y el orden de las columnas](#sección-175--composite-indexes-y-el-orden-de-las-columnas)
- [Sección 17.6 — Mini database engine con 3 tipos de índice](#sección-176--mini-database-engine-con-3-tipos-de-índice)
- [Sección 17.7 — Índices en producción: PostgreSQL, MySQL, Cassandra, Elasticsearch](#sección-177--índices-en-producción-postgresql-mysql-cassandra-elasticsearch)

---

## Sección 17.1 — Hash Index: Lookup Exacto en O(1)

### Ejercicio 17.1.1 — Implementar: hash index sobre un append-only log

**Tipo: Implementar**

```python
import os, struct, hashlib

class HashIndex:
    """
    Bitcask-style hash index: hash map in memory,
    data in an append-only log file on disk.
    """
    def __init__(self, path="db.log"):
        self.path = path
        self.index = {}  # key → (offset, length) in log file
        self.file = open(path, "ab+")
        self._rebuild_index()

    def _rebuild_index(self):
        """Scan the log to rebuild the in-memory index"""
        self.file.seek(0)
        offset = 0
        while True:
            header = self.file.read(8)  # key_len (4) + val_len (4)
            if len(header) < 8: break
            key_len, val_len = struct.unpack("!II", header)
            key = self.file.read(key_len).decode()
            value = self.file.read(val_len)
            total_len = 8 + key_len + val_len
            self.index[key] = (offset, total_len)
            offset += total_len

    def put(self, key: str, value: bytes):
        key_bytes = key.encode()
        header = struct.pack("!II", len(key_bytes), len(value))
        offset = self.file.seek(0, 2)  # seek to end
        self.file.write(header + key_bytes + value)
        self.file.flush()
        self.index[key] = (offset, 8 + len(key_bytes) + len(value))

    def get(self, key: str) -> bytes:
        if key not in self.index:
            return None
        offset, length = self.index[key]
        self.file.seek(offset)
        data = self.file.read(length)
        key_len, val_len = struct.unpack("!II", data[:8])
        return data[8 + key_len:]

    def delete(self, key: str):
        if key in self.index:
            self.put(key, b"__TOMBSTONE__")
            del self.index[key]

    def __len__(self):
        return len(self.index)

    def close(self):
        self.file.close()


# ═══ Test ═══
import time, os

try: os.remove("db.log")
except: pass

db = HashIndex("db.log")
n = 500_000

start = time.perf_counter_ns()
for i in range(n):
    db.put(f"key_{i}", f"value_{i}_{'x' * 50}".encode())
write_time = time.perf_counter_ns() - start

start = time.perf_counter_ns()
found = 0
for i in range(0, n, 5):
    if db.get(f"key_{i}") is not None: found += 1
read_time = time.perf_counter_ns() - start
queries = n // 5

print(f"Hash Index over append-only log:")
print(f"  Write {n:,}: {write_time // 1_000_000} ms ({write_time // n} ns/op)")
print(f"  Read {queries:,}: {read_time // 1_000_000} ms ({read_time // queries} ns/op)")
print(f"  Found: {found:,}")
print(f"  Log size: {os.path.getsize('db.log') // (1024*1024)} MB")
print(f"  Index entries: {len(db):,}")

db.close()
os.remove("db.log")
```

**Preguntas:**

1. ¿El índice está COMPLETAMENTE en memoria. ¿Para 1 billón de keys?

2. ¿Append-only log: las actualizaciones agregan al final. ¿Y la data vieja?

3. ¿Bitcask (Riak) usa exactamente este diseño. ¿Para qué workloads?

4. ¿Hash index no soporta range queries. ¿Por qué?

5. ¿Compaction del log: merge entries duplicadas. ¿Es como LSM-Tree compaction?

---

## Sección 17.2 — B+ Tree Index: el Estándar Relacional

### Ejercicio 17.2.1 — Analizar: por qué B+ Tree es el default

**Tipo: Analizar**

```
¿Por qué TODAS las bases de datos relacionales usan B+ Tree como default?

  1. SOPORTA TODAS LAS OPERACIONES:
     = (equality): navegar el árbol. O(log n).
     <, >, <=, >= (range): encontrar inicio, scan hojas. O(log n + k).
     ORDER BY: las hojas ya están ordenadas.
     MIN/MAX: primera/última hoja.
     BETWEEN: range scan de hojas enlazadas.
     LIKE 'prefix%': range scan.
     → El hash index solo soporta =. El B+ Tree soporta TODO.

  2. RENDIMIENTO PREDECIBLE:
     Altura 3-4 para billones de keys.
     3-4 I/Os por query. Siempre. No hay peor caso sorprendente.
     → El hash index tiene O(1) esperado pero colisiones pueden degradar.

  3. CONCURRENCIA PROBADA:
     Latch-based crabbing protocol (Cap.09).
     Blink-tree variant (PostgreSQL).
     → Décadas de optimización en producción.

  4. MANTENIMIENTO AUTOMÁTICO:
     Los DBAs no necesitan elegir. CREATE INDEX crea un B+ Tree.
     Funciona bien para el 90% de los casos.

  ¿Cuándo NO usar B+ Tree?
    - Writes extremadamente altos → LSM-Tree (Cassandra, RocksDB).
    - Solo equality lookups → Hash index (más rápido para =).
    - Columnas de baja cardinalidad → Bitmap index.
    - Full-text search → Inverted index (Elasticsearch).
    - Datos geoespaciales → R-Tree, GiST.

  PostgreSQL CREATE INDEX:
    CREATE INDEX idx ON users(email);         -- B+ Tree (default)
    CREATE INDEX idx ON users USING hash(id); -- Hash index
    CREATE INDEX idx ON docs USING gin(body); -- Inverted index (GIN)
    CREATE INDEX idx ON geo USING gist(loc);  -- Spatial index
```

**Preguntas:**

1. ¿B+ Tree soporta LIKE 'prefix%'. ¿LIKE '%middle%'?

2. ¿PostgreSQL hash index — ¿cuándo es más rápido que B+ Tree?

3. ¿GIN index de PostgreSQL — ¿es un inverted index?

4. ¿BRIN index de PostgreSQL — ¿qué es y cuándo se usa?

5. ¿Para un data engineer escribiendo queries, ¿necesita crear índices?

---

## Sección 17.3 — LSM-Tree Index: Write-Optimized

### Ejercicio 17.3.1 — Analizar: LSM-Tree como índice de base de datos

**Tipo: Analizar**

```
Recapitulación del Cap.10, ahora como "index engine":

  El LSM-Tree no es solo una estructura de datos.
  Es un INDEX ENGINE completo:
    - MemTable = índice in-memory (skip list sorted).
    - SSTable = índice on-disk (sorted file con index block).
    - Bloom filter = filtro para skip de SSTables innecesarias.
    - WAL = durabilidad.
    - Compaction = mantenimiento del índice.

  Comparación como index engine:

  Operación        B+ Tree (PostgreSQL)    LSM-Tree (Cassandra)
  ─────────        ──────────────────      ──────────────────
  Point read       1-2 I/Os (directo)      1-4 I/Os (MemTable + SSTables)
  Range scan       O(log n + k) secuencial O(log n + k × L) merge
  Insert           Random write (lento)    Sequential write (rápido)
  Update           In-place (random I/O)   Append (sequential I/O)
  Delete           In-place                Tombstone + compaction
  Space overhead   ~1× datos              ~1.1-2× datos (compaction)

  Cassandra como index engine:
    Primary key → partition key (hash) + clustering key (sorted).
    Partition key: consistent hashing (Cap.15) → qué nodo.
    Clustering key: LSM-Tree index → orden dentro de la partición.
    → SELECT * FROM events WHERE partition_key = X
      AND timestamp BETWEEN t1 AND t2;
    → Partition key localiza el nodo. Clustering key hace range scan.

  RocksDB como index engine embebido:
    CockroachDB: SQL → RocksDB key-value lookups.
    TiDB: SQL → RocksDB key-value lookups.
    Flink: state backend → RocksDB LSM-Tree.
    → El LSM-Tree es el motor DETRÁS de bases de datos SQL y streaming.
```

**Preguntas:**

1. ¿Cassandra partition key es hash. ¿No soporta range queries sobre partition key?

2. ¿RocksDB como base de CockroachDB — ¿es SQL sobre LSM-Tree?

3. ¿Flink state backend en RocksDB — ¿cada operador tiene su propio LSM-Tree?

4. ¿LSM-Tree vs B+ Tree para un OLTP workload 50/50 reads/writes?

5. ¿MyRocks (MySQL sobre RocksDB) — ¿es mejor que InnoDB?

---

## Sección 17.4 — Bitmap Index: Baja Cardinalidad

### Ejercicio 17.4.1 — Implementar: bitmap index en Python

**Tipo: Implementar**

```python
class BitmapIndex:
    """Bitmap index for low-cardinality columns"""
    def __init__(self, n_rows):
        self.n_rows = n_rows
        self.bitmaps = {}  # value → bytearray bitmap

    def build(self, values):
        for i, v in enumerate(values):
            if v not in self.bitmaps:
                self.bitmaps[v] = bytearray(((self.n_rows + 7) // 8))
            byte_idx, bit_idx = i // 8, i % 8
            self.bitmaps[v][byte_idx] |= (1 << bit_idx)

    def query_eq(self, value):
        """WHERE col = value"""
        bm = self.bitmaps.get(value)
        if bm is None: return []
        return self._bitmap_to_rows(bm)

    def query_and(self, val1, val2, bitmaps2):
        """WHERE col1 = val1 AND col2 = val2"""
        bm1 = self.bitmaps.get(val1)
        bm2 = bitmaps2.bitmaps.get(val2)
        if bm1 is None or bm2 is None: return []
        result = bytearray(len(bm1))
        for i in range(len(bm1)):
            result[i] = bm1[i] & bm2[i]  # bitwise AND
        return self._bitmap_to_rows(result)

    def query_or(self, val1, val2):
        """WHERE col = val1 OR col = val2"""
        bm1 = self.bitmaps.get(val1, bytearray(len(list(self.bitmaps.values())[0])))
        bm2 = self.bitmaps.get(val2, bytearray(len(bm1)))
        result = bytearray(len(bm1))
        for i in range(len(bm1)):
            result[i] = bm1[i] | bm2[i]  # bitwise OR
        return self._bitmap_to_rows(result)

    def count_eq(self, value):
        bm = self.bitmaps.get(value)
        if bm is None: return 0
        return sum(bin(byte).count('1') for byte in bm)

    def _bitmap_to_rows(self, bm):
        rows = []
        for byte_idx in range(len(bm)):
            if bm[byte_idx] == 0: continue
            for bit_idx in range(8):
                if bm[byte_idx] & (1 << bit_idx):
                    row = byte_idx * 8 + bit_idx
                    if row < self.n_rows: rows.append(row)
        return rows

    def memory_bytes(self):
        return sum(len(bm) for bm in self.bitmaps.values())


# ═══ Benchmark ═══
import random, time

n = 5_000_000
departments = ["Eng", "Sales", "Mkt", "Fin", "HR", "Legal", "Ops", "Support"]
statuses = ["active", "inactive", "on_leave"]
dept_col = [random.choice(departments) for _ in range(n)]
status_col = [random.choice(statuses) for _ in range(n)]

# Build bitmap indexes
dept_bm = BitmapIndex(n)
start = time.perf_counter_ns()
dept_bm.build(dept_col)
build_dept = time.perf_counter_ns() - start

status_bm = BitmapIndex(n)
status_bm.build(status_col)

print(f"Bitmap index for {n:,} rows:")
print(f"  Build dept: {build_dept // 1_000_000} ms")
print(f"  Dept bitmaps: {len(dept_bm.bitmaps)} values, {dept_bm.memory_bytes() // 1024} KB")
print(f"  Status bitmaps: {len(status_bm.bitmaps)} values, {status_bm.memory_bytes() // 1024} KB")

# COUNT WHERE dept = 'Eng'
start = time.perf_counter_ns()
count_bm = dept_bm.count_eq("Eng")
bm_time = time.perf_counter_ns() - start

start = time.perf_counter_ns()
count_scan = sum(1 for d in dept_col if d == "Eng")
scan_time = time.perf_counter_ns() - start

print(f"\nCOUNT WHERE dept='Eng': {count_bm}")
print(f"  Bitmap:    {bm_time // 1_000_000} ms")
print(f"  Full scan: {scan_time // 1_000_000} ms")
print(f"  Speedup:   {scan_time / bm_time:.1f}×")

# AND query: WHERE dept='Eng' AND status='active'
start = time.perf_counter_ns()
and_rows = dept_bm.query_and("Eng", "active", status_bm)
and_time = time.perf_counter_ns() - start

start = time.perf_counter_ns()
scan_rows = [i for i in range(n) if dept_col[i] == "Eng" and status_col[i] == "active"]
scan_and_time = time.perf_counter_ns() - start

print(f"\nWHERE dept='Eng' AND status='active': {len(and_rows)} rows")
print(f"  Bitmap AND: {and_time // 1_000_000} ms")
print(f"  Full scan:  {scan_and_time // 1_000_000} ms")
print(f"  Speedup:    {scan_and_time / and_time:.1f}×")
```

**Preguntas:**

1. ¿Bitmap AND/OR son operaciones de bits. ¿CPU las hace en 1 ciclo?

2. ¿5M filas = 625 KB por bitmap. ¿Para 1 billón de filas?

3. ¿Bitmap index solo funciona con baja cardinalidad. ¿Cuánto es "baja"?

4. ¿Apache Druid usa bitmap indexes. ¿Para qué tipo de queries?

5. ¿Roaring Bitmaps — ¿son bitmap indexes comprimidos?

---

## Sección 17.5 — Composite Indexes y el Orden de las Columnas

### Ejercicio 17.5.1 — Analizar: por qué INDEX(a, b) no sirve para WHERE b = X

**Tipo: Analizar**

```
Composite index: un B+ Tree con key compuesta.
  CREATE INDEX idx ON orders(customer_id, order_date);

  El B+ Tree ordena por (customer_id, order_date):
    (1, 2024-01-01), (1, 2024-01-15), (1, 2024-02-01),
    (2, 2024-01-05), (2, 2024-01-20), (3, 2024-01-10), ...

  WHERE customer_id = 1 AND order_date > '2024-01-10':
    → Navegar a customer_id=1, luego scan order_date > 01-10.
    → EFICIENTE: el B+ Tree soporta ambas condiciones.

  WHERE customer_id = 1:
    → Navegar a customer_id=1, retornar todas las fechas.
    → EFICIENTE: es un prefijo del index.

  WHERE order_date = '2024-01-15':
    → NO PUEDE usar el index eficientemente.
    → Necesita full scan del index (o table scan).
    → El B+ Tree está ordenado por customer_id PRIMERO.
      No puede saltar directamente a una fecha sin customer_id.

  Regla del PREFIJO:
    INDEX(a, b, c) soporta eficientemente:
    ✓ WHERE a = X
    ✓ WHERE a = X AND b = Y
    ✓ WHERE a = X AND b = Y AND c = Z
    ✓ WHERE a = X AND b > Y
    ✗ WHERE b = Y (no es un prefijo)
    ✗ WHERE c = Z (no es un prefijo)
    ✗ WHERE a = X AND c = Z (gap en el medio)

  Implicación para diseño de schemas:
    La columna más selectiva NO siempre va primero.
    La columna que más aparece en WHERE va primero.
    Si queries son WHERE customer_id = X AND date BETWEEN:
      → INDEX(customer_id, date). NO INDEX(date, customer_id).
```

**Preguntas:**

1. ¿INDEX(a, b) no soporta WHERE b = X. ¿Necesito INDEX(b)?

2. ¿INDEX(a, b, c) con WHERE a = X AND c = Z — ¿usa parte del index?

3. ¿Cassandra clustering key es un composite index. ¿Misma regla de prefijo?

4. ¿Para queries variados, ¿múltiples indexes o un composite grande?

5. ¿Covering index: ¿incluir todas las columnas del SELECT en el index?

---

## Sección 17.6 — Mini Database Engine con 3 Tipos de Índice

### Ejercicio 17.6.1 — Implementar: motor con hash, B-Tree, y bitmap index

**Tipo: Implementar**

```python
from collections import defaultdict
import bisect, time, random

class MiniDB:
    """Mini database engine with 3 index types"""

    def __init__(self):
        self.rows = []          # list of dicts
        self.hash_indexes = {}  # col → {value: [row_ids]}
        self.btree_indexes = {} # col → sorted [(value, row_id)]
        self.bitmap_indexes = {} # col → {value: set(row_ids)}

    def insert(self, row: dict):
        rid = len(self.rows)
        self.rows.append(row)
        for col, idx in self.hash_indexes.items():
            idx.setdefault(row[col], []).append(rid)
        for col in self.btree_indexes:
            bisect.insort(self.btree_indexes[col], (row[col], rid))
        for col, bm in self.bitmap_indexes.items():
            bm.setdefault(row[col], set()).add(rid)

    def create_hash_index(self, col):
        idx = defaultdict(list)
        for rid, row in enumerate(self.rows):
            idx[row[col]].append(rid)
        self.hash_indexes[col] = dict(idx)

    def create_btree_index(self, col):
        entries = sorted((row[col], rid) for rid, row in enumerate(self.rows))
        self.btree_indexes[col] = entries

    def create_bitmap_index(self, col):
        bm = defaultdict(set)
        for rid, row in enumerate(self.rows):
            bm[row[col]].add(rid)
        self.bitmap_indexes[col] = dict(bm)

    def query_eq_scan(self, col, val):
        return [rid for rid, row in enumerate(self.rows) if row[col] == val]

    def query_eq_hash(self, col, val):
        return self.hash_indexes.get(col, {}).get(val, [])

    def query_eq_bitmap(self, col, val):
        return list(self.bitmap_indexes.get(col, {}).get(val, set()))

    def query_range_scan(self, col, lo, hi):
        return [rid for rid, row in enumerate(self.rows) if lo <= row[col] <= hi]

    def query_range_btree(self, col, lo, hi):
        entries = self.btree_indexes.get(col, [])
        start = bisect.bisect_left(entries, (lo,))
        result = []
        for i in range(start, len(entries)):
            if entries[i][0] > hi: break
            result.append(entries[i][1])
        return result


# ═══ Benchmark ═══
db = MiniDB()
n = 1_000_000
depts = ["Eng", "Sales", "Mkt", "Fin", "HR", "Legal", "Ops", "Support"]

for i in range(n):
    db.insert({
        "id": i,
        "email": f"user_{i}@example.com",
        "department": random.choice(depts),
        "salary": random.randint(40000, 200000),
    })

db.create_hash_index("email")
db.create_btree_index("salary")
db.create_bitmap_index("department")

print(f"MiniDB: {n:,} rows, 3 indexes\n")

# Q1: WHERE email = 'user_42@example.com' (hash vs scan)
q = "user_42@example.com"
start = time.perf_counter_ns()
for _ in range(1000): r1 = db.query_eq_scan("email", q)
scan_t = (time.perf_counter_ns() - start) // 1000

start = time.perf_counter_ns()
for _ in range(1000): r2 = db.query_eq_hash("email", q)
hash_t = (time.perf_counter_ns() - start) // 1000

print(f"Q1: WHERE email = '...' (equality)")
print(f"  Scan:  {scan_t // 1000} μs")
print(f"  Hash:  {hash_t // 1000} μs  ({scan_t / max(hash_t,1):.0f}× faster)")

# Q2: WHERE salary BETWEEN 80000 AND 90000 (btree vs scan)
start = time.perf_counter_ns()
for _ in range(100): r3 = db.query_range_scan("salary", 80000, 90000)
scan_t = (time.perf_counter_ns() - start) // 100

start = time.perf_counter_ns()
for _ in range(100): r4 = db.query_range_btree("salary", 80000, 90000)
btree_t = (time.perf_counter_ns() - start) // 100

print(f"\nQ2: WHERE salary BETWEEN 80000 AND 90000 (range, ~{len(r4)} rows)")
print(f"  Scan:   {scan_t // 1_000_000} ms")
print(f"  B-Tree: {btree_t // 1_000_000} ms  ({scan_t / max(btree_t,1):.0f}× faster)")

# Q3: WHERE department = 'Eng' (bitmap vs scan)
start = time.perf_counter_ns()
for _ in range(100): r5 = db.query_eq_scan("department", "Eng")
scan_t = (time.perf_counter_ns() - start) // 100

start = time.perf_counter_ns()
for _ in range(100): r6 = db.query_eq_bitmap("department", "Eng")
bm_t = (time.perf_counter_ns() - start) // 100

print(f"\nQ3: WHERE department = 'Eng' (low cardinality, ~{len(r6)} rows)")
print(f"  Scan:   {scan_t // 1_000_000} ms")
print(f"  Bitmap: {bm_t // 1_000_000} ms  ({scan_t / max(bm_t,1):.0f}× faster)")
```

**Preguntas:**

1. ¿Hash para equality, B-Tree para range, bitmap para low cardinality. ¿Cuándo cada uno?

2. ¿Una base real tendría los 3 tipos. ¿PostgreSQL soporta los 3?

3. ¿El mini engine usa `bisect` para B-Tree. ¿Es un B-Tree real?

4. ¿Para un data warehouse, ¿qué tipo de índice domina?

5. ¿Elasticsearch: inverted index. ¿Sería un cuarto tipo en el engine?

---

## Sección 17.7 — Índices en Producción: PostgreSQL, MySQL, Cassandra, Elasticsearch

### Ejercicio 17.7.1 — Analizar: qué índice usa cada sistema

**Tipo: Analizar**

```
POSTGRESQL:
  Default: B+ Tree. CREATE INDEX crea B+ Tree.
  Hash: solo equality, WAL-logged desde PG10.
  GIN: inverted index para arrays, JSONB, full-text.
  GiST: spatial index para geometry, range types.
  BRIN: Block Range INdex para datos naturalmente ordenados
        (timestamps). Muy compacto: min/max por rango de páginas.
  → Para un data engineer: B+ Tree + BRIN para time series.

MYSQL / INNODB:
  Clustered: B+ Tree con datos en las hojas (Cap.09).
  Secondary: B+ Tree con (key, PK). Double lookup.
  Covering: secondary index con todas las columnas del SELECT.
  → Para un data engineer: entender clustered vs secondary.

CASSANDRA:
  Primary: partition key (hash) + clustering key (LSM-Tree sorted).
  Secondary index: local index por nodo. Generalmente lento.
  Materialized views: pre-computed queries.
  SAI (Storage-Attached Indexes): nuevo, más eficiente.
  → Para un data engineer: diseñar el partition key es el 90% del trabajo.

ELASTICSEARCH / LUCENE:
  Inverted index: term → list of document IDs.
  FST (Finite State Transducer): diccionario de terms comprimido (Cap.11).
  BKD tree: para datos numéricos y geo.
  Doc values: columnar storage para sorting y aggregations.
  → Para un data engineer: Elasticsearch es un inverted index + columnar.

RESUMEN:
  Exact lookup       → Hash index.
  Range / ORDER BY   → B+ Tree.
  Write-heavy        → LSM-Tree.
  Low cardinality    → Bitmap index.
  Full-text search   → Inverted index.
  Time series        → BRIN (PostgreSQL) o partition key (Cassandra).
  Columnar analytics → Parquet stats + column pruning.
```

**Preguntas:**

1. ¿BRIN para time series — ¿cómo funciona?

2. ¿Cassandra SAI vs secondary indexes — ¿cuál es la diferencia?

3. ¿Elasticsearch doc values es columnar. ¿Como Parquet?

4. ¿Para un pipeline Kafka → Cassandra → Spark, ¿cuántos tipos de índice?

5. ¿El proyecto final (Cap.18) implementa cuántos de estos?

---

### Ejercicio 17.7.2 — Resumen: las reglas de los índices

**Tipo: Leer**

```
Reglas nuevas del Cap.17:

  Regla 72: Un índice = acelerar reads a costa de writes y espacio.
    Sin índice: full scan O(n). Con índice: O(log n) o O(1).
    Cada INSERT/UPDATE debe actualizar TODOS los índices.
    Más índices = reads más rápidos, writes más lentos.

  Regla 73: B+ Tree es el default porque soporta TODO.
    Equality, range, ORDER BY, MIN/MAX, BETWEEN, LIKE 'prefix%'.
    Hash solo soporta equality. Bitmap solo baja cardinalidad.
    → Si no sabes qué elegir: B+ Tree.

  Regla 74: Composite index sigue la regla del prefijo.
    INDEX(a, b, c) soporta WHERE a, WHERE a AND b, WHERE a AND b AND c.
    NO soporta WHERE b sin a. El orden de las columnas importa.
    → La columna más usada en WHERE va primero.

  Regla 75: Cada sistema elige sus índices según su workload.
    PostgreSQL: B+ Tree + BRIN + GIN + GiST.
    Cassandra: hash (partition) + LSM-Tree (clustering) + Bloom filter.
    Elasticsearch: inverted index + FST + BKD + doc values.
    → No hay un índice universal. La combinación depende del workload.

  Próximo y último capítulo:
    Cap.18: Proyecto final — construir un key-value store completo.
    → Integra: WAL + skip list + SSTable + Bloom filter + compaction.
```

**Preguntas:**

1. ¿75 reglas en 17 capítulos. ¿Quedan ~5 reglas para el proyecto final?

2. ¿La Regla 75 (cada sistema elige sus índices) es la conclusión del libro?

3. ¿Este capítulo unifica los Cap.04, 06, 09, 10, 11, 16?

4. ¿El proyecto final es el Cap.18. ¿Qué implementa exactamente?

5. ¿Después del Cap.18, ¿qué más necesita un data engineer sobre estructuras?
