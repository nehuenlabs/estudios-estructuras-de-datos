# Estructuras de Datos Desde las Tripas
# Temario Completo

> No es un catálogo de "qué métodos tiene un TreeMap".
> Es un recorrido donde implementas cada estructura desde cero,
> entiendes por qué existe, mides cuándo supera a las alternativas,
> y conectas cada implementación con el sistema real que la usa en producción.

---

## Stack de lenguajes

| Lenguaje | Rol en el repositorio |
|----------|----------------------|
| Python   | Pseudocódigo ejecutable. Prototipo rápido. Entender la lógica sin ruido sintáctico. |
| Java     | Implementación de referencia. Misma plataforma que Kafka, Spark, Cassandra. JVM internals. |
| Scala    | Enfoque funcional. Inmutabilidad, pattern matching, ADTs. Estructuras persistentes. |
| Go       | Minimalismo. Concurrencia nativa. Estructuras concurrent-safe con goroutines y channels. |
| Rust     | Ownership. Zero-cost abstractions. Control total de memoria. Sin garbage collector. |

No todos los capítulos usan los 5 lenguajes.
Cada capítulo usa los lenguajes donde la comparación ilumina algo.

---

## Parte 1 — Fundamentos: Memoria, Costos, y las Reglas del Juego

### Capítulo 01 — Cómo Vive un Dato en Memoria

La base de todo: sin entender cómo la memoria física afecta el rendimiento
de las estructuras de datos, todo lo demás es abstracto.

- El modelo de memoria: stack vs heap
- Localidad de referencia: spatial y temporal locality
- Cache lines (64 bytes), cache misses, y por qué importan más que la complejidad O(n)
- Alineación de datos y padding en structs
- Cómo Java, Go, y Rust organizan objetos en memoria de forma distinta
  - Java: object headers (12-16 bytes), pointer indirection, GC roots
  - Go: structs por valor vs por referencia, escape analysis
  - Rust: ownership, stack allocation por defecto, zero-cost moves
- Benchmark: recorrer un array vs recorrer una linked list (misma O(n), rendimiento 10-50× distinto)
- Ejercicio: medir cache misses con `perf stat` (Linux) y explicar los resultados

**Conexión:** Esto explica por qué un array beats a una linked list en la práctica
a pesar de que la teoría dice que insertar en una linked list es O(1).

---

### Capítulo 02 — Complejidad: Más Allá de Big-O

Big-O es necesario pero insuficiente. Este capítulo enseña a medir de verdad.

- Big-O, Big-Ω, Big-Θ: qué dice cada uno y qué NO dice
- Por qué O(n) puede ser más rápido que O(1) para n < 10,000
  (constantes ocultas, cache effects, branch prediction)
- Complejidad amortizada: por qué ArrayList.add() es O(1) amortizado
  y cómo se demuestra con el método del banquero
- Complejidad en espacio: cuánta memoria usa la estructura, no solo cuánto tarda
- Benchmarking correcto: JMH (Java), criterion (Rust), pytest-benchmark (Python)
  - Warmup, iteraciones, percentiles, GC noise
- Ejercicio: implementar un dynamic array y demostrar empíricamente
  que la estrategia de duplicar capacidad es O(1) amortizado
- Ejercicio: comparar el rendimiento real de 5 operaciones en ArrayList vs LinkedList
  y explicar por qué los resultados contradicen la tabla de Big-O del textbook

**Conexión:** Después de este capítulo, el lector deja de confiar ciegamente
en las tablas de complejidad y empieza a medir.

---

### Capítulo 03 — El Array: la Estructura que Nadie Respeta

El array es la base de todo. Hash maps, heaps, B-Trees — todos usan arrays internamente.

- Array estático: memoria contigua, acceso O(1), el rey de la cache locality
- Dynamic array (ArrayList, Vec, slice):
  - Growth strategy: duplicar vs factor 1.5 vs incremento fijo
  - Implementar desde cero en Python, Java, y Rust
  - Shrinking: ¿cuándo reducir la capacidad? (hysteresis)
