# Guía de Ejercicios — Cap.10: LSM-Trees: la Alternativa Write-Optimized

> El B-Tree del Cap.09 es brillante para lecturas:
> 1-2 I/Os por point query, range scans secuenciales.
> Pero cada insert MODIFICA una página in-place en disco.
> Eso es un random write. Y los discos odian los random writes.
>
> HDD: sequential write ~100 MB/s. Random write ~1 MB/s. 100× diferencia.
> SSD: sequential write ~500 MB/s. Random write ~50 MB/s. 10× diferencia.
>
> Para workloads write-heavy — logs de eventos, métricas de sensores,
> time series, ingestion de datos a escala — el B-Tree se convierte
> en un cuello de botella por los random writes.
>
> El LSM-Tree (Log-Structured Merge-Tree) invierte el tradeoff:
> TODAS las escrituras son secuenciales. Nunca random writes.
> Los datos se acumulan en memoria (MemTable),
> luego se escriben al disco como un bloque secuencial (SSTable).
>
> El costo: las lecturas son más lentas.
> En vez de 1-2 I/Os, una lectura puede necesitar buscar
> en la MemTable + múltiples SSTables en disco.
> Los Bloom filters del Cap.06 mitigan este costo.
>
> Cassandra, RocksDB, LevelDB, HBase, ScyllaDB, BadgerDB:
> todos son LSM-Trees. Flink usa RocksDB como state backend.
> El WAL de un LSM-Tree es conceptualmente idéntico al commit log de Kafka.
>
> Este capítulo conecta CINCO capítulos anteriores en una estructura:
> Red-Black tree (Cap.07) como MemTable.
> Sorted array (Cap.03) como SSTable.
> Bloom filter (Cap.06) para evitar lecturas innecesarias.
> K-way merge (Cap.08) para compaction.
> Hash function (Cap.04) para partitioning.

---

## El modelo mental: B-Tree vs LSM-Tree

```
EL TRADEOFF FUNDAMENTAL:

  B-Tree (read-optimized):
    Write: encontrar la página correcta (random I/O) → modificar in-place.
    Read: navegar el árbol → 1-2 I/Os.
    → Writes son costosos (random). Reads son baratos (directo).

  LSM-Tree (write-optimized):
    Write: escribir en memoria (MemTable) → eventual flush secuencial.
    Read: buscar en MemTable + buscar en múltiples SSTables en disco.
    → Writes son baratos (secuencial). Reads son costosos (múltiples fuentes).

  Visualización:

  B-Tree write:                       LSM-Tree write:
  ┌──────────┐                        ┌──────────┐
  │ Encontrar │ ←─ random I/O         │ Escribir  │ ←─ RAM (instantáneo)
  │  página   │    (lento)            │ MemTable  │    (rápido)
  └──────────┘                        └──────────┘
       ↓                                   ↓ (cuando se llena)
  ┌──────────┐                        ┌──────────┐
  │ Modificar │ ←─ random write       │  Flush    │ ←─ sequential write
  │ in-place  │    (lento)            │ SSTable   │    (rápido)
  └──────────┘                        └──────────┘

  B-Tree read:                        LSM-Tree read:
  ┌──────────┐                        ┌──────────┐
  │ Navegar   │ ←─ 1-2 I/Os          │ MemTable  │ ←─ RAM
  │ B-Tree    │    (rápido)           │  lookup   │
  └──────────┘                        └──────────┘
                                           ↓ (si no está)
                                      ┌──────────┐
                                      │ Bloom     │ ←─ RAM
                                      │ filter    │
                                      └──────────┘
                                           ↓ (si "probablemente sí")
                                      ┌──────────┐
                                      │ SSTable   │ ←─ posiblemente I/O
                                      │ search    │    (más lento)
                                      └──────────┘
                                      (repetir para cada nivel)

  Amplification trilemma:
    No puedes optimizar las tres simultáneamente:
    - Write amplification: cuántas veces se reescribe un dato.
    - Read amplification: cuántas fuentes consultas por read.
    - Space amplification: cuánta data redundante almacenas.
    B-Tree: baja read amp, baja space amp, alta write amp.
    LSM-Tree (tiered): baja write amp, alta read amp, alta space amp.
    LSM-Tree (leveled): media write amp, media read amp, baja space amp.
```

---

## Tabla de contenidos

