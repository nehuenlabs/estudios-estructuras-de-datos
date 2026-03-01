# Guía de Ejercicios — Cap.03: El Array: la Estructura que Nadie Respeta

> Un hash map es un array con una función de hash encima.
> Un heap es un array interpretado como árbol.
> Un B-Tree es un árbol de arrays.
> Un ring buffer es un array con aritmética modular.
> Un Bloom filter es un array de bits.
> Un columnar store es un conjunto de arrays paralelos.
>
> El array es la estructura más importante de la informática.
> No porque sea interesante — es trivial.
> Sino porque es la base de todo lo demás.
>
> Este capítulo implementa el array y sus variantes desde cero:
> dynamic array, ring buffer, y bitset.
> Cada variante resuelve un problema que el array estático no puede,
> y cada una aparece como componente interno
> de las estructuras que implementaremos en los próximos 15 capítulos.
>
> Si subestimas el array, subestimas el fundamento.

---

## El modelo mental: memoria contigua es velocidad

```
Un array es la estructura más simple que existe:
N elementos del mismo tipo, almacenados uno tras otro en memoria.

  Dirección de memoria:
  ┌────┬────┬────┬────┬────┬────┬────┬────┐
  │ a₀ │ a₁ │ a₂ │ a₃ │ a₄ │ a₅ │ a₆ │ a₇ │
  └────┴────┴────┴────┴────┴────┴────┴────┘
  0x00  0x08  0x10  0x18  0x20  0x28  0x30  0x38

  Para acceder a a[i]: dirección = base + (i × tamaño_elemento)
  Una multiplicación y una suma. O(1). Sin punteros. Sin indirección.

  ¿Por qué esto importa? (Recordar Cap.01)
  - Datos contiguos → spatial locality → cache-friendly
  - Sin punteros → sin pointer chasing → sin cache misses aleatorios
  - Tamaño predecible → sin object headers → sin overhead de GC

  El array es la estructura contra la que se miden todas las demás.
  Si una linked list, un árbol, o un hash map no supera al array
  en la operación que necesitas, usa el array.
```

```
Las variantes del array que implementaremos:

  1. DYNAMIC ARRAY (ArrayList, Vec, slice)
     Problema: el array estático tiene tamaño fijo.
     Solución: crecer automáticamente cuando se llena.
     Costo: resize ocasional (O(1) amortizado, Cap.02).

  2. RING BUFFER (circular buffer)
     Problema: insertar/eliminar al inicio de un array es O(n).
     Solución: usar aritmética modular para "rotar" el inicio.
     Costo: tamaño fijo (no crece).

  3. BITSET (bit array)
     Problema: representar un conjunto de N elementos usa N bytes mínimo.
     Solución: usar 1 BIT por elemento → 8× menos memoria.
     Costo: solo soporta presencia/ausencia (sin valores asociados).

  Estas tres variantes son los building blocks de:
  - Hash maps (Cap.04-05): array + hash function
  - Heaps (Cap.08): array interpretado como árbol
  - Bloom filters (Cap.06): bitset + múltiples hash functions
  - Queues (Cap.13): ring buffer
  - Columnar stores (Cap.16): arrays paralelos
```

---

## Tabla de contenidos