- Ring buffer (circular buffer):
  - Implementar con un array y dos punteros (head, tail)
  - Uso real: buffers de I/O, consumer queues en Kafka, bounded channels en Go
- Bit arrays y bitsets:
  - Representar conjuntos con bits: 1 byte = 8 elementos
  - Operaciones de conjunto como operaciones bitwise (AND, OR, XOR)
  - Uso real: Bloom filters (próximo capítulo), bitmap indexes en columnar stores
- Ejercicio: implementar un dynamic array que soporte generics en Java, Scala, y Rust
- Ejercicio: implementar un ring buffer thread-safe en Go con goroutines

**Conexión:** El ring buffer aparece en Kafka (log segments) y en LMAX Disruptor.
El bitset es la base de los Bloom filters y de los bitmap indexes de Parquet.

---

## Parte 2 — Hashing: la Estructura Más Importante de la Informática

### Capítulo 04 — Hash Tables: de la Función Hash a la Implementación

Más del 50% de los accesos a datos en producción pasan por un hash table.

- ¿Qué es un hash? Función determinística que mapea datos a un índice
- Propiedades de una buena función hash: distribución uniforme, avalanche effect
- Funciones hash comunes: Java hashCode(), Murmur3, xxHash, SipHash (Rust default)
  - Por qué cada lenguaje elige una función distinta (seguridad vs velocidad)
- Colisiones: inevitables (birthday paradox)
- Resolución de colisiones — separate chaining:
  - Cada bucket es una linked list (o un árbol si la lista crece — Java 8+ TreeBins)
  - Implementar desde cero en Python y Java
- Resolución de colisiones — open addressing:
  - Linear probing: simple pero sufre de clustering
  - Quadratic probing: reduce clustering primario
  - Robin Hood hashing: reduce la varianza de lookup redistribuyendo probes
  - Implementar Robin Hood hashing en Rust
- Load factor y rehashing: cuándo crecer la tabla y cómo migrar
- Ejercicio: implementar un hash map completo con separate chaining en Java
- Ejercicio: implementar Robin Hood hashing en Rust y comparar varianza vs linear probing
- Ejercicio: benchmark de Java HashMap vs Rust HashMap vs Go map vs Python dict

**Conexión:** Java HashMap es el corazón de Spark shuffles (hash partitioning).
Robin Hood hashing se usa en la SwissTable de Rust y en Abseil (C++ de Google).

---

### Capítulo 05 — Hash Maps Avanzados: Concurrent, Ordered, Persistent

Variantes que resuelven problemas que el hash map básico no puede.

- ConcurrentHashMap (Java):
  - Segmented locking (Java 7) vs CAS + TreeBins (Java 8+)
  - Por qué no puedes simplemente poner un lock en un HashMap
  - Implementar una versión simplificada con striped locking
- Lock-free hash maps:
  - Compare-and-swap (CAS) como primitiva
  - Cliff Click's NonBlockingHashMap (concepto)
- Ordered hash maps:
  - LinkedHashMap (Java): insertion order via doubly-linked list
  - TreeMap (Java): sorted by key via Red-Black Tree (¿es realmente un hash map? No — es un detour al Cap.09)
- Persistent hash maps (Scala, Clojure):
  - Hash Array Mapped Trie (HAMT): la estructura detrás de Scala's immutable Map
  - Path copying: cómo modificar sin mutar, compartiendo estructura
  - Por qué la inmutabilidad es viable con structural sharing
  - Implementar un HAMT simplificado en Scala
- Ejercicio: benchmark ConcurrentHashMap vs synchronized HashMap en Java bajo contención
- Ejercicio: implementar un HAMT simplificado en Scala con path copying

**Conexión:** ConcurrentHashMap es el backbone de Spark's in-memory aggregation.
HAMT es la estructura detrás de las colecciones inmutables de Scala y Clojure.

---

### Capítulo 06 — Bloom Filters y Estructuras Probabilísticas

