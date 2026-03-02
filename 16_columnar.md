# Guía de Ejercicios — Cap.16: Columnar Layouts: Cómo se Almacenan los Datos Analíticos

> SELECT AVG(salary) FROM employees WHERE department = 'Engineering';
>
> En PostgreSQL (row-oriented): lee TODAS las columnas de TODAS las filas.
> Cada fila: id, name, email, department, salary, hire_date, address...
> Para calcular AVG(salary) solo necesitas 2 columnas: department y salary.
> Pero leíste 7 columnas. 5/7 del I/O fue desperdiciado.
>
> En un columnar store: lee SOLO las columnas salary y department.
> Cada columna es un array contiguo en disco.
> salary: [85000, 92000, 78000, 105000, ...]
> department: ["Eng", "Sales", "Eng", "Eng", ...]
>
> Para 1M filas × 7 columnas × 100 bytes/columna:
>   Row store: lee 700 MB.
>   Columnar store: lee 200 MB (2 columnas).
>   → 3.5× menos I/O.
>
> Pero hay más: los valores de una columna son del MISMO tipo.
> "Eng", "Eng", "Eng", "Sales", "Eng" → Run-Length Encoding: ("Eng", 3), ("Sales", 1), ("Eng", 1)
> Compresión 5:1 o más en columnas de baja cardinalidad.
>
> Parquet, ORC, Arrow: todos son columnar.
> Spark, Trino, BigQuery, Redshift, Snowflake, DuckDB: todos leen columnar.
> Este capítulo es el puente directo entre estructuras de datos y data engineering.

---

## El modelo mental: filas vs columnas

```
ROW-ORIENTED (PostgreSQL, MySQL, InnoDB):
  Fila 1: [id=1, name="Alice", dept="Eng",   salary=85000]
  Fila 2: [id=2, name="Bob",   dept="Sales", salary=92000]
  Fila 3: [id=3, name="Carol", dept="Eng",   salary=78000]

  En disco: [1,Alice,Eng,85000 | 2,Bob,Sales,92000 | 3,Carol,Eng,78000]
  → Todos los campos de una fila están CONTIGUOS.
  → Leer una fila completa: 1 I/O. Perfecto para OLTP.
  → Leer una columna de 1M filas: 1M I/Os (saltar entre filas).

COLUMN-ORIENTED (Parquet, ORC, Redshift, BigQuery):
  Columna id:     [1, 2, 3]
  Columna name:   ["Alice", "Bob", "Carol"]
  Columna dept:   ["Eng", "Sales", "Eng"]
  Columna salary: [85000, 92000, 78000]

  En disco: [1,2,3 | Alice,Bob,Carol | Eng,Sales,Eng | 85000,92000,78000]
  → Todos los valores de una columna están CONTIGUOS.
  → Leer una columna: sequential I/O. Perfecto para OLAP.
  → Leer una fila completa: N I/Os (una por columna).

  OLTP (transactions): INSERT/UPDATE/DELETE una fila → row store.
  OLAP (analytics):    SUM/AVG/COUNT una columna → columnar store.

  Compresión:
    Row store: [1,Alice,Eng,85000] — tipos mezclados. Difícil comprimir.
    Columnar: [Eng,Sales,Eng,Eng,Eng,Sales] — mismo tipo. Fácil comprimir.
    RLE: ("Eng",3),("Sales",1),("Eng",1) — 5 valores en 3 pares.
```

---

## Tabla de contenidos

