# Guía de Ejercicios — Cap.18: El Proyecto Final: Construir un Key-Value Store

> Este es el capítulo donde TODO se conecta.
>
> En 17 capítulos has estudiado las piezas:
> Cap.03: Arrays y memoria contigua → SSTable (sorted array en disco).
> Cap.04: Hash functions → partitioning de keys.
> Cap.06: Bloom filters → evitar lecturas innecesarias de SSTables.
> Cap.09: B+ Tree → índice on-disk para SSTables.
> Cap.10: LSM-Tree → arquitectura completa (MemTable + SSTable + WAL).
> Cap.12: Skip list → MemTable concurrent.
> Cap.13: Queues → background compaction.
> Cap.14: Concurrencia → RWLock para MemTable, mutex para WAL.
>
> Ahora las ensamblas en una sola pieza funcional:
> un key-value store que escribe datos de forma durable,
> los busca eficientemente, y los compacta en background.
>
> Es una versión simplificada de RocksDB / LevelDB.
> RocksDB es el motor detrás de Cassandra, CockroachDB,
> TiDB, y el state backend de Flink.
>
> Construirlo en Rust es un ejercicio de ingeniería completa:
> diseño, implementación, testing, benchmarking.
> No es un ejercicio académico. Es un sistema real, simplificado.

---

## El modelo mental: las piezas y cómo encajan

```
WRITE PATH:
  put(key, value)
    │
    ├─→ 1. Append to WAL (durabilidad)
    │      Archivo append-only. fsync para persistir.
    │      Si crash: replay WAL para reconstruir MemTable.
    │
    ├─→ 2. Insert en MemTable (velocidad)
    │      Skip list in-memory. Sorted para flush eficiente.
    │      Cuando size > threshold → flush.
    │
    └─→ 3. Flush MemTable → SSTable (persistencia)
           Escribir skip list como archivo sorted en disco.
           Crear Bloom filter para el SSTable.
           Crear index block (key → offset).
           MemTable se vacía. WAL se trunca.

READ PATH:
  get(key)
    │
    ├─→ 1. Buscar en MemTable (RAM)
    │      Si encontrada → retornar. DONE.
    │
    └─→ 2. Buscar en SSTables (disco, más reciente → más antiguo)
           Para cada SSTable:
             a) Bloom filter: ¿podría estar? Si no → skip.
             b) Index block: binary search → offset.
             c) Data block: leer y buscar.
           Si encontrada → retornar.
           Si tombstone → retornar None.
           Si ningún SSTable → retornar None.

COMPACTION (background):
  Cuando hay demasiados SSTables:
    Merge múltiples SSTables → 1 SSTable nuevo.
    Eliminar versiones antiguas y tombstones.
    Regenerar Bloom filter.
    → Menos SSTables → reads más rápidos.

COMPONENTES:
  ┌──────────────────────────────────────────────┐
  │                Key-Value Store                │
  │                                              │
  │  ┌────────┐  ┌──────────┐  ┌──────────────┐ │
  │  │  WAL   │  │ MemTable │  │  SSTable     │ │
  │  │(append)│  │(skip list)│  │  Manager     │ │
  │  └────────┘  └──────────┘  │ ┌──────────┐ │ │
  │                            │ │ SSTable 0 │ │ │
  │  ┌─────────────────────┐   │ │ SSTable 1 │ │ │
  │  │  Compaction Thread  │   │ │ SSTable 2 │ │ │
  │  │  (background merge) │   │ └──────────┘ │ │
  │  └─────────────────────┘   └──────────────┘ │
  └──────────────────────────────────────────────┘
```

---

## Tabla de contenidos