Cuando "probablemente sí, definitivamente no" es suficiente.

- El problema: ¿existe este elemento en un conjunto de mil millones de elementos?
  - Hash set: respuesta exacta, pero 8+ GB de memoria
  - Bloom filter: respuesta probabilística, pero 1 GB de memoria
- Cómo funciona: k funciones hash, m bits, false positive rate
  - Fórmula: probabilidad de false positive = (1 - e^(-kn/m))^k
  - Tuning: cómo elegir m y k dado n y un target de FP rate
- Implementar desde cero en Python y Java
- Counting Bloom filter: soporta deletes (counters en vez de bits)
- Cuckoo filter: alternativa con soporte de delete y mejor space efficiency
- HyperLogLog: estimar cardinalidad (COUNT DISTINCT) con memoria O(log log n)
  - Uso real: Redis PFCOUNT, BigQuery APPROX_COUNT_DISTINCT
- Ejercicio: implementar un Bloom filter y medir false positive rate empíricamente
- Ejercicio: implementar HyperLogLog y comparar precisión vs memoria
- Ejercicio: Bloom filter en Rust con zero allocations

**Conexión:** Cassandra usa Bloom filters para evitar lecturas a disco innecesarias.
Parquet usa Bloom filters para column-level predicate pushdown.
HyperLogLog es la estructura detrás de COUNT DISTINCT aproximado en BigQuery y Redis.

---

## Parte 3 — Árboles: Jerarquía, Búsqueda, y Almacenamiento

### Capítulo 07 — Árboles Binarios de Búsqueda: de BST a AVL a Red-Black

La familia de árboles que mantiene datos ordenados.

- BST básico: insert, search, delete, inorder traversal
  - Implementar en Python (claridad) y Java (referencia)
  - El problema: sin balanceo, el worst case es una linked list (O(n))
- AVL Trees: el primer árbol auto-balanceado
  - Factor de balance, rotaciones (simple y doble)
  - Implementar con rotaciones en Java
  - Garantía: altura siempre O(log n)
- Red-Black Trees:
  - Propiedades (5 reglas), coloreo, rotaciones, fix-up
  - Por qué Red-Black es menos estricto que AVL (menos rotaciones, mejor para insert-heavy)
  - Implementar insert + fix-up en Java
  - Uso real: Java TreeMap, Linux CFS scheduler, C++ std::map
- Left-Leaning Red-Black Trees (LLRB):
  - Simplificación de Sedgewick: la mitad de casos que un Red-Black estándar
  - Implementar en Scala con pattern matching (elegancia del enfoque funcional)
- Ejercicio: benchmark BST vs AVL vs Red-Black para n=1M inserts aleatorios vs ordenados
- Ejercicio: implementar un LLRB en Scala con insert y delete

**Conexión:** TreeMap de Java (Red-Black) se usa en Kafka para los offset indexes.
CFS scheduler de Linux usa un Red-Black tree para scheduling de procesos.

---

### Capítulo 08 — Heaps y Priority Queues: Orden Parcial

Cuando no necesitas orden total — solo saber cuál es el mínimo/máximo.

- Binary heap: el array como árbol implícito
  - Heap property, sift-up, sift-down
  - Implementar min-heap desde cero en Python, Java, y Go
  - Heapify: construir un heap en O(n) (no O(n log n))
- Priority queue:
  - La abstracción que usa el heap internamente
  - Java PriorityQueue, Python heapq
- D-ary heaps: ¿2 hijos o 4 hijos? Tradeoff entre profundidad y comparaciones
- Fibonacci heap: amortized O(1) decrease-key
  - Concepto y por qué importa para Dijkstra
  - Por qué casi nadie lo implementa en producción (constantes altas, complejidad)
- Indexed priority queue: soporta decrease-key eficientemente sin Fibonacci
  - Implementar en Java
- Ejercicio: implementar un min-heap y un max-heap genéricos en Rust
- Ejercicio: implementar un event scheduler (timer wheel) con un heap en Go
- Ejercicio: benchmark binary heap vs d-ary heap para diferentes valores de d