- [Sección 10.1 — Anatomía de un LSM-Tree: MemTable, SSTable, WAL](#sección-101--anatomía-de-un-lsm-tree-memtable-sstable-wal)
- [Sección 10.2 — Write path: de la escritura al disco](#sección-102--write-path-de-la-escritura-al-disco)
- [Sección 10.3 — Read path: buscar en múltiples fuentes](#sección-103--read-path-buscar-en-múltiples-fuentes)
- [Sección 10.4 — Implementar un LSM-Tree simplificado](#sección-104--implementar-un-lsm-tree-simplificado)
- [Sección 10.5 — Compaction: tiered vs leveled](#sección-105--compaction-tiered-vs-leveled)
- [Sección 10.6 — El trilemma de amplificación](#sección-106--el-trilemma-de-amplificación)
- [Sección 10.7 — LSM-Trees en producción: de Cassandra a Flink](#sección-107--lsm-trees-en-producción-de-cassandra-a-flink)

---

## Sección 10.1 — Anatomía de un LSM-Tree: MemTable, SSTable, WAL

### Ejercicio 10.1.1 — Leer: los tres componentes

**Tipo: Leer**

```
Un LSM-Tree tiene tres componentes fundamentales:

  1. MEMTABLE (in-memory, sorted):
     Un árbol balanceado en RAM (Red-Black tree, skip list, o AVL).
     Todas las escrituras van aquí primero.
     Mantiene los datos ORDENADOS para flush eficiente.
     Tamaño típico: 64 MB - 256 MB.

     ¿Por qué sorted?
     Cuando la MemTable se llena y se escribe a disco,
     los datos ya están ordenados → sequential write directo.
     Si no estuviera ordenada: habría que ordenar primero.

     Implementaciones reales:
     - LevelDB / RocksDB: skip list (permite concurrent reads durante writes).
     - Cassandra: skip list (ConcurrentSkipListMap de Java).
     - HBase: ConcurrentSkipListMap.
     ¿Por qué skip list y no Red-Black tree?
     Porque skip list permite inserts concurrentes sin lock global.
     Múltiples threads pueden escribir simultáneamente.

  2. SSTABLE (Sorted String Table, on-disk, immutable):
     Un archivo en disco con key-value pairs ORDENADOS.
     Una vez escrito, NUNCA se modifica (append-only).
     Estructura interna de un SSTable:
     ┌────────────────────────────────────────┐
     │  Data blocks (key-value pairs sorted)  │
     │  [k1:v1, k2:v2, k3:v3, ...]          │
     ├────────────────────────────────────────┤
     │  Index block (key → offset del data block) │
     ├────────────────────────────────────────┤
     │  Bloom filter (membership test)         │
     ├────────────────────────────────────────┤
     │  Metadata (min key, max key, count)     │
     └────────────────────────────────────────┘

     Búsqueda en un SSTable:
     1. Bloom filter: ¿podría estar aquí? Si no → skip.
     2. Index block: binary search → encontrar data block.
     3. Data block: scan → encontrar el key-value exacto.

  3. WAL (Write-Ahead Log):
     Antes de escribir en la MemTable, escribir en el WAL.
     El WAL es un archivo append-only en disco (sequential write).
     Si el sistema crashea antes de que la MemTable se flush,
     el WAL permite reconstruir la MemTable.
     → El WAL garantiza DURABILIDAD.

  Flujo:
    Write(key, value):
      1. Append al WAL (durabilidad).
      2. Insert en MemTable (velocidad).
      3. Cuando MemTable se llena → flush como SSTable.
      4. Truncar el WAL correspondiente.
```

**Preguntas:**

1. ¿La MemTable usa un skip list y no un Red-Black tree.
   ¿Solo por concurrencia?

2. ¿SSTable es immutable (nunca se modifica). ¿Cómo manejas deletes?

3. ¿El WAL es conceptualmente idéntico al commit log de Kafka?

4. ¿64-256 MB de MemTable. ¿Qué pasa si llegan más writes
   de lo que puedes flush?

5. ¿El Bloom filter DENTRO del SSTable es el del Cap.06?

---

### Ejercicio 10.1.2 — Analizar: por qué sequential writes son rápidos

**Tipo: Analizar**

```
La diferencia entre sequential y random I/O:

  HDD (spinning disk):
    Random write: mover el cabezal a la posición correcta (seek time ~5-10 ms)
                  + rotational latency (~2-4 ms) + transfer.
    Sequential write: el cabezal ya está en la posición correcta.
                      Solo transfer. ~100 MB/s.
    Ratio: sequential / random ≈ 100×

  SSD (NAND flash):
    Random write: encontrar el bloque correcto + escribir 4 KB page.
                  ~50 μs per page. ~50 MB/s effective.
    Sequential write: escribir pages en secuencia.
                      ~500 MB/s. Menos wear leveling overhead.
    Ratio: sequential / random ≈ 10×

  NVMe SSD:
    Ratio reducido a ~3-5× pero sequential sigue siendo más rápido.
    Y hay un factor adicional: las SSD hacen ERASE antes de WRITE.
    Random writes causan más erases → menor vida útil.
    → Sequential writes son mejores para la longevidad del SSD.

  El LSM-Tree convierte TODOS los writes en sequential:
    1. WAL: append-only (sequential).
    2. MemTable → SSTable flush: un bloque grande (sequential).
    3. Compaction: merge de SSTables (sequential read + sequential write).
    → Zero random writes.

  El B-Tree hace random writes:
    1. Encontrar la página del nodo hoja (random read).
    2. Modificar la página in-place (random write).
    3. Si split: escribir nuevas páginas (más random writes).

  Para ingestión de datos a escala:
    B-Tree: 50 MB/s writes (SSD random) → 50K inserts/sec (1 KB each).
    LSM-Tree: 500 MB/s writes (SSD sequential) → 500K inserts/sec.
    → 10× más throughput de escritura con LSM-Tree.
```

**Preguntas:**

1. ¿10× en SSD y 100× en HDD. ¿NVMe reduce la ventaja del LSM-Tree?

2. ¿"Zero random writes" del LSM-Tree — ¿incluso durante compaction?

3. ¿Kafka también es append-only y sequential. ¿Es un LSM-Tree?

4. ¿Para un pipeline de ingestión de 1M events/sec,
   ¿B-Tree o LSM-Tree?

5. ¿Las SSD modernas con NVMe — ¿hacen que el B-Tree
   sea suficiente para write-heavy?

---

## Sección 10.2 — Write Path: de la Escritura al Disco

### Ejercicio 10.2.1 — Leer: el camino de un write

**Tipo: Leer**

```
Un write(key="user_42", value="{name:'Alice'}"):

  Paso 1: APPEND AL WAL
    Escribir un record al final del archivo WAL:
    [timestamp | key_len | key | value_len | value]
    Costo: 1 sequential write (~1-5 μs en SSD).
    Propósito: durabilidad. Si el proceso crashea,
    el WAL permite reconstruir la MemTable.

  Paso 2: INSERT EN MEMTABLE
    Insertar (key, value) en el skip list / Red-Black tree.
    Costo: O(log n) comparaciones en RAM (~100-500 ns).
    La MemTable está COMPLETAMENTE en memoria.

  Paso 3: ACKNOWLEDGE AL CLIENTE
    El write es "durable" (gracias al WAL) y "visible" (en la MemTable).
    Retornar éxito al cliente.
    Latencia total: ~5-50 μs.

  Paso 4: FLUSH (eventual, asíncrono)
    Cuando la MemTable alcanza su tamaño máximo (ej: 64 MB):
    a) Convertir la MemTable actual en "immutable" (no más writes).
    b) Crear una NUEVA MemTable vacía para writes futuros.
    c) En background: escribir la MemTable inmutable como un SSTable en disco.
       Costo: sequential write de 64 MB ≈ 130 ms (SSD 500 MB/s).
    d) Truncar la porción del WAL que corresponde a esta MemTable.

  ¿Qué pasa durante el flush?
    Los writes nuevos van a la nueva MemTable.
    Los reads buscan en la nueva MemTable Y en la MemTable inmutable
    (que está siendo escrita al disco).
    → No hay downtime durante el flush.

  ¿Deletes?
    Un delete es un write de un TOMBSTONE:
    write(key="user_42", value=TOMBSTONE)
    El tombstone indica que la key fue eliminada.
    Los reads que encuentran un tombstone retornan "not found".
    El tombstone se propaga durante compaction y eventualmente
    se elimina cuando todas las versiones anteriores de la key
    han sido compactadas.
```

**Preguntas:**

1. ¿El write total toma ~5-50 μs. ¿Un B-Tree write toma cuánto?

2. ¿La MemTable inmutable durante el flush — ¿es un segundo skip list?

3. ¿Los tombstones ocupan espacio. ¿Cuándo se eliminan?

4. ¿Si el flush tarda mucho y la nueva MemTable se llena,
   ¿qué pasa?

5. ¿El WAL de un LSM-Tree es lo mismo que el WAL de PostgreSQL?

---

### Ejercicio 10.2.2 — Analizar: MemTable — skip list vs Red-Black tree

**Tipo: Analizar**

```
La MemTable necesita:
  - Insert: O(log n).
  - Búsqueda por key: O(log n).
  - Iteración ordenada: O(n) (para el flush).
  - (Idealmente) Concurrent reads y writes sin lock global.

  Skip list:
    Estructura: múltiples linked lists superpuestas.
    Cada nivel es una "express lane" que salta nodos.
    Nivel 0: todos los nodos (linked list completa).
    Nivel 1: ~50% de los nodos.
    Nivel 2: ~25% de los nodos.
    ...
    Búsqueda: empezar en el nivel más alto, avanzar,
              bajar cuando la clave siguiente es mayor que la buscada.
    Complejidad: O(log n) esperado.

    Nivel 3: ─────────────── 42 ─────────────────────────── 95 ──
    Nivel 2: ───── 15 ────── 42 ──── 60 ────────────────── 95 ──
    Nivel 1: ── 8, 15 ── 30, 42 ── 60 ── 73 ────── 88 ── 95 ──
    Nivel 0: 3, 8, 15, 22, 30, 42, 55, 60, 67, 73, 80, 88, 95, 99

    Ventaja para MemTable:
    - Concurrent inserts: cada insert modifica solo punteros locales.
      Con CAS (compare-and-swap), múltiples threads pueden insertar
      sin un lock global. LevelDB y RocksDB usan esto.
    - Iteración ordenada: recorrer el nivel 0 → todos los datos en orden.

  Red-Black tree:
    Ventaja: peor caso O(log n) garantizado (skip list es esperado).
    Desventaja: rotaciones durante insert modifican estructura global.
    → Difícil de hacer concurrent sin lock.
    → Java ConcurrentSkipListMap existe. No hay ConcurrentTreeMap.

  En la práctica:
    LevelDB:    skip list.
    RocksDB:    skip list (con concurrent inserts).
    Cassandra:  ConcurrentSkipListMap (Java).
    HBase:      ConcurrentSkipListMap.
    BadgerDB:   skip list.
    → El skip list ganó para MemTables.
```

**Preguntas:**

1. ¿Skip list es O(log n) ESPERADO, no worst case. ¿Es un problema?

2. ¿ConcurrentSkipListMap de Java — ¿es lock-free?

3. ¿Memoria del skip list vs Red-Black tree — ¿cuál usa más?

4. ¿Si la MemTable es single-threaded (un solo writer),
   ¿Red-Black tree sería mejor?

5. ¿El skip list del Cap.07 mencionado para Redis — ¿es el mismo concepto?

---

## Sección 10.3 — Read Path: Buscar en Múltiples Fuentes

### Ejercicio 10.3.1 — Leer: el camino de un read

**Tipo: Leer**

```
Un read(key="user_42"):

  Paso 1: BUSCAR EN MEMTABLE ACTIVA
    Skip list lookup: O(log n) en RAM. ~100-500 ns.
    Si encontrada → retornar. DONE.

  Paso 2: BUSCAR EN MEMTABLE INMUTABLE (si existe, durante flush)
    Si hay un flush en progreso, hay una MemTable inmutable.
    Buscar ahí también. ~100-500 ns.

  Paso 3: BUSCAR EN SSTABLES (del más reciente al más antiguo)
    Para cada SSTable, de más nuevo a más viejo:
    a) Verificar min/max key range:
       Si key < min_key o key > max_key → skip. Instantáneo.
    b) Consultar el BLOOM FILTER del SSTable:
       Si "definitely not" → skip. ~200 ns.
       Si "probably yes" → continuar.
    c) Buscar en el INDEX BLOCK:
       Binary search → encontrar el data block correcto. ~1-5 μs.
    d) Leer el DATA BLOCK del disco:
       Leer 4-16 KB → buscar la key. ~100 μs (SSD).
    Si encontrada → retornar. DONE.

  Paso 4: KEY NO EXISTE
    Si ningún SSTable tiene la key → retornar "not found".

  ¿Cuántos SSTables se consultan?
    Peor caso: TODOS los SSTables. Si hay 20 SSTables → 20 Bloom filter checks.
    Caso típico con Bloom filters (1% FP): 20 × 1% = 0.2 lecturas innecesarias.
    → En promedio, solo 1-2 SSTables se leen realmente.

  Sin Bloom filters:
    20 SSTables × ~100 μs = 2 ms por read.
  Con Bloom filters:
    1.2 SSTables × ~100 μs + 20 × 200 ns (Bloom) = ~124 μs por read.
    → 16× más rápido con Bloom filters.

  El read path es donde los Bloom filters del Cap.06
  son CRÍTICOS para el rendimiento del LSM-Tree.
```

**Preguntas:**

1. ¿Los SSTables se consultan del más reciente al más antiguo.
   ¿Por qué?

2. ¿Bloom filter reduce de 2 ms a 124 μs. ¿Qué FP rate usan en producción?

3. ¿Un read que busca una key inexistente es el peor caso. ¿Por qué?

4. ¿Compaction reduce el número de SSTables. ¿Eso mejora los reads?

5. ¿Con 100 SSTables y sin compaction, ¿qué tan lento es un read?

---

### Ejercicio 10.3.2 — Analizar: range scans en un LSM-Tree

**Tipo: Analizar**

```
Range scan: SELECT * FROM events WHERE timestamp BETWEEN t1 AND t2.

  En un B+ Tree:
    Encontrar la primera hoja: 2-3 I/Os.
    Scan hojas enlazadas: sequential I/O. Rápido.

  En un LSM-Tree:
    Necesitas hacer un MERGE de rangos de TODOS los niveles:
    1. Abrir un iterador en la MemTable: rango [t1, t2].
    2. Abrir un iterador en cada SSTable que PODRÍA contener [t1, t2].
    3. Hacer un K-way merge (Cap.08) de todos los iteradores.
    4. Retornar los resultados en orden, eliminando duplicados
       y aplicando tombstones.

  El K-way merge es necesario porque los datos están DISTRIBUIDOS
  entre múltiples SSTables. Cada SSTable tiene una porción
  del rango, posiblemente con versiones distintas del mismo key.

  Complejidad:
    B+ Tree range scan: O(log n + k) donde k = resultados.
    LSM-Tree range scan: O(L × log n + k × log L) donde L = niveles.
    → El factor L (niveles) hace el range scan más costoso.

  Optimización: leveled compaction (Sección 10.5).
    En leveled compaction, cada nivel tiene SSTables sin overlap.
    → Solo necesitas consultar UN SSTable por nivel para un rango.
    → K-way merge de L SSTables (uno por nivel) en vez de TODOS.
```

**Preguntas:**

1. ¿Range scan en LSM-Tree necesita K-way merge.
   ¿Es el heap del Cap.08?

2. ¿L niveles = L SSTables a merge. ¿Cuántos niveles típicamente?

3. ¿Para un query de analytics que escanea millones de rows,
   ¿LSM-Tree es mucho peor que B+ Tree?

4. ¿Cassandra range queries son lentas por esta razón?

5. ¿Leveled compaction mejora los range scans. ¿Cuánto?

---

## Sección 10.4 — Implementar un LSM-Tree Simplificado

### Ejercicio 10.4.1 — Implementar: LSM-Tree en Java con MemTable + SSTables

**Tipo: Implementar**

```java
import java.util.*;
import java.io.*;

public class SimpleLSM {

    // ═══ SSTable on disk (simplified: in-memory sorted array) ═══
    static class SSTable {
        final int id;
        final String[] keys;
        final String[] values;
        final long[] bloomFilter; // simplified bloom filter
        final int bloomM;
        final int bloomK;

        SSTable(int id, TreeMap<String, String> data) {
            this.id = id;
            this.keys = data.keySet().toArray(new String[0]);
            this.values = data.values().toArray(new String[0]);
            this.bloomM = keys.length * 10; // 10 bits per element
            this.bloomK = 7;
            this.bloomFilter = new long[(bloomM + 63) / 64];
            for (String key : keys) addToBloom(key);
        }

        private int[] bloomPositions(String key) {
            int h1 = key.hashCode();
            int h2 = h1 * 0x5BD1E995 + 0x47B6137B;
            int[] pos = new int[bloomK];
            for (int i = 0; i < bloomK; i++)
                pos[i] = Math.abs((h1 + i * h2) % bloomM);
            return pos;
        }

        private void addToBloom(String key) {
            for (int p : bloomPositions(key))
                bloomFilter[p / 64] |= (1L << (p % 64));
        }

        boolean mightContain(String key) {
            for (int p : bloomPositions(key))
                if ((bloomFilter[p / 64] & (1L << (p % 64))) == 0) return false;
            return true;
        }

        String get(String key) {
            if (!mightContain(key)) return null; // Bloom filter skip
            int idx = Arrays.binarySearch(keys, key);
            return idx >= 0 ? values[idx] : null;
        }

        String minKey() { return keys[0]; }
        String maxKey() { return keys[keys.length - 1]; }
        int size() { return keys.length; }
    }

    // ═══ LSM-Tree ═══

    private TreeMap<String, String> memTable = new TreeMap<>();
    private final List<SSTable> sstables = new ArrayList<>();
    private final int memTableMaxSize;
    private int sstableCounter = 0;
    private int totalWrites = 0;

    public SimpleLSM(int memTableMaxSize) {
        this.memTableMaxSize = memTableMaxSize;
    }

    public void put(String key, String value) {
        memTable.put(key, value);
        totalWrites++;
        if (memTable.size() >= memTableMaxSize) {
            flush();
        }
    }

    public void delete(String key) {
        put(key, "__TOMBSTONE__");
    }

    private void flush() {
        if (memTable.isEmpty()) return;
        SSTable sst = new SSTable(sstableCounter++, memTable);
        sstables.add(0, sst); // newest first
        memTable = new TreeMap<>();
    }

    public String get(String key) {
        // 1. Check MemTable
        if (memTable.containsKey(key)) {
            String val = memTable.get(key);
            return "__TOMBSTONE__".equals(val) ? null : val;
        }

        // 2. Check SSTables (newest first)
        for (SSTable sst : sstables) {
            // Min/max key range check
            if (key.compareTo(sst.minKey()) < 0 || key.compareTo(sst.maxKey()) > 0)
                continue;
            String val = sst.get(key); // includes Bloom filter check
            if (val != null) {
                return "__TOMBSTONE__".equals(val) ? null : val;
            }
        }

        return null; // not found
    }

    public List<String[]> rangeScan(String lo, String hi) {
        // Merge from all sources
        Map<String, String> merged = new TreeMap<>();

        // SSTables oldest first (so newer overwrites older)
        for (int i = sstables.size() - 1; i >= 0; i--) {
            SSTable sst = sstables.get(i);
            int start = Arrays.binarySearch(sst.keys, lo);
            if (start < 0) start = -(start + 1);
            for (int j = start; j < sst.keys.length && sst.keys[j].compareTo(hi) <= 0; j++) {
                merged.put(sst.keys[j], sst.values[j]);
            }
        }

        // MemTable (newest, overwrites all)
        for (var entry : memTable.subMap(lo, true, hi, true).entrySet()) {
            merged.put(entry.getKey(), entry.getValue());
        }

        // Filter tombstones
        List<String[]> result = new ArrayList<>();
        for (var entry : merged.entrySet()) {
            if (!"__TOMBSTONE__".equals(entry.getValue())) {
                result.add(new String[]{entry.getKey(), entry.getValue()});
            }
        }
        return result;
    }

    // Simple compaction: merge all SSTables into one
    public void compact() {
        if (sstables.size() <= 1) return;
        flush(); // flush MemTable first

        TreeMap<String, String> merged = new TreeMap<>();
        // Oldest first, then newer overwrites
        for (int i = sstables.size() - 1; i >= 0; i--) {
            SSTable sst = sstables.get(i);
            for (int j = 0; j < sst.size(); j++) {
                merged.put(sst.keys[j], sst.values[j]);
            }
        }
        // Remove tombstones
        merged.entrySet().removeIf(e -> "__TOMBSTONE__".equals(e.getValue()));

        sstables.clear();
        if (!merged.isEmpty()) {
            sstables.add(new SSTable(sstableCounter++, merged));
        }
    }

    public int sstableCount() { return sstables.size(); }
    public int memTableSize() { return memTable.size(); }

    public static void main(String[] args) {
        var lsm = new SimpleLSM(10_000);
        int n = 1_000_000;
        var rng = new Random(42);

        // Write benchmark
        long start = System.nanoTime();
        for (int i = 0; i < n; i++) {
            lsm.put("key_" + String.format("%010d", rng.nextInt(n * 10)),
                     "value_" + i);
        }
        long writeTime = System.nanoTime() - start;

        System.out.printf("LSM-Tree: %,d writes in %d ms (%.0f ns/op)%n",
            n, writeTime / 1_000_000, (double) writeTime / n);
        System.out.printf("SSTables: %d, MemTable size: %d%n",
            lsm.sstableCount(), lsm.memTableSize());

        // Read benchmark (before compaction)
        rng = new Random(42);
        start = System.nanoTime();
        int found = 0;
        for (int i = 0; i < 100_000; i++) {
            if (lsm.get("key_" + String.format("%010d", rng.nextInt(n * 10))) != null)
                found++;
        }
        long readBefore = System.nanoTime() - start;

        System.out.printf("Read (before compact): %d ms (%.0f ns/op), found=%d%n",
            readBefore / 1_000_000, (double) readBefore / 100_000, found);

        // Compact
        start = System.nanoTime();
        lsm.compact();
        long compactTime = System.nanoTime() - start;
        System.out.printf("Compaction: %d ms, SSTables after: %d%n",
            compactTime / 1_000_000, lsm.sstableCount());

        // Read benchmark (after compaction)
        rng = new Random(42);
        start = System.nanoTime();
        found = 0;
        for (int i = 0; i < 100_000; i++) {
            if (lsm.get("key_" + String.format("%010d", rng.nextInt(n * 10))) != null)
                found++;
        }
        long readAfter = System.nanoTime() - start;

        System.out.printf("Read (after compact): %d ms (%.0f ns/op), found=%d%n",
            readAfter / 1_000_000, (double) readAfter / 100_000, found);
        System.out.printf("Speedup from compaction: %.1f×%n",
            (double) readBefore / readAfter);
    }
}
// Resultado esperado:
// LSM-Tree: 1,000,000 writes in ~1000-2000 ms (~1000-2000 ns/op)
// SSTables: ~100, MemTable size: ~0
// Read (before compact): ~500-1500 ms (5000-15000 ns/op) — many SSTables
// Compaction: ~2000-4000 ms
// Read (after compact): ~100-300 ms (1000-3000 ns/op) — 1 SSTable
// Speedup: 3-5×
```

**Preguntas:**

1. ¿100 SSTables antes de compaction → reads 3-5× más lentos. ¿Esperado?

2. ¿La compaction reduce SSTables de 100 a 1. ¿Es siempre posible?

3. ¿Los Bloom filters en cada SSTable — ¿cuántas lecturas evitan?

4. ¿Esta implementación usa TreeMap in-memory como "disco".
   ¿Qué cambiaría con I/O real?

5. ¿`__TOMBSTONE__` como valor especial — ¿es lo que hace Cassandra?

---

### Ejercicio 10.4.2 — Implementar: benchmark LSM-Tree vs TreeMap (B-Tree proxy)

**Tipo: Implementar**

```java
import java.util.*;

public class LSMvsBTree {

    public static void main(String[] args) {
        int n = 2_000_000;
        var rng = new Random(42);

        // ═══ Write benchmark ═══
        System.out.println("═══ WRITE BENCHMARK ═══\n");

        // TreeMap (proxy for B-Tree: sorted, in-place updates)
        var treeMap = new TreeMap<String, String>();
        long start = System.nanoTime();
        rng = new Random(42);
        for (int i = 0; i < n; i++) {
            String key = "k" + String.format("%010d", rng.nextInt(n * 5));
            treeMap.put(key, "v" + i);
        }
        long treeWrite = System.nanoTime() - start;

        // SimpleLSM
        var lsm = new SimpleLSM(50_000);
        start = System.nanoTime();
        rng = new Random(42);
        for (int i = 0; i < n; i++) {
            String key = "k" + String.format("%010d", rng.nextInt(n * 5));
            lsm.put(key, "v" + i);
        }
        long lsmWrite = System.nanoTime() - start;

        System.out.printf("TreeMap write: %d ms (%.0f ns/op)%n",
            treeWrite / 1_000_000, (double) treeWrite / n);
        System.out.printf("LSM write:     %d ms (%.0f ns/op)%n",
            lsmWrite / 1_000_000, (double) lsmWrite / n);
        System.out.printf("LSM/TreeMap:   %.2f×%n", (double) lsmWrite / treeWrite);

        // ═══ Read benchmark ═══
        System.out.println("\n═══ READ BENCHMARK ═══\n");
        int queries = 200_000;

        rng = new Random(99);
        start = System.nanoTime();
        int found1 = 0;
        for (int i = 0; i < queries; i++) {
            if (treeMap.get("k" + String.format("%010d", rng.nextInt(n * 5))) != null)
                found1++;
        }
        long treeRead = System.nanoTime() - start;

        rng = new Random(99);
        start = System.nanoTime();
        int found2 = 0;
        for (int i = 0; i < queries; i++) {
            if (lsm.get("k" + String.format("%010d", rng.nextInt(n * 5))) != null)
                found2++;
        }
        long lsmRead = System.nanoTime() - start;

        System.out.printf("TreeMap read: %d ms (%.0f ns/op), found=%d%n",
            treeRead / 1_000_000, (double) treeRead / queries, found1);
        System.out.printf("LSM read:     %d ms (%.0f ns/op), found=%d%n",
            lsmRead / 1_000_000, (double) lsmRead / queries, found2);
        System.out.printf("TreeMap/LSM:  %.1f× faster reads%n",
            (double) lsmRead / treeRead);

        System.out.printf("\nSSTables: %d%n", lsm.sstableCount());
    }
}
// Resultado esperado:
// TreeMap write: ~3000-5000 ms
// LSM write:     ~2000-3500 ms (faster: sequential flush)
// LSM/TreeMap:   ~0.7-0.8× (LSM writes slightly faster)
//
// TreeMap read: ~600-1000 ms
// LSM read:     ~3000-8000 ms (slower: multiple SSTables)
// TreeMap/LSM:  5-10× faster reads
//
// → LSM faster writes, B-Tree faster reads. The fundamental tradeoff.
```

**Preguntas:**

1. ¿LSM writes ligeramente más rápidos. ¿Con disco real la diferencia
   sería 10×?

2. ¿TreeMap reads 5-10× más rápidos. ¿Con Bloom filters + compaction
   la diferencia se reduce?

3. ¿Para un workload 90% writes / 10% reads, ¿LSM gana overall?

4. ¿Para un workload 10% writes / 90% reads, ¿B-Tree gana overall?

5. ¿Cassandra (LSM) vs PostgreSQL (B-Tree) — ¿el tradeoff
   explica cuándo usar cada uno?

---

## Sección 10.5 — Compaction: Tiered vs Leveled

### Ejercicio 10.5.1 — Leer: por qué compaction es necesaria

**Tipo: Leer**

```
Sin compaction, los SSTables se acumulan:
  Después de 1M writes con MemTable de 10K:
  → 100 SSTables en disco.
  → Cada read potencialmente consulta 100 Bloom filters + SSTables.
  → Reads cada vez más lentos.
  → Data redundante (versiones antiguas del mismo key).

Compaction = merge de múltiples SSTables en uno (o pocos) nuevos.
  Lee N SSTables → K-way merge → escribe 1 SSTable nuevo.
  Elimina duplicados (versiones antiguas).
  Elimina tombstones (deletes completados).
  → Menos SSTables → reads más rápidos.
  → Menos espacio en disco → menos redundancia.
  → Costo: lee y reescribe todos los datos → write amplification.

Dos estrategias dominantes:

  1. SIZE-TIERED COMPACTION (Cassandra default):
     Cuando hay ~4 SSTables de tamaño similar → merge en uno más grande.
     Tiers: [small, small, small, small] → [medium]
            [medium, medium, medium, medium] → [large]
     → Simple. Baja write amplification (~10×).
     → Alta space amplification (necesitas espacio para los datos
       originales + los compactados temporalmente).
     → Alta read amplification (múltiples SSTables por tier).

  2. LEVELED COMPACTION (RocksDB default, Cassandra opción):
     Los SSTables se organizan en NIVELES.
     Nivel 0: SSTables del flush (pueden overlappear).
     Nivel 1: SSTables sin overlap, max 10 MB total.
     Nivel 2: SSTables sin overlap, max 100 MB total.
     Nivel 3: SSTables sin overlap, max 1 GB total.
     ...
     Cada nivel es 10× el anterior.

     Compaction: merge un SSTable del nivel L con los SSTables
     del nivel L+1 que overlappean → nuevos SSTables en nivel L+1.
     → Cada nivel tiene SSTables SIN overlap.
     → Un read solo consulta 1 SSTable por nivel.
     → Baja read amplification y baja space amplification.
     → Alta write amplification (~30×): cada dato se reescribe
       cuando se mueve entre niveles.
```

**Preguntas:**

1. ¿Tiered: write amp ~10×, leveled: write amp ~30×.
   ¿Cuál es mejor para write-heavy?

2. ¿"SSTables sin overlap" en leveled compaction.
   ¿Eso hace que un read solo consulte 1 SSTable por nivel?

3. ¿Cassandra usa tiered por defecto. ¿RocksDB usa leveled.
   ¿Por qué la diferencia?

4. ¿Compaction consume CPU y I/O. ¿Puede afectar latencia
   de queries en primer plano?

5. ¿El "10× por nivel" de leveled compaction
   — ¿es configurable?

---

### Ejercicio 10.5.2 — Analizar: el impacto de compaction en números

**Tipo: Analizar**

```
Escenario: 100 GB de datos, keys de 100 bytes, values de 900 bytes.

  SIZE-TIERED:
    SSTables: ~4 por tier, ~4-5 tiers.
    Total SSTables: ~16-20.
    Space amplification: hasta 2× (100 GB datos + 100 GB durante compaction).
    Write amplification: ~10× (cada byte se reescribe ~10 veces total).
    Read amplification: read consulta ~4 SSTables en el tier más grande.
    → Write throughput: alto. Space usage: alto. Read latency: medio.

  LEVELED:
    Niveles: L0 (~64 MB) → L1 (~640 MB) → L2 (~6.4 GB) → L3 (~64 GB).
    SSTables por nivel: L1=10, L2=100, L3=1000.
    Space amplification: ~1.1× (10% extra).
    Write amplification: ~30× (cada dato se reescribe al cruzar niveles).
    Read amplification: 1 SSTable por nivel = ~4 SSTables máx.
    → Write throughput: medio. Space usage: bajo. Read latency: bajo.

  COMPARACIÓN DIRECTA:
                     Size-Tiered    Leveled
  Write amp          ~10×           ~30×
  Read amp           ~20 SSTables   ~4 SSTables
  Space amp          ~2×            ~1.1×
  Write throughput   ★★★            ★★
  Read latency       ★★             ★★★
  Space efficiency   ★              ★★★

  Regla de decisión:
  - Write-heavy (logs, time series, ingestion): size-tiered.
  - Read-heavy (point queries, interactive): leveled.
  - Space-constrained (limited disk): leveled.
```

**Preguntas:**

1. ¿Write amplification de 30× — ¿significa que 100 GB de datos
   causan 3 TB de escrituras a disco?

2. ¿Para SSD con vida limitada por writes (wear leveling),
   ¿write amp de 30× es un problema?

3. ¿Cassandra cambió a leveled compaction para workloads read-heavy?

4. ¿RocksDB (Flink state backend) usa leveled.
   ¿Flink es read-heavy o write-heavy?

5. ¿Existe una estrategia que combine lo mejor de ambas?

---

## Sección 10.6 — El Trilemma de Amplificación

### Ejercicio 10.6.1 — Analizar: RUM conjecture — no puedes tener todo

**Tipo: Analizar**

```
RUM Conjecture (Athanassoulis et al., 2016):
  No puedes optimizar Read, Update, y Memory simultáneamente.
  Cualquier estructura de datos optimiza como máximo 2 de 3.

  READ optimal:     datos en 1 lugar, sin duplicados. Lectura directa.
  UPDATE optimal:   writes secuenciales, sin reorganización.
  MEMORY optimal:   sin duplicación, sin índices extra.

  Mapa de estructuras:

  Estructura        Read    Write    Space
  ─────────         ────    ─────    ─────
  B-Tree            ★★★     ★★       ★★★
  LSM (tiered)      ★       ★★★      ★
  LSM (leveled)     ★★      ★★       ★★★
  Sorted array      ★★★     ★        ★★★
  Hash table        ★★★     ★★★      ★★
  Log (append-only) ★       ★★★      ★★★

  Implicaciones para el diseño de sistemas:

  1. Kafka (append-only log):
     Excelente writes y space. Reads requieren offset scan.
     → Optimizado para writes y space, sacrifica random reads.

  2. Cassandra (LSM, tiered):
     Excelente writes. Reads y space son el costo.
     → Optimizado para ingestión masiva de datos.

  3. PostgreSQL (B-Tree):
     Excelente reads y space. Writes son el costo.
     → Optimizado para queries interactivos.

  4. Redis (hash table in-memory):
     Excelente reads y writes. Space es el costo (todo en RAM).
     → Optimizado para baja latencia, sacrifica espacio.

  No hay estructura perfecta.
  La elección correcta depende del WORKLOAD.
```

**Preguntas:**

1. ¿La RUM conjecture es un resultado teórico o una observación empírica?

2. ¿Un hash table tiene ★★★ en read Y write.
   ¿Dónde está el tradeoff?

3. ¿Para un sistema que necesita reads rápidos Y writes rápidos
   Y poco espacio, ¿qué opciones hay?

4. ¿Tiered compaction favorece writes y leveled favorece reads.
   ¿Existe un punto medio?

5. ¿LSM-Tree con leveled compaction se parece a un B-Tree
   en rendimiento?

---

## Sección 10.7 — LSM-Trees en Producción: de Cassandra a Flink

### Ejercicio 10.7.1 — Analizar: el ecosistema de LSM-Trees

**Tipo: Analizar**

```
LSM-Trees dominan el almacenamiento moderno de write-heavy workloads:

  CASSANDRA:
    Storage engine: LSM-Tree con SSTables.
    MemTable: ConcurrentSkipListMap.
    Compaction: size-tiered (default), leveled (opción).
    Bloom filter: configurable (bloom_filter_fp_chance).
    Uso: time series, IoT, logs, messaging (Discord, Netflix, Apple).

  ROCKSDB:
    Storage engine embebido: LSM-Tree.
    MemTable: skip list con concurrent writes.
    Compaction: leveled (default), universal (similar a tiered).
    Bloom filter: block-based, configurable.
    Uso: Flink state backend, CockroachDB, TiDB, MyRocks (MySQL).

  LEVELDB:
    Predecesor de RocksDB (creado por Google, Jeff Dean & Sanjay Ghemawat).
    Simpler. Single-threaded compaction.
    Uso: Chrome (IndexedDB), Ethereum (state storage).

  HBASE:
    LSM-Tree sobre HDFS.
    MemTable: ConcurrentSkipListMap.
    SSTables: HFiles en HDFS.
    Compaction: size-tiered y optimized-bucketing.
    Uso: Facebook messages (originalmente), AdTech, genomics.

  SCYLLADB:
    Rewrite de Cassandra en C++.
    Misma arquitectura LSM-Tree, 10× mejor rendimiento.
    Uso: alto throughput donde Cassandra no es suficiente.

  BADGERDB (Go):
    LSM-Tree con value-log separation.
    Keys en LSM-Tree. Values en un append-only log separado.
    → Reduce write amplification de keys y values juntos.
    Uso: Dgraph (graph database).

  FLINK + ROCKSDB:
    Flink usa RocksDB como state backend para keyed state.
    Cada operador de Flink mantiene su estado en un LSM-Tree local.
    El state se checkpointa periódicamente a S3/HDFS.
    → El LSM-Tree permite writes rápidos de estado durante streaming.
    → Los checkpoints son snapshots del LSM-Tree.
```

**Preguntas:**

1. ¿RocksDB es el LSM-Tree embebido más popular.
   ¿Por qué no LevelDB?

2. ¿BadgerDB separa keys y values. ¿Eso reduce write amplification?

3. ¿Flink + RocksDB: el state backend es un LSM-Tree local.
   ¿Qué pasa si el nodo falla?

4. ¿ScyllaDB es 10× más rápido que Cassandra. ¿Solo por C++ vs Java?

5. ¿CockroachDB y TiDB usan RocksDB. ¿Son bases SQL sobre LSM-Tree?

---

### Ejercicio 10.7.2 — Analizar: B-Tree vs LSM-Tree — la decisión final

**Tipo: Analizar**

```
Guía de decisión:

  ┌─────────────────────────────────────────────────────────────┐
  │ ¿Tu workload es read-heavy o write-heavy?                  │
  │                                                             │
  │  READ-HEAVY (80%+ reads):                                   │
  │    → B-Tree (PostgreSQL, MySQL/InnoDB).                     │
  │    Reads directos en 1-2 I/Os. Latencia predecible.        │
  │    Writes más lentos pero aceptables.                       │
  │                                                             │
  │  WRITE-HEAVY (80%+ writes):                                 │
  │    → LSM-Tree (Cassandra, RocksDB, HBase).                 │
  │    Writes secuenciales, alto throughput.                    │
  │    Reads más lentos, pero Bloom filters ayudan.            │
  │                                                             │
  │  MIXED (50/50):                                             │
  │    → Depende de latencia vs throughput.                     │
  │    Baja latencia requerida → B-Tree.                       │
  │    Alto throughput requerido → LSM-Tree (leveled).          │
  │                                                             │
  │  APPEND-ONLY (logs, time series):                           │
  │    → LSM-Tree o Kafka (log puro).                          │
  │    Los datos se escriben una vez y se leen ocasionalmente.  │
  └─────────────────────────────────────────────────────────────┘

  Resumen por sistema:
    PostgreSQL:  B-Tree.  Para OLTP read-heavy, SQL, transactions.
    MySQL:       B-Tree.  Para OLTP general, web applications.
    Cassandra:   LSM-Tree. Para write-heavy, time series, global distribution.
    RocksDB:     LSM-Tree. Para embedding en otros sistemas (Flink, CockroachDB).
    HBase:       LSM-Tree. Para big data sobre HDFS.
    SQLite:      B-Tree.  Para embebido, mobile, single-file database.
    BoltDB:      B-Tree (COW). Para embebido en Go (etcd).
```

**Preguntas:**

1. ¿Para un data warehouse (OLAP), ¿ni B-Tree ni LSM-Tree?
   ¿Columnar stores?

2. ¿CockroachDB es SQL sobre RocksDB (LSM-Tree).
   ¿Es un compromiso?

3. ¿Para un sistema de logging (write-only, reads raros),
   ¿LSM-Tree o simplemente append files?

4. ¿Spark (Parquet) no usa ni B-Tree ni LSM-Tree.
   ¿Qué usa?

5. ¿En 10 años, ¿LSM-Trees reemplazarán a los B-Trees
   o coexistirán?

---

### Ejercicio 10.7.3 — Resumen: las reglas del LSM-Tree

**Tipo: Leer**

```
Reglas nuevas del Cap.10:

  Regla 41: LSM-Tree convierte random writes en sequential writes.
    Writes van a la MemTable (RAM) → flush como SSTable (sequential).
    100× más throughput de escritura que B-Tree en HDD.
    10× en SSD. La diferencia se reduce con NVMe pero persiste.

  Regla 42: El read path del LSM-Tree depende de Bloom filters.
    Sin Bloom filters: reads consultan TODOS los SSTables.
    Con Bloom filters: reads consultan 1-2 SSTables.
    El Bloom filter del Cap.06 es CRÍTICO para el rendimiento
    del LSM-Tree. Sin él, el LSM-Tree es inutilizable.

  Regla 43: Compaction es el costo de los writes baratos.
    Size-tiered: baja write amp (~10×), alta space amp (~2×).
    Leveled: alta write amp (~30×), baja space amp (~1.1×).
    No hay compaction gratis. Es el precio de los writes secuenciales.

  Regla 44: B-Tree para reads, LSM-Tree para writes.
    No hay estructura perfecta. La elección depende del workload.
    PostgreSQL (B-Tree): OLTP, transactions, queries interactivos.
    Cassandra (LSM-Tree): ingestión masiva, time series, logs.
    El tradeoff es fundamental, no accidental.

  Regla 45: El LSM-Tree integra 5 capítulos de este libro.
    MemTable = Red-Black tree / skip list (Cap.07).
    SSTable = sorted array en disco (Cap.03).
    Bloom filter para skip de SSTables (Cap.06).
    K-way merge para compaction (Cap.08).
    Hash function para partitioning (Cap.04).
    → El LSM-Tree es la primera estructura del libro
      que demuestra cómo TODO se conecta.

  La Parte 3 (Árboles) está completa.
    Cap.07: BST / AVL / Red-Black → datos ordenados en memoria.
    Cap.08: Heaps → acceso al extremo, merge de streams.
    Cap.09: B-Trees → datos ordenados en disco (read-optimized).
    Cap.10: LSM-Trees → datos en disco (write-optimized).

  Parte 4: Estructuras para Búsqueda y Texto.
    Cap.11: Tries → búsqueda por prefijo, routing tables.
    Cap.12: Inverted indexes → búsqueda full-text, Elasticsearch.
```

**Preguntas:**

1. ¿Las 45 reglas hasta ahora cubren fundamentos + hashing
   + árboles + almacenamiento?

2. ¿La Regla 45 (5 capítulos integrados) es la más importante
   del capítulo?

3. ¿La Regla 44 (B-Tree reads, LSM writes) debería memorizarse?

4. ¿Un data engineer que trabaja con Spark + Kafka + Cassandra
   interactúa directamente con LSM-Trees?

5. ¿El Cap.18 (key-value store) implementará un LSM-Tree completo?