- [Sección 3.1 — El array estático: por qué lo simple es poderoso](#sección-31--el-array-estático-por-qué-lo-simple-es-poderoso)
- [Sección 3.2 — Dynamic array: crecer sin perder velocidad](#sección-32--dynamic-array-crecer-sin-perder-velocidad)
- [Sección 3.3 — Implementar un dynamic array desde cero](#sección-33--implementar-un-dynamic-array-desde-cero)
- [Sección 3.4 — Ring buffer: la cola circular](#sección-34--ring-buffer-la-cola-circular)
- [Sección 3.5 — Ring buffer en producción: de Kafka a LMAX](#sección-35--ring-buffer-en-producción-de-kafka-a-lmax)
- [Sección 3.6 — Bitset: un bit por elemento](#sección-36--bitset-un-bit-por-elemento)
- [Sección 3.7 — Arrays en el mundo real: dónde están y por qué importan](#sección-37--arrays-en-el-mundo-real-dónde-están-y-por-qué-importan)

---

## Sección 3.1 — El Array Estático: Por Qué lo Simple es Poderoso

### Ejercicio 3.1.1 — Leer: anatomía de un array en 5 lenguajes

**Tipo: Leer**

```
El array estático tiene las mismas propiedades en todos los lenguajes,
pero su representación en memoria varía significativamente.

  PYTHON — No tiene arrays estáticos nativos.
    list: array de PUNTEROS a objetos PyObject en el heap.
    Cada elemento es un puntero (8 bytes) → objeto (28+ bytes).
    → No es un "array real". Es un array de referencias.
    → Mala cache locality (pointer chasing en cada acceso).

    array.array: array de valores primitivos (como C).
    array.array('l', [1, 2, 3]) → 3 longs contiguos en memoria.
    → Buena cache locality, pero la iteración Python es lenta.

    numpy.ndarray: array de valores primitivos + operaciones vectorizadas.
    np.array([1, 2, 3], dtype=np.int64) → 3 int64 contiguos.
    → Buena cache locality + operaciones en C/Fortran. El "array real" de Python.

  JAVA — Dos tipos de arrays:
    int[]: array de primitivos. Datos contiguos. 16 bytes de header.
    Integer[]: array de REFERENCIAS a objetos Integer en el heap.
    → int[] es 5-15× más rápido que Integer[] para iteración (Cap.01).
    → Java no tiene "array de structs" sin boxing hasta Valhalla.

  SCALA — Misma JVM que Java:
    Array[Int]: compila a int[] de JVM. Primitivo. Rápido.
    Array[String]: compila a String[] de JVM. Referencias.
    List[Int]: NO es un array. Es una linked list. Lento.
    Vector[Int]: NO es un array. Es un trie. O(log32 n) acceso.
    → Para rendimiento, Array[T] es la única opción en Scala.

  GO — Arrays por valor, slices por referencia:
    [5]int{1,2,3,4,5}: array de tamaño fijo. Valor, no referencia.
    Se copia cuando se pasa a una función.
    []int{1,2,3,4,5}: slice. Referencia a un array subyacente.
    Internamente: (puntero, longitud, capacidad) = 24 bytes.
    → Sin headers. Sin boxing. Datos contiguos.

  RUST — Arrays y slices:
    [i64; 5]: array de tamaño fijo. Vive en el stack. Sin heap.
    Vec<i64>: dynamic array. Datos contiguos en el heap.
    &[i64]: slice. Referencia a una porción de un array o Vec.
    → Sin headers. Sin GC. Datos contiguos. Máxima cache locality.
```

**Preguntas:**

1. ¿`list` de Python es un array de punteros. ¿Eso significa
   que `list[0]` y `list[1]` pueden apuntar a cualquier tipo distinto?

2. ¿`Array[Int]` de Scala es literalmente un `int[]` de JVM
   o hay una capa intermedia?

3. ¿Un array `[5]int` de Go se copia al pasarlo a una función.
   ¿Para un array de 1M elementos, eso es un problema?

4. ¿`[i64; 1000000]` en Rust vive en el stack?
   ¿Cabe en el stack?

5. ¿Cuál de los 5 lenguajes tiene el array más "honesto"
   (datos contiguos sin overhead)?

---

### Ejercicio 3.1.2 — Implementar: medir las operaciones básicas del array

**Tipo: Implementar**

```java
// Las 4 operaciones fundamentales de un array y su costo real

public class ArrayOperations {

    static final int N = 10_000_000;

    public static void main(String[] args) {
        int[] data = new int[N];
        var rng = new java.util.Random(42);
        for (int i = 0; i < N; i++) data[i] = rng.nextInt();

        int runs = 50;
        long start, elapsed;

        // ═══ 1. ACCESO POR ÍNDICE: O(1) ═══
        // base + (i × 4) = una suma. Siempre el mismo costo.
        int[] indices = new int[1_000_000];
        for (int i = 0; i < indices.length; i++) indices[i] = rng.nextInt(N);

        // Warmup
        long dummy = 0;
        for (int i : indices) dummy += data[i];

        start = System.nanoTime();
        for (int r = 0; r < runs; r++)
            for (int i : indices) dummy += data[i];
        elapsed = System.nanoTime() - start;
        System.out.printf("Random access:      %5.1f ns/op%n",
            (double) elapsed / (indices.length * (long) runs));

        // ═══ 2. RECORRIDO SECUENCIAL: O(n) ═══
        start = System.nanoTime();
        for (int r = 0; r < runs; r++)
            for (int v : data) dummy += v;
        elapsed = System.nanoTime() - start;
        System.out.printf("Sequential iterate: %5.1f ns/elem%n",
            (double) elapsed / (N * (long) runs));

        // ═══ 3. INSERTAR AL FINAL: O(1) ═══
        // (simulado: ya tenemos el array, solo medir la escritura)
        start = System.nanoTime();
        for (int r = 0; r < runs; r++)
            for (int i = 0; i < N; i++) data[i] = i;
        elapsed = System.nanoTime() - start;
        System.out.printf("Write sequential:   %5.1f ns/elem%n",
            (double) elapsed / (N * (long) runs));

        // ═══ 4. INSERTAR EN EL MEDIO: O(n) — arraycopy ═══
        int[] small = new int[100_000];
        int insertOps = 10_000;
        start = System.nanoTime();
        for (int i = 0; i < insertOps; i++) {
            int pos = small.length / 2;
            System.arraycopy(small, pos, small, pos + 1, small.length - pos - 1);
            small[pos] = i;
        }
        elapsed = System.nanoTime() - start;
        System.out.printf("Insert middle (n=100K): %5.0f ns/op%n",
            (double) elapsed / insertOps);

        if (dummy == Long.MIN_VALUE) System.out.println(dummy);
    }
}
// Resultado esperado:
// Random access:        5-15 ns/op (cache misses para array de 40 MB)
// Sequential iterate:   0.3-0.5 ns/elem (auto-vectorización SIMD)
// Write sequential:     0.3-0.5 ns/elem
// Insert middle (n=100K): 10,000-20,000 ns/op (~10-20 μs por memmove de 200 KB)
```

**Preguntas:**

1. ¿Random access (5-15 ns) es 10-30× más lento que sequential (0.3-0.5 ns).
   ¿La causa es cache misses o algo más?

2. ¿Sequential iterate y write tienen el mismo costo?
   ¿El hardware trata lecturas y escrituras igual?

3. ¿`System.arraycopy` para insert middle — ¿usa instrucciones SIMD?

4. ¿Insert middle de 100K elementos tarda ~15 μs. ¿Es mucho o poco
   comparado con un insert en un Red-Black tree?

5. ¿Para un array de 40 MB (10M ints), ¿cuántos cache misses
   genera un random access?

---

### Ejercicio 3.1.3 — Analizar: cuándo el array estático es suficiente

**Tipo: Analizar**

```
Antes de saltar a un dynamic array, hash map, o árbol,
pregúntate: ¿el array estático resuelve mi problema?

  CASO 1: Lookup table con dominio conocido
    Queremos mapear HTTP status codes (100-599) a descripciones.
    HashMap<Integer, String>: ~80 bytes por entry, hashing, indirección.
    String[600]: acceso directo por índice. 0 hash. 0 colisiones.
    table[200] = "OK". table[404] = "Not Found".
    → El array es 20× más rápido y usa 20× menos memoria.
    Funciona porque el dominio es pequeño y denso (100-599).

  CASO 2: Contadores por categoría
    Contar eventos por hora del día (0-23).
    HashMap<Integer, Long>: overhead, boxing, hashing.
    long[24]: 192 bytes. Acceso directo. Sin overhead.
    counters[hora]++ es una operación de 1 ns.
    → Para dominios pequeños y densos, el array como lookup table
      es imbatible.

  CASO 3: Datos ordenados que no cambian
    Un catálogo de 10,000 productos ordenados por ID.
    TreeMap<Integer, Product>: O(log n) búsqueda, 40+ bytes/entry overhead.
    Product[] ordenado + binary search: O(log n) búsqueda, 0 bytes overhead.
    → Si los datos no cambian (o cambian raramente),
      un sorted array es mejor que un TreeMap.

  CASO 4: Buffer de tamaño fijo
    Almacenar las últimas 1000 mediciones de un sensor.
    ArrayList: crece si no lo controlas. Overhead de boxing.
    double[1000]: exactamente 8 KB. Sin GC. Sin sorpresas.
    → Para buffers de tamaño conocido, el array estático es perfecto.

  Regla: si conoces el tamaño, conoces el dominio, o los datos
  no cambian, usa un array. No necesitas más.
```

**Preguntas:**

1. ¿Un array como lookup table funciona si el dominio es disperso
   (ej: IDs de usuario de 1 a 10,000,000 pero solo 100,000 activos)?

2. ¿Databases usan arrays como lookup tables internamente?

3. ¿`enum` en Java se puede usar como índice de un array?
   ¿Es idiomático?

4. ¿Un sorted array de 10M productos con binary search
   es viable para un servicio web con <5 ms de latencia?

5. ¿Cuándo el array estático deja de ser suficiente
   y necesitas algo más?

---

### Ejercicio 3.1.4 — Implementar: array como lookup table vs HashMap

**Tipo: Implementar**

```go
// Go: comparar array lookup vs map lookup para un dominio denso
package main

import (
    "fmt"
    "math/rand"
    "time"
)

const domainSize = 1000
const lookups = 10_000_000

func benchArrayLookup(table [domainSize]int64, keys []int) int64 {
    var total int64
    for _, k := range keys {
        total += table[k]
    }
    return total
}

func benchMapLookup(m map[int]int64, keys []int) int64 {
    var total int64
    for _, k := range keys {
        total += m[k]
    }
    return total
}

func main() {
    // Setup
    var table [domainSize]int64
    m := make(map[int]int64, domainSize)
    for i := 0; i < domainSize; i++ {
        table[i] = int64(i * 7)
        m[i] = int64(i * 7)
    }

    keys := make([]int, lookups)
    rng := rand.New(rand.NewSource(42))
    for i := range keys {
        keys[i] = rng.Intn(domainSize)
    }

    // Warmup
    benchArrayLookup(table, keys[:1000])
    benchMapLookup(m, keys[:1000])

    // Benchmark array
    start := time.Now()
    sink := benchArrayLookup(table, keys)
    arrayTime := time.Since(start)

    // Benchmark map
    start = time.Now()
    sink += benchMapLookup(m, keys)
    mapTime := time.Since(start)

    fmt.Printf("Array lookup: %.1f ns/op\n",
        float64(arrayTime.Nanoseconds())/float64(lookups))
    fmt.Printf("Map lookup:   %.1f ns/op\n",
        float64(mapTime.Nanoseconds())/float64(lookups))
    fmt.Printf("Ratio:        %.1f×\n",
        float64(mapTime.Nanoseconds())/float64(arrayTime.Nanoseconds()))
    _ = sink
}
// Resultado esperado:
// Array lookup: ~2-3 ns/op (datos caben en L2/L3, acceso directo)
// Map lookup:   ~25-40 ns/op (hashing + posible chaining)
// Ratio:        ~10-15×
```

**Preguntas:**

1. ¿10-15× de diferencia para un dominio de 1000 elementos?
   ¿El ratio se mantiene para dominios de 100,000?

2. ¿El array de 1000 elementos cabe en L1 cache?
   ¿Eso explica el rendimiento?

3. ¿En Java, ¿el ratio sería similar entre `int[]` y `HashMap<Integer, Long>`?

4. ¿Un `switch` con 1000 cases se compila a un array lookup?

5. ¿Cuándo el map se justifica a pesar de ser 10× más lento?

---

### Ejercicio 3.1.5 — Implementar: sorted array + binary search vs TreeMap

**Tipo: Implementar**

```rust
// Rust: sorted Vec + binary_search vs BTreeMap
use std::collections::BTreeMap;
use std::time::Instant;

fn main() {
    let sizes = [1_000, 10_000, 100_000, 1_000_000, 10_000_000];
    let lookups = 500_000;

    println!("{:<12} {:>14} {:>14} {:>8}",
        "Size", "Vec+bsearch", "BTreeMap", "Ratio");
    println!("{}", "─".repeat(52));

    for &n in &sizes {
        // Sorted Vec
        let vec: Vec<(i64, i64)> = (0..n).map(|i| (i as i64, i as i64 * 3)).collect();

        // BTreeMap
        let btree: BTreeMap<i64, i64> = (0..n).map(|i| (i as i64, i as i64 * 3)).collect();

        // Keys to lookup (pseudo-random, deterministic)
        let keys: Vec<i64> = (0..lookups).map(|i| ((i * 7919) % n) as i64).collect();

        // Warmup
        for &k in keys.iter().take(1000) {
            let _ = vec.binary_search_by_key(&k, |&(key, _)| key);
            let _ = btree.get(&k);
        }

        // Benchmark Vec + binary_search
        let start = Instant::now();
        let mut sink = 0i64;
        for &k in &keys {
            if let Ok(idx) = vec.binary_search_by_key(&k, |&(key, _)| key) {
                sink = sink.wrapping_add(vec[idx].1);
            }
        }
        let vec_elapsed = start.elapsed();
        let vec_ns = vec_elapsed.as_nanos() as f64 / lookups as f64;

        // Benchmark BTreeMap
        let start = Instant::now();
        for &k in &keys {
            if let Some(&v) = btree.get(&k) {
                sink = sink.wrapping_add(v);
            }
        }
        let btree_elapsed = start.elapsed();
        let btree_ns = btree_elapsed.as_nanos() as f64 / lookups as f64;

        let ratio = btree_ns / vec_ns;
        println!("{:<12} {:>12.1} ns {:>12.1} ns {:>7.1}×",
            format!("{}",n), vec_ns, btree_ns, ratio);

        std::hint::black_box(sink);
    }
}
// Resultado esperado:
// Size         Vec+bsearch      BTreeMap    Ratio
// ────────────────────────────────────────────────
// 1000              8.0 ns         18.0 ns    2.3×
// 10000            12.0 ns         35.0 ns    2.9×
// 100000           18.0 ns         55.0 ns    3.1×
// 1000000          25.0 ns         75.0 ns    3.0×
// 10000000         40.0 ns        110.0 ns    2.8×
//
// El sorted Vec es consistentemente 2.5-3× más rápido que BTreeMap.
// Ambos son O(log n). La diferencia es cache locality.
```

**Preguntas:**

1. ¿El sorted Vec gana siempre. ¿Entonces por qué existe BTreeMap?

2. ¿Si necesitas inserts frecuentes, ¿el sorted Vec sigue ganando?

3. ¿El ratio se mantiene estable (~3×) independiente de n?
   ¿Por qué?

4. ¿Para 10M elementos, binary search hace ~23 comparaciones.
   ¿Cuántas son cache misses?

5. ¿En Java, ¿`Collections.binarySearch()` sobre un ArrayList
   tiene el mismo rendimiento relativo vs TreeMap?

---

## Sección 3.2 — Dynamic Array: Crecer Sin Perder Velocidad

### Ejercicio 3.2.1 — Leer: la anatomía de ArrayList, Vec, y slice

**Tipo: Leer**

```
Un dynamic array es un array que crece automáticamente.
Internamente: un array estático + un size + un capacity.

  ┌──────────────────────────────────────────────┐
  │  Dynamic Array                                │
  │                                               │
  │  data: ──→ [ a₀ │ a₁ │ a₂ │ a₃ │ __ │ __ ]  │
  │  size: 4                                      │
  │  capacity: 6                                  │
  └──────────────────────────────────────────────┘

  size = cuántos elementos tiene.
  capacity = cuántos elementos caben antes del próximo resize.
  data = puntero al array estático en el heap.

  Operaciones:
    get(i):  return data[i]                         → O(1)
    set(i):  data[i] = valor                        → O(1)
    add(x):  if size == capacity: resize()
             data[size++] = x                       → O(1) amortizado
    insert(i, x): arraycopy(data, i, data, i+1, size-i)
                   data[i] = x; size++              → O(n)
    remove(i): arraycopy(data, i+1, data, i, size-i-1)
               size--                               → O(n)

  Implementaciones por lenguaje:

  Java ArrayList<E>:
    - Object[] internamente (siempre boxing para primitivos)
    - Growth factor: 1.5 (oldCapacity + oldCapacity >> 1)
    - Capacity mínima: 10 (default constructor)
    - No hace shrinking automático (trimToSize() es manual)

  Rust Vec<T>:
    - Datos contiguos en el heap. Sin boxing.
    - Growth factor: 2 (duplica)
    - Capacity inicial: 0 (no asigna hasta el primer push)
    - Drop libera la memoria automáticamente (RAII)

  Go slice:
    - Header: (pointer, length, capacity) = 24 bytes en stack
    - Datos contiguos en el heap. Sin boxing.
    - Growth factor: 2 para cap < 256, luego ~1.25
    - append() puede retornar un NUEVO slice si necesita resize
      (trampas de aliasing)

  Scala ArrayBuffer[T]:
    - Array[T] internamente (JVM array)
    - Growth factor: 2
    - Para Int/Long/Double: boxea si usas genéricos
      (pero Array[Int] directo no boxea)

  Python list:
    - PyObject** internamente (array de punteros)
    - Growth factor: ~1.125 (conservador)
    - Cada elemento es un puntero → objeto en heap
    - El dynamic array más lento de los 5 por overhead de Python
```

**Preguntas:**

1. ¿Java ArrayList empieza con capacidad 10 incluso si no agregas nada?

2. ¿Rust Vec con capacidad 0 — ¿cuándo asigna heap por primera vez?

3. ¿Go `append()` puede causar bugs sutiles con aliasing?
   ¿Cómo?

4. ¿Python list con growth factor 1.125 — ¿hace más resizes
   que Java con 1.5?

5. ¿Scala ArrayBuffer boxea los Int? ¿Siempre?

---

### Ejercicio 3.2.2 — Analizar: growth factor — el tradeoff fundamental

**Tipo: Analizar**

```
El growth factor controla cuánto crece el array en cada resize.
Es el tradeoff central del dynamic array:

  Factor alto (2.0, 3.0):
    + Menos resizes (log_2(n) para factor 2)
    + Menos copias totales
    + Menos spikes de latencia
    - Más memoria desperdiciada (hasta 50% con factor 2)
    - Las asignaciones previas NUNCA se pueden reutilizar

  Factor bajo (1.25, 1.5):
    + Menos memoria desperdiciada
    + Las asignaciones previas PUEDEN reutilizarse (factor < φ ≈ 1.618)
    - Más resizes
    - Más copias totales

  El golden ratio (φ ≈ 1.618):
    Con factor < φ, la suma de las capacidades anteriores
    es mayor que la nueva capacidad.
    → El allocator PUEDE reutilizar la memoria liberada.
    Con factor > φ, la nueva asignación es siempre más grande
    que toda la memoria previamente liberada.
    → El allocator NO puede reutilizar. Necesita memoria fresca.

    Ejemplo con factor 2.0, capacidades: 1, 2, 4, 8, 16, 32, 64
    Cuando necesitas 64: memoria previamente liberada = 1+2+4+8+16+32 = 63.
    63 < 64. No cabe. Necesitas un bloque nuevo.

    Ejemplo con factor 1.5, capacidades: 1, 2, 3, 5, 8, 12, 18, 27
    Cuando necesitas 27: liberada = 1+2+3+5+8+12+18 = 49.
    49 > 27. Cabe. El allocator puede reutilizar.

  Decisiones reales:
    Java ArrayList: 1.5     (conservador, reutilizable)
    Rust Vec:       2.0     (rápido, no reutilizable)
    Python list:    ~1.125  (muy conservador)
    Go slice:       2.0→1.25 (híbrido: agresivo al inicio, conservador después)
    C++ vector:     varía (MSVC: 1.5, GCC: 2.0)
```

**Preguntas:**

1. ¿El golden ratio como límite para reutilización de memoria
   asume un allocator específico o aplica a cualquiera?

2. ¿Facebook folly::fbvector usa factor 1.5 explícitamente.
   ¿Es por la reutilización?

3. ¿Go cambió su growth factor de 2.0 a un esquema híbrido.
   ¿En qué versión?

4. ¿Para un pipeline de datos que asigna arrays temporales
   en cada batch, ¿el growth factor importa?

5. ¿Pre-asignar con `new ArrayList<>(expectedSize)` elimina
   la relevancia del growth factor?

---

### Ejercicio 3.2.3 — Implementar: shrinking — cuándo reducir la capacidad

**Tipo: Implementar**

```java
// Un dynamic array que crece Y se reduce
// Hysteresis: no reducir inmediatamente para evitar thrashing

public class ShrinkableArray<T> {

    private Object[] data;
    private int size;
    private static final int MIN_CAPACITY = 16;
    private int shrinkCount = 0;
    private int growCount = 0;

    public ShrinkableArray() {
        this.data = new Object[MIN_CAPACITY];
        this.size = 0;
    }

    public void add(T element) {
        if (size == data.length) grow();
        data[size++] = element;
    }

    @SuppressWarnings("unchecked")
    public T remove(int index) {
        T removed = (T) data[index];
        System.arraycopy(data, index + 1, data, index, size - index - 1);
        data[--size] = null; // let GC collect
        maybeShrink();
        return removed;
    }

    public T removeLast() {
        @SuppressWarnings("unchecked")
        T removed = (T) data[--size];
        data[size] = null;
        maybeShrink();
        return removed;
    }

    private void grow() {
        int newCap = data.length * 2;
        data = java.util.Arrays.copyOf(data, newCap);
        growCount++;
    }

    private void maybeShrink() {
        // Hysteresis: reducir solo cuando size < capacity/4
        // Esto evita thrashing si alguien hace add/remove/add/remove
        // en el límite.
        // Reducir a capacity/2 (no a size) — deja margen.
        if (data.length > MIN_CAPACITY && size < data.length / 4) {
            int newCap = Math.max(data.length / 2, MIN_CAPACITY);
            data = java.util.Arrays.copyOf(data, newCap);
            shrinkCount++;
        }
    }

    public int size() { return size; }
    public int capacity() { return data.length; }
    public int getGrowCount() { return growCount; }
    public int getShrinkCount() { return shrinkCount; }

    public static void main(String[] args) {
        var arr = new ShrinkableArray<Integer>();

        // Fase 1: crecer a 100,000
        for (int i = 0; i < 100_000; i++) arr.add(i);
        System.out.printf("After adding 100K: size=%d, cap=%d, grows=%d%n",
            arr.size(), arr.capacity(), arr.getGrowCount());

        // Fase 2: eliminar hasta 1,000
        while (arr.size() > 1_000) arr.removeLast();
        System.out.printf("After removing to 1K: size=%d, cap=%d, shrinks=%d%n",
            arr.size(), arr.capacity(), arr.getShrinkCount());

        // Fase 3: crecer de nuevo a 50,000
        for (int i = 0; i < 49_000; i++) arr.add(i);
        System.out.printf("After re-adding to 50K: size=%d, cap=%d, grows=%d%n",
            arr.size(), arr.capacity(), arr.getGrowCount());
    }
}
// Resultado esperado:
// After adding 100K: size=100000, cap=131072, grows=14
// After removing to 1K: size=1000, cap=2048, shrinks=6
// After re-adding to 50K: size=50000, cap=65536, grows=19
//
// Sin shrinking, la capacidad habría permanecido en 131072
// después de remover hasta 1000 elementos → 130,000 slots desperdiciados.
// Con shrinking, se redujo a 2048 → 1,048 slots desperdiciados.
```

**Preguntas:**

1. ¿Por qué reducir en size < capacity/4 y no en size < capacity/2?

2. ¿Qué es "thrashing" en el contexto de un dynamic array?

3. ¿Java ArrayList hace shrinking automático? ¿Rust Vec?

4. ¿`trimToSize()` de Java es mejor que shrinking automático?

5. ¿Para un pipeline de datos que procesa batches de tamaño variable,
   ¿el shrinking es útil o contraproducente?

---

### Ejercicio 3.2.4 — Analizar: Go slices — las trampas del aliasing

**Tipo: Analizar**

```go
// Go slices tienen una trampa que ningún otro lenguaje tiene:
// append() puede retornar un slice que comparte datos con el original
// o uno completamente nuevo. Depende de la capacidad.

package main

import "fmt"

func main() {
    // TRAMPA 1: append puede no crear un nuevo backing array
    original := make([]int, 3, 5) // len=3, cap=5
    original[0] = 1
    original[1] = 2
    original[2] = 3

    derived := append(original, 4) // cap=5, hay espacio → NO crea nuevo array
    derived[0] = 99                 // ¡MODIFICA original también!

    fmt.Println("original:", original) // [99 2 3] — ¡SORPRESA!
    fmt.Println("derived:", derived)   // [99 2 3 4]

    // TRAMPA 2: append puede crear un nuevo backing array
    full := make([]int, 3, 3) // len=3, cap=3
    full[0] = 1
    full[1] = 2
    full[2] = 3

    extended := append(full, 4) // cap=3, NO hay espacio → crea nuevo array
    extended[0] = 99             // NO modifica full (arrays distintos)

    fmt.Println("full:", full)         // [1 2 3] — no cambió
    fmt.Println("extended:", extended) // [99 2 3 4]

    // TRAMPA 3: subslice comparte datos
    data := []int{10, 20, 30, 40, 50}
    sub := data[1:3] // sub = [20 30], pero comparte el backing array

    sub[0] = 999
    fmt.Println("data:", data) // [10 999 30 40 50] — data cambió

    // SOLUCIÓN: copiar explícitamente
    safeSub := make([]int, 2)
    copy(safeSub, data[1:3])
    safeSub[0] = 888
    fmt.Println("data:", data) // [10 999 30 40 50] — no cambió
}
```

**Preguntas:**

1. ¿Este problema de aliasing existe en Rust Vec?
   (Pista: ownership y borrowing)

2. ¿Java ArrayList tiene el mismo problema con `subList()`?

3. ¿En producción, ¿los bugs de aliasing de slices de Go
   son comunes?

4. ¿`slices.Clone()` (Go 1.21+) resuelve el problema?

5. ¿Un rule de linter puede detectar aliasing peligroso en Go?

---

### Ejercicio 3.2.5 — Analizar: cuándo NO usar un dynamic array

**Tipo: Analizar**

```
El dynamic array es el default. Pero tiene limitaciones:

  PROBLEMA 1: Insert/delete en el medio es O(n)
    Cada insert/delete mueve n/2 elementos en promedio.
    Para n = 10M, eso son 40 MB de datos movidos por operación.
    → Si inserts/deletes en el medio son frecuentes,
      considera un balanced BST (O(log n)) o una skip list.

  PROBLEMA 2: Resize causa latency spikes
    Un resize copia todos los elementos.
    Para n = 10M: copiar 80 MB tarda ~30-50 ms.
    → Si la latencia predecible importa,
      pre-asigna o usa un ring buffer de tamaño fijo.

  PROBLEMA 3: Fragmentación de memoria
    Después de muchos grow + shrink cycles, el allocator
    puede tener bloques libres dispersos que no se pueden combinar.
    → Para buffers de larga duración, pre-asignar y reutilizar.

  PROBLEMA 4: No es thread-safe
    Dos threads haciendo add() simultáneamente pueden corromper el array.
    → Usa CopyOnWriteArrayList (Java), sync.Mutex (Go),
      Arc<RwLock<Vec>> (Rust), o un concurrent queue.

  PROBLEMA 5: Capacidad desconocida con datos enormes
    Si no sabes cuántos elementos vendrán y pueden ser billones,
    un dynamic array puede no funcionar (no cabe en RAM).
    → Usa streaming (procesar sin almacenar todo) o external sorting.

  Regla: usa dynamic array por defecto.
  Cambia solo cuando uno de estos 5 problemas
  se manifieste en tu caso de uso concreto.
```

**Preguntas:**

1. ¿`CopyOnWriteArrayList` de Java es un dynamic array thread-safe?
   ¿Cuál es su costo?

2. ¿En Go, ¿un slice protegido por `sync.RWMutex` es suficiente?

3. ¿Un "gap buffer" (como en editores de texto) resuelve el problema
   de insert en el medio?

4. ¿Para un pipeline de Spark que procesa particiones,
   ¿alguno de estos 5 problemas aplica?

5. ¿Un dynamic array de dynamic arrays (ej: `Vec<Vec<T>>`)
   ¿es una buena idea o un anti-pattern?

---

## Sección 3.3 — Implementar un Dynamic Array Desde Cero

### Ejercicio 3.3.1 — Implementar: dynamic array genérico en Java

**Tipo: Implementar**

```java
// Implementación completa con generics, iterator, y bounds checking

public class DynArray<T> implements Iterable<T> {

    private Object[] data;
    private int size;
    private static final int DEFAULT_CAPACITY = 8;

    public DynArray() {
        this.data = new Object[DEFAULT_CAPACITY];
        this.size = 0;
    }

    public DynArray(int initialCapacity) {
        if (initialCapacity < 0) throw new IllegalArgumentException("Negative capacity");
        this.data = new Object[Math.max(initialCapacity, 1)];
        this.size = 0;
    }

    // ═══ Core operations ═══

    public void add(T element) {
        ensureCapacity(size + 1);
        data[size++] = element;
    }

    public void insert(int index, T element) {
        rangeCheckForAdd(index);
        ensureCapacity(size + 1);
        System.arraycopy(data, index, data, index + 1, size - index);
        data[index] = element;
        size++;
    }

    @SuppressWarnings("unchecked")
    public T get(int index) {
        rangeCheck(index);
        return (T) data[index];
    }

    public void set(int index, T element) {
        rangeCheck(index);
        data[index] = element;
    }

    @SuppressWarnings("unchecked")
    public T remove(int index) {
        rangeCheck(index);
        T removed = (T) data[index];
        int numMoved = size - index - 1;
        if (numMoved > 0) {
            System.arraycopy(data, index + 1, data, index, numMoved);
        }
        data[--size] = null; // help GC
        return removed;
    }

    public T removeLast() {
        if (size == 0) throw new java.util.NoSuchElementException();
        @SuppressWarnings("unchecked")
        T removed = (T) data[--size];
        data[size] = null;
        return removed;
    }

    // ═══ Capacity management ═══

    private void ensureCapacity(int minCapacity) {
        if (minCapacity > data.length) {
            int newCapacity = data.length + (data.length >> 1); // factor 1.5
            if (newCapacity < minCapacity) newCapacity = minCapacity;
            data = java.util.Arrays.copyOf(data, newCapacity);
        }
    }

    // ═══ Bounds checking ═══

    private void rangeCheck(int index) {
        if (index < 0 || index >= size)
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
    }

    private void rangeCheckForAdd(int index) {
        if (index < 0 || index > size)
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
    }

    // ═══ Utility ═══

    public int size() { return size; }
    public boolean isEmpty() { return size == 0; }
    public int capacity() { return data.length; }

    public boolean contains(T element) {
        for (int i = 0; i < size; i++) {
            if (java.util.Objects.equals(data[i], element)) return true;
        }
        return false;
    }

    public int indexOf(T element) {
        for (int i = 0; i < size; i++) {
            if (java.util.Objects.equals(data[i], element)) return i;
        }
        return -1;
    }

    // ═══ Iterator (para for-each) ═══

    @Override
    public java.util.Iterator<T> iterator() {
        return new java.util.Iterator<T>() {
            private int cursor = 0;

            @Override public boolean hasNext() { return cursor < size; }

            @Override
            @SuppressWarnings("unchecked")
            public T next() {
                if (cursor >= size) throw new java.util.NoSuchElementException();
                return (T) data[cursor++];
            }
        };
    }

    @Override
    public String toString() {
        var sb = new StringBuilder("[");
        for (int i = 0; i < size; i++) {
            if (i > 0) sb.append(", ");
            sb.append(data[i]);
        }
        return sb.append("]").toString();
    }

    // ═══ Test ═══

    public static void main(String[] args) {
        var arr = new DynArray<String>();
        arr.add("alpha");
        arr.add("beta");
        arr.add("gamma");
        arr.insert(1, "bravo");
        System.out.println(arr);              // [alpha, bravo, beta, gamma]
        System.out.println(arr.get(2));        // beta
        arr.remove(0);
        System.out.println(arr);              // [bravo, beta, gamma]
        System.out.println("size=" + arr.size() + ", cap=" + arr.capacity());

        // Test con 1M elements
        var nums = new DynArray<Integer>(0);
        for (int i = 0; i < 1_000_000; i++) nums.add(i);
        System.out.printf("1M elements: size=%d, cap=%d%n", nums.size(), nums.capacity());

        // Verify iteration
        long total = 0;
        for (int v : nums) total += v;
        System.out.printf("Sum: %d (expected: %d)%n", total, 499999500000L);
    }
}
```

**Preguntas:**

1. ¿`data[--size] = null` es necesario para que el GC funcione?
   ¿Qué pasa si no lo haces?

2. ¿`System.arraycopy` es más rápido que un loop manual? ¿Por qué?

3. ¿El Iterator no es fail-fast (no detecta modificación concurrente).
   ¿Cómo lo agregarías?

4. ¿`rangeCheck` en cada get/set — ¿el JIT lo puede eliminar?

5. ¿Esta implementación es comparable a `java.util.ArrayList`?
   ¿Qué le falta?

---

### Ejercicio 3.3.2 — Implementar: dynamic array en Rust con ownership

**Tipo: Implementar**

```rust
// Un Vec simplificado que muestra ownership y Drop
pub struct MiVec<T> {
    ptr: *mut T,    // raw pointer a los datos en el heap
    len: usize,     // elementos actuales
    cap: usize,     // capacidad total
}

impl<T> MiVec<T> {
    pub fn new() -> Self {
        MiVec { ptr: std::ptr::null_mut(), len: 0, cap: 0 }
    }

    pub fn with_capacity(capacity: usize) -> Self {
        if capacity == 0 {
            return Self::new();
        }
        let layout = std::alloc::Layout::array::<T>(capacity).unwrap();
        let ptr = unsafe { std::alloc::alloc(layout) as *mut T };
        if ptr.is_null() { std::alloc::handle_alloc_error(layout); }
        MiVec { ptr, len: 0, cap: capacity }
    }

    pub fn push(&mut self, value: T) {
        if self.len == self.cap {
            self.grow();
        }
        unsafe {
            self.ptr.add(self.len).write(value);
        }
        self.len += 1;
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 { return None; }
        self.len -= 1;
        unsafe { Some(self.ptr.add(self.len).read()) }
    }

    pub fn get(&self, index: usize) -> Option<&T> {
        if index >= self.len { return None; }
        unsafe { Some(&*self.ptr.add(index)) }
    }

    pub fn len(&self) -> usize { self.len }
    pub fn capacity(&self) -> usize { self.cap }
    pub fn is_empty(&self) -> bool { self.len == 0 }

    fn grow(&mut self) {
        let new_cap = if self.cap == 0 { 4 } else { self.cap * 2 };
        let new_layout = std::alloc::Layout::array::<T>(new_cap).unwrap();

        let new_ptr = if self.cap == 0 {
            unsafe { std::alloc::alloc(new_layout) as *mut T }
        } else {
            let old_layout = std::alloc::Layout::array::<T>(self.cap).unwrap();
            unsafe {
                std::alloc::realloc(
                    self.ptr as *mut u8,
                    old_layout,
                    new_layout.size(),
                ) as *mut T
            }
        };

        if new_ptr.is_null() { std::alloc::handle_alloc_error(new_layout); }
        self.ptr = new_ptr;
        self.cap = new_cap;
    }
}

// Drop: liberar la memoria cuando MiVec sale del scope
impl<T> Drop for MiVec<T> {
    fn drop(&mut self) {
        if self.cap > 0 {
            // Drop each element
            for i in 0..self.len {
                unsafe { self.ptr.add(i).drop_in_place(); }
            }
            // Free the allocation
            let layout = std::alloc::Layout::array::<T>(self.cap).unwrap();
            unsafe { std::alloc::dealloc(self.ptr as *mut u8, layout); }
        }
    }
}

fn main() {
    let mut v = MiVec::new();
    for i in 0..1_000_000 {
        v.push(i as i64);
    }
    println!("len={}, cap={}", v.len(), v.capacity());
    println!("v[500000] = {:?}", v.get(500_000));

    // v se libera automáticamente al salir de main (Drop)
    // Sin GC. Sin leak. El compilador garantiza que drop() se llama.
}
```

**Preguntas:**

1. ¿`unsafe` es necesario para implementar un dynamic array en Rust?
   ¿Por qué?

2. ¿`realloc` puede reutilizar la memoria existente sin copiar?

3. ¿`drop_in_place` para cada elemento — ¿es necesario para `i64`?
   ¿Para `String`?

4. ¿Esta implementación es thread-safe? ¿Qué falta?

5. ¿El `Vec<T>` real de Rust usa `RawVec` internamente.
   ¿Qué agrega?

---

### Ejercicio 3.3.3 — Implementar: dynamic array en Scala — enfoque funcional

**Tipo: Implementar**

```scala
// Scala: un dynamic array inmutable con structural sharing
// (similar a un Vector simplificado)

sealed trait FunArray[+A] {
  def apply(index: Int): A
  def length: Int
  def appended[B >: A](elem: B): FunArray[B]
  def updated[B >: A](index: Int, elem: B): FunArray[B]
}

case class ArrayLeaf[A](data: Array[Any], len: Int) extends FunArray[A] {
  def apply(index: Int): A = {
    if (index < 0 || index >= len) throw new IndexOutOfBoundsException(s"$index")
    data(index).asInstanceOf[A]
  }

  def length: Int = len

  def appended[B >: A](elem: B): FunArray[B] = {
    if (len < data.length) {
      // Hay espacio: copiar y agregar (inmutabilidad → nueva copia)
      val newData = java.util.Arrays.copyOf(data, data.length)
      newData(len) = elem.asInstanceOf[AnyRef]
      ArrayLeaf(newData, len + 1)
    } else {
      // Sin espacio: crecer
      val newCap = if (data.length == 0) 4 else data.length * 2
      val newData = java.util.Arrays.copyOf(data, newCap)
      newData(len) = elem.asInstanceOf[AnyRef]
      ArrayLeaf(newData, len + 1)
    }
  }

  def updated[B >: A](index: Int, elem: B): FunArray[B] = {
    if (index < 0 || index >= len) throw new IndexOutOfBoundsException(s"$index")
    val newData = java.util.Arrays.copyOf(data, data.length)
    newData(index) = elem.asInstanceOf[AnyRef]
    ArrayLeaf(newData, len)
  }
}

object FunArray {
  def empty[A]: FunArray[A] = ArrayLeaf[A](Array.empty[Any], 0)

  def apply[A](elems: A*): FunArray[A] = {
    var arr = empty[A]
    for (e <- elems) arr = arr.appended(e)
    arr
  }
}

// Uso
object Main extends App {
  val a = FunArray(1, 2, 3, 4, 5)
  println(s"a(2) = ${a(2)}")        // 3
  println(s"a.length = ${a.length}") // 5

  val b = a.appended(6)              // a no cambia
  println(s"a.length = ${a.length}") // 5 (inmutable)
  println(s"b.length = ${b.length}") // 6

  val c = a.updated(0, 99)
  println(s"a(0) = ${a(0)}")         // 1 (inmutable)
  println(s"c(0) = ${c(0)}")         // 99

  // Benchmark: append 100K elements
  val start = System.nanoTime()
  var arr = FunArray.empty[Int]
  for (i <- 0 until 100_000) arr = arr.appended(i)
  val elapsed = (System.nanoTime() - start) / 1_000_000
  println(s"100K appends: ${elapsed} ms (inmutable, cada append copia)")
}
```

**Preguntas:**

1. ¿Cada `appended()` copia todo el array? ¿Es O(n) por operación?

2. ¿`scala.collection.immutable.Vector` resuelve este problema?
   ¿Cómo?

3. ¿La inmutabilidad de este dynamic array lo hace thread-safe
   automáticamente?

4. ¿Para 100K appends, ¿esta implementación es viable?
   ¿O demasiado lenta?

5. ¿Cuándo elegirías un array inmutable sobre uno mutable en Scala?

---

### Ejercicio 3.3.4 — Implementar: benchmark comparativo de las 3 implementaciones

**Tipo: Implementar**

```java
// Comparar: nuestra DynArray vs ArrayList vs int[] para 1M elementos
// Medir: add, iterate, random access

public class DynArrayBenchmark {

    static final int N = 1_000_000;
    static final int RUNS = 20;

    public static void main(String[] args) {
        // ═══ ADD ═══
        System.out.println("═══ ADD (1M elements) ═══");

        // int[] (pre-sized)
        long start = System.nanoTime();
        for (int r = 0; r < RUNS; r++) {
            int[] arr = new int[N];
            for (int i = 0; i < N; i++) arr[i] = i;
        }
        long elapsed = System.nanoTime() - start;
        System.out.printf("int[] (pre-sized):  %5.1f ms%n",
            (double) elapsed / RUNS / 1_000_000);

        // ArrayList (no pre-sized)
        start = System.nanoTime();
        for (int r = 0; r < RUNS; r++) {
            var list = new java.util.ArrayList<Integer>();
            for (int i = 0; i < N; i++) list.add(i);
        }
        elapsed = System.nanoTime() - start;
        System.out.printf("ArrayList (default): %5.1f ms%n",
            (double) elapsed / RUNS / 1_000_000);

        // ArrayList (pre-sized)
        start = System.nanoTime();
        for (int r = 0; r < RUNS; r++) {
            var list = new java.util.ArrayList<Integer>(N);
            for (int i = 0; i < N; i++) list.add(i);
        }
        elapsed = System.nanoTime() - start;
        System.out.printf("ArrayList (pre-sized): %5.1f ms%n",
            (double) elapsed / RUNS / 1_000_000);

        // DynArray (nuestra implementación)
        start = System.nanoTime();
        for (int r = 0; r < RUNS; r++) {
            var arr = new DynArray<Integer>();
            for (int i = 0; i < N; i++) arr.add(i);
        }
        elapsed = System.nanoTime() - start;
        System.out.printf("DynArray (ours):     %5.1f ms%n",
            (double) elapsed / RUNS / 1_000_000);

        // ═══ ITERATE ═══
        System.out.println("\n═══ ITERATE (sum 1M elements) ═══");

        int[] intArr = new int[N];
        var arrayList = new java.util.ArrayList<Integer>(N);
        var dynArray = new DynArray<Integer>(N);
        for (int i = 0; i < N; i++) {
            intArr[i] = i;
            arrayList.add(i);
            dynArray.add(i);
        }

        // Warmup
        long dummy = 0;
        for (int w = 0; w < 10; w++) {
            for (int v : intArr) dummy += v;
            for (int v : arrayList) dummy += v;
            for (int v : dynArray) dummy += v;
        }

        start = System.nanoTime();
        for (int r = 0; r < RUNS; r++)
            for (int v : intArr) dummy += v;
        elapsed = System.nanoTime() - start;
        System.out.printf("int[]:     %5.1f ms  (%.1f ns/elem)%n",
            (double) elapsed / RUNS / 1_000_000,
            (double) elapsed / RUNS / N);

        start = System.nanoTime();
        for (int r = 0; r < RUNS; r++)
            for (int v : arrayList) dummy += v;
        elapsed = System.nanoTime() - start;
        System.out.printf("ArrayList: %5.1f ms  (%.1f ns/elem)%n",
            (double) elapsed / RUNS / 1_000_000,
            (double) elapsed / RUNS / N);

        start = System.nanoTime();
        for (int r = 0; r < RUNS; r++)
            for (Object v : dynArray) dummy += (Integer) v;
        elapsed = System.nanoTime() - start;
        System.out.printf("DynArray:  %5.1f ms  (%.1f ns/elem)%n",
            (double) elapsed / RUNS / 1_000_000,
            (double) elapsed / RUNS / N);

        if (dummy == Long.MIN_VALUE) System.out.println(dummy);
    }
}
// Resultado esperado:
// ═══ ADD ═══
// int[] (pre-sized):    3.0 ms
// ArrayList (default):  15.0 ms (autoboxing + resizes)
// ArrayList (pre-sized): 10.0 ms (autoboxing, sin resizes)
// DynArray (ours):      12.0 ms (autoboxing + resizes)
//
// ═══ ITERATE ═══
// int[]:      2.0 ms  (0.4 ns/elem)  — SIMD, primitivos contiguos
// ArrayList: 10.0 ms  (3.5 ns/elem)  — unboxing + pointer chasing
// DynArray:  11.0 ms  (3.8 ns/elem)  — similar a ArrayList
```

**Preguntas:**

1. ¿Nuestra DynArray es competitiva con ArrayList?
   ¿Qué le falta para ser igual?

2. ¿`int[]` es 8× más rápido que ArrayList para iteración.
   ¿El autoboxing es el cuello de botella?

3. ¿Pre-sizing el ArrayList reduce el tiempo de add.
   ¿Cuánto se ahorra en resizes?

4. ¿Si tuviéramos un DynArray de `int` (sin boxing),
   ¿cerraríamos la brecha con `int[]`?

5. ¿Valhalla de Java permitiría un ArrayList<int> sin boxing?

---

### Ejercicio 3.3.5 — Analizar: las APIs que esconden el dynamic array

**Tipo: Analizar**

```
El dynamic array está en TODAS partes, frecuentemente oculto:

  Java:
    ArrayList<E>           — el dynamic array explícito
    StringBuilder          — dynamic array de chars
    ByteArrayOutputStream  — dynamic array de bytes
    ArrayList de Collections.unmodifiableList() — immutable wrapper

  Rust:
    Vec<T>                 — el dynamic array explícito
    String                 — Vec<u8> con garantía UTF-8
    VecDeque<T>            — ring buffer sobre Vec (Sección 3.4)
    BinaryHeap<T>          — heap sobre Vec (Cap.08)

  Go:
    []T (slice)            — el dynamic array explícito
    strings.Builder        — dynamic array de bytes para construir strings
    bytes.Buffer           — dynamic array de bytes para I/O

  Python:
    list                   — el dynamic array explícito
    bytearray              — dynamic array de bytes
    collections.deque      — doubly-linked list (NO dynamic array)
    io.BytesIO             — dynamic array de bytes para I/O

  Scala:
    ArrayBuffer[T]         — el dynamic array mutable
    Vector[T]              — tree-based immutable (NO dynamic array, pero parecido)
    StringBuilder          — dynamic array de chars

  En todos los lenguajes:
  - El constructor de strings con concatenación
    (StringBuilder, strings.Builder, String.join)
    es un dynamic array internamente.
  - Los buffers de I/O son dynamic arrays de bytes.
  - Los heaps son dynamic arrays interpretados como árboles.

  → Optimizar el dynamic array optimiza TODOS estos componentes.
```

**Preguntas:**

1. ¿`String` de Rust es literalmente un `Vec<u8>`?
   ¿Con qué restricciones?

2. ¿`StringBuilder` de Java crece con factor 2.
   ¿Es más agresivo que ArrayList (factor 1.5)?

3. ¿`collections.deque` de Python NO es un dynamic array?
   ¿Qué es entonces?

4. ¿Un BinaryHeap de Rust usa un Vec. ¿Eso significa que push
   al heap tiene los mismos resizes que push al Vec?

5. ¿Si optimizas la implementación de Vec en Rust,
   ¿automáticamente optimizas String, BinaryHeap, y VecDeque?

---

## Sección 3.4 — Ring Buffer: la Cola Circular

### Ejercicio 3.4.1 — Leer: por qué el array no sirve como cola

**Tipo: Leer**

```
Una cola (queue) tiene dos operaciones: enqueue (agregar al final)
y dequeue (sacar del inicio).

  Con un dynamic array:
    enqueue: add al final → O(1) amortizado. Perfecto.
    dequeue: remove del inicio → O(n). Desastre.
             Hay que mover TODOS los elementos una posición.

    Visualización de dequeue en un array:
    [a, b, c, d, e]  → remove a → [b, c, d, e, _]
     ↑  ↑  ↑  ↑                    ↑  ↑  ↑  ↑
     cada elemento se mueve una posición: O(n)

  Con un ring buffer:
    enqueue: escribir en data[tail], tail = (tail + 1) % capacity → O(1)
    dequeue: leer data[head], head = (head + 1) % capacity → O(1)

    Visualización de dequeue en un ring buffer:
    [a, b, c, d, e]  head=0, tail=5
     ↑ head

    Dequeue: return data[0]='a', head = (0+1) % 5 = 1
    [_, b, c, d, e]  head=1, tail=5
        ↑ head

    Enqueue 'f': data[5%5=0] = 'f', tail = (5+1) % 5 = 1... wait.
    Aquí necesitamos manejar full vs empty.

  El truco: el ring buffer "envuelve" el final al inicio.
  No mueve datos. Solo mueve los punteros head y tail.

  ┌────┬────┬────┬────┬────┐
  │ __ │ b  │ c  │ d  │ __ │   head=1, tail=4
  └────┴────┴────┴────┴────┘
         ↑ head       ↑ tail

  Enqueue 'e': data[4] = 'e', tail = (4+1) % 5 = 0
  ┌────┬────┬────┬────┬────┐
  │ __ │ b  │ c  │ d  │ e  │   head=1, tail=0
  └────┴────┴────┴────┴────┘
    ↑ tail  ↑ head             (tail "envolvió" al inicio)

  Enqueue 'f': data[0] = 'f', tail = (0+1) % 5 = 1
  ┌────┬────┬────┬────┬────┐
  │ f  │ b  │ c  │ d  │ e  │   head=1, tail=1
  └────┴────┴────┴────┴────┘
         ↑ head=tail           LLENO (head == tail)

  Problema: head == tail puede significar LLENO o VACÍO.
  Solución 1: mantener un campo size separado.
  Solución 2: dejar un slot vacío (capacidad = N+1 para N elementos).
  Solución 3: usar un flag "full" booleano.
```

**Preguntas:**

1. ¿La aritmética modular `(index + 1) % capacity` tiene costo?
   ¿Es más cara que un if?

2. ¿Si la capacidad es potencia de 2, `% capacity` se puede
   reemplazar por `& (capacity - 1)`. ¿Cuánto más rápido es?

3. ¿La "Solución 2" (slot vacío) desperdicia un elemento.
   ¿Es un costo aceptable?

4. ¿`ArrayDeque` de Java usa un ring buffer?

5. ¿Un ring buffer puede crecer dinámicamente?
   ¿Cómo manejas el resize con wrap-around?

---

### Ejercicio 3.4.2 — Implementar: ring buffer desde cero en Go

**Tipo: Implementar**

```go
// Ring buffer genérico en Go con capacidad fija (potencia de 2)
package main

import (
    "fmt"
    "errors"
)

type RingBuffer[T any] struct {
    data []T
    head int     // índice del primer elemento
    tail int     // índice del siguiente slot libre
    size int     // elementos actuales
    mask int     // capacity - 1 (para & en vez de %)
}

func NewRingBuffer[T any](capacity int) *RingBuffer[T] {
    // Redondear a la siguiente potencia de 2
    cap := 1
    for cap < capacity {
        cap <<= 1
    }
    return &RingBuffer[T]{
        data: make([]T, cap),
        mask: cap - 1,
    }
}

func (rb *RingBuffer[T]) Enqueue(value T) error {
    if rb.size == len(rb.data) {
        return errors.New("ring buffer full")
    }
    rb.data[rb.tail] = value
    rb.tail = (rb.tail + 1) & rb.mask  // & mask en vez de % capacity
    rb.size++
    return nil
}

func (rb *RingBuffer[T]) Dequeue() (T, error) {
    var zero T
    if rb.size == 0 {
        return zero, errors.New("ring buffer empty")
    }
    value := rb.data[rb.head]
    rb.data[rb.head] = zero  // help GC for pointer types
    rb.head = (rb.head + 1) & rb.mask
    rb.size--
    return value, nil
}

func (rb *RingBuffer[T]) Peek() (T, error) {
    var zero T
    if rb.size == 0 {
        return zero, errors.New("ring buffer empty")
    }
    return rb.data[rb.head], nil
}

func (rb *RingBuffer[T]) Size() int       { return rb.size }
func (rb *RingBuffer[T]) Capacity() int   { return len(rb.data) }
func (rb *RingBuffer[T]) IsEmpty() bool   { return rb.size == 0 }
func (rb *RingBuffer[T]) IsFull() bool    { return rb.size == len(rb.data) }

func main() {
    rb := NewRingBuffer[int](8)

    // Enqueue
    for i := 1; i <= 8; i++ {
        if err := rb.Enqueue(i * 10); err != nil {
            fmt.Println("Full:", err)
        }
    }
    fmt.Printf("After filling: size=%d, cap=%d\n", rb.Size(), rb.Capacity())

    // Dequeue 3
    for i := 0; i < 3; i++ {
        v, _ := rb.Dequeue()
        fmt.Printf("Dequeued: %d\n", v)
    }

    // Enqueue 3 more (wrap-around)
    for i := 9; i <= 11; i++ {
        rb.Enqueue(i * 10)
    }
    fmt.Printf("After wrap-around: size=%d\n", rb.Size())

    // Drain
    for !rb.IsEmpty() {
        v, _ := rb.Dequeue()
        fmt.Printf("%d ", v)
    }
    fmt.Println()
}
// Output:
// After filling: size=8, cap=8
// Dequeued: 10
// Dequeued: 20
// Dequeued: 30
// After wrap-around: size=8
// 40 50 60 70 80 90 100 110
```

**Preguntas:**

1. ¿`& rb.mask` es más rápido que `% len(rb.data)`?
   ¿Cuánto en nanosegundos?

2. ¿Forzar potencia de 2 desperdicia capacidad si el usuario pide 5?

3. ¿`rb.data[rb.head] = zero` es necesario en Go?
   ¿El GC no lo maneja?

4. ¿Este ring buffer es thread-safe? ¿Qué falta?

5. ¿Cómo agregarías grow/resize a este ring buffer
   manteniendo el wrap-around correcto?

---

### Ejercicio 3.4.3 — Implementar: ring buffer thread-safe en Go

**Tipo: Implementar**

```go
// Ring buffer thread-safe con mutex para producer-consumer
package main

import (
    "fmt"
    "sync"
    "time"
)

type SafeRingBuffer[T any] struct {
    data    []T
    head    int
    tail    int
    size    int
    mask    int
    mu      sync.Mutex
    notFull  *sync.Cond
    notEmpty *sync.Cond
}

func NewSafeRingBuffer[T any](capacity int) *SafeRingBuffer[T] {
    cap := 1
    for cap < capacity {
        cap <<= 1
    }
    rb := &SafeRingBuffer[T]{
        data: make([]T, cap),
        mask: cap - 1,
    }
    rb.notFull = sync.NewCond(&rb.mu)
    rb.notEmpty = sync.NewCond(&rb.mu)
    return rb
}

// Enqueue bloquea si el buffer está lleno
func (rb *SafeRingBuffer[T]) Enqueue(value T) {
    rb.mu.Lock()
    defer rb.mu.Unlock()

    for rb.size == len(rb.data) {
        rb.notFull.Wait()  // esperar hasta que haya espacio
    }

    rb.data[rb.tail] = value
    rb.tail = (rb.tail + 1) & rb.mask
    rb.size++
    rb.notEmpty.Signal()  // notificar que hay datos
}

// Dequeue bloquea si el buffer está vacío
func (rb *SafeRingBuffer[T]) Dequeue() T {
    rb.mu.Lock()
    defer rb.mu.Unlock()

    for rb.size == 0 {
        rb.notEmpty.Wait()  // esperar hasta que haya datos
    }

    value := rb.data[rb.head]
    var zero T
    rb.data[rb.head] = zero
    rb.head = (rb.head + 1) & rb.mask
    rb.size--
    rb.notFull.Signal()  // notificar que hay espacio
    return value
}

func (rb *SafeRingBuffer[T]) Size() int {
    rb.mu.Lock()
    defer rb.mu.Unlock()
    return rb.size
}

func main() {
    rb := NewSafeRingBuffer[int](1024)
    total := 1_000_000
    var wg sync.WaitGroup

    // Producer
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < total; i++ {
            rb.Enqueue(i)
        }
    }()

    // Consumer
    var sum int64
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < total; i++ {
            sum += int64(rb.Dequeue())
        }
    }()

    start := time.Now()
    wg.Wait()
    elapsed := time.Since(start)

    expected := int64(total-1) * int64(total) / 2
    fmt.Printf("Sum: %d (expected: %d, match: %v)\n", sum, expected, sum == expected)
    fmt.Printf("Throughput: %.0f ops/sec\n", float64(total)/elapsed.Seconds())
    fmt.Printf("Latency: %.0f ns/op\n", float64(elapsed.Nanoseconds())/float64(total))
}
// Output esperado:
// Sum: 499999500000 (expected: 499999500000, match: true)
// Throughput: ~5-15M ops/sec
// Latency: ~60-200 ns/op
```

**Preguntas:**

1. ¿El mutex limita el throughput? ¿Cuánto?

2. ¿`sync.Cond` con `Wait()` y `Signal()` — ¿es producer-consumer
   clásico?

3. ¿Usar channels de Go (`chan T`) sería más simple?
   ¿Más rápido o más lento?

4. ¿El LMAX Disruptor logra esto SIN mutex. ¿Cómo?

5. ¿Para un buffer de 1024 elementos entre producer y consumer,
   ¿es este enfoque suficiente para producción?

---

### Ejercicio 3.4.4 — Implementar: ArrayDeque en Java — el ring buffer del JDK

**Tipo: Implementar**

```java
// Implementar un deque (double-ended queue) como ring buffer
// Similar a java.util.ArrayDeque

public class MiArrayDeque<T> {

    private Object[] data;
    private int head; // índice del primer elemento
    private int tail; // índice del siguiente slot libre (después del último)
    // Invariante: capacidad es siempre potencia de 2
    // Esto permite & (capacity-1) en vez de % capacity

    public MiArrayDeque() {
        this.data = new Object[16]; // capacidad inicial
        this.head = 0;
        this.tail = 0;
    }

    public MiArrayDeque(int minCapacity) {
        int cap = Integer.highestOneBit(Math.max(minCapacity, 2) - 1) << 1;
        this.data = new Object[cap];
        this.head = 0;
        this.tail = 0;
    }

    // ═══ Operaciones de deque ═══

    public void addFirst(T element) {
        head = (head - 1) & (data.length - 1);
        data[head] = element;
        if (head == tail) grow();
    }

    public void addLast(T element) {
        data[tail] = element;
        tail = (tail + 1) & (data.length - 1);
        if (tail == head) grow();
    }

    @SuppressWarnings("unchecked")
    public T removeFirst() {
        if (head == tail) throw new java.util.NoSuchElementException();
        T result = (T) data[head];
        data[head] = null;
        head = (head + 1) & (data.length - 1);
        return result;
    }

    @SuppressWarnings("unchecked")
    public T removeLast() {
        if (head == tail) throw new java.util.NoSuchElementException();
        tail = (tail - 1) & (data.length - 1);
        T result = (T) data[tail];
        data[tail] = null;
        return result;
    }

    @SuppressWarnings("unchecked")
    public T peekFirst() {
        if (head == tail) return null;
        return (T) data[head];
    }

    @SuppressWarnings("unchecked")
    public T peekLast() {
        if (head == tail) return null;
        return (T) data[(tail - 1) & (data.length - 1)];
    }

    public int size() {
        return (tail - head) & (data.length - 1);
    }

    public boolean isEmpty() { return head == tail; }

    // ═══ Grow ═══

    private void grow() {
        int oldCap = data.length;
        int newCap = oldCap << 1; // duplicar
        Object[] newData = new Object[newCap];

        // Copiar: head → fin del array, luego inicio → tail
        int rightLen = oldCap - head;
        System.arraycopy(data, head, newData, 0, rightLen);
        System.arraycopy(data, 0, newData, rightLen, head);

        data = newData;
        head = 0;
        tail = oldCap;
    }

    public static void main(String[] args) {
        var deque = new MiArrayDeque<Integer>();

        // Usar como stack (LIFO)
        deque.addLast(1); deque.addLast(2); deque.addLast(3);
        System.out.printf("Stack pop: %d, %d, %d%n",
            deque.removeLast(), deque.removeLast(), deque.removeLast());

        // Usar como queue (FIFO)
        deque.addLast(10); deque.addLast(20); deque.addLast(30);
        System.out.printf("Queue dequeue: %d, %d, %d%n",
            deque.removeFirst(), deque.removeFirst(), deque.removeFirst());

        // Benchmark: 1M ops como queue
        long start = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) deque.addLast(i);
        for (int i = 0; i < 1_000_000; i++) deque.removeFirst();
        long elapsed = System.nanoTime() - start;
        System.out.printf("1M enqueue + 1M dequeue: %d ms%n", elapsed / 1_000_000);
    }
}
```

**Preguntas:**

1. ¿`(head - 1) & (data.length - 1)` — ¿funciona para head=0?

2. ¿El grow() copia en dos partes. ¿Por qué no un solo arraycopy?

3. ¿ArrayDeque de Java es más rápido que LinkedList como queue?
   ¿Cuánto?

4. ¿`VecDeque` de Rust tiene la misma implementación?

5. ¿Un deque thread-safe necesitaría locks en addFirst, addLast,
   removeFirst, y removeLast. ¿O se puede hacer lock-free?

---

### Ejercicio 3.4.5 — Analizar: dónde aparecen los ring buffers en producción

**Tipo: Analizar**

```
El ring buffer es ubicuo en sistemas de producción:

  KAFKA LOG SEGMENTS:
    Kafka almacena mensajes en "log segments" — archivos append-only.
    Cada partición tiene un log que actúa como un ring buffer lógico:
    los mensajes antiguos se eliminan por retention policy (tiempo o tamaño).
    No es un ring buffer en memoria, pero el concepto es el mismo:
    un buffer circular donde los datos viejos se sobrescriben.

  LMAX DISRUPTOR:
    Un ring buffer en memoria con tamaño fijo (potencia de 2).
    Producers escriben en posiciones secuenciales.
    Consumers leen desde sus propias posiciones.
    Sin locks: usa sequence numbers y memory barriers (CAS).
    Throughput: >100M ops/sec en una sola thread.
    → La estructura de datos detrás de los exchanges financieros.

  TCP BUFFERS:
    Los buffers de envío y recepción de TCP en el kernel
    son ring buffers. Los datos se escriben en el tail
    y se leen desde el head. Cuando el buffer se llena,
    TCP aplica backpressure (reduce el window size).

  LINUX IO_URING:
    El mecanismo de I/O asíncrono de Linux usa dos ring buffers:
    un Submission Queue (SQ) y un Completion Queue (CQ).
    El userspace escribe requests en el SQ ring.
    El kernel escribe completions en el CQ ring.
    → I/O de alto rendimiento sin system calls por operación.

  BOUNDED CHANNELS (Go):
    ch := make(chan int, 100)
    Internamente: un ring buffer de 100 slots con sincronización.
    Producer bloquea si el buffer está lleno.
    Consumer bloquea si el buffer está vacío.

  El ring buffer es la estructura preferida cuando:
  (a) El tamaño es conocido o acotado.
  (b) Producers y consumers operan a velocidades distintas.
  (c) La latencia predecible importa (sin resizes).
```

**Preguntas:**

1. ¿El LMAX Disruptor sin locks — ¿cómo evita data races?

2. ¿`io_uring` usa ring buffers en shared memory entre kernel y userspace?

3. ¿Un channel de Go con buffer 0 (`make(chan int)`) NO es un ring buffer?

4. ¿Kafka con `retention.bytes` es conceptualmente un ring buffer
   a nivel de disco?

5. ¿Para un pipeline de streaming (Flink, Spark Streaming),
   ¿los buffers internos entre operadores son ring buffers?

---

## Sección 3.6 — Bitset: Un Bit por Elemento

### Ejercicio 3.6.1 — Leer: representar conjuntos con bits

**Tipo: Leer**

```
Un bitset representa un conjunto de enteros usando 1 BIT por elemento.
El bit N está encendido (1) si N pertenece al conjunto.

  Ejemplo: conjunto {2, 5, 7} en un universo de 0-7
  Bit:     7 6 5 4 3 2 1 0
  Valor:   1 0 1 0 0 1 0 0  = 0b10100100 = 0xA4

  Operaciones de conjunto = operaciones BITWISE:
    Unión (A ∪ B):         A | B    (OR)
    Intersección (A ∩ B):  A & B    (AND)
    Diferencia (A \ B):    A & ~B   (AND NOT)
    Complemento (~A):      ~A       (NOT)
    Pertenencia (x ∈ A):   A & (1 << x) != 0

  Rendimiento:
    Estas operaciones procesan 64 elementos simultáneamente
    (un long de 64 bits). Con SIMD (AVX-512): 512 bits a la vez.

  Memoria:
    1M elementos: 1M bits = 125 KB
    vs boolean[]: 1M bytes = 1 MB (8× más)
    vs HashSet<Integer>: ~40 MB (320× más)

  Limitaciones:
    - Solo funciona para universos densos (0 a N)
    - Almacenar {0, 1000000} requiere 125 KB incluso con solo 2 elementos
    - No almacena valores asociados (solo presencia/ausencia)

  Uso real:
    Bloom filters (Cap.06): array de bits con k hash functions
    Bitmap indexes (Cap.16/17): una columna = un bitset por valor distinto
    Roaring Bitmaps: bitsets comprimidos para conjuntos sparse
    java.util.BitSet: implementación del JDK
```

**Preguntas:**

1. ¿Un bitset para un universo de 0 a 4 mil millones (32-bit ints)
   ocupa cuánta memoria?

2. ¿`boolean[]` de Java almacena 1 byte por booleano, no 1 bit?

3. ¿Las operaciones bitwise en un long procesan 64 elementos en 1 ciclo.
   ¿Es más rápido que un hash set por elemento?

4. ¿Roaring Bitmaps resuelve el problema del universo sparse?

5. ¿Parquet usa bitmap encoding para columnas de baja cardinalidad?

---

### Ejercicio 3.6.2 — Implementar: bitset desde cero en Rust

**Tipo: Implementar**

```rust
// Bitset que soporta conjuntos de enteros de 0 a N
pub struct BitSet {
    words: Vec<u64>,  // cada u64 almacena 64 bits
    num_bits: usize,  // universo: 0..num_bits
}

impl BitSet {
    pub fn new(num_bits: usize) -> Self {
        let num_words = (num_bits + 63) / 64; // ceil division
        BitSet {
            words: vec![0u64; num_words],
            num_bits,
        }
    }

    pub fn set(&mut self, bit: usize) {
        assert!(bit < self.num_bits, "bit {} out of range {}", bit, self.num_bits);
        self.words[bit / 64] |= 1u64 << (bit % 64);
    }

    pub fn clear(&mut self, bit: usize) {
        assert!(bit < self.num_bits);
        self.words[bit / 64] &= !(1u64 << (bit % 64));
    }

    pub fn get(&self, bit: usize) -> bool {
        assert!(bit < self.num_bits);
        (self.words[bit / 64] & (1u64 << (bit % 64))) != 0
    }

    pub fn count_ones(&self) -> usize {
        self.words.iter().map(|w| w.count_ones() as usize).sum()
    }

    // Operaciones de conjunto (retornan nuevo BitSet)
    pub fn union(&self, other: &BitSet) -> BitSet {
        assert_eq!(self.num_bits, other.num_bits);
        let words = self.words.iter().zip(&other.words)
            .map(|(a, b)| a | b)
            .collect();
        BitSet { words, num_bits: self.num_bits }
    }

    pub fn intersection(&self, other: &BitSet) -> BitSet {
        assert_eq!(self.num_bits, other.num_bits);
        let words = self.words.iter().zip(&other.words)
            .map(|(a, b)| a & b)
            .collect();
        BitSet { words, num_bits: self.num_bits }
    }

    pub fn difference(&self, other: &BitSet) -> BitSet {
        assert_eq!(self.num_bits, other.num_bits);
        let words = self.words.iter().zip(&other.words)
            .map(|(a, b)| a & !b)
            .collect();
        BitSet { words, num_bits: self.num_bits }
    }
}

fn main() {
    let mut a = BitSet::new(1_000_000);
    let mut b = BitSet::new(1_000_000);

    // Set algunos bits
    for i in (0..1_000_000).step_by(2) { a.set(i); }  // pares
    for i in (0..1_000_000).step_by(3) { b.set(i); }  // múltiplos de 3

    println!("A (pares):       {} bits set", a.count_ones());
    println!("B (mult. de 3):  {} bits set", b.count_ones());

    let union = a.union(&b);
    let inter = a.intersection(&b);
    let diff = a.difference(&b);

    println!("A ∪ B:           {} bits set", union.count_ones());
    println!("A ∩ B:           {} bits set", inter.count_ones());
    println!("A \\ B:           {} bits set", diff.count_ones());

    // Benchmark: intersección de 1M bits
    let start = std::time::Instant::now();
    for _ in 0..10_000 {
        let _ = std::hint::black_box(a.intersection(&b));
    }
    let elapsed = start.elapsed();
    println!("\nIntersection 1M bits × 10K: {:.1} μs/op",
        elapsed.as_micros() as f64 / 10_000.0);

    // Memory usage
    println!("Memory: {} KB for {} bits",
        a.words.len() * 8 / 1024, a.num_bits);
}
// Output esperado:
// A (pares):       500000 bits set
// B (mult. de 3):  333334 bits set
// A ∪ B:           666667 bits set
// A ∩ B:           166667 bits set (múltiplos de 6)
// A \ B:           333333 bits set
//
// Intersection 1M bits × 10K: ~1-3 μs/op
// Memory: 122 KB for 1000000 bits
```

**Preguntas:**

1. ¿`count_ones()` de Rust usa la instrucción POPCNT del CPU?

2. ¿La intersección de 1M bits en ~2 μs — ¿es más rápido
   que intersectar dos HashSets de 500K elementos?

3. ¿122 KB para 1M bits vs ~40 MB para un HashSet<Integer> de 1M.
   ¿325× menos memoria?

4. ¿Este bitset se puede auto-vectorizar (SIMD)?

5. ¿`java.util.BitSet` tiene rendimiento comparable?

---

### Ejercicio 3.6.3 — Implementar: bitset en Java con benchmark vs HashSet

**Tipo: Implementar**

```java
import java.util.*;

public class BitSetVsHashSet {

    public static void main(String[] args) {
        int universe = 1_000_000;
        int setSize = 500_000;
        var rng = new Random(42);

        // Crear ambos conjuntos con los mismos elementos
        var bitSet = new BitSet(universe);
        var hashSet = new HashSet<Integer>(setSize * 2);
        var elements = new int[setSize];
        for (int i = 0; i < setSize; i++) {
            int val = rng.nextInt(universe);
            elements[i] = val;
            bitSet.set(val);
            hashSet.add(val);
        }

        int lookups = 1_000_000;
        int[] queries = new int[lookups];
        for (int i = 0; i < lookups; i++) queries[i] = rng.nextInt(universe);

        // Warmup
        for (int q : queries) { bitSet.get(q); hashSet.contains(q); }

        int runs = 20;

        // Benchmark: contains
        long start = System.nanoTime();
        long dummy = 0;
        for (int r = 0; r < runs; r++)
            for (int q : queries) dummy += bitSet.get(q) ? 1 : 0;
        long bitTime = System.nanoTime() - start;

        start = System.nanoTime();
        for (int r = 0; r < runs; r++)
            for (int q : queries) dummy += hashSet.contains(q) ? 1 : 0;
        long hashTime = System.nanoTime() - start;

        System.out.printf("Contains (%d lookups × %d runs):%n", lookups, runs);
        System.out.printf("  BitSet:  %.1f ns/op%n",
            (double) bitTime / (lookups * (long) runs));
        System.out.printf("  HashSet: %.1f ns/op%n",
            (double) hashTime / (lookups * (long) runs));
        System.out.printf("  Ratio:   %.1f×%n",
            (double) hashTime / bitTime);

        // Memory (approximate)
        long bitSetMem = universe / 8; // bits → bytes
        long hashSetMem = (long) hashSet.size() * 40; // ~40 bytes per entry
        System.out.printf("%nMemory:%n");
        System.out.printf("  BitSet:  %d KB%n", bitSetMem / 1024);
        System.out.printf("  HashSet: %d KB%n", hashSetMem / 1024);
        System.out.printf("  Ratio:   %.0f×%n", (double) hashSetMem / bitSetMem);

        if (dummy == Long.MIN_VALUE) System.out.println(dummy);
    }
}
// Resultado esperado:
// Contains (1M lookups × 20 runs):
//   BitSet:   3-5 ns/op
//   HashSet:  25-40 ns/op
//   Ratio:    6-10×
//
// Memory:
//   BitSet:  122 KB
//   HashSet: 19531 KB (~19 MB)
//   Ratio:   160×
```

**Preguntas:**

1. ¿BitSet es 6-10× más rápido Y 160× menos memoria.
   ¿Por qué no se usa siempre?

2. ¿BitSet solo funciona para enteros. ¿Cómo representarías
   un conjunto de Strings?

3. ¿`BitSet.and()` de Java modifica in-place.
   ¿Es más rápido que crear uno nuevo?

4. ¿Para un universo de 4 mil millones (todos los int32),
   ¿el BitSet requiere 512 MB? ¿Es viable?

5. ¿Roaring Bitmaps comprime el bitset para universos sparse.
   ¿Cómo?

---

### Ejercicio 3.6.4 — Analizar: de bitsets a Bloom filters

**Tipo: Analizar**

```
Un Bloom filter (Cap.06) es un bitset + k hash functions.

  Bitset:       set(x)    → encender bit x
                get(x)    → ¿está encendido bit x?

  Bloom filter: add(x)    → calcular k hashes de x
                             encender bit hash₁(x), hash₂(x), ..., hashₖ(x)
                contains(x) → calcular k hashes de x
                              ¿están TODOS los bits encendidos?
                              Sí → "probablemente contiene x"
                              No → "definitivamente NO contiene x"

  La diferencia:
  - Bitset: almacena enteros directamente. Exacto. Universo = 0..N.
  - Bloom filter: almacena "fingerprints" de cualquier tipo. Probabilístico.
    Puede tener false positives (dice "sí" pero el elemento no está).
    Nunca tiene false negatives (si dice "no", el elemento no está).

  ¿Por qué importa esto?
  Cassandra usa Bloom filters en cada SSTable para evitar lecturas a disco.
  Sin Bloom filter: cada búsqueda puede requerir leer 10+ SSTables de disco.
  Con Bloom filter: verifica en memoria (ns) si el SSTable PODRÍA tener la key.
  Si dice "no" → skip this SSTable (ahorra 10 ms de I/O).
  Si dice "probablemente sí" → leer el SSTable (posible false positive, pero raro).

  El bitset que implementaste en los ejercicios anteriores
  es el componente central del Bloom filter que implementarás en el Cap.06.
```

**Preguntas:**

1. ¿Un Bloom filter de 1M bits con k=7 hash functions
   tiene qué false positive rate para 100K elementos?

2. ¿El bitset del ejercicio anterior puede usarse directamente
   como base de un Bloom filter?

3. ¿Un Bloom filter con k=1 es solo un bitset con hash?

4. ¿Counting Bloom filter usa un array de contadores en vez de bits.
   ¿Cuánto más memoria?

5. ¿Parquet usa Bloom filters para column predicate pushdown.
   ¿Cuánta memoria ahorra vs leer toda la columna?

---

### Ejercicio 3.6.5 — Analizar: bitmap indexes en columnar stores

**Tipo: Analizar**

```
Un bitmap index usa un bitset por cada valor distinto de una columna.

  Tabla: 8 filas, columna "color" con 3 valores distintos
  
  Fila:     0    1    2    3    4    5    6    7
  Color:   red  blue red  green red  blue green red

  Bitmap index:
  red:     1    0    1    0    1    0    0    1   = 0b10101001
  blue:    0    1    0    0    0    1    0    0   = 0b00100010
  green:   0    0    0    1    0    0    1    0   = 0b01000100

  Query: WHERE color = 'red' AND size = 'large'

  red bitmap:   10101001
  large bitmap: 01101001  (del bitmap index de la columna size)
  AND:          00101001  → filas 0, 3, 5 satisfacen ambas condiciones

  Una operación AND de 64 bits evalúa 64 filas simultáneamente.
  Para 1M filas: 1M / 64 = 15,625 operaciones AND.
  A 1 ns por operación: ~16 μs para evaluar un filtro sobre 1M filas.

  vs scan secuencial comparando strings: ~1-5 ms.
  → El bitmap index es 100× más rápido para queries de baja cardinalidad.

  Limitaciones:
  - Solo eficiente para baja cardinalidad (pocos valores distintos).
  - Columna "email" con 1M valores distintos → 1M bitsets × 1M bits = 125 GB.
    → Desastre. Usa B-Tree index para alta cardinalidad.
  - Updates son costosos (cambiar un valor requiere modificar 2 bitsets).

  Uso real:
  - Apache Druid: bitmap indexes para queries OLAP rápidas.
  - Oracle: bitmap indexes para data warehouses.
  - Parquet: dictionary encoding + bitpacking (no bitmap indexes directamente,
    pero el concepto es similar).
```

**Preguntas:**

1. ¿Un bitmap index para una columna con 2 valores (boolean)
   necesita 1 o 2 bitsets?

2. ¿Roaring Bitmaps mejorarían el rendimiento de bitmap indexes
   para columnas con muchos valores distintos?

3. ¿Apache Druid es más rápido que PostgreSQL para queries OLAP
   porque usa bitmap indexes?

4. ¿Un bitmap index es viable para una tabla que recibe
   10K inserts/segundo?

5. ¿Combinar bitmap indexes con Bloom filters en un columnar store
   ¿tiene sentido? ¿Cuándo usarías cada uno?

---

## Sección 3.7 — Arrays en el Mundo Real: Dónde Están y Por Qué Importan

### Ejercicio 3.7.1 — Analizar: el array como fundamento de todo

**Tipo: Analizar**

```
Mapa de dependencias — qué estructuras usan arrays internamente:

  Array estático
    ├── Dynamic array (ArrayList, Vec, slice)
    │     ├── StringBuilder / String
    │     ├── BinaryHeap (Cap.08)
    │     └── Stack (LIFO con dynamic array)
    ├── Ring buffer (ArrayDeque, VecDeque)
    │     ├── Bounded queues
    │     ├── Go channels (buffered)
    │     └── LMAX Disruptor
    ├── Hash map (Cap.04) — array de buckets
    │     ├── Hash set
    │     └── Concurrent hash map
    ├── Bitset
    │     ├── Bloom filter (Cap.06)
    │     ├── Bitmap indexes (Cap.17)
    │     └── Roaring Bitmaps
    ├── B-Tree nodo (Cap.09) — array de claves + array de hijos
    ├── SSTable (Cap.10) — array ordenado en disco
    └── Columnar store (Cap.16) — arrays paralelos por columna

  TODOS usan arrays internamente.
  Optimizar el acceso a un array optimiza TODOS estos componentes.

  ¿Qué NO usa arrays?
  - Linked lists (nodos dispersos en heap)
  - Red-Black trees (nodos dispersos en heap)
  - Skip lists (nodos dispersos en heap)
  → Todas las estructuras "sin array" tienen peor cache locality.
  → Todas existen porque resuelven operaciones que el array no puede
    hacer eficientemente (insert O(log n), merge O(log n), etc.).
```

**Preguntas:**

1. ¿El B-Tree usa arrays para CADA NODO. ¿Eso explica
   su buena cache locality?

2. ¿Un hash map con open addressing es "más array"
   que uno con chaining? ¿Eso lo hace más rápido?

3. ¿Si las CPUs tuvieran caches de 1 GB,
   ¿las linked lists serían tan lentas?

4. ¿Persistent data structures (inmutables) pueden usar arrays?
   ¿O necesitan nodos individuales?

5. ¿En 10 años, ¿las estructuras basadas en arrays seguirán
   dominando? ¿O el hardware cambiará las reglas?

---

### Ejercicio 3.7.2 — Resumen: las reglas del array

**Tipo: Leer**

```
Reglas actualizadas después de los Capítulos 01-03:

  Regla 1 (Cap.01): La estructura más rápida causa menos cache misses.
  Regla 2 (Cap.02): Big-O es necesario pero insuficiente. Mide.
  Regla 3 (Cap.02): Las constantes importan tanto como la clase.
  Regla 4 (Cap.02): O(1) amortizado ≠ O(1) siempre.
  Regla 5 (Cap.02): Mide con frameworks serios.
  Regla 6 (Cap.02): El espacio es el tradeoff olvidado.
  Regla 7 (Cap.01): La estructura importa más que el lenguaje.

  NUEVAS del Cap.03:

  Regla 8: Usa un array por defecto.
    Si no tienes una razón específica para usar algo más,
    usa un array. Es más rápido, usa menos memoria,
    y es más simple que cualquier otra estructura.

  Regla 9: El array es el building block de todo.
    Hash maps, heaps, B-Trees, Bloom filters, columnar stores
    — todos usan arrays internamente.
    Entender el array es entender la base.

  Regla 10: Tamaño fijo → ring buffer. Tamaño variable → dynamic array.
    Si conoces el tamaño máximo y necesitas FIFO: ring buffer.
    Si no conoces el tamaño: dynamic array.
    No uses linked list para ninguno de los dos.

  Regla 11: Bitset cuando solo necesitas presencia/ausencia.
    Para conjuntos de enteros en un universo denso,
    el bitset usa 160× menos memoria que un HashSet
    y es 6-10× más rápido.

  A partir del Cap.04, construimos SOBRE arrays:
  hash maps, heaps, árboles, Bloom filters.
  Cada estructura agrega complejidad encima de un array
  para resolver una operación que el array solo no puede.
```

**Preguntas:**

1. ¿La Regla 8 ("usa un array por defecto") es controversial?

2. ¿Existe un caso donde una linked list sea mejor que un array
   Y mejor que un ring buffer Y mejor que un dynamic array?

3. ¿Si el array es el building block de todo,
   ¿por qué los cursos de CS empiezan con linked lists?

4. ¿Las 11 reglas son suficientes para todo el libro
   o necesitarás más?

5. ¿Cuál de las 11 reglas habría cambiado tu forma de programar
   si la hubieras aprendido antes?