**Conexión:** Los heaps son la estructura detrás de los schedulers (Airflow task scheduling),
merge sort externo (Spark shuffle merge), y Kafka's DelayedOperationPurgatory.

---

### Capítulo 09 — B-Trees: la Estructura que Domina el Almacenamiento

La estructura más importante de las bases de datos.

- ¿Por qué no usar un Red-Black tree para discos?
  - Un nodo = un acceso a disco. Red-Black tiene altura O(log₂ n). B-Tree tiene altura O(log_B n).
  - Con B=1000, un B-Tree de 1 billón de claves tiene altura 3.
  - Menos accesos a disco = más rápido. Punto.
- B-Tree:
  - Propiedades: nodo con M hijos, claves ordenadas, hojas al mismo nivel
  - Insert con split, delete con merge/redistribution
  - Implementar en Java (versión simplificada)
- B+ Tree:
  - Todos los datos en las hojas. Nodos internos son solo índices.
  - Hojas enlazadas para range scans eficientes
  - Implementar en Java y Rust
  - Uso real: InnoDB (MySQL), PostgreSQL indexes, SQLite, NTFS/ext4
- B-Tree en la práctica:
  - Page size y su relación con el branching factor
  - Write amplification: cuántas escrituras a disco por una inserción lógica
  - Copy-on-write B-Trees: inmutabilidad para concurrencia (LMDB, boltdb)
- Ejercicio: implementar un B+ Tree con insert, search, y range scan en Java
- Ejercicio: implementar un copy-on-write B-Tree simplificado en Rust
- Ejercicio: medir la altura de un B+ Tree con distintos branching factors

**Conexión:** Cada SELECT con WHERE en PostgreSQL/MySQL atraviesa un B+ Tree.
BoltDB (Go, usado en etcd) usa copy-on-write B+ Trees.
Los indexes de Parquet usan estructuras similares para min/max filtering.

---

### Capítulo 10 — LSM-Trees: la Alternativa Write-Optimized

La estructura que domina el almacenamiento moderno de write-heavy workloads.

- El tradeoff fundamental: B-Tree optimiza reads, LSM-Tree optimiza writes
- Arquitectura de un LSM-Tree:
  - MemTable (in-memory, sorted): usualmente un Red-Black tree o skip list
  - Flush a disco como SSTable (Sorted String Table): archivos inmutables
  - Compaction: merge de SSTables (tiered vs leveled compaction)
- Write path: write → WAL → MemTable → flush → SSTable
- Read path: MemTable → Bloom filter → SSTable lookup (de más reciente a más antiguo)
- Implementar un LSM-Tree simplificado:
  - MemTable con TreeMap (Java)
  - Flush a disco como archivo sorted
  - Read con merge de resultados
- Compaction strategies:
  - Size-tiered (Cassandra default): simple, más space amplification
  - Leveled (RocksDB default): menos space amplification, más write amplification
- Write amplification, read amplification, space amplification: el trilemma
- Ejercicio: implementar MemTable + SSTable flush + read merge en Java
- Ejercicio: implementar Bloom filter para skip de SSTables (integrar Cap.06)
- Ejercicio: benchmark write throughput de B+ Tree vs LSM-Tree para 1M inserts

**Conexión:** Cassandra, RocksDB, LevelDB, HBase, ScyllaDB — todos son LSM-Trees.
Flink usa RocksDB (LSM-Tree) como state backend.
El WAL de un LSM-Tree es conceptualmente idéntico al commit log de Kafka.

---

## Parte 4 — Estructuras para Búsqueda y Texto

### Capítulo 11 — Tries y Estructuras para Strings

Cuando las claves son strings, los hash maps no son la única opción.

- Trie (prefix tree):
  - Cada nodo es un carácter, cada camino root→leaf es una clave
  - Insert, search, prefix search
  - Implementar en Python y Java