- [Sección 16.1 — Row vs column: el tradeoff fundamental](#sección-161--row-vs-column-el-tradeoff-fundamental)
- [Sección 16.2 — Implementar un mini columnar store](#sección-162--implementar-un-mini-columnar-store)
- [Sección 16.3 — Técnicas de compresión columnar](#sección-163--técnicas-de-compresión-columnar)
- [Sección 16.4 — Parquet internals: row groups, pages, metadata](#sección-164--parquet-internals-row-groups-pages-metadata)
- [Sección 16.5 — Apache Arrow: columnar in-memory](#sección-165--apache-arrow-columnar-in-memory)
- [Sección 16.6 — Benchmark: columnar vs row para analytics](#sección-166--benchmark-columnar-vs-row-para-analytics)
- [Sección 16.7 — Columnar en producción: de Spark a DuckDB](#sección-167--columnar-en-producción-de-spark-a-duckdb)

---

## Sección 16.1 — Row vs Column: el Tradeoff Fundamental

### Ejercicio 16.1.1 — Analizar: por qué columnar gana para analytics

**Tipo: Analizar**

```
Escenario: tabla employees, 10M filas, 20 columnas, ~200 bytes/fila.
Total datos: 10M × 200 = 2 GB.

  Query: SELECT AVG(salary) FROM employees WHERE department = 'Engineering';

  ROW STORE (PostgreSQL):
    Leer: 2 GB (todas las filas, todas las columnas).
    Columnas usadas: 2 de 20 (salary, department).
    I/O desperdiciado: 90% (18 columnas innecesarias).

  COLUMNAR STORE (Parquet):
    Leer: columna department (~100 MB) + columna salary (~80 MB) = ~180 MB.
    I/O desperdiciado: 0%.
    → 11× menos I/O.

  Pero eso es sin compresión. Con compresión:
    department: 5 valores distintos. RLE + dictionary → ~5 MB.
    salary: delta encoding + bit-packing → ~20 MB.
    Total: ~25 MB. → 80× menos I/O que row store.

  ¿Y para OLTP?
    INSERT INTO employees VALUES (...)

    Row store: escribir 1 fila en 1 página. 1 I/O.
    Columnar: escribir 1 valor en CADA columna. 20 I/Os.
    → Row store gana 20× para inserts.

  Resumen:
    Operación        Row Store    Columnar     Ratio
    ─────────        ─────────    ────────     ─────
    Read 1 row       1 I/O        N I/Os       Row 20×
    Read 1 column    N I/Os       1 I/O        Col 20×
    Insert 1 row     1 I/O        N I/Os       Row 20×
    Analytics query   Todo         Solo cols    Col 10-100×
    Compression       1-2×         5-20×        Col 5-10×
```

**Preguntas:**

1. ¿80× menos I/O con compresión. ¿Es un caso típico?

2. ¿Columnar es terrible para INSERT. ¿Cómo maneja Spark los writes?

3. ¿DuckDB es columnar pero soporta OLTP. ¿Cómo?

4. ¿BigQuery columnar con 10 PB. ¿Cuánto ahorra la compresión?

5. ¿PostgreSQL tiene columnar extensions (citus, timescaledb). ¿Son populares?

---

## Sección 16.2 — Implementar un Mini Columnar Store

### Ejercicio 16.2.1 — Implementar: columnar vs row store en Python

**Tipo: Implementar**

```python
import time
import random
import struct
import array

class RowStore:
    """Row-oriented: each row is a dict"""
    def __init__(self):
        self.rows = []

    def insert(self, row: dict):
        self.rows.append(row)

    def scan_column(self, col: str):
        return [row[col] for row in self.rows]

    def aggregate_sum(self, col: str):
        return sum(row[col] for row in self.rows)

    def filter_and_agg(self, filter_col, filter_val, agg_col):
        total, count = 0, 0
        for row in self.rows:
            if row[filter_col] == filter_val:
                total += row[agg_col]
                count += 1
        return total / count if count > 0 else 0


class ColumnarStore:
    """Column-oriented: each column is a separate array"""
    def __init__(self, schema: list):
        self.schema = schema
        self.columns = {col: [] for col in schema}
        self.size = 0

    def insert(self, row: dict):
        for col in self.schema:
            self.columns[col].append(row[col])
        self.size += 1

    def scan_column(self, col: str):
        return self.columns[col]

    def aggregate_sum(self, col: str):
        return sum(self.columns[col])

    def filter_and_agg(self, filter_col, filter_val, agg_col):
        fc = self.columns[filter_col]
        ac = self.columns[agg_col]
        total, count = 0, 0
        for i in range(self.size):
            if fc[i] == filter_val:
                total += ac[i]
                count += 1
        return total / count if count > 0 else 0


# ═══ Benchmark ═══
n = 2_000_000
departments = ["Engineering", "Sales", "Marketing", "Finance", "HR"]
schema = ["id", "name", "department", "salary", "bonus", "years", "rating"]

row_store = RowStore()
col_store = ColumnarStore(schema)

for i in range(n):
    row = {
        "id": i,
        "name": f"emp_{i}",
        "department": random.choice(departments),
        "salary": random.randint(50000, 150000),
        "bonus": random.randint(1000, 20000),
        "years": random.randint(1, 30),
        "rating": random.randint(1, 5),
    }
    row_store.insert(row)
    col_store.insert(row)

# SUM(salary)
start = time.perf_counter_ns()
s1 = row_store.aggregate_sum("salary")
row_sum = time.perf_counter_ns() - start

start = time.perf_counter_ns()
s2 = col_store.aggregate_sum("salary")
col_sum = time.perf_counter_ns() - start

print(f"SUM(salary) over {n:,} rows:")
print(f"  Row store: {row_sum // 1_000_000} ms")
print(f"  Columnar:  {col_sum // 1_000_000} ms")
print(f"  Speedup:   {row_sum / col_sum:.1f}×")

# AVG(salary) WHERE department = 'Engineering'
start = time.perf_counter_ns()
a1 = row_store.filter_and_agg("department", "Engineering", "salary")
row_filter = time.perf_counter_ns() - start

start = time.perf_counter_ns()
a2 = col_store.filter_and_agg("department", "Engineering", "salary")
col_filter = time.perf_counter_ns() - start

print(f"\nAVG(salary) WHERE dept='Engineering':")
print(f"  Row store: {row_filter // 1_000_000} ms")
print(f"  Columnar:  {col_filter // 1_000_000} ms")
print(f"  Speedup:   {row_filter / col_filter:.1f}×")
```

**Preguntas:**

1. ¿Columnar es más rápido para SUM. ¿Porque accede solo 1 columna?

2. ¿En Python todo es objetos. ¿Con arrays nativos (numpy) la diferencia sería mayor?

3. ¿El filter_and_agg accede 2 columnas. ¿Con 20 columnas la ventaja sería mayor?

4. ¿Para INSERT, ¿row store es más rápido?

5. ¿Arrow y Parquet usan arrays contiguos de bytes, no listas de Python?

---

## Sección 16.3 — Técnicas de Compresión Columnar

### Ejercicio 16.3.1 — Implementar: RLE, dictionary encoding, bit-packing

**Tipo: Implementar**

```python
import struct

# ═══ RUN-LENGTH ENCODING ═══
def rle_encode(values):
    """Encode as (value, count) pairs"""
    if not values: return []
    runs = []
    current, count = values[0], 1
    for v in values[1:]:
        if v == current:
            count += 1
        else:
            runs.append((current, count))
            current, count = v, 1
    runs.append((current, count))
    return runs

def rle_decode(runs):
    return [v for v, c in runs for _ in range(c)]

# ═══ DICTIONARY ENCODING ═══
def dict_encode(values):
    """Replace values with integer indices"""
    dictionary = {}
    indices = []
    for v in values:
        if v not in dictionary:
            dictionary[v] = len(dictionary)
        indices.append(dictionary[v])
    return dictionary, indices

def dict_decode(dictionary, indices):
    reverse = {v: k for k, v in dictionary.items()}
    return [reverse[i] for i in indices]

# ═══ BIT-PACKING ═══
def bit_pack(values, bits_per_value):
    """Pack integers using minimum bits"""
    packed = bytearray()
    buffer, buffer_bits = 0, 0
    for v in values:
        buffer |= (v << buffer_bits)
        buffer_bits += bits_per_value
        while buffer_bits >= 8:
            packed.append(buffer & 0xFF)
            buffer >>= 8
            buffer_bits -= 8
    if buffer_bits > 0:
        packed.append(buffer & 0xFF)
    return bytes(packed)

def bit_unpack(packed, bits_per_value, count):
    values = []
    buffer, buffer_bits, byte_idx = 0, 0, 0
    mask = (1 << bits_per_value) - 1
    for _ in range(count):
        while buffer_bits < bits_per_value:
            buffer |= (packed[byte_idx] << buffer_bits)
            buffer_bits += 8
            byte_idx += 1
        values.append(buffer & mask)
        buffer >>= bits_per_value
        buffer_bits -= bits_per_value
    return values

# ═══ DELTA ENCODING ═══
def delta_encode(values):
    if not values: return []
    deltas = [values[0]]
    for i in range(1, len(values)):
        deltas.append(values[i] - values[i-1])
    return deltas

def delta_decode(deltas):
    values = [deltas[0]]
    for d in deltas[1:]:
        values.append(values[-1] + d)
    return values


# ═══ Demo ═══
import random

n = 1_000_000

# RLE: department column (low cardinality, sorted)
depts = sorted(random.choices(["Eng", "Sales", "Mkt", "Fin", "HR"], k=n))
rle = rle_encode(depts)
original_size = n * 8  # ~8 bytes per string avg
rle_size = len(rle) * 12  # ~12 bytes per run
print(f"RLE (sorted departments, {n:,} values):")
print(f"  Original: {original_size:,} bytes, RLE: {rle_size:,} bytes")
print(f"  Ratio: {original_size / rle_size:.0f}:1 ({len(rle)} runs)")

# Dictionary: department column (not sorted)
depts_unsorted = random.choices(["Eng", "Sales", "Mkt", "Fin", "HR"], k=n)
dictionary, indices = dict_encode(depts_unsorted)
dict_size = sum(len(k) for k in dictionary) + len(indices) * 1  # 1 byte per index
print(f"\nDictionary encoding ({n:,} values, {len(dictionary)} unique):")
print(f"  Original: {original_size:,} bytes, Encoded: {dict_size:,} bytes")
print(f"  Ratio: {original_size / dict_size:.1f}:1")

# Bit-packing: indices only need 3 bits (5 values → ceil(log2(5)) = 3)
packed = bit_pack(indices, 3)
print(f"\nBit-packing (3 bits per value):")
print(f"  Indices: {n * 4:,} bytes (32-bit ints)")
print(f"  Packed:  {len(packed):,} bytes")
print(f"  Ratio: {n * 4 / len(packed):.1f}:1")

# Dictionary + bit-packing combined
combined_size = sum(len(k) for k in dictionary) + len(packed)
print(f"\nDictionary + bit-packing combined:")
print(f"  Original: {original_size:,} bytes, Compressed: {combined_size:,} bytes")
print(f"  Ratio: {original_size / combined_size:.0f}:1")

# Delta: timestamp column (monotonically increasing)
timestamps = sorted(random.sample(range(1_700_000_000, 1_700_100_000), min(n, 100_000)))
deltas = delta_encode(timestamps)
max_delta = max(abs(d) for d in deltas[1:])
bits_needed = max_delta.bit_length()
print(f"\nDelta encoding ({len(timestamps):,} timestamps):")
print(f"  Max delta: {max_delta} ({bits_needed} bits)")
print(f"  Original: {len(timestamps) * 8:,} bytes (64-bit)")
print(f"  Delta + bit-pack ({bits_needed} bits): {len(timestamps) * bits_needed // 8:,} bytes")
print(f"  Ratio: {64 / bits_needed:.1f}:1")
```

**Preguntas:**

1. ¿RLE con datos sorted da 166K:1 compresión. ¿Parquet ordena las columnas?

2. ¿Dictionary + bit-packing combinados: ¿es lo que Parquet hace?

3. ¿Delta encoding para timestamps: ¿los deltas son más pequeños?

4. ¿Bit-packing 3 bits vs 32 bits: 10× compresión. ¿Hardware support?

5. ¿Para columnas de alta cardinalidad (emails), ¿qué compresión funciona?

---

## Sección 16.4 — Parquet Internals: Row Groups, Pages, Metadata

### Ejercicio 16.4.1 — Leer: estructura interna de un archivo Parquet

**Tipo: Leer**

```
Un archivo Parquet:

  ┌─────────────────────────────────────────────┐
  │ Magic: "PAR1" (4 bytes)                     │
  ├─────────────────────────────────────────────┤
  │ Row Group 1                                  │
  │   Column Chunk: id      [pages...]          │
  │   Column Chunk: name    [pages...]          │
  │   Column Chunk: salary  [pages...]          │
  ├─────────────────────────────────────────────┤
  │ Row Group 2                                  │
  │   Column Chunk: id      [pages...]          │
  │   Column Chunk: name    [pages...]          │
  │   Column Chunk: salary  [pages...]          │
  ├─────────────────────────────────────────────┤
  │ Footer                                       │
  │   Schema, row group metadata, column stats   │
  │   (min/max per column chunk for predicate    │
  │    pushdown)                                 │
  ├─────────────────────────────────────────────┤
  │ Footer length (4 bytes)                     │
  │ Magic: "PAR1" (4 bytes)                     │
  └─────────────────────────────────────────────┘

  ROW GROUP: un subconjunto de filas (típicamente 128 MB).
    → Unidad de paralelización. Spark lee 1 row group por task.

  COLUMN CHUNK: todos los valores de una columna en un row group.
    → Unidad de I/O. Leer solo las columnas que necesitas.

  PAGE: subdivisión de un column chunk (~1 MB).
    → Unidad de compresión y encoding.
    Tipos: data page, dictionary page, index page.

  FOOTER: metadata al final del archivo.
    → Contiene el schema, y para cada column chunk:
      min value, max value, null count, distinct count.
    → Predicate pushdown: si WHERE salary > 100000 y el max
      de un column chunk es 80000 → skip todo el chunk.
    → Spark lee el footer PRIMERO para planificar qué leer.

  Encoding por tipo de dato:
    Strings:       DICTIONARY + RLE.
    Integers:      DELTA + BIT-PACKING.
    Timestamps:    DELTA + BIT-PACKING.
    Booleans:      RLE (runs de true/false).
    Floats:        PLAIN (sin encoding especial).
```

**Preguntas:**

1. ¿Row group de 128 MB — ¿configurable en Spark?

2. ¿Footer al final del archivo — ¿por qué no al inicio?

3. ¿Predicate pushdown con min/max — ¿es como el Bloom filter?

4. ¿Spark lee 1 row group por task. ¿Cuántos tasks para 1 TB?

5. ¿Parquet vs ORC — ¿cuál tiene mejor compresión?

---

## Sección 16.5 — Apache Arrow: Columnar In-Memory

### Ejercicio 16.5.1 — Analizar: Parquet en disco, Arrow en memoria

**Tipo: Analizar**

```
Parquet: formato columnar para DISCO. Optimizado para I/O y compresión.
Arrow: formato columnar para MEMORIA. Optimizado para CPU y zero-copy.

  Parquet:
    - Datos comprimidos (RLE, dictionary, delta).
    - Para leer: descomprimir → procesar.
    - Optimizado para minimizar I/O.
    - Formato de archivo (serialización).

  Arrow:
    - Datos NO comprimidos. Arrays contiguos en memoria.
    - Para leer: acceso directo. Sin deserialización.
    - Optimizado para operaciones vectorizadas (SIMD).
    - Formato de memoria (in-process).

  Pipeline típico:
    Disco (Parquet) → Deserialize → Memoria (Arrow) → CPU → Resultado.

    Sin Arrow: cada sistema tiene su propio formato en memoria.
    Spark lee Parquet → formato Spark interno.
    Pandas lee Parquet → formato Pandas interno.
    DuckDB lee Parquet → formato DuckDB interno.
    → Conversiones costosas entre sistemas.

    Con Arrow: formato UNIVERSAL en memoria.
    Spark lee Parquet → Arrow.
    Pandas lee Parquet → Arrow (PyArrow).
    DuckDB lee Parquet → Arrow.
    → Zero-copy entre sistemas. Sin conversión.

  DuckDB + Arrow:
    DuckDB opera directamente sobre Arrow buffers.
    Pandas DataFrames backed by Arrow → DuckDB query sin copia.
    → SQL sobre un DataFrame sin mover datos.

  Spark + Arrow:
    PySpark usa Arrow para transferir datos entre JVM y Python.
    Sin Arrow: serializar → transferir → deserializar (lento).
    Con Arrow: shared memory → zero-copy (rápido).
```

**Preguntas:**

1. ¿Arrow no comprime datos. ¿Usa más memoria que Parquet?

2. ¿Zero-copy entre Pandas y DuckDB con Arrow. ¿Cómo funciona?

3. ¿PySpark sin Arrow vs con Arrow — ¿cuánta diferencia?

4. ¿Arrow Flight — ¿es Arrow sobre la red?

5. ¿Polars (alternativa a Pandas) usa Arrow internamente?

---

## Sección 16.6 — Benchmark: Columnar vs Row para Analytics

### Ejercicio 16.6.1 — Implementar: benchmark con compresión

**Tipo: Implementar**

```python
import time, random, struct, array

n = 5_000_000
departments = ["Eng", "Sales", "Mkt", "Fin", "HR", "Legal", "Ops", "Support"]

# Generate data
ids = list(range(n))
depts = [random.choice(departments) for _ in range(n)]
salaries = [random.randint(40000, 200000) for _ in range(n)]
bonuses = [random.randint(0, 50000) for _ in range(n)]
years = [random.randint(1, 40) for _ in range(n)]
ratings = [random.randint(1, 5) for _ in range(n)]

# Row store: list of tuples
rows = list(zip(ids, depts, salaries, bonuses, years, ratings))

# Columnar: separate arrays
col_salaries = array.array('i', salaries)
col_bonuses = array.array('i', bonuses)
col_years = array.array('i', years)

print(f"Dataset: {n:,} rows × 6 columns\n")

# Q1: SUM(salary)
start = time.perf_counter_ns()
s1 = sum(r[2] for r in rows)
row_t = time.perf_counter_ns() - start

start = time.perf_counter_ns()
s2 = sum(col_salaries)
col_t = time.perf_counter_ns() - start

print(f"Q1: SUM(salary)")
print(f"  Row:      {row_t // 1_000_000} ms")
print(f"  Columnar: {col_t // 1_000_000} ms  ({row_t / col_t:.1f}× faster)")

# Q2: AVG(salary) WHERE department = 'Eng'
start = time.perf_counter_ns()
total, cnt = 0, 0
for r in rows:
    if r[1] == "Eng": total += r[2]; cnt += 1
avg1 = total / cnt
row_t = time.perf_counter_ns() - start

start = time.perf_counter_ns()
total, cnt = 0, 0
for i in range(n):
    if depts[i] == "Eng": total += col_salaries[i]; cnt += 1
avg2 = total / cnt
col_t = time.perf_counter_ns() - start

print(f"\nQ2: AVG(salary) WHERE dept='Eng'")
print(f"  Row:      {row_t // 1_000_000} ms")
print(f"  Columnar: {col_t // 1_000_000} ms  ({row_t / col_t:.1f}× faster)")

# Q3: COUNT(*) GROUP BY department
start = time.perf_counter_ns()
counts1 = {}
for r in rows:
    counts1[r[1]] = counts1.get(r[1], 0) + 1
row_t = time.perf_counter_ns() - start

start = time.perf_counter_ns()
counts2 = {}
for d in depts:
    counts2[d] = counts2.get(d, 0) + 1
col_t = time.perf_counter_ns() - start

print(f"\nQ3: COUNT(*) GROUP BY department")
print(f"  Row:      {row_t // 1_000_000} ms")
print(f"  Columnar: {col_t // 1_000_000} ms  ({row_t / col_t:.1f}× faster)")

# Memory comparison
import sys
row_mem = sys.getsizeof(rows) + sum(sys.getsizeof(r) for r in rows[:1000]) * n // 1000
col_mem = col_salaries.buffer_info()[1] * len(col_salaries) * 3 + sys.getsizeof(depts)
print(f"\nMemory (estimate):")
print(f"  Row store: ~{row_mem // (1024*1024)} MB")
print(f"  Columnar:  ~{col_mem // (1024*1024)} MB (numeric only)")
```

**Preguntas:**

1. ¿Columnar es 1.5-3× más rápido en Python puro. ¿Con Arrow/numpy?

2. ¿Q3 (GROUP BY) — ¿columnar debería ganar más porque accede solo 1 columna?

3. ¿Memory: columnar con arrays tipados ocupa menos. ¿Por qué?

4. ¿Con 100 columnas y query que usa 2, ¿la ventaja sería 50×?

5. ¿DuckDB ejecutando SQL sobre DataFrames — ¿usa estas optimizaciones?

---

## Sección 16.7 — Columnar en Producción: de Spark a DuckDB

### Ejercicio 16.7.1 — Analizar: el ecosistema columnar

**Tipo: Analizar**

```
FORMATOS DE ARCHIVO:
  Parquet:  Hadoop ecosystem. Spark, Trino, BigQuery, Snowflake.
            Row groups + column chunks + footer. Snappy/Zstd compression.
  ORC:      Hive ecosystem. Optimized Row Columnar.
            Stripes + index + footer. Better for Hive, comparable to Parquet.
  Arrow IPC: Arrow's file format. For interchange, not long-term storage.

MOTORES DE QUERY:
  Spark:      Lee Parquet/ORC. Procesa con Tungsten (binary format).
              Pushdown predicates + column pruning → lee solo lo necesario.
  Trino:      Lee Parquet/ORC de S3/HDFS. Columnar scan engine.
  DuckDB:     Lee Parquet directamente. In-process OLAP.
              Vectorized execution sobre columnas.
  BigQuery:   Almacena en Capacitor (formato columnar propio).
              Dremel paper (2010) → inspiró Parquet.
  Snowflake:  Micro-partitions columnar. Auto-clustering.
  Redshift:   Columnar con zone maps (min/max per block).
  ClickHouse: Columnar con merge tree. Ultra-rápido para analytics.

FORMATOS EN MEMORIA:
  Arrow:      Standard columnar in-memory. Pandas 2.0, Polars, DuckDB.
  Tungsten:   Spark's binary format. Off-heap, manual memory management.

  La tendencia: TODO se está moviendo a columnar para analytics.
  Parquet es el formato de archivo estándar.
  Arrow es el formato en memoria estándar.
  Juntos cubren disco → memoria → CPU.
```

**Preguntas:**

1. ¿Parquet es el estándar de facto. ¿ORC tiene alguna ventaja?

2. ¿BigQuery Capacitor vs Parquet — ¿por qué su propio formato?

3. ¿ClickHouse merge tree — ¿es un LSM-Tree columnar?

4. ¿Polars usa Arrow. ¿Es más rápido que Pandas?

5. ¿Para un data engineer, ¿cuándo NO usar Parquet?

---

### Ejercicio 16.7.2 — Resumen: las reglas del almacenamiento columnar

**Tipo: Leer**

```
Reglas nuevas del Cap.16:

  Regla 68: Row store para OLTP, columnar para OLAP.
    OLTP (1 fila): row store lee todo junto. 1 I/O.
    OLAP (1 columna de 1M filas): columnar lee solo esa columna.
    Analytics sobre row store: lee 20 columnas cuando necesitas 2.
    → 10× menos I/O con columnar para queries analíticos.

  Regla 69: La compresión columnar es dramáticamente mejor.
    Mismos tipos contiguos → RLE, dictionary, delta, bit-packing.
    Columna de 5 departamentos: dictionary + bit-packing → 20:1.
    Timestamps ordenados: delta + bit-packing → 8:1.
    Row store mezcla tipos → 2:1 máximo.

  Regla 70: Parquet = columnar en disco. Arrow = columnar en memoria.
    Parquet: row groups + column chunks + footer con stats.
    Arrow: arrays contiguos sin compresión. Zero-copy entre sistemas.
    Pipeline: Disco (Parquet) → Memoria (Arrow) → CPU.

  Regla 71: Predicate pushdown + column pruning = leer solo lo necesario.
    Column pruning: si el query usa 2 de 20 columnas → lee solo 2.
    Predicate pushdown: si max(salary) en un chunk < 100K y
    WHERE salary > 100K → skip el chunk entero.
    Spark, Trino, DuckDB: todos hacen esto automáticamente.

  La Parte 6 continúa:
    Cap.17: Índices — unificar B+Tree, hash, LSM, bitmap.
    Cap.18: Proyecto final — key-value store completo.
```

**Preguntas:**

1. ¿71 reglas en 16 capítulos. ¿Quedan ~9 reglas para 2 capítulos?

2. ¿La Regla 71 (predicate pushdown) es lo que hace Spark automáticamente?

3. ¿Este capítulo es el más directamente relevante para data engineering?

4. ¿El Cap.17 unifica cuántos capítulos anteriores?

5. ¿El proyecto final necesita columnar o solo key-value?
