# Estructuras de Datos desde las Tripas

**Guía práctica de estructuras de datos para ingenieros de datos y sistemas.**

No es un curso de algoritmos para entrevistas. Es un recorrido de 18 capítulos por las estructuras que están *adentro* de los sistemas que usas todos los días: el B+ Tree de PostgreSQL, el LSM-Tree de Cassandra, el Bloom filter de RocksDB, el skip list de Redis, el ring buffer de Kafka, el columnar layout de Parquet.

Cada capítulo sigue el mismo patrón: entender *por qué* existe la estructura, implementarla desde cero en código real (Python, Java, Go, Rust, Scala), medir su rendimiento con benchmarks, y conectarla con los sistemas de producción que la usan.

---

## Estructura del repositorio

```
estructuras-de-datos-desde-las-tripas/
│
├── README.md
│
├── parte-1-fundamentos/
│   ├── 01_memoria.md              # Memoria, caché, locality — por qué importa el hardware
│   ├── 02_complejidad.md          # Big-O con benchmarks reales, no solo teoría
│   └── 03_array.md                # Arrays, bitsets, sorted arrays — la base de todo
│
├── parte-2-hashing/
│   ├── 04_hash_tables.md          # Hash tables: open addressing, chaining, robin hood
│   ├── 05_hash_maps_avanzados.md  # ConcurrentHashMap, Swiss Table, consistent hashing intro
│   └── 06_bloom_filters.md        # Bloom filters, counting Bloom, HyperLogLog
│
├── parte-3-arboles/
│   ├── 07_arboles_busqueda.md     # BST, AVL, Red-Black, LLRB
│   ├── 08_heaps.md                # Binary heap, priority queue, K-way merge
│   ├── 09_btrees.md               # B-Tree, B+ Tree — cómo funcionan las bases de datos
│   └── 10_lsm_trees.md            # LSM-Tree — la alternativa write-optimized
│
├── parte-4-busqueda/
│   ├── 11_tries.md                # Tries, radix trees, suffix arrays, FST
│   └── 12_skip_lists.md           # Skip lists, concurrent skip lists, Redis ZSET
│
├── parte-5-concurrencia/
│   ├── 13_queues.md               # Ring buffer, blocking queue, work-stealing, Disruptor
│   ├── 14_thread_safe.md          # Mutex, striped locking, CAS, Rust ownership
│   └── 15_distributed.md          # Consistent hashing, Merkle trees, CRDTs
│
└── parte-6-datos-a-escala/
    ├── 16_columnar.md             # Row vs column, compresión, Parquet, Arrow
    ├── 17_indices.md              # Hash, B+ Tree, LSM, bitmap — unificación
    └── 18_kv_store.md             # Proyecto final: key-value store completo en Rust
```

---

## Qué hay en cada parte

### Parte 1 — Fundamentos (Cap. 01–03)

Lo que la mayoría de cursos omite: cómo la memoria, el caché, y el hardware determinan qué estructura es rápida y cuál no. Un array es rápido no porque sea O(1) sino porque es contiguo en memoria y el prefetcher lo adora. Un linked list es lento no porque sea O(n) sino porque cada nodo está en una dirección distinta y destruye la caché.

### Parte 2 — Hashing (Cap. 04–06)

Del hash map básico al `ConcurrentHashMap` de Java 8 (CAS, no striped locking), al `Swiss Table` de Google (SIMD para probing), al Bloom filter que permite a Cassandra evitar el 99% de las lecturas innecesarias, al HyperLogLog que cuenta 1 billón de elementos únicos en 12 KB.

### Parte 3 — Árboles (Cap. 07–10)

La progresión completa: BST → AVL → Red-Black → B-Tree → B+ Tree → LSM-Tree. Cada estructura resuelve un problema que la anterior no puede. El B+ Tree optimiza reads en disco. El LSM-Tree optimiza writes convirtiendo random writes en sequential writes (100× más throughput en HDD, 10× en SSD). Juntos explican por qué PostgreSQL y Cassandra existen y para qué sirve cada uno.

### Parte 4 — Búsqueda y Texto (Cap. 11–12)