- Compressed trie (radix tree / Patricia tree):
  - Colapsar nodos con un solo hijo: reduce memoria significativamente
  - Implementar en Java
  - Uso real: routing tables de Linux (LPC trie), HTTP routers (Go chi, httprouter)
- Ternary search tree:
  - Alternativa al trie que usa menos memoria con alfabetos grandes
- Suffix arrays y suffix trees:
  - Buscar cualquier substring en O(m log n) con un suffix array
  - Concepto de suffix tree para pattern matching
  - Uso real: búsqueda full-text, bioinformática
- Ejercicio: implementar un trie con insert, search, y autocomplete (prefix search)
- Ejercicio: implementar un radix tree en Go (como un HTTP router simplificado)
- Ejercicio: benchmark trie vs hash map para prefix search con 1M strings

**Conexión:** Los HTTP routers de Go (chi, gin) usan radix trees.
Apache Lucene (Elasticsearch) usa finite state transducers (evolución del trie).

---

### Capítulo 12 — Skip Lists: Simplicidad que Compite con Árboles

La estructura probabilística que reemplaza a los árboles balanceados.

- ¿Qué es? Una linked list con "atajos" — niveles adicionales que permiten saltar
- Probabilistic balancing: lanzar una moneda para decidir la altura de cada nodo
  - Sin rotaciones, sin rebalanceo, sin la complejidad de Red-Black trees
- Implementar desde cero en Python, Java, y Rust
- Análisis: expected O(log n) para search, insert, delete
- Concurrent skip lists:
  - Más fácil de hacer lock-free que un árbol balanceado
  - Java ConcurrentSkipListMap: la alternativa concurrent a TreeMap
- Ejercicio: implementar una skip list con insert, search, delete, y range scan
- Ejercicio: comparar performance skip list vs TreeMap vs ConcurrentSkipListMap
- Ejercicio: implementar una skip list lock-free simplificada en Rust

**Conexión:** Redis sorted sets (ZSET) usan skip lists.
LevelDB/RocksDB usan skip lists como MemTable (alternativa al Red-Black tree).
Kafka usa ConcurrentSkipListMap para ciertos indexes internos.

---

## Parte 5 — Estructuras para Concurrencia y Sistemas Distribuidos

### Capítulo 13 — Queues: de FIFO a Lock-Free a Distributed

La estructura más subestimada y más usada en sistemas reales.

- Array-based queue y linked-list queue: implementar ambas
- Deque (double-ended queue):
  - Implementar con ring buffer
  - Work-stealing deque (usado en schedulers: Java ForkJoinPool, Tokio, Go runtime)
- Blocking queue:
  - ArrayBlockingQueue (Java): bounded, lock-based
  - LinkedBlockingQueue (Java): unbounded, separate locks para head y tail
  - Implementar con condition variables en Java y Go
- Lock-free queues:
  - Michael-Scott queue: el algoritmo lock-free clásico con CAS
  - LMAX Disruptor: ring buffer sin locks, mecánica de secuencias
  - Concepto e implementación simplificada en Java
- Ejercicio: implementar un work-stealing deque en Go con goroutines
- Ejercicio: implementar una bounded blocking queue en Java y Rust
- Ejercicio: benchmark blocking queue vs lock-free queue bajo alta contención

**Conexión:** Kafka es, en esencia, un distributed log (queue persistente).
LMAX Disruptor se usa en sistemas de trading de baja latencia.
Java ForkJoinPool (que Spark usa) tiene work-stealing deques internamente.

---

### Capítulo 14 — Estructuras Thread-Safe: Patrones y Tradeoffs

Hacer una estructura concurrent-safe es más que ponerle un mutex.

- Mutex / RWLock: la solución obvia y sus problemas
  - Writer starvation en RWLock
  - Lock contention bajo carga
- Striped locking: dividir el lock en N segmentos (ConcurrentHashMap Java 7)
- Copy-on-write: inmutabilidad como estrategia de concurrencia
  - CopyOnWriteArrayList (Java): reads sin lock, writes copian todo
  - Cuándo es viable (reads >>> writes)
