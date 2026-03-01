# Guía de Ejercicios — Cap.06: Bloom Filters y Estructuras Probabilísticas

> Cassandra recibe una query: ¿existe la key "user_789012" en esta SSTable?
>
> La SSTable tiene 10 millones de entries y está en disco.
> Leerla para verificar cuesta ~10 ms de I/O.
> Y hay 20 SSTables por nodo.
> Sin optimización: 20 × 10 ms = 200 ms por lookup. Inaceptable.
>
> Solución: un Bloom filter de 10 MB en memoria por SSTable.
> Verificar si la key PODRÍA estar: ~200 ns. En memoria. Sin I/O.
> Si dice "no" → skip this SSTable. Ahorro: 10 ms.
> Si dice "probablemente sí" → leer la SSTable (quizás false positive).
>
> Con Bloom filters, Cassandra reduce el número de lecturas a disco
> de 20 a 1-2 por query. De 200 ms a 10-20 ms.
>
> El truco: el Bloom filter NUNCA dice "sí" cuando la respuesta es "no".
> Pero PUEDE decir "probablemente sí" cuando la respuesta es "no"
> (false positive). La tasa de error es configurable: 1%, 0.1%, 0.01%.
>
> Este capítulo implementa las tres estructuras probabilísticas
> más importantes de la infraestructura de datos:
> 1. Bloom filter: "¿pertenece X al conjunto?" (Cap.03 bitset + Cap.04 hash)
> 2. Counting Bloom filter: Bloom con soporte de delete.
> 3. HyperLogLog: "¿cuántos elementos distintos hay?" con 12 KB de memoria.

---

## El modelo mental: exactitud vs memoria

```
Pregunta: ¿el elemento X pertenece al conjunto S?

  RESPUESTA EXACTA (HashSet):
    Almacenar todos los elementos.
    1 billón de strings de 20 bytes = 20 GB + overhead ≈ 40+ GB.
    Respuesta: 100% correcta. Siempre.

  RESPUESTA PROBABILÍSTICA (Bloom filter):
    Almacenar "huellas" de los elementos en un bitset.
    1 billón de elementos con 1% de false positive = 1.2 GB.
    Respuesta: "definitivamente no" (100% correcto) o
               "probablemente sí" (99% correcto, 1% false positive).

  Tradeoff:
    40 GB → respuesta exacta.
    1.2 GB → respuesta 99% correcta.
    33× menos memoria por 1% de error.

  ¿Cuándo es aceptable?
  - Cassandra: "¿esta SSTable contiene la key?"
    False positive: leemos una SSTable innecesariamente (10 ms extra).
    False negative: perdemos datos (inaceptable).
    → Bloom filter perfecto: false positives son baratos,
      false negatives son inaceptables.

  - Chrome Safe Browsing: "¿esta URL está en la lista negra?"
    False positive: bloqueamos una URL legítima (molesto pero reversible).
    False negative: dejamos pasar una URL maliciosa (peligroso).
    → Bloom filter: reduce 99% de las consultas al servidor.
      Los positivos se verifican con el servidor.

  - Parquet predicate pushdown: "¿este row group contiene value X?"
    False positive: leemos un row group innecesariamente.
    False negative: perdemos resultados del query.
    → Bloom filter: reduce I/O significativamente.

Pregunta diferente: ¿cuántos elementos DISTINTOS hay en el stream?

  RESPUESTA EXACTA (HashSet):
    Almacenar todos los elementos distintos.
    1 billón de IPs de usuario = ~16 GB.

  RESPUESTA APROXIMADA (HyperLogLog):
    12 KB de memoria. Error estándar: ~0.8%.
    Para 1 billón de IPs: responde "1,008,000,000" ± 0.8%.

  16 GB → exacto. 12 KB → 99.2% preciso.
  1,300,000× menos memoria.
```

---

## Tabla de contenidos