Tries para búsqueda por prefijo (autocomplete de Google, HTTP routers de Go), radix trees para routing tables de Linux, suffix arrays para bioinformática, FST de Lucene/Elasticsearch. Skip lists: la estructura más simple que compite con árboles balanceados y gana en concurrencia — por eso Redis la eligió y por eso Java tiene `ConcurrentSkipListMap` pero no `ConcurrentTreeMap`.

### Parte 5 — Concurrencia y Distribución (Cap. 13–15)

Queues como tejido conectivo de sistemas concurrentes: ring buffer (Kafka), blocking queue (producer-consumer), work-stealing deque (ForkJoinPool de Spark, Go scheduler, Tokio). Patrones thread-safe: de mutex global a striped locking a CAS lock-free a Rust ownership. Estructuras distribuidas: consistent hashing (Cassandra, DynamoDB), Merkle trees (Git, anti-entropy repair), CRDTs (Figma, Redis Enterprise).

### Parte 6 — Datos a Escala (Cap. 16–18)

Row vs columnar: por qué Parquet lee 80× menos datos que PostgreSQL para un `AVG(salary)`. Compresión columnar: RLE, dictionary encoding, bit-packing, delta encoding. Arrow como formato in-memory universal. Unificación de todos los tipos de índice. Proyecto final: un key-value store completo en Rust que integra WAL + MemTable (skip list) + SSTable (sorted file) + Bloom filter + compaction — una versión simplificada de RocksDB.

---

## Números

| Métrica | Valor |
|---------|-------|
| Capítulos | 18 |
| Líneas de contenido | ~28,000 |
| Ejercicios | ~330 |
| Reglas acumuladas | 80 |
| Lenguajes | Python, Java, Go, Rust, Scala |
| Sistemas referenciados | PostgreSQL, Cassandra, RocksDB, Redis, Kafka, Spark, Flink, Elasticsearch, DuckDB, y ~30 más |

---

## Las 80 reglas — selección

Cada capítulo destila su contenido en reglas numeradas. Algunas:

- **Regla 1:** Un `int` no es un número. Es 4 bytes en una dirección de memoria.
- **Regla 15:** Un hash map O(1) puede ser más lento que un array O(n) si n < 50, porque el hash tiene overhead constante que domina.
- **Regla 22:** Un Bloom filter con 10 bits/elemento y 7 hashes tiene 0.8% de false positives. Sin Bloom filter, Cassandra leería todos los SSTables.
- **Regla 38:** B+ Tree: todas las keys en las hojas, hojas enlazadas. Range scan = encontrar la primera hoja, seguir los punteros. Por eso PostgreSQL usa B+ Tree.
- **Regla 44:** B-Tree para reads, LSM-Tree para writes. No hay estructura perfecta. La elección depende del workload.
- **Regla 51:** Skip list gana en concurrencia. Inserts locales + CAS. Por eso existe `ConcurrentSkipListMap` pero no `ConcurrentTreeMap`.
- **Regla 58:** Kafka es una queue persistente distribuida. Append-only log en disco. O(1) writes y reads.
- **Regla 70:** Parquet = columnar en disco. Arrow = columnar en memoria. Pipeline: Disco → Arrow → CPU.
- **Regla 80:** Las estructuras de datos son el lenguaje de los sistemas.

---

## Cómo usar este repositorio

**Cada capítulo es independiente pero acumulativo.** Los ejercicios se clasifican en tres tipos:

- **Leer:** Entender la estructura, su motivación, y sus tradeoffs.
- **Analizar:** Conectar la teoría con sistemas reales y comparar alternativas.
- **Implementar:** Código ejecutable con benchmarks. Copiar, correr, modificar.

**Recomendación:** seguir el orden. Cada capítulo usa estructuras de los anteriores. El proyecto final (Cap. 18) integra 8 de los 17 capítulos previos.

---

## Conexión con otros repositorios

Este es el cuarto repositorio de una serie sobre ingeniería de datos:

1. **Fundamentos de Ingeniería de Datos** — conceptos, arquitecturas, ciclo de vida del dato.
2. **SQL desde las Tripas** — SQL como lenguaje para pensar en datos, no solo para consultar.
3. **Java para Ingeniería de Datos** — Java aplicado a sistemas de datos (JVM, concurrencia, frameworks).
4. **Estructuras de Datos desde las Tripas** — este repositorio.

---

## Licencia

Material educativo de uso personal. El código de los ejercicios es libre para uso, modificación, y aprendizaje.