- [Sección 18.1 — Diseño: las decisiones de arquitectura](#sección-181--diseño-las-decisiones-de-arquitectura)
- [Sección 18.2 — WAL: durabilidad ante crashes](#sección-182--wal-durabilidad-ante-crashes)
- [Sección 18.3 — MemTable: skip list in-memory](#sección-183--memtable-skip-list-in-memory)
- [Sección 18.4 — SSTable: sorted file con Bloom filter e index](#sección-184--sstable-sorted-file-con-bloom-filter-e-index)
- [Sección 18.5 — Ensamblar: el key-value store completo](#sección-185--ensamblar-el-key-value-store-completo)
- [Sección 18.6 — Compaction y testing](#sección-186--compaction-y-testing)
- [Sección 18.7 — Benchmark y análisis: qué simplificamos vs RocksDB](#sección-187--benchmark-y-análisis-qué-simplificamos-vs-rocksdb)

---

## Sección 18.1 — Diseño: las Decisiones de Arquitectura

### Ejercicio 18.1.1 — Leer: las decisiones de diseño

**Tipo: Leer**

```
Decisiones de diseño para nuestro key-value store:

  LENGUAJE: Rust.
    ¿Por qué? Performance + ownership model.
    Arc<RwLock<MemTable>> para concurrent reads/writes.
    Sin GC: control total de la memoria.
    (Alternativa: Java con ConcurrentSkipListMap. Más simple.)

  MEMTABLE: BTreeMap de Rust (o skip list custom del Cap.12).
    Threshold: 4 MB. Cuando la MemTable supera 4 MB → flush.
    En producción (RocksDB): 64 MB default.

  WAL: archivo append-only.
    Formato: [key_len: u32][val_len: u32][key: bytes][val: bytes]
    fsync después de cada write (durabilidad máxima).
    En producción: fsync por batch (más throughput, menos durabilidad).

  SSTABLE: archivo sorted con 3 secciones.
    [Data section: key-value pairs sorted]
    [Index section: key → offset del data section]
    [Bloom filter: serialized]
    [Footer: data_offset, index_offset, bloom_offset, entry_count]

  BLOOM FILTER: 10 bits por key, 7 hash functions.
    False positive rate ~0.8%.
    En producción (RocksDB): configurable, typically 10 bits/key.

  COMPACTION: simplificada.
    Cuando hay ≥ 4 SSTables → merge todos en 1.
    En producción: leveled compaction con múltiples niveles.

  CONCURRENCY:
    MemTable: RwLock (múltiples readers, un writer).
    WAL: Mutex (un writer a la vez).
    SSTables: inmutables → no necesitan locks.
    Compaction: background thread.

  LIMITACIONES (vs RocksDB):
    - Sin leveled compaction (solo full merge).
    - Sin compression (Snappy/LZ4/Zstd).
    - Sin snapshots ni transactions.
    - Sin column families.
    - Sin rate limiting para compaction.
    - Sin block cache.
```

**Preguntas:**

1. ¿4 MB MemTable vs 64 MB de RocksDB. ¿La diferencia importa?

2. ¿fsync por write vs por batch. ¿Cuánta diferencia en throughput?

3. ¿Sin compression: ¿cuánto más grande son los SSTables?

4. ¿Sin leveled compaction: ¿cuánto peores son los reads?

5. ¿En cuántas líneas de Rust se implementa este diseño?

---

## Sección 18.2 — WAL: Durabilidad ante Crashes

### Ejercicio 18.2.1 — Implementar: WAL en Rust

**Tipo: Implementar**

```rust
use std::fs::{File, OpenOptions};
use std::io::{self, BufWriter, Read, Seek, SeekFrom, Write};
use std::path::Path;

pub struct WAL {
    writer: BufWriter<File>,
    path: String,
}

impl WAL {
    pub fn new(path: &str) -> io::Result<Self> {
        let file = OpenOptions::new()
            .create(true).append(true).open(path)?;
        Ok(WAL {
            writer: BufWriter::new(file),
            path: path.to_string(),
        })
    }

    pub fn append(&mut self, key: &[u8], value: &[u8]) -> io::Result<()> {
        let key_len = key.len() as u32;
        let val_len = value.len() as u32;
        self.writer.write_all(&key_len.to_le_bytes())?;
        self.writer.write_all(&val_len.to_le_bytes())?;
        self.writer.write_all(key)?;
        self.writer.write_all(value)?;
        self.writer.flush()?;
        // In production: fsync here for durability
        // self.writer.get_ref().sync_data()?;
        Ok(())
    }

    pub fn replay(&self) -> io::Result<Vec<(Vec<u8>, Vec<u8>)>> {
        let mut file = File::open(&self.path)?;
        let mut entries = Vec::new();
        let mut buf4 = [0u8; 4];
        loop {
            if file.read_exact(&mut buf4).is_err() { break; }
            let key_len = u32::from_le_bytes(buf4) as usize;
            if file.read_exact(&mut buf4).is_err() { break; }
            let val_len = u32::from_le_bytes(buf4) as usize;
            let mut key = vec![0u8; key_len];
            let mut val = vec![0u8; val_len];
            file.read_exact(&mut key)?;
            file.read_exact(&mut val)?;
            entries.push((key, val));
        }
        Ok(entries)
    }

    pub fn clear(&mut self) -> io::Result<()> {
        let file = OpenOptions::new()
            .write(true).truncate(true).open(&self.path)?;
        self.writer = BufWriter::new(file);
        Ok(())
    }
}
```

**Preguntas:**

1. ¿`BufWriter` bufferea escrituras. ¿`flush` garantiza que llega al OS?

2. ¿`sync_data()` es fsync. ¿Cuánto impacta el performance?

3. ¿`replay` reconstruye la MemTable después de un crash?

4. ¿El WAL crece indefinidamente. ¿Cuándo se trunca?

5. ¿Este WAL es conceptualmente idéntico al de PostgreSQL?

---

## Sección 18.3 — MemTable: Skip List In-Memory

### Ejercicio 18.3.1 — Implementar: MemTable con BTreeMap de Rust

**Tipo: Implementar**

```rust
use std::collections::BTreeMap;

pub struct MemTable {
    data: BTreeMap<Vec<u8>, Vec<u8>>,
    size_bytes: usize,
}

impl MemTable {
    pub fn new() -> Self {
        MemTable { data: BTreeMap::new(), size_bytes: 0 }
    }

    pub fn put(&mut self, key: Vec<u8>, value: Vec<u8>) {
        let entry_size = key.len() + value.len() + 16; // overhead estimate
        if let Some(old) = self.data.insert(key.clone(), value) {
            self.size_bytes -= old.len();
        }
        self.size_bytes += entry_size;
    }

    pub fn get(&self, key: &[u8]) -> Option<&Vec<u8>> {
        self.data.get(key)
    }

    pub fn delete(&mut self, key: Vec<u8>) {
        // Tombstone: special value
        self.put(key, b"__TOMBSTONE__".to_vec());
    }

    pub fn is_full(&self, threshold: usize) -> bool {
        self.size_bytes >= threshold
    }

    pub fn size_bytes(&self) -> usize { self.size_bytes }
    pub fn len(&self) -> usize { self.data.len() }

    pub fn iter(&self) -> impl Iterator<Item = (&Vec<u8>, &Vec<u8>)> {
        self.data.iter()
    }

    pub fn clear(&mut self) {
        self.data.clear();
        self.size_bytes = 0;
    }
}
```

**Preguntas:**

1. ¿BTreeMap de Rust vs ConcurrentSkipListMap de Java. ¿Cuál es mejor?

2. ¿`size_bytes` es un estimado. ¿Cómo lo calcula RocksDB?

3. ¿Tombstone como valor especial. ¿Hay una forma más limpia?

4. ¿`iter()` retorna en orden sorted. ¿Es crucial para el flush?

5. ¿Para concurrent access: `Arc<RwLock<MemTable>>`. ¿Es suficiente?

---

## Sección 18.4 — SSTable: Sorted File con Bloom Filter e Index

### Ejercicio 18.4.1 — Implementar: SSTable writer y reader

**Tipo: Implementar**

```rust
use std::collections::HashMap;
use std::fs::File;
use std::io::{self, BufReader, BufWriter, Read, Seek, SeekFrom, Write};

// Simplified Bloom filter (from Cap.06)
pub struct BloomFilter {
    bits: Vec<u8>,
    num_hashes: usize,
    num_bits: usize,
}

impl BloomFilter {
    pub fn new(expected_items: usize, bits_per_item: usize) -> Self {
        let num_bits = expected_items * bits_per_item;
        BloomFilter {
            bits: vec![0u8; (num_bits + 7) / 8],
            num_hashes: 7,
            num_bits,
        }
    }

    fn positions(&self, key: &[u8]) -> Vec<usize> {
        let mut h1 = 0u64;
        let mut h2 = 0u64;
        for (i, &b) in key.iter().enumerate() {
            h1 = h1.wrapping_mul(31).wrapping_add(b as u64);
            h2 = h2.wrapping_mul(37).wrapping_add(b as u64).wrapping_add(i as u64);
        }
        (0..self.num_hashes)
            .map(|i| (h1.wrapping_add((i as u64).wrapping_mul(h2)) % self.num_bits as u64) as usize)
            .collect()
    }

    pub fn insert(&mut self, key: &[u8]) {
        for pos in self.positions(key) {
            self.bits[pos / 8] |= 1 << (pos % 8);
        }
    }

    pub fn might_contain(&self, key: &[u8]) -> bool {
        self.positions(key).iter()
            .all(|&pos| self.bits[pos / 8] & (1 << (pos % 8)) != 0)
    }

    pub fn to_bytes(&self) -> Vec<u8> {
        let mut out = Vec::new();
        out.extend(&(self.num_bits as u32).to_le_bytes());
        out.extend(&(self.num_hashes as u32).to_le_bytes());
        out.extend(&self.bits);
        out
    }

    pub fn from_bytes(data: &[u8]) -> Self {
        let num_bits = u32::from_le_bytes(data[0..4].try_into().unwrap()) as usize;
        let num_hashes = u32::from_le_bytes(data[4..8].try_into().unwrap()) as usize;
        BloomFilter {
            bits: data[8..].to_vec(),
            num_hashes,
            num_bits,
        }
    }
}

pub struct SSTableWriter {
    path: String,
}

impl SSTableWriter {
    pub fn write(path: &str, entries: &[(&[u8], &[u8])]) -> io::Result<()> {
        let mut file = BufWriter::new(File::create(path)?);
        let mut index: Vec<(Vec<u8>, u64)> = Vec::new();
        let mut bloom = BloomFilter::new(entries.len().max(1), 10);
        let data_start = 0u64;

        // Write data section
        let mut offset = 0u64;
        for &(key, value) in entries {
            bloom.insert(key);
            index.push((key.to_vec(), offset));
            let key_len = key.len() as u32;
            let val_len = value.len() as u32;
            file.write_all(&key_len.to_le_bytes())?;
            file.write_all(&val_len.to_le_bytes())?;
            file.write_all(key)?;
            file.write_all(value)?;
            offset += 8 + key.len() as u64 + value.len() as u64;
        }
        let index_start = offset;

        // Write index section
        let index_count = index.len() as u32;
        file.write_all(&index_count.to_le_bytes())?;
        for (key, off) in &index {
            file.write_all(&(key.len() as u32).to_le_bytes())?;
            file.write_all(key)?;
            file.write_all(&off.to_le_bytes())?;
        }
        let bloom_start = file.seek(SeekFrom::Current(0))? as u64;

        // Write bloom filter
        let bloom_bytes = bloom.to_bytes();
        file.write_all(&bloom_bytes)?;

        // Write footer
        file.write_all(&data_start.to_le_bytes())?;
        file.write_all(&index_start.to_le_bytes())?;
        file.write_all(&bloom_start.to_le_bytes())?;
        file.write_all(&(entries.len() as u64).to_le_bytes())?;

        file.flush()?;
        Ok(())
    }
}

// SSTable reader would load the index + bloom filter into memory,
// then do point lookups by:
// 1. Check bloom filter
// 2. Binary search index for offset
// 3. Seek to offset, read key-value pair
```

**Preguntas:**

1. ¿El index se carga en memoria. ¿Para SSTables de 256 MB?

2. ¿Bloom filter 10 bits/key para 100K keys = 125 KB. ¿Aceptable?

3. ¿El footer tiene offsets fijos al final. ¿Cómo se lee?

4. ¿Binary search en el index: O(log n). ¿Es suficiente?

5. ¿RocksDB usa block index (sparse). ¿Es diferente?

---

## Sección 18.5 — Ensamblar: el Key-Value Store Completo

### Ejercicio 18.5.1 — Implementar: KVStore que integra WAL + MemTable + SSTables

**Tipo: Implementar**

```rust
// Pseudocódigo estructural — la implementación completa
// combina los módulos de las secciones anteriores.

pub struct KVStore {
    memtable: Arc<RwLock<MemTable>>,
    wal: Arc<Mutex<WAL>>,
    sstables: Arc<RwLock<Vec<SSTableReader>>>,
    db_path: String,
    memtable_threshold: usize,  // 4 MB
    sstable_counter: AtomicU64,
}

impl KVStore {
    pub fn open(path: &str) -> io::Result<Self> {
        // 1. Create directory if not exists
        // 2. Open WAL, replay entries into MemTable
        // 3. Load existing SSTables (sorted by creation time)
        // 4. Return KVStore ready for use
    }

    pub fn put(&self, key: &[u8], value: &[u8]) -> io::Result<()> {
        // 1. Append to WAL (under mutex)
        {
            let mut wal = self.wal.lock().unwrap();
            wal.append(key, value)?;
        }
        // 2. Insert into MemTable (under write lock)
        let should_flush;
        {
            let mut mt = self.memtable.write().unwrap();
            mt.put(key.to_vec(), value.to_vec());
            should_flush = mt.is_full(self.memtable_threshold);
        }
        // 3. Flush if threshold reached
        if should_flush {
            self.flush()?;
        }
        Ok(())
    }

    pub fn get(&self, key: &[u8]) -> io::Result<Option<Vec<u8>>> {
        // 1. Check MemTable (under read lock)
        {
            let mt = self.memtable.read().unwrap();
            if let Some(val) = mt.get(key) {
                if val == b"__TOMBSTONE__" { return Ok(None); }
                return Ok(Some(val.clone()));
            }
        }
        // 2. Check SSTables (newest first)
        {
            let tables = self.sstables.read().unwrap();
            for sst in tables.iter().rev() {
                if let Some(val) = sst.get(key)? {
                    if val == b"__TOMBSTONE__" { return Ok(None); }
                    return Ok(Some(val));
                }
            }
        }
        Ok(None)
    }

    pub fn delete(&self, key: &[u8]) -> io::Result<()> {
        self.put(key, b"__TOMBSTONE__")
    }

    fn flush(&self) -> io::Result<()> {
        // 1. Swap MemTable: create new empty, take the full one
        // 2. Write full MemTable as SSTable
        // 3. Add SSTable to the list
        // 4. Clear WAL
        // 5. Trigger compaction if too many SSTables
        Ok(())
    }

    pub fn compact(&self) -> io::Result<()> {
        // 1. Take all SSTables
        // 2. K-way merge (all entries, newer wins)
        // 3. Remove tombstones
        // 4. Write as 1 new SSTable
        // 5. Replace old SSTables with new one
        // 6. Delete old SSTable files
        Ok(())
    }
}
```

**Preguntas:**

1. ¿`Arc<RwLock<MemTable>>` permite concurrent reads durante writes?

2. ¿El flush swapea la MemTable. ¿Los writes durante el flush?

3. ¿Los SSTables son inmutables. ¿Por qué no necesitan locks?

4. ¿`get` busca en MemTable primero, luego SSTables. ¿Siempre?

5. ¿Este diseño es exactamente lo que hace LevelDB?

---

## Sección 18.6 — Compaction y Testing

### Ejercicio 18.6.1 — Analizar: testing del key-value store

**Tipo: Analizar**

```
TESTS NECESARIOS:

  1. CORRECTNESS (funcional):
     - put + get retorna el valor correcto.
     - put + put (update) retorna el último valor.
     - delete + get retorna None.
     - get de key inexistente retorna None.
     - Muchos put + verificar todos con get.

  2. PERSISTENCE (durabilidad):
     - put varios keys → cerrar el DB → reabrir → get retorna valores.
     - El WAL replay reconstruye la MemTable.
     - Después de flush: datos en SSTable persisten sin WAL.

  3. CRASH RECOVERY:
     - Simular crash (kill process) después de WAL write pero antes de flush.
     - Reabrir → WAL replay → datos disponibles.
     - Simular crash durante flush → estado consistente.

  4. COMPACTION:
     - Múltiples SSTables con keys duplicadas → compaction → 1 SSTable.
     - Tombstones eliminados durante compaction.
     - get correcto antes y después de compaction.

  5. CONCURRENCY:
     - N threads haciendo put simultáneamente.
     - N threads haciendo get mientras otros hacen put.
     - Flush durante reads concurrentes.

  6. PERFORMANCE:
     - Throughput: writes/sec, reads/sec.
     - Latencia: p50, p99 de get y put.
     - Comparar con RocksDB (la referencia).

  Resultado esperado de benchmark:
    Nuestro KV store:     ~100K-500K writes/sec, ~200K-1M reads/sec.
    RocksDB:              ~500K-2M writes/sec, ~1M-5M reads/sec.
    Diferencia: 2-5× más lento que RocksDB.
    → Aceptable para un proyecto educativo.
    → La diferencia viene de: compression, block cache,
      leveled compaction, concurrent compaction, SIMD optimizations.
```

**Preguntas:**

1. ¿2-5× más lento que RocksDB. ¿Qué optimizaciones faltan?

2. ¿Crash recovery: ¿cómo verificamos que funciona sin realmente crashear?

3. ¿Concurrent reads durante flush: ¿la MemTable inmutable es accesible?

4. ¿Para testing de persistence: ¿necesitamos archivos reales o mock?

5. ¿RocksDB tiene millones de líneas. ¿Nuestro store tiene cuántas?

---

## Sección 18.7 — Benchmark y Análisis: Qué Simplificamos vs RocksDB

### Ejercicio 18.7.1 — Analizar: nuestro store vs RocksDB

**Tipo: Analizar**

```
LO QUE IMPLEMENTAMOS:
  ✓ WAL para durabilidad.
  ✓ MemTable (BTreeMap sorted) para writes rápidos.
  ✓ SSTable con index block y Bloom filter.
  ✓ Read path: MemTable → Bloom filter → SSTables.
  ✓ Tombstones para deletes.
  ✓ Full compaction (merge all SSTables).
  ✓ Concurrent reads con RwLock.

LO QUE SIMPLIFICAMOS:
  ✗ Leveled compaction: múltiples niveles con size ratio.
    → Nuestro full merge es O(n) cada vez. Leveled amortiza.
  ✗ Compression (Snappy/LZ4/Zstd): reduce I/O 2-5×.
  ✗ Block cache: cachear bloques de SSTable en memoria.
  ✗ Rate limiter: limitar I/O de compaction para no afectar reads.
  ✗ Column families: múltiples "tablas" en un DB.
  ✗ Snapshots: leer un estado consistente en un punto en el tiempo.
  ✗ Transactions: put/get atómicos de múltiples keys.
  ✗ Prefix Bloom filter: Bloom filter para prefijos de keys.
  ✗ Compaction filters: lógica custom durante compaction.
  ✗ SSTable compression: Snappy por defecto en RocksDB.
  ✗ Direct I/O: bypass OS page cache para control de memoria.
  ✗ SIMD: operaciones vectorizadas para comparación de keys.

MAPA DE CAPÍTULOS INTEGRADOS:
  Cap.03 (Arrays)         → SSTable = sorted array en disco.
  Cap.04 (Hash)           → Hash functions en Bloom filter.
  Cap.06 (Bloom filter)   → Bloom filter por SSTable.
  Cap.09 (B+ Tree)        → Index block del SSTable.
  Cap.10 (LSM-Tree)       → Arquitectura completa.
  Cap.12 (Skip list)      → MemTable (o BTreeMap como proxy).
  Cap.13 (Queues)         → Background compaction thread.
  Cap.14 (Concurrency)    → RwLock, Mutex, Arc.

  8 de 17 capítulos están directamente representados.
  Los otros 9 proveen contexto y fundamentos:
  Cap.01 (Memoria), Cap.02 (Complejidad), Cap.05 (Hash avanzado),
  Cap.07 (BST/AVL/RBT), Cap.08 (Heaps), Cap.11 (Tries),
  Cap.15 (Distribuido), Cap.16 (Columnar), Cap.17 (Índices).
```

**Preguntas:**

1. ¿8 de 17 capítulos integrados directamente. ¿Es suficiente?

2. ¿Agregar compression (LZ4) sería la mejora de mayor impacto?

3. ¿Block cache mejoraría reads pero usaría más memoria?

4. ¿Leveled compaction vs full merge: ¿cuánta diferencia para reads?

5. ¿Si alguien implementa este proyecto completo en Rust,
   ¿cuántas líneas de código serían?

---

### Ejercicio 18.7.2 — Resumen: las reglas finales y el mapa completo

**Tipo: Leer**

```
Reglas finales del Cap.18:

  Regla 76: Un key-value store es WAL + MemTable + SSTables.
    WAL: durabilidad (append-only, fsync).
    MemTable: writes rápidos (sorted in-memory).
    SSTable: reads eficientes (sorted on-disk + Bloom filter + index).
    → Estas tres piezas, juntas, son el motor de almacenamiento
      detrás de Cassandra, RocksDB, LevelDB, y Flink state.

  Regla 77: Las estructuras de datos no existen en aislamiento.
    Un sistema real COMBINA múltiples estructuras:
    Skip list + Bloom filter + sorted array + hash function + WAL.
    Cada estructura resuelve UN problema. El sistema resuelve TODOS.
    → La ingeniería es elegir y combinar las piezas correctas.

  Regla 78: La diferencia entre "funciona" y "producción" es 10×.
    Nuestro store: ~100-500K ops/sec. RocksDB: ~1-5M ops/sec.
    La diferencia: compression, block cache, leveled compaction,
    concurrent compaction, SIMD, direct I/O, tuning.
    → Entender la arquitectura es el 80%. El tuning es el otro 20%.

  Regla 79: Cada sistema de datos es una combinación de estas estructuras.
    PostgreSQL = B+ Tree + heap + WAL + buffer pool.
    Cassandra = consistent hashing + LSM-Tree + Bloom filter + Merkle tree.
    Elasticsearch = inverted index + FST + BKD tree + doc values.
    Kafka = append-only log + index files + page cache.
    Spark = columnar (Parquet/Arrow) + hash partitioning + sort-merge.
    → Conocer las estructuras es conocer CÓMO funcionan estos sistemas.

  Regla 80: Las estructuras de datos son el lenguaje de los sistemas.
    Cuando alguien dice "B+ Tree index", sabes qué significa.
    Cuando alguien dice "LSM-Tree con leveled compaction", sabes el tradeoff.
    Cuando alguien dice "Bloom filter con 1% FP", sabes el costo.
    → Este libro te dio el vocabulario. Los sistemas te dan el contexto.

═══ MAPA COMPLETO: 80 reglas en 18 capítulos ═══

  Parte 1 — Fundamentos (Cap.01-03): Reglas 1-12
    Memoria, complejidad, arrays. El hardware importa.

  Parte 2 — Hashing (Cap.04-06): Reglas 13-25
    Hash maps, concurrent maps, Bloom filters, HyperLogLog.

  Parte 3 — Árboles (Cap.07-10): Reglas 26-45
    BST, AVL, Red-Black, B-Tree, B+ Tree, LSM-Tree.

  Parte 4 — Búsqueda y Texto (Cap.11-12): Reglas 46-53
    Tries, radix trees, suffix arrays, skip lists.

  Parte 5 — Concurrencia y Distribución (Cap.13-15): Reglas 54-67
    Queues, thread-safe patterns, consistent hashing, Merkle, CRDTs.

  Parte 6 — Datos a Escala (Cap.16-18): Reglas 68-80
    Columnar layouts, índices unificados, key-value store completo.

  De la Regla 1 ("un int no es un número, es 4 bytes en una dirección")
  a la Regla 80 ("las estructuras de datos son el lenguaje de los sistemas"):
  un camino completo desde la memoria hasta los sistemas distribuidos.
```

**Preguntas:**

1. ¿80 reglas cubren todo lo esencial de estructuras para un data engineer?

2. ¿Si solo pudieras memorizar 10 reglas, ¿cuáles serían?

3. ¿El mapa de 18 capítulos: ¿hay huecos importantes?

4. ¿Después de este libro, ¿qué sigue para profundizar?

5. ¿Este proyecto final se puede extender como un side project real?