- Lock-free con CAS:
  - AtomicReference, AtomicInteger: las primitivas
  - ABA problem y cómo resolverlo (tagged pointers, epoch-based reclamation)
- Epoch-based memory reclamation (Rust crossbeam):
  - El problema de liberar memoria en estructuras lock-free
  - Concepto y uso en crossbeam-epoch
- Rust's ownership model como solución de concurrencia:
  - Send + Sync traits: el compilador previene data races
  - Arc<Mutex<T>> vs lock-free: cuándo cada uno
- Ejercicio: implementar un hash map con striped locking en Java
- Ejercicio: comparar Mutex vs RWLock vs lock-free para un contador compartido en Go y Rust
- Ejercicio: implementar un stack lock-free con CAS en Java (Treiber stack)

**Conexión:** Todo sistema distribuido (Kafka brokers, Spark executors, Flink TaskManagers)
opera con estructuras concurrent internamente. Este capítulo enseña los patrones.

---

### Capítulo 15 — Consistent Hashing y Estructuras para Sistemas Distribuidos

Las estructuras que hacen posible la distribución de datos entre nodos.

- Consistent hashing:
  - El problema: distribuir datos entre N nodos y sobrevivir a la adición/remoción de nodos
  - Hash ring: cada nodo se ubica en un anillo, cada key va al nodo más cercano
  - Virtual nodes: resolver el desbalance con nodos virtuales
  - Implementar en Python, Java, y Go
- Merkle trees (hash trees):
  - Cada hoja es un hash de un bloque de datos
  - Cada nodo interno es el hash de sus hijos
  - Verificar la integridad de un dataset enorme comparando solo la raíz
  - Implementar en Java y Rust
  - Uso real: Cassandra anti-entropy repair, Git (content-addressable storage), blockchain
- CRDTs (Conflict-free Replicated Data Types):
  - G-Counter: cada nodo incrementa su propio contador, merge = max por nodo
  - PN-Counter: counter que soporta incrementos y decrementos
  - OR-Set (Observed-Remove Set): agregar y remover elementos sin conflictos
  - Implementar G-Counter y OR-Set en Scala (inmutabilidad natural)
- Ejercicio: implementar consistent hashing con virtual nodes en Go
- Ejercicio: implementar un Merkle tree y verificar integridad de un archivo grande en Rust
- Ejercicio: implementar un G-Counter y un OR-Set en Scala

**Conexión:** Kafka usa consistent hashing para partitioning.
Cassandra usa consistent hashing + Merkle trees + CRDTs (counters).
DynamoDB fue diseñado sobre consistent hashing (Dynamo paper, 2007).

---

## Parte 6 — Estructuras para Datos a Escala

### Capítulo 16 — Columnar Layouts: Cómo se Almacenan los Datos Analíticos

El puente directo entre estructuras de datos y data engineering.

- Row-oriented vs column-oriented storage:
  - Row: bueno para OLTP (leer/escribir una fila completa)
  - Column: bueno para OLAP (leer una columna de millones de filas)
- Implementar un columnar store simplificado:
  - Almacenar cada columna como un array contiguo
  - Ejecutar un SUM sobre una columna vs sobre filas (benchmark)
- Técnicas de compresión columnar:
  - Run-Length Encoding (RLE): columnas con muchos valores repetidos
  - Dictionary encoding: reemplazar strings por integers
  - Bit-packing: almacenar integers con el mínimo de bits necesarios
  - Delta encoding: almacenar diferencias entre valores consecutivos
  - Implementar cada técnica y medir la ratio de compresión
- Parquet internals:
  - Row groups, column chunks, pages
  - Repetition y definition levels (para datos anidados)
  - La conexión con las técnicas de compresión implementadas
- Ejercicio: implementar dictionary encoding + bit-packing en Python y Rust
- Ejercicio: implementar un mini columnar store que soporte SUM, AVG, COUNT, y filtering
- Ejercicio: leer un archivo Parquet real y mapear cada sección a las estructuras implementadas