- [Sección 6.1 — Bloom filter: la idea y la matemática](#sección-61--bloom-filter-la-idea-y-la-matemática)
- [Sección 6.2 — Implementar un Bloom filter desde cero](#sección-62--implementar-un-bloom-filter-desde-cero)
- [Sección 6.3 — Tuning: elegir m y k para tu caso de uso](#sección-63--tuning-elegir-m-y-k-para-tu-caso-de-uso)
- [Sección 6.4 — Counting Bloom filter: soportar deletes](#sección-64--counting-bloom-filter-soportar-deletes)
- [Sección 6.5 — Cuckoo filter: la alternativa moderna](#sección-65--cuckoo-filter-la-alternativa-moderna)
- [Sección 6.6 — HyperLogLog: contar distintos con 12 KB](#sección-66--hyperloglog-contar-distintos-con-12-kb)
- [Sección 6.7 — Estructuras probabilísticas en producción](#sección-67--estructuras-probabilísticas-en-producción)

---

## Sección 6.1 — Bloom Filter: la Idea y la Matemática

### Ejercicio 6.1.1 — Leer: cómo funciona un Bloom filter

**Tipo: Leer**

```
Un Bloom filter combina:
  - Un bitset de m bits (Cap.03)
  - k funciones hash independientes (Cap.04)

  ADD(element):
    Para cada función hash h₁, h₂, ..., hₖ:
      bit_position = hᵢ(element) % m
      bitset[bit_position] = 1

    Ejemplo: m=16 bits, k=3 hash functions
    add("hello"):
      h₁("hello") % 16 = 3   → encender bit 3
      h₂("hello") % 16 = 7   → encender bit 7
      h₃("hello") % 16 = 11  → encender bit 11

    Bitset: 0001000100010000
            ^   ^   ^
            3   7   11

  CONTAINS(element):
    Para cada función hash h₁, h₂, ..., hₖ:
      bit_position = hᵢ(element) % m
      if bitset[bit_position] == 0: return DEFINITELY NOT
    return PROBABLY YES

    contains("hello"):
      bit 3 = 1 ✓, bit 7 = 1 ✓, bit 11 = 1 ✓ → PROBABLY YES

    contains("world"):
      h₁("world") % 16 = 3 (bit 3 = 1 ✓)
      h₂("world") % 16 = 9 (bit 9 = 0 ✗) → DEFINITELY NOT
      (Un solo bit en 0 es suficiente para confirmar ausencia.)

    contains("xyz"):
      h₁("xyz") % 16 = 3 (bit 3 = 1 — encendido por "hello")
      h₂("xyz") % 16 = 7 (bit 7 = 1 — encendido por "hello")
      h₃("xyz") % 16 = 11 (bit 11 = 1 — encendido por "hello")
      → PROBABLY YES ← ¡FALSE POSITIVE!
      "xyz" nunca fue insertado, pero sus k bits coinciden.

  Propiedades fundamentales:
    FALSE NEGATIVE:  IMPOSIBLE. Si el elemento fue insertado,
                     sus k bits están encendidos. Siempre.
    FALSE POSITIVE:  POSIBLE. Bits encendidos por otros elementos
                     pueden coincidir con los k bits del query.
    DELETE:          IMPOSIBLE. Apagar un bit podría afectar
                     a otros elementos que comparten ese bit.
```

**Preguntas:**

1. ¿Con m=16 y k=3, ¿cuántos elementos puedes insertar
   antes de que la mayoría de bits estén encendidos?

2. ¿Si todos los m bits están encendidos, ¿el Bloom filter
   retorna "PROBABLY YES" para CUALQUIER query?

3. ¿Aumentar k (más hash functions) siempre reduce los false positives?

4. ¿Un Bloom filter con k=1 es solo un bitset con hash.
   ¿Qué false positive rate tiene?

5. ¿"Definitivamente no" es más útil que "probablemente sí"?
   ¿En qué escenarios?

---

### Ejercicio 6.1.2 — Leer: la matemática del false positive rate

**Tipo: Leer**

```
Después de insertar n elementos en un Bloom filter de m bits con k hashes:

  Probabilidad de que un bit específico siga en 0:
    p = (1 - 1/m)^(kn) ≈ e^(-kn/m)

  Probabilidad de un false positive (todos los k bits encendidos para un elemento no insertado):
    FP = (1 - p)^k = (1 - e^(-kn/m))^k

  El k ÓPTIMO que minimiza FP para m y n dados:
    k_opt = (m/n) × ln(2) ≈ 0.693 × (m/n)

  Con k óptimo, la FP rate se simplifica:
    FP ≈ (0.6185)^(m/n)

  Tabla de referencia (k óptimo):

    m/n (bits/elem)   k óptimo   FP rate
    ────────────────   ────────   ───────
    4                  3          14.7%
    8                  6          2.15%
    10                 7          0.82%
    12                 8          0.31%
    16                 11         0.046%
    20                 14         0.006%
    24                 17         0.001%

  Ejemplos prácticos:

    CASO 1: Cassandra SSTable Bloom filter
    n = 10M entries, target FP = 1%
    m/n = 10 bits/entry → m = 100M bits = 12.5 MB
    k = 7
    → 12.5 MB de memoria evita ~99% de lecturas a disco innecesarias.

    CASO 2: Chrome Safe Browsing
    n = 1M URLs maliciosas, target FP = 0.01%
    m/n = 20 bits/URL → m = 20M bits = 2.5 MB
    k = 14
    → 2.5 MB en el navegador. 99.99% de queries respondidas localmente.

    CASO 3: Parquet column Bloom filter
    n = 1M valores distintos en un row group, target FP = 5%
    m/n = 6 bits/valor → m = 6M bits = 750 KB
    k = 4
    → 750 KB por columna por row group. Ahorra leer row groups de 128 MB.
```

**Preguntas:**

1. ¿10 bits por elemento da 0.82% de FP rate. ¿Es un buen tradeoff?

2. ¿Si duplicas m (bits), ¿el FP rate se reduce a la mitad?

3. ¿k=7 es la cantidad más común en producción. ¿Coincidencia?

4. ¿Para n=1 billón, ¿cuánta memoria necesita un Bloom filter con 1% FP?

5. ¿Existe un límite inferior teórico de memoria para un FP rate dado?

---

### Ejercicio 6.1.3 — Analizar: k hash functions sin k funciones distintas

**Tipo: Analizar**

```
Implementar k funciones hash independientes es costoso.
Truco: usar solo 2 hash functions y combinarlas.

  Kirsch-Mitzenmacher (2006):
    hᵢ(x) = h₁(x) + i × h₂(x)   para i = 0, 1, ..., k-1

  Demostrado: para Bloom filters, esta combinación lineal
  es tan buena como k funciones independientes.

  Implementación práctica:
    Calcular UN hash de 128 bits (ej: MurmurHash3_x64_128).
    Tomar los primeros 64 bits como h₁.
    Tomar los últimos 64 bits como h₂.
    Generar k hashes: hᵢ(x) = (h₁ + i × h₂) % m

  Costo: 1 hash computation en vez de k.
  Para k=7 con hash de 20 ns: 20 ns en vez de 140 ns.

  Esta es la técnica usada por:
  - Google Guava BloomFilter
  - Apache Spark BloomFilter
  - Cassandra BloomFilter
  - La mayoría de implementaciones de producción

  Alternativa: double hashing con split
    hash_128 = MurmurHash3(element)
    h₁ = upper_64_bits(hash_128)
    h₂ = lower_64_bits(hash_128)
    positions[i] = (h₁ + i * h₂) % m

  Alternativa: enhanced double hashing
    hᵢ(x) = h₁(x) + i × h₂(x) + i²   (término cuadrático extra)
    Dillinger & Manolios (2004): reduce correlaciones residuales.
```

**Preguntas:**

1. ¿Kirsch-Mitzenmacher funciona para CUALQUIER k
   o tiene un límite?

2. ¿Un hash de 128 bits partido en dos mitades de 64 bits
   — ¿son independientes?

3. ¿MurmurHash3_x64_128 es la opción estándar para Bloom filters?

4. ¿El término cuadrático i² mejora significativamente
   la distribución?

5. ¿En Rust, ¿cómo obtienes un hash de 128 bits
   del trait Hash?

---

## Sección 6.2 — Implementar un Bloom Filter Desde Cero

### Ejercicio 6.2.1 — Implementar: Bloom filter en Python

**Tipo: Implementar**

```python
import math
import hashlib
import struct

class BloomFilter:
    def __init__(self, expected_elements: int, fp_rate: float = 0.01):
        # Calcular m y k óptimos
        self.n = expected_elements
        self.fp_rate = fp_rate
        self.m = self._optimal_m(expected_elements, fp_rate)
        self.k = self._optimal_k(self.m, expected_elements)
        self.bitset = bytearray(self.m // 8 + 1)
        self.size = 0

    @staticmethod
    def _optimal_m(n: int, fp: float) -> int:
        """m = -(n × ln(fp)) / (ln(2))²"""
        return int(-n * math.log(fp) / (math.log(2) ** 2)) + 1

    @staticmethod
    def _optimal_k(m: int, n: int) -> int:
        """k = (m/n) × ln(2)"""
        return max(1, int(m / n * math.log(2) + 0.5))

    def _hashes(self, element: str) -> list[int]:
        """Generate k hash positions using double hashing"""
        data = element.encode('utf-8')
        # Use MD5 for 128 bits (not for security, just distribution)
        digest = hashlib.md5(data).digest()
        h1 = struct.unpack('<Q', digest[:8])[0]  # first 64 bits
        h2 = struct.unpack('<Q', digest[8:])[0]   # last 64 bits
        return [(h1 + i * h2) % self.m for i in range(self.k)]

    def add(self, element: str):
        for pos in self._hashes(element):
            self.bitset[pos // 8] |= (1 << (pos % 8))
        self.size += 1

    def contains(self, element: str) -> bool:
        """Returns True if element is PROBABLY in the set, False if DEFINITELY NOT"""
        for pos in self._hashes(element):
            if not (self.bitset[pos // 8] & (1 << (pos % 8))):
                return False
        return True

    def false_positive_rate(self) -> float:
        """Theoretical FP rate for current fill level"""
        return (1 - math.exp(-self.k * self.size / self.m)) ** self.k

    def fill_ratio(self) -> float:
        """Fraction of bits that are set"""
        set_bits = sum(bin(byte).count('1') for byte in self.bitset)
        return set_bits / self.m

    def memory_bytes(self) -> int:
        return len(self.bitset)


# ═══ Test ═══
n = 1_000_000
bf = BloomFilter(n, fp_rate=0.01)
print(f"Bloom filter: m={bf.m:,} bits ({bf.memory_bytes():,} bytes), k={bf.k}")
print(f"Bits per element: {bf.m / n:.1f}")

# Insert n elements
for i in range(n):
    bf.add(f"element_{i}")

print(f"Fill ratio: {bf.fill_ratio():.1%}")
print(f"Theoretical FP rate: {bf.false_positive_rate():.4%}")

# Test false positive rate empirically
test_size = 100_000
false_positives = 0
for i in range(n, n + test_size):
    if bf.contains(f"element_{i}"):
        false_positives += 1

empirical_fp = false_positives / test_size
print(f"Empirical FP rate: {empirical_fp:.4%} ({false_positives}/{test_size})")

# Verify zero false negatives
false_negatives = 0
for i in range(min(10_000, n)):
    if not bf.contains(f"element_{i}"):
        false_negatives += 1
print(f"False negatives: {false_negatives} (should be 0)")

# Output esperado:
# Bloom filter: m=9,585,059 bits (1,198,133 bytes), k=7
# Bits per element: 9.6
# Fill ratio: ~50% (ideal para k óptimo)
# Theoretical FP rate: ~1.0%
# Empirical FP rate: ~0.95-1.05% (close to theoretical)
# False negatives: 0 (guaranteed)
```

**Preguntas:**

1. ¿El fill ratio de ~50% es coincidencia o es lo esperado con k óptimo?

2. ¿MD5 para el hash — ¿es demasiado lento para producción?

3. ¿El FP rate empírico coincide con el teórico. ¿Siempre?

4. ¿`bytearray` de Python es eficiente como bitset?

5. ¿Para 1M elementos con 1% FP, usamos ~1.2 MB.
   ¿Un HashSet usaría cuánto?

---

### Ejercicio 6.2.2 — Implementar: Bloom filter en Java con Murmur3

**Tipo: Implementar**

```java
public class BloomFilter {

    private final long[] bits;  // long[] como bitset (64 bits por long)
    private final int m;        // número total de bits
    private final int k;        // número de hash functions
    private int size;

    public BloomFilter(int expectedElements, double fpRate) {
        this.m = optimalM(expectedElements, fpRate);
        this.k = optimalK(m, expectedElements);
        this.bits = new long[(m + 63) / 64];  // ceil division
        this.size = 0;
    }

    static int optimalM(int n, double fp) {
        return (int) Math.ceil(-n * Math.log(fp) / (Math.log(2) * Math.log(2)));
    }

    static int optimalK(int m, int n) {
        return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
    }

    // Murmur3-inspired hash (simplified 64-bit)
    private long murmurMix(long h) {
        h ^= h >>> 33;
        h *= 0xff51afd7ed558ccdL;
        h ^= h >>> 33;
        h *= 0xc4ceb9fe1a85ec53L;
        h ^= h >>> 33;
        return h;
    }

    private int[] hashPositions(String element) {
        byte[] data = element.getBytes(java.nio.charset.StandardCharsets.UTF_8);
        long hash1 = murmurMix(java.util.Arrays.hashCode(data));
        long hash2 = murmurMix(hash1 + 0x9E3779B97F4A7C15L);

        int[] positions = new int[k];
        for (int i = 0; i < k; i++) {
            long combined = hash1 + (long) i * hash2;
            positions[i] = (int) ((combined & 0x7FFFFFFFFFFFFFFFL) % m);
        }
        return positions;
    }

    public void add(String element) {
        for (int pos : hashPositions(element)) {
            bits[pos / 64] |= (1L << (pos % 64));
        }
        size++;
    }

    public boolean mightContain(String element) {
        for (int pos : hashPositions(element)) {
            if ((bits[pos / 64] & (1L << (pos % 64))) == 0) {
                return false;
            }
        }
        return true;
    }

    public int size() { return size; }
    public int bitSize() { return m; }
    public int hashCount() { return k; }
    public long memoryBytes() { return bits.length * 8L; }

    public double fillRatio() {
        int setBits = 0;
        for (long word : bits) setBits += Long.bitCount(word);
        return (double) setBits / m;
    }

    public static void main(String[] args) {
        int n = 10_000_000;
        var bf = new BloomFilter(n, 0.01);
        System.out.printf("m=%,d bits (%,d bytes), k=%d, bits/elem=%.1f%n",
            bf.bitSize(), bf.memoryBytes(), bf.hashCount(),
            (double) bf.bitSize() / n);

        // Insert
        long start = System.nanoTime();
        for (int i = 0; i < n; i++) bf.add("elem_" + i);
        long insertTime = System.nanoTime() - start;
        System.out.printf("Insert %,d: %d ms (%.0f ns/op)%n",
            n, insertTime / 1_000_000, (double) insertTime / n);

        // Test FP rate
        int testSize = 1_000_000;
        int fp = 0;
        start = System.nanoTime();
        for (int i = n; i < n + testSize; i++) {
            if (bf.mightContain("elem_" + i)) fp++;
        }
        long queryTime = System.nanoTime() - start;

        System.out.printf("Query %,d: %d ms (%.0f ns/op)%n",
            testSize, queryTime / 1_000_000, (double) queryTime / testSize);
        System.out.printf("Fill ratio: %.1f%%%n", bf.fillRatio() * 100);
        System.out.printf("False positives: %d/%,d = %.3f%%%n",
            fp, testSize, (double) fp / testSize * 100);
    }
}
// Output esperado:
// m=95,850,584 bits (11,981,323 bytes), k=7, bits/elem=9.6
// Insert 10,000,000: ~3000 ms (~300 ns/op)
// Query 1,000,000: ~300 ms (~300 ns/op)
// Fill ratio: ~50.0%
// False positives: ~10000/1,000,000 = ~1.000%
```

**Preguntas:**

1. ¿300 ns/op para add y query — ¿es competitivo con HashMap.get() (28 ns)?

2. ¿`long[]` como bitset — ¿es más eficiente que `java.util.BitSet`?

3. ¿`Long.bitCount()` usa la instrucción POPCNT del CPU?

4. ¿El hash Murmur3 simplificado tiene distribución suficientemente buena?

5. ¿Para 10M elementos, 12 MB de Bloom filter vs ~800 MB de HashMap.
   ¿67× menos memoria?

---

### Ejercicio 6.2.3 — Implementar: Bloom filter en Rust con zero allocations en query

**Tipo: Implementar**

```rust
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

pub struct BloomFilter {
    bits: Vec<u64>,
    m: usize,       // total bits
    k: usize,       // hash functions
    size: usize,    // elements added
}

impl BloomFilter {
    pub fn new(expected_elements: usize, fp_rate: f64) -> Self {
        let m = optimal_m(expected_elements, fp_rate);
        let k = optimal_k(m, expected_elements);
        let words = (m + 63) / 64;
        BloomFilter { bits: vec![0u64; words], m, k, size: 0 }
    }

    pub fn add<T: Hash>(&mut self, element: &T) {
        let (h1, h2) = double_hash(element);
        for i in 0..self.k {
            let pos = ((h1.wrapping_add((i as u64).wrapping_mul(h2))) % self.m as u64) as usize;
            self.bits[pos / 64] |= 1u64 << (pos % 64);
        }
        self.size += 1;
    }

    pub fn might_contain<T: Hash>(&self, element: &T) -> bool {
        let (h1, h2) = double_hash(element);
        for i in 0..self.k {
            let pos = ((h1.wrapping_add((i as u64).wrapping_mul(h2))) % self.m as u64) as usize;
            if self.bits[pos / 64] & (1u64 << (pos % 64)) == 0 {
                return false;
            }
        }
        true
    }

    pub fn fill_ratio(&self) -> f64 {
        let set: usize = self.bits.iter().map(|w| w.count_ones() as usize).sum();
        set as f64 / self.m as f64
    }

    pub fn memory_bytes(&self) -> usize { self.bits.len() * 8 }
    pub fn size(&self) -> usize { self.size }
    pub fn bit_count(&self) -> usize { self.m }
    pub fn hash_count(&self) -> usize { self.k }
}

fn double_hash<T: Hash>(element: &T) -> (u64, u64) {
    let mut h1 = DefaultHasher::new();
    element.hash(&mut h1);
    let hash1 = h1.finish();

    // Second hash: mix the first hash
    let hash2 = hash1.wrapping_mul(0x9E3779B97F4A7C15)
        .wrapping_add(0x517CC1B727220A95);
    (hash1, hash2)
}

fn optimal_m(n: usize, fp: f64) -> usize {
    (-(n as f64) * fp.ln() / (2.0f64.ln().powi(2))).ceil() as usize
}

fn optimal_k(m: usize, n: usize) -> usize {
    ((m as f64 / n as f64) * 2.0f64.ln()).round().max(1.0) as usize
}

fn main() {
    let n = 10_000_000;
    let mut bf = BloomFilter::new(n, 0.01);
    println!("m={} bits ({} bytes), k={}, bits/elem={:.1}",
        bf.bit_count(), bf.memory_bytes(), bf.hash_count(),
        bf.bit_count() as f64 / n as f64);

    // Insert
    let start = std::time::Instant::now();
    for i in 0..n {
        bf.add(&format!("elem_{}", i));
    }
    let insert_time = start.elapsed();
    println!("Insert {}: {:?} ({:.0} ns/op)",
        n, insert_time, insert_time.as_nanos() as f64 / n as f64);

    // Test FP
    let test_size = 1_000_000;
    let start = std::time::Instant::now();
    let mut fp = 0usize;
    for i in n..n + test_size {
        if bf.might_contain(&format!("elem_{}", i)) { fp += 1; }
    }
    let query_time = start.elapsed();

    println!("Query {}: {:?} ({:.0} ns/op)",
        test_size, query_time, query_time.as_nanos() as f64 / test_size as f64);
    println!("Fill ratio: {:.1}%", bf.fill_ratio() * 100.0);
    println!("False positives: {}/{} = {:.3}%",
        fp, test_size, fp as f64 / test_size as f64 * 100.0);
}
// cargo run --release
// Output esperado:
// Insert 10M: ~1500 ms (~150 ns/op)
// Query 1M: ~150 ms (~150 ns/op)
// Fill ratio: ~50%
// FP rate: ~1.0%
```

**Preguntas:**

1. ¿Rust es ~2× más rápido que Java para Bloom filter ops.
   ¿Por la ausencia de GC o por el hash function?

2. ¿`might_contain` no asigna heap (zero allocations).
   ¿Eso importa para un query path?

3. ¿`count_ones()` de Rust compila a POPCNT?

4. ¿El double hashing con `wrapping_mul` + constante
   produce hashes suficientemente independientes?

5. ¿Usar `&str` en vez de `format!()` haría el benchmark
   significativamente más rápido?

---

### Ejercicio 6.2.4 — Analizar: Bloom filter vs HashSet — el tradeoff completo

**Tipo: Analizar**

```
Comparación completa para n = 10M strings de 20 bytes:

  Métrica              HashSet (Java)     Bloom filter (1% FP)
  ─────────────        ──────────────     ────────────────────
  Memoria              ~800 MB            ~12 MB
  Add (ns/op)          ~55                ~300
  Contains (ns/op)     ~28 (hit)          ~300
  Contains (ns/op)     ~22 (miss)         ~300 (or ~50 early exit)
  False negatives      0                  0
  False positives      0                  ~1%
  Supports delete      Sí                 No
  Supports iterate     Sí                 No
  Supports get value   Sí                 No (solo membership)

  Cuándo usar Bloom filter en vez de HashSet:

  1. Memoria es el constraint (#1 razón).
     800 MB vs 12 MB = 67× menos memoria.
     Si no cabe en RAM, un HashSet no es opción.

  2. Solo necesitas membership test.
     "¿Existe esta key?" — no necesitas el value asociado.
     Bloom filter responde solo SÍ/NO.

  3. False positives son aceptables.
     Si un false positive causa una operación extra
     (ej: leer una SSTable innecesariamente) pero no un error,
     el Bloom filter es perfecto.

  4. Pre-filtro antes de una operación costosa.
     Bloom filter como primera línea: descarta 99% de negativos.
     Los positivos se verifican con la estructura exacta.
     → Reduce significativamente la carga en la estructura exacta.

  Cuándo NO usar Bloom filter:
  - Necesitas el valor asociado (usa hash map).
  - Necesitas iterar los elementos (usa hash set o array).
  - False positives no son aceptables (usa hash set).
  - Necesitas delete (usa counting Bloom filter o cuckoo filter).
```

**Preguntas:**

1. ¿67× menos memoria — ¿es siempre el ratio o depende del tipo de datos?

2. ¿El Bloom filter como pre-filtro antes de un HashSet
   — ¿cuándo tiene sentido?

3. ¿300 ns/op del Bloom filter es más lento que 28 ns del HashSet.
   ¿Eso importa si ahorra I/O de 10 ms?

4. ¿Un Bloom filter distribuido (cada nodo con su propio BF)
   funciona correctamente?

5. ¿Se puede usar un Bloom filter como "negative cache"
   para evitar queries a una base de datos?

---

### Ejercicio 6.2.5 — Implementar: verificar empíricamente la fórmula de FP rate

**Tipo: Implementar**

```python
import math

class SimpleBloom:
    def __init__(self, m, k):
        self.m = m
        self.k = k
        self.bits = bytearray(m // 8 + 1)

    def _positions(self, element):
        import hashlib, struct
        d = hashlib.md5(element.encode()).digest()
        h1 = struct.unpack('<Q', d[:8])[0]
        h2 = struct.unpack('<Q', d[8:])[0]
        return [(h1 + i * h2) % self.m for i in range(self.k)]

    def add(self, element):
        for p in self._positions(element):
            self.bits[p // 8] |= (1 << (p % 8))

    def contains(self, element):
        return all(self.bits[p // 8] & (1 << (p % 8)) for p in self._positions(element))

# Test: vary m/n and measure FP rate
n = 100_000
test_size = 50_000

print(f"{'m/n':>6}  {'k':>4}  {'Theory FP':>10}  {'Empirical FP':>12}  {'Match':>6}")
print("─" * 48)

for bits_per_elem in [4, 6, 8, 10, 12, 16, 20]:
    m = n * bits_per_elem
    k = max(1, round(bits_per_elem * math.log(2)))

    bf = SimpleBloom(m, k)
    for i in range(n):
        bf.add(f"in_{i}")

    # Test FP
    fp = sum(1 for i in range(test_size) if bf.contains(f"out_{i}"))
    empirical = fp / test_size
    theoretical = (1 - math.exp(-k * n / m)) ** k

    match = "✓" if abs(empirical - theoretical) / max(theoretical, 0.001) < 0.3 else "✗"
    print(f"{bits_per_elem:>6}  {k:>4}  {theoretical:>10.4%}  {empirical:>12.4%}  {match:>6}")

# Output esperado:
# m/n      k    Theory FP  Empirical FP   Match
# ────────────────────────────────────────────────
#    4      3     14.69%       14.5-15.0%      ✓
#    6      4      5.61%        5.3-5.9%       ✓
#    8      6      2.15%        2.0-2.3%       ✓
#   10      7      0.82%        0.7-0.9%       ✓
#   12      8      0.31%        0.25-0.35%     ✓
#   16     11      0.046%       0.03-0.06%     ✓
#   20     14      0.006%       0.00-0.01%     ✓
```

**Preguntas:**

1. ¿La fórmula teórica coincide con la empírica. ¿Siempre?

2. ¿Para m/n=20, el FP rate es ~0.006%. ¿Vale la pena
   gastar 2.5× más memoria que m/n=10 por 130× menos FP?

3. ¿El fill ratio es ~50% para k óptimo. ¿Es una propiedad general?

4. ¿Si la hash function tiene mala distribución,
   ¿el FP rate empírico será peor que el teórico?

5. ¿Esta verificación empírica debería ser un test unitario
   en toda implementación de Bloom filter?

---

## Sección 6.3 — Tuning: Elegir m y k Para tu Caso de Uso

### Ejercicio 6.3.1 — Implementar: calculadora de parámetros de Bloom filter

**Tipo: Implementar**

```python
import math

def bloom_params(n: int, fp_rate: float) -> dict:
    """Calculate optimal Bloom filter parameters"""
    m = int(math.ceil(-n * math.log(fp_rate) / (math.log(2) ** 2)))
    k = int(round(m / n * math.log(2)))
    k = max(1, k)
    memory_bytes = m // 8 + 1
    memory_mb = memory_bytes / (1024 * 1024)
    actual_fp = (1 - math.exp(-k * n / m)) ** k
    return {
        'n': n, 'target_fp': fp_rate,
        'm': m, 'k': k,
        'bits_per_elem': m / n,
        'memory_bytes': memory_bytes, 'memory_mb': memory_mb,
        'actual_fp': actual_fp,
    }

# ═══ Tabla de referencia para escenarios reales ═══
scenarios = [
    ("Cassandra SSTable (10M keys, 1% FP)", 10_000_000, 0.01),
    ("Cassandra SSTable (10M keys, 0.1% FP)", 10_000_000, 0.001),
    ("Chrome Safe Browsing (1M URLs, 0.01% FP)", 1_000_000, 0.0001),
    ("Parquet row group (1M values, 5% FP)", 1_000_000, 0.05),
    ("Parquet row group (1M values, 1% FP)", 1_000_000, 0.01),
    ("Big data dedup (1B records, 1% FP)", 1_000_000_000, 0.01),
    ("Big data dedup (1B records, 0.1% FP)", 1_000_000_000, 0.001),
    ("Email spam filter (10M addresses, 0.1% FP)", 10_000_000, 0.001),
]

print(f"{'Scenario':<48} {'n':>12} {'m (bits)':>14} {'k':>4} {'Memory':>10} {'FP rate':>10}")
print("─" * 104)

for name, n, fp in scenarios:
    p = bloom_params(n, fp)
    if p['memory_mb'] >= 1:
        mem_str = f"{p['memory_mb']:.1f} MB"
    else:
        mem_str = f"{p['memory_bytes'] // 1024} KB"
    print(f"{name:<48} {n:>12,} {p['m']:>14,} {p['k']:>4} {mem_str:>10} {p['actual_fp']:>10.4%}")
```

**Preguntas:**

1. ¿1 billón de records con 1% FP = ~1.2 GB.
   ¿Cabe en un solo nodo?

2. ¿Reducir FP de 1% a 0.1% cuesta cuánta memoria extra?

3. ¿Parquet con 5% FP es suficiente para predicate pushdown?

4. ¿El k varía de 4 a 14 entre escenarios.
   ¿El costo de k=14 es relevante?

5. ¿Para un servicio con 10 ms de latencia máxima,
   ¿un Bloom filter con k=14 es demasiado lento?

---

### Ejercicio 6.3.2 — Analizar: scaling — qué pasa cuando n crece más de lo esperado

**Tipo: Analizar**

```
Un Bloom filter se dimensiona para n elementos.
¿Qué pasa si insertas más que n?

  Con n_expected = 1M y FP_target = 1%:
    m = 9.6M bits, k = 7

  Si insertas 1M elementos: FP ≈ 1.0% ✓ (como se diseñó)
  Si insertas 2M elementos: FP ≈ 4.5% (4.5× peor)
  Si insertas 5M elementos: FP ≈ 28% (28× peor)
  Si insertas 10M elementos: FP ≈ 68% (casi inútil)

  El FP rate CRECE EXPONENCIALMENTE con el overfill.
  No falla gradualmente — falla dramáticamente.

  Soluciones:

  1. SCALABLE BLOOM FILTER (Almeida et al., 2007):
     Crear una serie de Bloom filters de tamaño creciente.
     Cuando el primero se llena, crear uno nuevo más grande.
     contains() consulta TODOS los filters.
     → Costo: múltiples lookups (uno por filter).
     → Ventaja: no necesitas saber n de antemano.

  2. OVER-PROVISIONING:
     Dimensionar para 2× o 3× el n esperado.
     Si n_real < n_expected: memoria desperdiciada pero FP bajo.
     → Simple. Lo más común en producción.

  3. REBUILD:
     Cuando el fill ratio supera 60%, crear un Bloom filter
     nuevo más grande y re-insertar todos los elementos.
     → Requiere acceso a los elementos originales (o almacenarlos aparte).

  En Cassandra: cada SSTable tiene su propio Bloom filter.
  Cuando una SSTable se compacta (merge), el Bloom filter se reconstruye.
  → No hay riesgo de overfill: el BF se dimensiona para la SSTable final.
```

**Preguntas:**

1. ¿FP de 68% con 10× overfill — ¿el Bloom filter es inútil?

2. ¿Over-provisioning 3× — ¿es 3× más memoria para "seguridad"?

3. ¿Scalable Bloom filter consulta todos los sub-filters.
   ¿Cuántos sub-filters son aceptables?

4. ¿Cassandra reconstruye BF en compaction. ¿Es costoso?

5. ¿Existe un "dynamic Bloom filter" que crece automáticamente
   como un dynamic array?

---

## Sección 6.4 — Counting Bloom Filter: Soportar Deletes

### Ejercicio 6.4.1 — Leer: por qué el Bloom filter estándar no soporta delete

**Tipo: Leer**

```
  add("hello"):  encender bits 3, 7, 11
  add("world"):  encender bits 7, 9, 14  (bit 7 compartido con "hello")

  Bitset: 0001001101010100
                ^
                bit 7: encendido por AMBOS "hello" y "world"

  delete("hello"): ¿apagar bits 3, 7, 11?
    Si apagamos bit 7: contains("world") retorna FALSE.
    ¡FALSE NEGATIVE! Inaceptable.

  Solución: COUNTING BLOOM FILTER.
  En vez de 1 bit por posición, usar un CONTADOR de c bits.

    add("hello"):  bits[3]++, bits[7]++, bits[11]++
    add("world"):  bits[7]++, bits[9]++, bits[14]++

    Contadores: [..., 3→1, ..., 7→2, ..., 9→1, ..., 11→1, ..., 14→1, ...]

    delete("hello"): bits[3]--, bits[7]--, bits[11]--
    Contadores: [..., 3→0, ..., 7→1, ..., 9→1, ..., 11→0, ..., 14→1, ...]

    contains("world"): bits[7] > 0 ✓, bits[9] > 0 ✓, bits[14] > 0 ✓ → YES ✓
    Correcto: "world" sigue presente.

  Tamaño del contador:
    c = 4 bits: soporta hasta 15 incrementos por posición.
    Probabilidad de overflow (>15) con n razonable: < 10^(-7).
    4 bits por posición = 4× la memoria de un Bloom filter estándar.

    Para n = 1M con 1% FP:
    Bloom filter estándar: ~1.2 MB
    Counting Bloom filter (4 bits): ~4.8 MB
    → 4× más memoria para soportar deletes.
```

**Preguntas:**

1. ¿4 bits por posición = 4× más memoria. ¿Vale la pena?

2. ¿Qué pasa si un contador hace overflow (supera 15)?

3. ¿Si haces delete de un elemento que nunca fue insertado,
   ¿corrompes el counting Bloom filter?

4. ¿Un counting Bloom filter con contadores de 8 bits
   soporta cuántos inserts antes de overflow?

5. ¿Cassandra usa counting Bloom filters? ¿O estándar?

---

### Ejercicio 6.4.2 — Implementar: counting Bloom filter en Java

**Tipo: Implementar**

```java
public class CountingBloomFilter {

    private final byte[] counters; // 4 bits per counter, 2 per byte
    private final int m;
    private final int k;
    private int size;

    public CountingBloomFilter(int expectedElements, double fpRate) {
        this.m = (int) Math.ceil(-expectedElements * Math.log(fpRate) / (Math.log(2) * Math.log(2)));
        this.k = Math.max(1, (int) Math.round((double) m / expectedElements * Math.log(2)));
        this.counters = new byte[(m + 1) / 2]; // 2 counters per byte (4 bits each)
        this.size = 0;
    }

    private int getCounter(int pos) {
        int byteIdx = pos / 2;
        if (pos % 2 == 0) return counters[byteIdx] & 0x0F;
        else return (counters[byteIdx] >>> 4) & 0x0F;
    }

    private void setCounter(int pos, int value) {
        int byteIdx = pos / 2;
        value = Math.min(value, 15); // clamp to 4 bits
        if (pos % 2 == 0) {
            counters[byteIdx] = (byte) ((counters[byteIdx] & 0xF0) | (value & 0x0F));
        } else {
            counters[byteIdx] = (byte) ((counters[byteIdx] & 0x0F) | ((value & 0x0F) << 4));
        }
    }

    private int[] hashPositions(String element) {
        int h1 = element.hashCode();
        int h2 = h1 * 0x5BD1E995 + 0x47B6137B;
        h1 ^= h1 >>> 16;
        h2 ^= h2 >>> 16;
        int[] pos = new int[k];
        for (int i = 0; i < k; i++) {
            pos[i] = Math.abs((h1 + i * h2) % m);
        }
        return pos;
    }

    public void add(String element) {
        for (int pos : hashPositions(element)) {
            int current = getCounter(pos);
            if (current < 15) setCounter(pos, current + 1);
        }
        size++;
    }

    public boolean mightContain(String element) {
        for (int pos : hashPositions(element)) {
            if (getCounter(pos) == 0) return false;
        }
        return true;
    }

    public void remove(String element) {
        // Only remove if element might be present
        if (!mightContain(element)) return;
        for (int pos : hashPositions(element)) {
            int current = getCounter(pos);
            if (current > 0) setCounter(pos, current - 1);
        }
        size--;
    }

    public int size() { return size; }
    public long memoryBytes() { return counters.length; }

    public static void main(String[] args) {
        int n = 100_000;
        var cbf = new CountingBloomFilter(n, 0.01);
        System.out.printf("Counting BF: m=%d, k=%d, memory=%d KB%n",
            cbf.m, cbf.k, cbf.memoryBytes() / 1024);

        // Insert
        for (int i = 0; i < n; i++) cbf.add("elem_" + i);

        // Verify membership
        System.out.println("contains(elem_42): " + cbf.mightContain("elem_42")); // true

        // Delete
        cbf.remove("elem_42");
        System.out.println("After delete, contains(elem_42): " + cbf.mightContain("elem_42")); // false (most likely)

        // Verify others unaffected
        int missing = 0;
        for (int i = 0; i < n; i++) {
            if (i == 42) continue;
            if (!cbf.mightContain("elem_" + i)) missing++;
        }
        System.out.println("False negatives after delete: " + missing); // should be 0
    }
}
```

**Preguntas:**

1. ¿4 bits por contador empaquetados en bytes — ¿el acceso bit es costoso?

2. ¿`remove` verifica `mightContain` primero. ¿Es necesario?

3. ¿Después de delete, ¿el false negative de 0 se mantiene
   para los demás elementos?

4. ¿La memoria es 4× la del Bloom filter estándar. ¿Exacto?

5. ¿Para un stream que necesita add Y remove,
   ¿el counting Bloom filter es la única opción?

---

## Sección 6.5 — Cuckoo Filter: la Alternativa Moderna

### Ejercicio 6.5.1 — Leer: cómo funciona un cuckoo filter

**Tipo: Leer**

```
Un cuckoo filter usa cuckoo hashing (dos posiciones posibles por elemento)
con FINGERPRINTS (hashes cortos) en vez de los datos completos.

  Estructura:
    Array de buckets. Cada bucket tiene b slots (típicamente b=4).
    Cada slot almacena un fingerprint de f bits (típicamente f=8-16).

  ADD(element):
    fp = fingerprint(element)  (ej: 8 bits del hash)
    i₁ = hash₁(element) % num_buckets
    i₂ = i₁ XOR hash(fp) % num_buckets  (XOR trick para relocation)

    Si bucket[i₁] tiene un slot libre: insertar fp en bucket[i₁].
    Sino, si bucket[i₂] tiene un slot libre: insertar fp en bucket[i₂].
    Sino: CUCKOO — desplazar un fingerprint existente:
      Elegir un fp existente de bucket[i₁] o bucket[i₂].
      Moverlo a su posición alternativa (XOR trick).
      Si la alternativa está llena, repetir (hasta un máximo).
      Si se alcanza el máximo de desplazamientos: el filtro está lleno.

  CONTAINS(element):
    fp = fingerprint(element)
    return fp ∈ bucket[i₁] OR fp ∈ bucket[i₂]

  DELETE(element):
    fp = fingerprint(element)
    Si fp ∈ bucket[i₁]: borrar fp de bucket[i₁]. Done.
    Si fp ∈ bucket[i₂]: borrar fp de bucket[i₂]. Done.
    (Si el fingerprint no está en ninguno: el elemento no existe.)

  Ventajas vs Bloom filter:
    ✓ Soporta DELETE (sin contadores).
    ✓ Mejor space efficiency para FP < 3%.
    ✓ Lookup más rápido (2 accesos vs k accesos).

  Desventajas vs Bloom filter:
    ✗ Puede llenarse (insert falla si demasiados cuckoo kicks).
    ✗ Más complejo de implementar.
    ✗ No soporta intersección/unión con operaciones bitwise.

  Paper: "Cuckoo Filter: Practically Better Than Bloom"
         (Fan, Andersen, Kaminsky, Mitzenmacher, 2014)
```

**Preguntas:**

1. ¿El XOR trick `i₂ = i₁ XOR hash(fp)` — ¿por qué funciona?

2. ¿Fingerprint de 8 bits tiene solo 256 valores posibles.
   ¿Eso limita la capacidad?

3. ¿El cuckoo filter puede llenarse. ¿Qué load factor máximo?

4. ¿2 memory accesses (cuckoo) vs 7 memory accesses (Bloom con k=7).
   ¿Cuánto más rápido?

5. ¿Por qué Cassandra usa Bloom filters y no cuckoo filters?

---

### Ejercicio 6.5.2 — Analizar: Bloom vs Cuckoo — cuándo usar cada uno

**Tipo: Analizar**

```
Comparación para n = 10M elementos:

  Métrica               Bloom (1% FP)       Cuckoo (1% FP)
  ──────                ─────────────        ──────────────
  Memory                ~12 MB               ~10 MB
  Add (ns)              ~200-300             ~150-250
  Contains (ns)         ~200-300             ~50-100
  Delete                No                   Sí
  Insert failure        Nunca                Posible (when full)
  Union/Intersection    Sí (bitwise OR/AND)  No
  Space efficiency      Mejor para FP > 3%   Mejor para FP < 3%

  Usar Bloom filter cuando:
  - No necesitas delete.
  - Necesitas unión/intersección de filtros (bitwise operations).
  - FP rate > 3%.
  - Quieres implementación más simple.
  - Quieres garantía de que insert nunca falla.

  Usar Cuckoo filter cuando:
  - Necesitas delete.
  - FP rate < 3%.
  - Lookup speed es crítico.
  - Puedes manejar insert failures (resize o rebuild).

  En la práctica:
  - Cassandra: Bloom (no necesita delete, FP ~1%).
  - Parquet: Bloom (no necesita delete).
  - Chrome Safe Browsing: podría ser Cuckoo (necesita updates = deletes).
  - Redis: Bloom filter module (RedisBloom) soporta ambos.
```

**Preguntas:**

1. ¿10 MB vs 12 MB para 10M elementos — ¿la diferencia es significativa?

2. ¿Lookup 50-100 ns del cuckoo vs 200-300 ns del Bloom.
   ¿3× más rápido? ¿Por qué?

3. ¿"Unión de Bloom filters" — ¿para qué se usa en producción?

4. ¿RedisBloom soporta ambos. ¿Cuál recomienda por defecto?

5. ¿El paper de cuckoo filter dice "practically better". ¿Siempre?

---

## Sección 6.6 — HyperLogLog: Contar Distintos con 12 KB

### Ejercicio 6.6.1 — Leer: el problema de COUNT DISTINCT

**Tipo: Leer**

```
Pregunta: ¿cuántas IPs distintas visitaron mi sitio hoy?

  Solución exacta: HashSet<String> de todas las IPs.
    100M IPs × ~80 bytes/entry = 8 GB.
    Respuesta exacta: 42,387,291 IPs distintas.

  Solución HyperLogLog: 12 KB de memoria.
    Respuesta aproximada: 42,500,000 ± 340,000 (error estándar ~0.81%).
    8 GB vs 12 KB = 660,000× menos memoria.

  ¿Cómo es posible?

  Intuición: la probabilidad observada.
    Si hasheas muchos valores aleatorios y miras los bits del hash:
    - 50% de los hashes empiezan con 0
    - 25% empiezan con 00
    - 12.5% empiezan con 000
    - Si ves un hash que empieza con 0000000000 (10 ceros),
      probablemente hasheaste al menos 2^10 = 1024 valores distintos.

  El "run of leading zeros" más largo te da una estimación de log₂(n).

  Pero un solo estimador tiene alta varianza.
  Solución: usar MUCHOS estimadores y promediar.

  HyperLogLog:
    1. Dividir los hashes en m "buckets" (registros).
       Los primeros p bits del hash seleccionan el bucket.
       Los bits restantes se usan para contar leading zeros.

    2. Para cada bucket, almacenar el MÁXIMO de leading zeros observado.
       Cada bucket necesita ~6 bits (para almacenar hasta 64 zeros).

    3. Para estimar la cardinalidad:
       Calcular la media armónica de 2^(máximo leading zeros) por bucket.
       Aplicar correcciones para rangos bajos y altos.

  Parámetros:
    m = 2^p buckets (p = precision parameter)
    Memoria: m × 6 bits = m × 0.75 bytes
    Error estándar: 1.04 / √m

    p=4:   16 buckets,   12 bytes,  error ~26%
    p=10:  1024 buckets, 768 bytes, error ~3.25%
    p=14:  16384 buckets, 12 KB,    error ~0.81%  ← estándar (Redis PFCOUNT)
    p=16:  65536 buckets, 48 KB,    error ~0.41%

  Redis PFCOUNT usa p=14 → 12 KB → error ~0.81%.
  BigQuery APPROX_COUNT_DISTINCT usa HLL++ (optimización de Google).
```

**Preguntas:**

1. ¿"Media armónica" en vez de media aritmética — ¿por qué?

2. ¿12 KB para estimar cardinalidad de mil millones de elementos.
   ¿Es magia o matemáticas?

3. ¿HyperLogLog puede SUBESTIMAR la cardinalidad?

4. ¿Dos HyperLogLog se pueden MERGE?
   ¿Cómo?

5. ¿BigQuery HLL++ — ¿qué mejora sobre HLL estándar?

---

### Ejercicio 6.6.2 — Implementar: HyperLogLog desde cero

**Tipo: Implementar**

```python
import math
import hashlib
import struct

class HyperLogLog:
    def __init__(self, precision=14):
        self.p = precision
        self.m = 1 << precision      # number of registers
        self.registers = [0] * self.m
        # Alpha correction factor
        if self.m == 16:
            self.alpha = 0.673
        elif self.m == 32:
            self.alpha = 0.697
        elif self.m == 64:
            self.alpha = 0.709
        else:
            self.alpha = 0.7213 / (1 + 1.079 / self.m)

    def _hash(self, element: str) -> int:
        """64-bit hash"""
        d = hashlib.sha1(element.encode()).digest()
        return struct.unpack('<Q', d[:8])[0]

    def _leading_zeros(self, value: int, bits: int = 64) -> int:
        """Count leading zeros after removing the p prefix bits"""
        if value == 0:
            return bits - self.p
        count = 0
        # Count zeros from the MSB of remaining bits
        for i in range(bits - self.p - 1, -1, -1):
            if value & (1 << i):
                break
            count += 1
        return count + 1  # +1 because we count the position of first 1-bit

    def add(self, element: str):
        h = self._hash(element)
        # First p bits determine the register
        register_idx = h & (self.m - 1)
        # Remaining bits used for leading zeros count
        remaining = h >> self.p
        leading = self._leading_zeros(remaining, 64 - self.p)
        self.registers[register_idx] = max(self.registers[register_idx], leading)

    def count(self) -> int:
        """Estimate cardinality"""
        # Raw estimate using harmonic mean
        indicator = sum(2.0 ** (-r) for r in self.registers)
        raw_estimate = self.alpha * self.m * self.m / indicator

        # Small range correction
        if raw_estimate <= 2.5 * self.m:
            zeros = self.registers.count(0)
            if zeros > 0:
                return int(self.m * math.log(self.m / zeros))
            return int(raw_estimate)

        # Large range correction (for 64-bit hashes, rarely needed)
        if raw_estimate > (1 << 32) / 30:
            return int(-(1 << 32) * math.log(1 - raw_estimate / (1 << 32)))

        return int(raw_estimate)

    def merge(self, other: 'HyperLogLog'):
        """Merge another HLL into this one (for distributed counting)"""
        assert self.p == other.p
        for i in range(self.m):
            self.registers[i] = max(self.registers[i], other.registers[i])

    def memory_bytes(self) -> int:
        return self.m  # 1 byte per register (could be 6 bits)


# ═══ Test ═══
for true_cardinality in [1_000, 10_000, 100_000, 1_000_000, 10_000_000]:
    hll = HyperLogLog(precision=14)
    for i in range(true_cardinality):
        hll.add(f"element_{i}")

    estimated = hll.count()
    error = abs(estimated - true_cardinality) / true_cardinality * 100

    print(f"True: {true_cardinality:>12,}  Estimated: {estimated:>12,}  "
          f"Error: {error:>5.2f}%  Memory: {hll.memory_bytes():,} bytes")

# Output esperado:
# True:        1,000  Estimated:        ~1,005  Error:  ~0.5%  Memory: 16,384 bytes
# True:       10,000  Estimated:       ~10,050  Error:  ~0.5%  Memory: 16,384 bytes
# True:      100,000  Estimated:      ~100,800  Error:  ~0.8%  Memory: 16,384 bytes
# True:    1,000,000  Estimated:    ~1,008,000  Error:  ~0.8%  Memory: 16,384 bytes
# True:   10,000,000  Estimated:   ~10,080,000  Error:  ~0.8%  Memory: 16,384 bytes

# ═══ Merge test ═══
hll1 = HyperLogLog(14)
hll2 = HyperLogLog(14)
for i in range(500_000): hll1.add(f"user_{i}")
for i in range(300_000, 800_000): hll2.add(f"user_{i}")
# Union = 800K distinct (0-800K), overlap = 200K

hll1.merge(hll2)
print(f"\nMerge test: estimated union = {hll1.count():,} (expected ~800,000)")
```

**Preguntas:**

1. ¿La precisión de ~0.8% se mantiene para cualquier cardinalidad?

2. ¿El merge de dos HLL — ¿el resultado es tan preciso
   como un HLL construido desde cero con todos los datos?

3. ¿16 KB de memoria para 10M elementos. ¿Es constante
   independiente de la cardinalidad?

4. ¿El small range correction usa "linear counting" para n < 2.5m.
   ¿Por qué?

5. ¿HyperLogLog puede contar la intersección de dos conjuntos?

---

### Ejercicio 6.6.3 — Analizar: HyperLogLog en producción

**Tipo: Analizar**

```
HyperLogLog es ubicuo en infraestructura de datos:

  REDIS PFCOUNT:
    PFADD mykey "elem1" "elem2" "elem3"  → adds to HLL
    PFCOUNT mykey                          → ~3 (approximate count)
    PFMERGE destkey srckey1 srckey2        → merge two HLLs

    Implementación: p=14, 16384 registros de 6 bits = 12 KB.
    Error estándar: 0.81%.
    → Contar usuarios únicos diarios: 12 KB vs 100+ MB con HashSet.

  BIGQUERY APPROX_COUNT_DISTINCT:
    SELECT APPROX_COUNT_DISTINCT(user_id) FROM events
    Usa HLL++ (Heule, Nunkesser, Hall, 2013 — Google Research).
    Mejoras: sparse representation para cardinalidades bajas,
    bias correction mejorada.
    → COUNT DISTINCT sobre petabytes en segundos.

  PRESTO / TRINO:
    approx_distinct(column) — HyperLogLog internamente.
    approx_distinct(column, 0.01) — especificar error deseado.

  APACHE FLINK:
    HyperLogLog en streaming para contar distintos en ventanas.
    Merge de HLLs entre workers.

  DRUID:
    Almacena HLL sketches como columnas pre-computadas.
    Queries de COUNT DISTINCT se resuelven mergeando sketches.
    → Sub-segundo para billones de registros.

  Patrón común:
    1. Cada nodo/partición mantiene su propio HLL sketch.
    2. Para el resultado global: MERGE todos los sketches.
    3. El merge es asociativo y conmutativo → paralelizable.
    → Perfecto para sistemas distribuidos.

  Variantes avanzadas:
    HLL++ (Google): bias correction, sparse mode.
    Apache DataSketches: HLL, CPC (compressed), Theta sketches.
    Theta sketch: soporta intersección (HLL no puede).
```

**Preguntas:**

1. ¿Redis PFCOUNT usa solo 12 KB por key.
   ¿Cuántas HLL keys puedes tener en 1 GB de RAM?

2. ¿BigQuery HLL++ sparse mode — ¿qué ahorra para cardinalidades bajas?

3. ¿Merge de HLLs es asociativo y conmutativo.
   ¿Eso significa que el orden de merge no importa?

4. ¿Theta sketch soporta intersección. ¿Cómo?

5. ¿Para un dashboard de analytics con 100 métricas de "usuarios únicos",
   ¿cuánta memoria necesitan 100 HLL sketches?

---

## Sección 6.7 — Estructuras Probabilísticas en Producción

### Ejercicio 6.7.1 — Analizar: el ecosistema de sketches

**Tipo: Analizar**

```
Las estructuras probabilísticas forman un ecosistema:

  MEMBERSHIP (¿pertenece X al conjunto?):
    Bloom filter:    m bits, k hashes. No delete. ~200 ns.
    Cuckoo filter:   fingerprints + cuckoo hashing. Delete. ~70 ns.
    Quotient filter:  alternativa cache-friendly.

  CARDINALITY (¿cuántos elementos distintos?):
    HyperLogLog:     12 KB, 0.81% error. Merge O(m).
    HLL++:           sparse + bias correction. Usado por Google.
    CPC sketch:      ~40% mejor que HLL en espacio. Apache DataSketches.

  FREQUENCY (¿cuántas veces aparece X?):
    Count-Min Sketch: array 2D de contadores. Sobre-estima, nunca sub-estima.
    Count sketch:     variante que puede sub-estimar (con signo).
    Frequent items:   Space-Saving, Misra-Gries.

  QUANTILES (¿cuál es la mediana / p99?):
    t-digest:        merge-friendly, preciso en los extremos.
    KLL sketch:      provable guarantees. Apache DataSketches.
    DDSketch:        relative error guarantee. Datadog.

  Cada sketch responde UNA pregunta con POCA memoria.
  Para un sistema de monitoring con 1000 métricas:

  Exacto (HashSet + contadores):
    1000 métricas × 100MB/métrica = 100 GB de RAM.

  Probabilístico (sketches):
    1000 × (12 KB HLL + 100 KB CMS + 50 KB t-digest) = ~160 MB.
    625× menos memoria. Error < 1%.

  → Los sketches son la base del monitoring moderno.
```

**Preguntas:**

1. ¿Count-Min Sketch para frequency estimation — ¿cómo funciona?

2. ¿t-digest para quantiles — ¿es preciso para p99 y p99.9?

3. ¿Apache DataSketches implementa todos estos sketches?

4. ¿Datadog usa DDSketch para latency histograms.
   ¿Por qué no t-digest?

5. ¿Un pipeline de datos debería computar sketches
   en la ingestión o al query time?

---

### Ejercicio 6.7.2 — Analizar: Bloom filters en Cassandra y Parquet

**Tipo: Analizar**

```
CASSANDRA — Bloom filter por SSTable:

  Cada SSTable (Sorted String Table) es un archivo en disco
  con millones de key-value pairs ordenados.
  Un nodo Cassandra puede tener 20+ SSTables por tabla.

  Sin Bloom filter:
    read(key) → buscar en CADA SSTable → 20 lecturas a disco.
    Cada lectura: binary search en SSTable index + I/O = ~5-10 ms.
    Total: 100-200 ms por lectura.

  Con Bloom filter (1 por SSTable):
    read(key) → verificar BF de cada SSTable (en memoria, ~200 ns).
    Si BF dice "no" → skip SSTable. Ahorro: 5-10 ms.
    Si BF dice "probablemente sí" → leer SSTable.
    Con FP=1%: 20 SSTables × 1% FP = 0.2 lecturas innecesarias promedio.
    Total: 1.2 lecturas a disco = 6-12 ms.

    Speedup: 10-20× por agregar ~12 MB de Bloom filters en RAM.

  Configuración en Cassandra:
    bloom_filter_fp_chance = 0.01  (default, 1% FP)
    bloom_filter_fp_chance = 0.001 (0.1% FP, más memoria)
    bloom_filter_fp_chance = 0.1   (10% FP, menos memoria)

PARQUET — Bloom filter por columna por row group:

  Un archivo Parquet tiene row groups de ~128 MB.
  Cada row group tiene columnas almacenadas por separado.

  Query: SELECT * FROM table WHERE user_id = 'abc123'

  Sin Bloom filter:
    Leer la columna user_id de CADA row group.
    Si hay 100 row groups: 100 × 128 MB = 12.8 GB de I/O.

  Con Bloom filter (1 por columna por row group):
    Verificar BF: ¿'abc123' podría estar en este row group?
    Si "no" → skip row group (128 MB de I/O ahorrado).
    Si "sí" → leer solo la columna user_id del row group.

    Con 100 row groups y el valor en solo 1:
    Sin BF: 12.8 GB de I/O.
    Con BF: 128 MB de I/O + 99 × ~300 ns de BF check.
    Speedup: ~100×.

  Habilitado en Parquet con:
    parquet.bloom.filter.enabled = true
    parquet.bloom.filter.max.bytes = 1048576  (1 MB por columna)
```

**Preguntas:**

1. ¿Cassandra con 1% FP pierde 0.2 lecturas innecesarias de 20 SSTables.
   ¿Es significativo?

2. ¿Parquet Bloom filter de 1 MB por columna por row group
   — ¿cuántos elementos cubre?

3. ¿Si el query filtra por RANGO (user_id > 'a' AND user_id < 'b'),
   ¿el Bloom filter ayuda?

4. ¿Spark respeta los Bloom filters de Parquet automáticamente?

5. ¿HBase usa Bloom filters como Cassandra?

---

### Ejercicio 6.7.3 — Resumen: las reglas de las estructuras probabilísticas

**Tipo: Leer**

```
Reglas nuevas del Cap.06:

  Regla 22: Las estructuras probabilísticas intercambian exactitud por memoria.
    Bloom filter: 67× menos memoria que HashSet por 1% de error.
    HyperLogLog: 660,000× menos memoria que HashSet por 0.81% de error.
    El tradeoff es configurable: más memoria = menos error.

  Regla 23: "Definitivamente no" es más valioso que "probablemente sí".
    El Bloom filter es útil NO por confirmar presencia,
    sino por DESCARTAR ausencia rápidamente.
    Cassandra, Parquet, Chrome: todos usan el "no" del Bloom filter
    para evitar operaciones costosas.

  Regla 24: Las estructuras probabilísticas son merge-friendly.
    Bloom filter: unión = OR de los bitsets.
    HyperLogLog: merge = max de los registros.
    → Perfectas para sistemas distribuidos.
    Cada nodo computa su sketch local, luego merge global.

  Regla 25: Bloom filter + bitset + hash function = los 3 capítulos juntos.
    Bitset (Cap.03) → almacenamiento eficiente.
    Hash function (Cap.04) → mapeo a posiciones.
    Bloom filter (Cap.06) → la combinación que resuelve membership.
    Este es el primer ejemplo del libro donde múltiples
    capítulos se combinan en una estructura completa.

  La Parte 2 (Hashing) está completa.
  Hash maps, concurrent maps, Bloom filters, HyperLogLog.
  Todo construido sobre arrays (Cap.03) y funciones hash (Cap.04).

  A partir del Cap.07, entramos en la Parte 3: Árboles.
  Los árboles resuelven lo que los hash maps no pueden:
  datos ordenados, range queries, y almacenamiento en disco.
```

**Preguntas:**

1. ¿Las 25 reglas son un "cheat sheet" útil para un data engineer?

2. ¿La Regla 24 (merge-friendly) es la razón
   por la que las estructuras probabilísticas dominan
   en sistemas distribuidos?

3. ¿Falta alguna estructura probabilística importante?

4. ¿Count-Min Sketch merece su propio capítulo?

5. ¿Los capítulos 03 (bitset) + 04 (hash) + 06 (Bloom)
   demuestran cómo los fundamentos se combinan?