**Conexión:** Este capítulo conecta directamente con el Cap.02 del libro de data engineering
(formatos y memoria). Arrow, Parquet, y ORC son implementaciones de estas técnicas.

---

### Capítulo 17 — Índices: Cómo las Bases de Datos Encuentran tus Datos

Unificar todo: B-Trees, hash indexes, Bloom filters, bitmap indexes.

- Hash index: rápido para lookups exactos, inútil para rangos
  - Bitcask (Riak): un hash index sobre un log
  - Implementar un hash index sobre un append-only log file
- B+ Tree index: el estándar para bases de datos relacionales
  - Cómo PostgreSQL organiza sus indexes (páginas de 8KB, MVCC)
  - Relación entre buffer pool, B+ Tree, y disco
- LSM-Tree index: SSTables + MemTable + Bloom filter
  - Recapitular Cap.10, ahora como un "index engine" completo
- Bitmap index: ideal para columnas con baja cardinalidad
  - Implementar con bitsets del Cap.03
  - Uso real: Oracle, Apache Druid
- Composite indexes y el order de las columnas
  - Por qué INDEX(a, b) sirve para WHERE a = X pero no para WHERE b = Y
- Ejercicio: implementar un hash index sobre un append-only log en Rust
- Ejercicio: implementar un bitmap index y benchmark contra B+ Tree para queries de baja cardinalidad
- Ejercicio: construir un "mini database engine" que soporte 3 tipos de índice

**Conexión:** Este capítulo unifica los capítulos 04, 06, 09, 10, y 16 en el contexto
de cómo una base de datos real encuentra datos. PostgreSQL, MySQL, Cassandra,
RocksDB — cada uno elige una combinación distinta de estas estructuras.

---

### Capítulo 18 — El Proyecto Final: Construir un Key-Value Store

Integrar todo lo aprendido en una implementación funcional.

- Diseño: un key-value store con:
  - Write path: WAL → MemTable (skip list) → SSTable flush
  - Read path: MemTable → Bloom filter → SSTable binary search
  - Compaction: merge de SSTables (leveled compaction simplificada)
  - Concurrencia: reads concurrentes con writes (RWLock en MemTable)
- Implementar en Rust (performance + ownership model):
  - WAL: append-only file con fsync
  - MemTable: skip list (Cap.12)
  - SSTable: sorted file con block index (Cap.10)
  - Bloom filter: por SSTable para skip reads (Cap.06)
  - Compaction: background thread que merge SSTables
- Testing:
  - Correctness: put/get/delete con verificación
  - Crash recovery: matar el proceso y verificar durabilidad via WAL
  - Concurrency: reads y writes paralelos
  - Performance: benchmark throughput vs RocksDB (la referencia)
- Análisis: qué simplificaciones hicimos vs un sistema real (RocksDB, LevelDB)
  - Compaction scheduling, bloom filter tuning, compression, snapshots, transactions

**Conexión:** Has construido una versión simplificada de RocksDB.
Cassandra, Flink, y CockroachDB usan RocksDB internamente.
Entender esta implementación es entender el motor de almacenamiento
que mueve datos en los sistemas que estudiaste en el libro de data engineering.

---

## Resumen: el mapa de estructuras y sus conexiones

```
Cap.01-03: Fundamentos (memoria, complejidad, arrays)
    ↓
Cap.04-06: Hashing (hash maps, concurrent maps, Bloom filters, HyperLogLog)
    ↓
Cap.07-10: Árboles (BST, AVL, Red-Black, B-Tree, B+ Tree, LSM-Tree)
    ↓
Cap.11-12: Búsqueda (tries, skip lists)
    ↓
Cap.13-15: Concurrencia y distribución (queues, lock-free, consistent hashing, CRDTs)
    ↓
Cap.16-17: Almacenamiento (columnar, índices)
    ↓
Cap.18: Integración (key-value store completo)

Cada capítulo usa estructuras de los anteriores.
El proyecto final (Cap.18) integra: skip list + Bloom filter + SSTable + WAL + compaction.
```
