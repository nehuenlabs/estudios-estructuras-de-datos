# Guía de Ejercicios — Cap.05: Hash Maps Avanzados: Concurrent, Ordered, Persistent

> El capítulo anterior construyó un hash map desde cero.
> Un hash map básico, single-threaded, sin orden, mutable.
>
> En producción, eso no es suficiente.
>
> Spark agrega datos en paralelo desde decenas de threads
> en un mismo executor — necesita un hash map concurrente.
>
> Un LRU cache necesita saber cuál fue el último elemento accedido
> — necesita un hash map que recuerde el orden de inserción o acceso.
>
> Las colecciones inmutables de Scala y Clojure necesitan
> "modificar" un hash map sin mutar el original
> — necesitan un hash map persistente con structural sharing.
>
> Este capítulo cubre las tres variantes avanzadas:
> 1. Concurrent: acceso seguro desde múltiples threads.
> 2. Ordered: mantener orden de inserción o de clave.
> 3. Persistent: inmutabilidad eficiente con structural sharing.
>
> Cada variante resuelve un problema que el hash map básico no puede,
> y cada una tiene un costo en rendimiento o memoria
> que necesitas entender antes de elegirla.

---

## El modelo mental: tres dimensiones del hash map

```
El hash map básico del Cap.04 tiene tres limitaciones:

  1. NO ES THREAD-SAFE
     Dos threads haciendo put() simultáneamente
     pueden corromper el backing array, perder datos,
     o causar infinite loops durante resize.
     "Simplemente ponerle un lock" funciona pero es lento.

  2. NO TIENE ORDEN
     Los elementos se almacenan según su hash, no según
     cuándo se insertaron o cuál es su clave.
     Iterar un HashMap de Java da un orden "aleatorio"
     que cambia después de cada resize.

  3. MUTAR ES DESTRUIR
     put() modifica el hash map in-place.
     Si alguien más tiene una referencia al mismo map,
     ve la modificación inmediatamente.
     En programación funcional, esto es un problema.

  Tres variantes, tres soluciones:

  CONCURRENT HASH MAP (Java ConcurrentHashMap, Go sync.Map):
    Múltiples threads leen y escriben simultáneamente.
    Técnicas: striped locking, CAS (compare-and-swap), lock-free.
    Costo: overhead de sincronización (~2-10× más lento bajo contención).

  ORDERED HASH MAP (Java LinkedHashMap, Python dict 3.7+):
    Mantiene el orden de inserción o de acceso.
    Técnica: hash map + doubly-linked list a través de los entries.
    Costo: 16 bytes extra por entry (prev + next pointers).

  PERSISTENT HASH MAP (Scala immutable.HashMap, Clojure PersistentHashMap):
    "Modificar" retorna un NUEVO hash map que comparte
    la mayor parte de la estructura con el original.
    Técnica: Hash Array Mapped Trie (HAMT) con path copying.
    Costo: O(log₃₂ n) por operación en vez de O(1).
```

---

## Tabla de contenidos

- [Sección 5.1 — Por qué synchronized HashMap no funciona](#sección-51--por-qué-synchronized-hashmap-no-funciona)
- [Sección 5.2 — ConcurrentHashMap: striped locking y CAS](#sección-52--concurrenthashmap-striped-locking-y-cas)
- [Sección 5.3 — Implementar un hash map con striped locking](#sección-53--implementar-un-hash-map-con-striped-locking)
- [Sección 5.4 — Concurrent maps en Go y Rust](#sección-54--concurrent-maps-en-go-y-rust)
- [Sección 5.5 — Ordered hash maps: LinkedHashMap y LRU cache](#sección-55--ordered-hash-maps-linkedhashmap-y-lru-cache)
- [Sección 5.6 — Persistent hash maps: HAMT y structural sharing](#sección-56--persistent-hash-maps-hamt-y-structural-sharing)
- [Sección 5.7 — El benchmark final: cuándo usar cada variante](#sección-57--el-benchmark-final-cuándo-usar-cada-variante)

---

## Sección 5.1 — Por Qué Synchronized HashMap No Funciona

### Ejercicio 5.1.1 — Leer: los problemas de un lock global

**Tipo: Leer**

```
Primera intuición: "ponle un lock al HashMap".

  synchronized HashMap:
    Cada operación (get, put, remove) adquiere un lock global.
    Solo un thread a la vez puede operar sobre el map.

  Collections.synchronizedMap(new HashMap<>()) hace exactamente esto.
  ¿Funciona? Sí, es correcto. ¿Es rápido? No.

  Problema 1: SERIALIZACIÓN TOTAL
    8 threads intentan hacer get() simultáneamente.
    Solo 1 ejecuta. Los otros 7 esperan.
    Throughput: ~igual que 1 thread. Los 7 threads extra son inútiles.

    Para get(), que es una operación de solo lectura,
    bloquear es innecesario. Múltiples lectores podrían operar
    en paralelo sin riesgo.

  Problema 2: LOCK CONTENTION
    Bajo alta carga, los threads pasan más tiempo esperando el lock
    que ejecutando la operación.
    Costo de lock/unlock en Java (uncontended): ~20 ns.
    Costo de lock/unlock (contended, 8 threads): ~500-2000 ns.
    → El lock cuesta más que el get() mismo (~28 ns).

  Problema 3: RESIZE BLOQUEA TODO
    Un put() que activa un resize necesita copiar todos los entries.
    Durante el resize (potencialmente millisegundos),
    TODOS los threads están bloqueados.

  Problema 4: COMPOUND OPERATIONS
    "Si la key no existe, insertarla" requiere dos operaciones:
      if (!map.containsKey(key)) {
          map.put(key, value);
      }
    Con synchronized, otro thread puede insertar entre containsKey y put.
    Necesitas sincronizar el bloque completo.
    → putIfAbsent() de ConcurrentHashMap lo resuelve atómicamente.

  Conclusión: un lock global es correcto pero lento.
  Lo que necesitamos es GRANULARIDAD FINA:
  bloquear solo la parte del map que estamos modificando.
```

**Preguntas:**

1. ¿`Collections.synchronizedMap()` es thread-safe?
   ¿Es útil en algún escenario?

2. ¿Un `ReentrantReadWriteLock` mejoraría el escenario de
   muchos lectores y pocos escritores?

3. ¿El costo de lock contended (500-2000 ns) depende de cuántos threads?

4. ¿`putIfAbsent()` es una "compound operation atómica".
   ¿Qué otras operaciones compuestas necesita un hash map concurrente?

5. ¿Un HashMap en un sistema single-threaded necesita sincronización?

---

### Ejercicio 5.1.2 — Implementar: demostrar la corrupción sin sincronización

**Tipo: Implementar**

```java
// Demostrar que HashMap sin sincronización se corrompe

import java.util.*;
import java.util.concurrent.*;

public class UnsafeHashMapDemo {

    public static void main(String[] args) throws Exception {
        int numThreads = 8;
        int opsPerThread = 100_000;

        // ═══ HashMap sin sincronización: UNSAFE ═══
        var unsafeMap = new HashMap<Integer, Integer>();
        var latch1 = new CountDownLatch(numThreads);
        var executor = Executors.newFixedThreadPool(numThreads);

        for (int t = 0; t < numThreads; t++) {
            final int threadId = t;
            executor.submit(() -> {
                for (int i = 0; i < opsPerThread; i++) {
                    int key = threadId * opsPerThread + i;
                    unsafeMap.put(key, key);
                }
                latch1.countDown();
            });
        }
        latch1.await();

        int expected = numThreads * opsPerThread;
        System.out.printf("UNSAFE HashMap: expected=%d, actual=%d, lost=%d%n",
            expected, unsafeMap.size(), expected - unsafeMap.size());

        // ═══ ConcurrentHashMap: SAFE ═══
        var safeMap = new ConcurrentHashMap<Integer, Integer>();
        var latch2 = new CountDownLatch(numThreads);

        for (int t = 0; t < numThreads; t++) {
            final int threadId = t;
            executor.submit(() -> {
                for (int i = 0; i < opsPerThread; i++) {
                    int key = threadId * opsPerThread + i;
                    safeMap.put(key, key);
                }
                latch2.countDown();
            });
        }
        latch2.await();

        System.out.printf("SAFE ConcurrentHashMap: expected=%d, actual=%d, lost=%d%n",
            expected, safeMap.size(), expected - safeMap.size());

        executor.shutdown();
    }
}
// Resultado (varía):
// UNSAFE HashMap: expected=800000, actual=~750000-790000, lost=~10000-50000
// SAFE ConcurrentHashMap: expected=800000, actual=800000, lost=0
//
// El HashMap pierde entries silenciosamente.
// En casos peores: infinite loop durante resize (thread se cuelga).
```

**Preguntas:**

1. ¿Cómo pierde entries el HashMap sin sincronización?
   ¿En qué operación?

2. ¿El infinite loop durante resize — ¿cómo ocurre?

3. ¿`HashMap` de Java documenta que no es thread-safe?

4. ¿El test puede pasar sin pérdidas por pura suerte?

5. ¿En Go, ¿un `map` sin sync da un error explícito
   o se corrompe silenciosamente?

---

### Ejercicio 5.1.3 — Implementar: benchmark synchronized vs ConcurrentHashMap

**Tipo: Implementar**

```java
import java.util.*;
import java.util.concurrent.*;

public class ConcurrentBenchmark {

    interface MapAdapter {
        void put(int key, int value);
        Integer get(int key);
    }

    static long benchmarkThroughput(MapAdapter map, int threads, int opsPerThread, double writeRatio) throws Exception {
        var executor = Executors.newFixedThreadPool(threads);
        var latch = new CountDownLatch(threads);
        var rng = ThreadLocalRandom.current();

        long start = System.nanoTime();
        for (int t = 0; t < threads; t++) {
            executor.submit(() -> {
                var localRng = ThreadLocalRandom.current();
                for (int i = 0; i < opsPerThread; i++) {
                    int key = localRng.nextInt(1_000_000);
                    if (localRng.nextDouble() < writeRatio) {
                        map.put(key, key);
                    } else {
                        map.get(key);
                    }
                }
                latch.countDown();
            });
        }
        latch.await();
        long elapsed = System.nanoTime() - start;
        executor.shutdown();
        return elapsed;
    }

    public static void main(String[] args) throws Exception {
        int opsPerThread = 1_000_000;

        // Pre-populate
        var syncMap = Collections.synchronizedMap(new HashMap<Integer, Integer>());
        var concMap = new ConcurrentHashMap<Integer, Integer>();
        for (int i = 0; i < 500_000; i++) {
            syncMap.put(i, i);
            concMap.put(i, i);
        }

        MapAdapter syncAdapter = new MapAdapter() {
            public void put(int k, int v) { syncMap.put(k, v); }
            public Integer get(int k) { return syncMap.get(k); }
        };
        MapAdapter concAdapter = new MapAdapter() {
            public void put(int k, int v) { concMap.put(k, v); }
            public Integer get(int k) { return concMap.get(k); }
        };

        System.out.printf("%-10s  %-10s  %-14s  %-14s  %-8s%n",
            "Threads", "Write %", "Sync (ms)", "Conc (ms)", "Speedup");
        System.out.println("─".repeat(62));

        for (int threads : new int[]{1, 2, 4, 8, 16}) {
            for (double writeRatio : new double[]{0.0, 0.10, 0.50, 1.0}) {
                long syncTime = benchmarkThroughput(syncAdapter, threads, opsPerThread, writeRatio);
                long concTime = benchmarkThroughput(concAdapter, threads, opsPerThread, writeRatio);
                System.out.printf("%-10d  %-10.0f%%  %-14d  %-14d  %-8.1f×%n",
                    threads, writeRatio * 100,
                    syncTime / 1_000_000, concTime / 1_000_000,
                    (double) syncTime / concTime);
            }
        }
    }
}
// Resultado esperado (orientativo):
// Threads   Write %   Sync (ms)       Conc (ms)       Speedup
// ──────────────────────────────────────────────────────────────
// 1         0%        250             280             0.9×
// 1         50%       280             300             0.9×
// 4         0%        900             150             6.0×
// 4         10%       950             180             5.3×
// 4         50%       1000            250             4.0×
// 8         0%        1800            160             11.3×
// 8         10%       1900            200             9.5×
// 8         50%       2000            350             5.7×
// 16        0%        3500            180             19.4×
// 16        50%       3800            500             7.6×
//
// Con 1 thread: ConcurrentHashMap tiene overhead (~10% más lento).
// Con 8+ threads: ConcurrentHashMap es 6-19× más rápido.
// Read-heavy (0% writes): mayor speedup (reads no bloquean).
```

**Preguntas:**

1. ¿Con 1 thread, synchronized es marginalmente más rápido.
   ¿Por qué?

2. ¿Con 16 threads y 0% writes, el speedup es ~19×.
   ¿Cómo logra ConcurrentHashMap que 16 reads corran en paralelo?

3. ¿El speedup disminuye con más writes. ¿Por qué?

4. ¿Para Spark que usa 8+ cores por executor,
   ¿el speedup de ConcurrentHashMap es relevante?

5. ¿El benchmark usa `ThreadLocalRandom`. ¿Qué pasaría
   con `Random` compartido?

---

### Ejercicio 5.1.4 — Analizar: Go y su detector de data races

**Tipo: Analizar**

```
Go toma un enfoque diferente a Java:
  - No hay "synchronized map" en la librería estándar.
  - El runtime tiene un DETECTOR de data races para maps.
  - Si dos goroutines acceden un map simultáneamente
    (al menos una escritura), el programa PANICS.

  Ejemplo:
    m := make(map[int]int)
    go func() { m[1] = 1 }()   // goroutine 1: write
    go func() { _ = m[1] }()   // goroutine 2: read
    // → fatal error: concurrent map read and map write

  Go NO te deja corromper silenciosamente (a diferencia de Java HashMap).
  El programa se detiene inmediatamente.

  Soluciones en Go:

  1. sync.Mutex:
     var mu sync.Mutex
     mu.Lock()
     m[key] = value
     mu.Unlock()
     → Lock global. Simple pero lento bajo contención.

  2. sync.RWMutex:
     var mu sync.RWMutex
     mu.RLock()     // múltiples lectores en paralelo
     _ = m[key]
     mu.RUnlock()
     mu.Lock()      // un solo escritor, bloquea lectores
     m[key] = value
     mu.Unlock()
     → Mejor para read-heavy workloads.

  3. sync.Map (Go 1.9+):
     var m sync.Map
     m.Store(key, value)      // thread-safe
     v, ok := m.Load(key)     // thread-safe
     → Optimizado para dos patrones:
       (a) write-once, read-many (config, caches)
       (b) disjoint key sets por goroutine (cada goroutine escribe su rango)
     → NO optimizado para escrituras frecuentes al mismo key range.

  4. Sharded map (manual):
     Crear N maps, cada uno con su propio mutex.
     Shard = hash(key) % N.
     → Equivalente a striped locking de ConcurrentHashMap.
     → Más código, pero mejor rendimiento.
```

**Preguntas:**

1. ¿El race detector de Go detecta TODOS los data races?
   ¿O solo los que ocurren durante la ejecución?

2. ¿`sync.Map` es más lento que `map` + `sync.RWMutex`
   para escrituras frecuentes?

3. ¿Por qué Go no tiene un equivalente a ConcurrentHashMap
   en la librería estándar?

4. ¿El patrón "sharded map" con 16 shards es suficiente
   para 100 goroutines?

5. ¿`go run -race` activa el detector. ¿Hay overhead de rendimiento?

---

### Ejercicio 5.1.5 — Analizar: Rust — data races imposibles en tiempo de compilación

**Tipo: Analizar**

```
Rust previene data races en TIEMPO DE COMPILACIÓN.
No en runtime como Go, no en documentación como Java.

  Regla de Rust:
    En cualquier momento, puedes tener:
    - UNA referencia mutable (&mut T), O
    - CUALQUIER NÚMERO de referencias inmutables (&T).
    - NUNCA ambas simultáneamente.

  Consecuencia: compartir un HashMap entre threads
  requiere wrappers explícitos.

  Opción 1: Arc<Mutex<HashMap<K, V>>>
    Arc: referencia compartida con reference counting atómico.
    Mutex: lock exclusivo.
    → Equivalente a synchronized HashMap de Java.
    → Simple, funciona, pero un solo thread a la vez.

  Opción 2: Arc<RwLock<HashMap<K, V>>>
    RwLock: múltiples lectores O un escritor.
    → Equivalente a ReadWriteLock de Java.
    → Mejor para read-heavy workloads.

  Opción 3: DashMap (crate externa)
    Sharded hash map concurrente.
    Internamente: Vec<RwLock<HashMap<K, V>>> con sharding.
    → Equivalente a ConcurrentHashMap de Java.
    → La opción más rápida para la mayoría de casos.

  Opción 4: Crossbeam / lock-free maps
    Estructuras lock-free usando CAS.
    → Máximo rendimiento, máxima complejidad.

  El compilador de Rust RECHAZA código con data races:
    use std::collections::HashMap;
    let mut map = HashMap::new();
    let r = &map;        // referencia inmutable
    map.insert(1, 2);    // ERROR: no puedes mutar mientras hay referencia inmutable
    println!("{:?}", r);

  Este error se detecta en COMPILACIÓN. No en tests. No en producción.
  → Rust es el único lenguaje de los 5 que GARANTIZA ausencia de data races
    sin overhead en runtime.
```

**Preguntas:**

1. ¿`Arc<Mutex<HashMap>>` tiene overhead. ¿Cuánto vs un HashMap solo?

2. ¿`DashMap` es el ConcurrentHashMap de Rust?
   ¿Por qué no está en la librería estándar?

3. ¿Si Rust previene data races en compilación,
   ¿es posible escribir código con data races en Rust?

4. ¿`unsafe` puede crear data races en Rust?

5. ¿Para un sistema de streaming en Rust,
   ¿cuál de las 4 opciones elegirías?

---

## Sección 5.2 — ConcurrentHashMap: Striped Locking y CAS

### Ejercicio 5.2.1 — Leer: la evolución de ConcurrentHashMap en Java

**Tipo: Leer**

```
ConcurrentHashMap ha evolucionado significativamente:

  JAVA 5-7: Segmented Locking
    La tabla se divide en N segmentos (default: 16).
    Cada segmento tiene su propio lock.
    → 16 threads pueden operar simultáneamente
      SI acceden a segmentos distintos.
    → Si dos threads acceden al mismo segmento, uno espera.

    Estructura (Java 7):
    ┌──────────────────────────────────────┐
    │  Segment[0] ─→ [bucket0, bucket1, ...]  lock_0  │
    │  Segment[1] ─→ [bucket2, bucket3, ...]  lock_1  │
    │  ...                                            │
    │  Segment[15] ─→ [bucket30, bucket31, ...] lock_15 │
    └──────────────────────────────────────┘

    Lectura: NO requiere lock (volatile reads).
    Escritura: lock del segmento correspondiente.

  JAVA 8+: CAS + Per-Bucket Locking
    Elimina los segmentos. Usa un array de Node[] (como HashMap).
    Cada BUCKET es su propia unidad de sincronización.

    Lectura: lockless (volatile + CAS).
    Escritura: synchronized en el PRIMER NODO del bucket.
    → Granularidad mucho más fina que Java 7.
    → Si la tabla tiene 1M buckets, 1M threads podrían operar
      en paralelo (si acceden a buckets distintos).

    Técnicas clave:

    1. VOLATILE NODE ARRAY:
       El array de buckets es volatile.
       Las lecturas ven la versión más reciente sin lock.

    2. CAS para el primer nodo:
       Si el bucket está vacío, CAS inserta el primer nodo
       sin lock (operación atómica del hardware).
       CAS = Compare-And-Swap:
         "Si el valor actual es X, cámbialo a Y. Si no, falla."

    3. SYNCHRONIZED en el primer nodo del bucket:
       Si el bucket YA tiene entries, se sincroniza en el primer nodo.
       Solo los threads que acceden al MISMO bucket compiten.

    4. TREE BINS (igual que HashMap):
       Chains de ≥8 nodos se convierten en Red-Black trees.

    5. CONCURRENT RESIZE:
       El resize NO bloquea toda la tabla.
       Múltiples threads COOPERAN en el resize:
       cada thread migra un rango de buckets.
       Mientras el resize está en progreso, los threads
       que necesitan un bucket ya migrado acceden a la nueva tabla,
       y los que necesitan uno no migrado acceden a la vieja.
```

**Preguntas:**

1. ¿16 segmentos de Java 7 → per-bucket de Java 8.
   ¿Cuánto mejora la concurrencia?

2. ¿CAS es una instrucción del hardware? ¿Qué pasa
   si falla?

3. ¿Concurrent resize donde múltiples threads cooperan
   — ¿es lock-free?

4. ¿La lectura es realmente sin lock en Java 8?
   ¿Qué garantiza la consistencia?

5. ¿Para un map con 100 entries y 16 threads,
   ¿cuánta contención hay?

---

### Ejercicio 5.2.2 — Analizar: Compare-And-Swap (CAS) — la primitiva atómica

**Tipo: Analizar**

```
CAS es la base de la programación lock-free.

  CAS(memory_location, expected_value, new_value):
    ATÓMICAMENTE:
      if *memory_location == expected_value:
          *memory_location = new_value
          return true   (éxito)
      else:
          return false  (falló, alguien modificó antes)

  Es una instrucción del CPU:
    x86: CMPXCHG (compare and exchange)
    ARM: LDREX + STREX (load/store exclusive) o CAS (ARMv8.1)

  Costo: ~5-10 ns (uncontended), ~50-200 ns (contended).
  Vs mutex: ~20 ns (uncontended), ~500-2000 ns (contended).

  Uso en ConcurrentHashMap (Java 8):
    // Insertar en bucket vacío
    Node[] table = this.table;
    int index = hash & (table.length - 1);
    if (table[index] == null) {
        if (CAS(table, index, null, newNode)) {
            // Éxito: insertamos sin lock
            return;
        }
        // Fallo: otro thread insertó primero. Reintentar.
    }

  Patrón CAS retry loop:
    while (true) {
        V current = atomicRef.get();
        V newValue = compute(current);
        if (atomicRef.compareAndSet(current, newValue)) {
            break;  // éxito
        }
        // Fallo: reintentar con el valor actual
    }

  Ventaja: sin lock, sin deadlock, sin inversión de prioridad.
  Desventaja: CAS loop puede SPINNING (gastar CPU sin progresar)
  bajo alta contención. → Peor que un lock en ese caso.

  Java: AtomicInteger, AtomicLong, AtomicReference, VarHandle.
  Rust: AtomicUsize, AtomicPtr (std::sync::atomic).
  Go: sync/atomic.CompareAndSwapInt64, atomic.Value.
```

**Preguntas:**

1. ¿CAS es más rápido que un lock. ¿Siempre?

2. ¿El CAS retry loop — ¿puede rodar indefinidamente (livelock)?

3. ¿ABA problem: CAS cree que el valor no cambió
   pero cambió de A→B→A. ¿Es un problema real?

4. ¿Java `AtomicInteger.incrementAndGet()` usa CAS internamente?

5. ¿En Rust, `Ordering::SeqCst` vs `Ordering::Relaxed` en CAS
   — ¿cuál es la diferencia de rendimiento?

---

### Ejercicio 5.2.3 — Analizar: las operaciones atómicas de ConcurrentHashMap

**Tipo: Analizar**

```
ConcurrentHashMap ofrece operaciones que HashMap no puede:

  1. putIfAbsent(key, value):
     ATÓMICAMENTE: si la key no existe, insertar.
     Si la key existe, retornar el valor existente sin modificar.
     → Equivalente a "INSERT IF NOT EXISTS" en SQL.
     → Sin esto, necesitarías:
       synchronized(map) { if (!map.containsKey(k)) map.put(k, v); }

  2. computeIfAbsent(key, mappingFunction):
     ATÓMICAMENTE: si la key no existe, calcular el valor con la función
     e insertarlo. Si existe, retornar el valor existente.
     → Ideal para caches: computeIfAbsent("user_123", k -> loadFromDB(k))
     → La función se ejecuta SOLO si la key no existe.

  3. merge(key, value, remappingFunction):
     ATÓMICAMENTE: si la key existe, combinar el valor viejo con el nuevo
     usando la función. Si no existe, insertar el valor directamente.
     → Ideal para contadores: merge(key, 1, Integer::sum)

  4. compute(key, remappingFunction):
     ATÓMICAMENTE: calcular un nuevo valor basado en la key y el valor
     actual (o null si no existe).
     → El más flexible, pero el más lento (siempre ejecuta la función).

  5. forEach / search / reduce (paralelos):
     Operaciones bulk que ejecutan en paralelo sobre los entries.
     forEach(parallelismThreshold, (k, v) -> { ... })
     → Si el map tiene más entries que el threshold,
       se ejecuta en el ForkJoinPool.

  Estas operaciones son IMPOSIBLES de implementar correctamente
  con un HashMap + lock externo sin race conditions.
  ConcurrentHashMap las implementa atómicamente por diseño.
```

**Preguntas:**

1. ¿`computeIfAbsent` es perfecto para un cache thread-safe?
   ¿Puede ejecutar la función dos veces para el mismo key?

2. ¿`merge(key, 1, Integer::sum)` como contador — ¿es más rápido
   que `AtomicLong`?

3. ¿Las operaciones bulk (forEach parallel) usan el ForkJoinPool
   común. ¿Es un problema?

4. ¿Python tiene un equivalente a `computeIfAbsent`?

5. ¿Spark usa alguna de estas operaciones atómicas
   en sus aggregators internos?

---

## Sección 5.3 — Implementar un Hash Map con Striped Locking

### Ejercicio 5.3.1 — Implementar: striped locking hash map en Java

**Tipo: Implementar**

```java
import java.util.concurrent.locks.ReentrantLock;

public class StripedHashMap<K, V> {

    private static class Node<K, V> {
        final K key;
        volatile V value;
        final int hash;
        volatile Node<K, V> next;
        Node(K key, V value, int hash, Node<K, V> next) {
            this.key = key; this.value = value; this.hash = hash; this.next = next;
        }
    }

    private volatile Node<K, V>[] table;
    private final ReentrantLock[] locks;
    private volatile int size;
    private final int numStripes;

    @SuppressWarnings("unchecked")
    public StripedHashMap(int capacity, int numStripes) {
        this.numStripes = numStripes;
        int cap = Integer.highestOneBit(Math.max(capacity, 16) - 1) << 1;
        this.table = new Node[cap];
        this.locks = new ReentrantLock[numStripes];
        for (int i = 0; i < numStripes; i++) {
            locks[i] = new ReentrantLock();
        }
    }

    public StripedHashMap() { this(64, 32); }

    private int hash(K key) {
        int h = key.hashCode();
        return h ^ (h >>> 16);
    }

    private int bucketIndex(int hash) {
        return hash & (table.length - 1);
    }

    private ReentrantLock lockFor(int hash) {
        return locks[(hash & 0x7FFFFFFF) % numStripes];
    }

    public V put(K key, V value) {
        int h = hash(key);
        ReentrantLock lock = lockFor(h);
        lock.lock();
        try {
            int idx = bucketIndex(h);
            for (Node<K, V> e = table[idx]; e != null; e = e.next) {
                if (e.hash == h && key.equals(e.key)) {
                    V old = e.value;
                    e.value = value;
                    return old;
                }
            }
            table[idx] = new Node<>(key, value, h, table[idx]);
            size++;
            return null;
        } finally {
            lock.unlock();
        }
    }

    public V get(K key) {
        int h = hash(key);
        int idx = bucketIndex(h);
        // Lectura sin lock (volatile array + volatile next)
        for (Node<K, V> e = table[idx]; e != null; e = e.next) {
            if (e.hash == h && key.equals(e.key)) {
                return e.value;
            }
        }
        return null;
    }

    public V remove(K key) {
        int h = hash(key);
        ReentrantLock lock = lockFor(h);
        lock.lock();
        try {
            int idx = bucketIndex(h);
            Node<K, V> prev = null;
            for (Node<K, V> e = table[idx]; e != null; e = e.next) {
                if (e.hash == h && key.equals(e.key)) {
                    if (prev == null) table[idx] = e.next;
                    else prev.next = e.next;
                    size--;
                    return e.value;
                }
                prev = e;
            }
            return null;
        } finally {
            lock.unlock();
        }
    }

    public int size() { return size; }

    public static void main(String[] args) throws Exception {
        var map = new StripedHashMap<Integer, Integer>(1_000_000, 64);
        int threads = 8;
        int opsPerThread = 125_000;
        var latch = new java.util.concurrent.CountDownLatch(threads);
        var exec = java.util.concurrent.Executors.newFixedThreadPool(threads);

        long start = System.nanoTime();
        for (int t = 0; t < threads; t++) {
            final int off = t * opsPerThread;
            exec.submit(() -> {
                for (int i = 0; i < opsPerThread; i++) map.put(off + i, i);
                latch.countDown();
            });
        }
        latch.await();
        long elapsed = System.nanoTime() - start;

        System.out.printf("Inserted %d entries with %d threads in %d ms%n",
            map.size(), threads, elapsed / 1_000_000);
        System.out.printf("%.0f ns/op%n",
            (double) elapsed / (threads * opsPerThread));
        exec.shutdown();
    }
}
```

**Preguntas:**

1. ¿32 stripes para 8 threads — ¿es suficiente? ¿Demasiado?

2. ¿`get()` no toma lock. ¿Es correcto? ¿Qué garantiza `volatile`?

3. ¿`volatile Node<K,V> next` — ¿es necesario para la lectura sin lock?

4. ¿Esta implementación soporta resize? ¿Qué falta?

5. ¿Cómo se compara con ConcurrentHashMap real en rendimiento?

---

### Ejercicio 5.3.2 — Implementar: sharded map concurrente en Go

**Tipo: Implementar**

```go
package main

import (
    "fmt"
    "hash/fnv"
    "sync"
    "time"
)

const NUM_SHARDS = 64

type Shard[V any] struct {
    mu   sync.RWMutex
    data map[string]V
}

type ShardedMap[V any] struct {
    shards [NUM_SHARDS]*Shard[V]
}

func NewShardedMap[V any]() *ShardedMap[V] {
    sm := &ShardedMap[V]{}
    for i := 0; i < NUM_SHARDS; i++ {
        sm.shards[i] = &Shard[V]{data: make(map[string]V)}
    }
    return sm
}

func (sm *ShardedMap[V]) getShard(key string) *Shard[V] {
    h := fnv.New64a()
    h.Write([]byte(key))
    return sm.shards[h.Sum64()%NUM_SHARDS]
}

func (sm *ShardedMap[V]) Set(key string, value V) {
    shard := sm.getShard(key)
    shard.mu.Lock()
    shard.data[key] = value
    shard.mu.Unlock()
}

func (sm *ShardedMap[V]) Get(key string) (V, bool) {
    shard := sm.getShard(key)
    shard.mu.RLock()
    val, ok := shard.data[key]
    shard.mu.RUnlock()
    return val, ok
}

func (sm *ShardedMap[V]) Delete(key string) {
    shard := sm.getShard(key)
    shard.mu.Lock()
    delete(shard.data, key)
    shard.mu.Unlock()
}

func (sm *ShardedMap[V]) Len() int {
    total := 0
    for _, s := range sm.shards {
        s.mu.RLock()
        total += len(s.data)
        s.mu.RUnlock()
    }
    return total
}

func main() {
    sm := NewShardedMap[int]()
    n := 1_000_000
    goroutines := 8
    perGoroutine := n / goroutines

    var wg sync.WaitGroup

    start := time.Now()
    for g := 0; g < goroutines; g++ {
        wg.Add(1)
        go func(offset int) {
            defer wg.Done()
            for i := 0; i < perGoroutine; i++ {
                key := fmt.Sprintf("key_%d", offset+i)
                sm.Set(key, offset+i)
            }
        }(g * perGoroutine)
    }
    wg.Wait()
    elapsed := time.Since(start)

    fmt.Printf("Inserted %d entries with %d goroutines in %v\n",
        sm.Len(), goroutines, elapsed)
    fmt.Printf("%.0f ns/op\n",
        float64(elapsed.Nanoseconds())/float64(n))

    // Verify
    v, ok := sm.Get("key_42")
    fmt.Printf("Get(key_42) = %d, found=%v\n", v, ok)
}
```

**Preguntas:**

1. ¿64 shards es un buen número? ¿Depende del número de goroutines?

2. ¿`RWMutex` en cada shard — ¿es necesario o basta con `Mutex`?

3. ¿`Len()` toma todos los read locks. ¿Puede dar un resultado inconsistente?

4. ¿Cómo se compara con `sync.Map` de Go para este workload?

5. ¿`fmt.Sprintf` para generar keys domina el benchmark?

---

### Ejercicio 5.3.3 — Implementar: concurrent hash map en Rust con DashMap

**Tipo: Implementar**

```rust
// Rust: comparar Mutex<HashMap>, RwLock<HashMap>, y DashMap
// Cargo.toml: [dependencies] dashmap = "5", rayon = "1"

use std::collections::HashMap;
use std::sync::{Arc, Mutex, RwLock};
use std::time::Instant;

fn bench_mutex(n: usize, threads: usize) -> std::time::Duration {
    let map = Arc::new(Mutex::new(HashMap::with_capacity(n)));
    let per_thread = n / threads;
    let mut handles = vec![];

    let start = Instant::now();
    for t in 0..threads {
        let map = Arc::clone(&map);
        handles.push(std::thread::spawn(move || {
            for i in 0..per_thread {
                let key = (t * per_thread + i) as i64;
                map.lock().unwrap().insert(key, key * 2);
            }
        }));
    }
    for h in handles { h.join().unwrap(); }
    start.elapsed()
}

fn bench_rwlock(n: usize, threads: usize) -> std::time::Duration {
    let map = Arc::new(RwLock::new(HashMap::with_capacity(n)));
    let per_thread = n / threads;
    let mut handles = vec![];

    let start = Instant::now();
    for t in 0..threads {
        let map = Arc::clone(&map);
        handles.push(std::thread::spawn(move || {
            for i in 0..per_thread {
                let key = (t * per_thread + i) as i64;
                map.write().unwrap().insert(key, key * 2);
            }
        }));
    }
    for h in handles { h.join().unwrap(); }
    start.elapsed()
}

fn bench_dashmap(n: usize, threads: usize) -> std::time::Duration {
    let map = Arc::new(dashmap::DashMap::with_capacity(n));
    let per_thread = n / threads;
    let mut handles = vec![];

    let start = Instant::now();
    for t in 0..threads {
        let map = Arc::clone(&map);
        handles.push(std::thread::spawn(move || {
            for i in 0..per_thread {
                let key = (t * per_thread + i) as i64;
                map.insert(key, key * 2);
            }
        }));
    }
    for h in handles { h.join().unwrap(); }
    start.elapsed()
}

fn main() {
    let n = 1_000_000;
    println!("Inserting {} entries\n", n);
    println!("{:<10} {:>12} {:>12} {:>12}",
        "Threads", "Mutex", "RwLock", "DashMap");
    println!("{}", "─".repeat(50));

    for threads in [1, 2, 4, 8] {
        let t1 = bench_mutex(n, threads);
        let t2 = bench_rwlock(n, threads);
        let t3 = bench_dashmap(n, threads);
        println!("{:<10} {:>10} ms {:>10} ms {:>10} ms",
            threads,
            t1.as_millis(), t2.as_millis(), t3.as_millis());
    }
}
// Resultado esperado:
// Threads       Mutex      RwLock     DashMap
// ──────────────────────────────────────────────
// 1             80 ms       85 ms      90 ms
// 2            160 ms      165 ms      55 ms
// 4            300 ms      310 ms      35 ms
// 8            550 ms      560 ms      25 ms
```

**Preguntas:**

1. ¿Mutex y RwLock se DEGRADAN con más threads (write-only).
   ¿Por qué?

2. ¿DashMap MEJORA con más threads. ¿Cómo?

3. ¿RwLock no ayuda porque el workload es 100% writes?

4. ¿Para un workload 90% reads / 10% writes,
   ¿RwLock sería competitivo con DashMap?

5. ¿DashMap usa cuántos shards por defecto?

---

## Sección 5.4 — Concurrent Maps en Go y Rust

### Ejercicio 5.4.1 — Analizar: sync.Map de Go — cuándo usarlo

**Tipo: Analizar**

```
sync.Map tiene una implementación particular:
dos maps internos — "read" (read-only, lock-free)
y "dirty" (requiere lock).

  Patrón: write-once, read-many
    Primera escritura: va al dirty map (con lock).
    Primera lectura (miss en read): promueve dirty → read (lock).
    Lecturas siguientes: del read map (SIN lock, atómico).
    → Si las keys se escriben una vez y se leen muchas veces,
      sync.Map es excelente.

  Patrón: escrituras frecuentes al mismo rango de keys
    Cada escritura va al dirty map (con lock).
    Las lecturas hacen miss en read → lock para leer dirty.
    → sync.Map es PEOR que un map + Mutex.

  Cuándo usar sync.Map:
    ✓ Caches donde las entries se establecen una vez
    ✓ Mapas de configuración (startup: write, runtime: read)
    ✓ Mapas donde cada goroutine escribe un rango disjunto de keys

  Cuándo NO usar sync.Map:
    ✗ Contadores incrementados frecuentemente
    ✗ Mapas con inserts/deletes constantes
    ✗ Mapas donde la mayoría de keys se actualizan

  Benchmark mental:
    sync.Map read (hot path):  ~15 ns (atomic load, sin lock)
    sync.Map write (miss):     ~200 ns (lock, copy, unlock)
    map + RWMutex read:        ~40 ns (RLock + read + RUnlock)
    map + RWMutex write:       ~80 ns (Lock + write + Unlock)
    sharded map read:          ~25 ns (shard lookup + RLock + read + RUnlock)
    sharded map write:         ~50 ns (shard lookup + Lock + write + Unlock)
```

**Preguntas:**

1. ¿`sync.Map` es type-safe con generics de Go 1.18+?

2. ¿El patrón "two-map" (read + dirty) es similar
   a copy-on-write?

3. ¿Para un DNS resolver cache, ¿sync.Map es ideal?

4. ¿La documentación de Go dice "for specific use cases".
   ¿Cuántos programadores Go usan sync.Map correctamente?

5. ¿orcaman/concurrent-map es una alternativa popular.
   ¿Es un sharded map?

---

### Ejercicio 5.4.2 — Analizar: cuándo usar cada variante concurrente

**Tipo: Analizar**

```
Guía de decisión para hash maps concurrentes:

  ┌─────────────────────────────────────────────────────────┐
  │ ¿Múltiples threads acceden al map?                      │
  │   No → usa HashMap normal (Java), HashMap (Rust), map (Go) │
  │   Sí → ↓                                                │
  │                                                         │
  │ ¿Cuál es el ratio reads/writes?                         │
  │                                                         │
  │   99% reads, 1% writes (config, cache estable):         │
  │     Java: ConcurrentHashMap                             │
  │     Go: sync.Map                                        │
  │     Rust: Arc<RwLock<HashMap>>                          │
  │                                                         │
  │   80% reads, 20% writes (cache activo):                 │
  │     Java: ConcurrentHashMap                             │
  │     Go: sharded map (manual o orcaman/concurrent-map)   │
  │     Rust: DashMap                                       │
  │                                                         │
  │   50% reads, 50% writes (aggregations):                 │
  │     Java: ConcurrentHashMap                             │
  │     Go: sharded map                                     │
  │     Rust: DashMap                                       │
  │                                                         │
  │   100% writes (bulk loading):                           │
  │     Java: ConcurrentHashMap o partitioned loading       │
  │     Go: sharded map                                     │
  │     Rust: DashMap o partitioned Vec → merge             │
  └─────────────────────────────────────────────────────────┘

  Regla de oro:
    ConcurrentHashMap (Java) funciona bien para TODO.
    Los otros lenguajes requieren más decisiones.
```

**Preguntas:**

1. ¿ConcurrentHashMap de Java es siempre la respuesta correcta
   para concurrencia en Java?

2. ¿Para Spark aggregations, ¿qué variante se usa internamente?

3. ¿"Partitioned loading" (cada thread construye su propio map,
   luego merge) — ¿es más rápido que un map concurrente?

4. ¿Scala Parallel Collections usan ConcurrentHashMap internamente?

5. ¿Un actor model (Akka) elimina la necesidad de maps concurrentes?

---

## Sección 5.5 — Ordered Hash Maps: LinkedHashMap y LRU Cache

### Ejercicio 5.5.1 — Leer: LinkedHashMap — hash map con memoria de orden

**Tipo: Leer**

```
LinkedHashMap de Java = HashMap + doubly-linked list.

  Cada Entry tiene, además de (key, value, hash, next en bucket),
  dos punteros extra: before y after.
  Estos punteros forman una doubly-linked list que conecta
  TODOS los entries en orden de inserción.

  Estructura:
  HashMap buckets:          Linked list (insertion order):
  [0] → null                head ↔ entry_A ↔ entry_C ↔ entry_B ↔ tail
  [1] → entry_A → null
  [2] → entry_B → entry_C → null
  [3] → null

  entry_A fue insertado primero, entry_C segundo, entry_B tercero.
  Iterar el LinkedHashMap recorre la linked list: A → C → B.
  Iterar un HashMap normal recorre los buckets: A → B → C (orden de hash).

  Dos modos:

  1. INSERTION ORDER (default):
     Los entries se recorren en el orden en que fueron insertados.
     put(key, value) agrega al FINAL de la linked list.
     put(key, newValue) para key existente NO cambia la posición.

  2. ACCESS ORDER (accessOrder = true):
     Cada get() o put() mueve el entry al FINAL de la linked list.
     El entry más recientemente accedido está al final.
     El entry menos recientemente accedido está al inicio.
     → Perfecto para un LRU cache.

  LRU Cache con LinkedHashMap:
    Overridear removeEldestEntry(eldest):
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > MAX_SIZE;
    }
    Cuando el map supera MAX_SIZE, se elimina el eldest
    (el menos recientemente accedido, al inicio de la linked list).

  Costo:
    Memoria: +16 bytes por entry (before + after pointers).
    Tiempo: put/get/remove idénticos a HashMap (~28-35 ns).
    Iteración: O(n) sobre entries reales (no buckets vacíos).
    → Iterar LinkedHashMap es más rápido que HashMap
      si el load factor es bajo (muchos buckets vacíos).
```

**Preguntas:**

1. ¿+16 bytes por entry para mantener el orden — ¿es mucho?

2. ¿Iteración de LinkedHashMap es O(n entries) vs HashMap que es
   O(n entries + capacity). ¿Cuándo importa la diferencia?

3. ¿Python dict (3.7+) mantiene insertion order.
   ¿Es como LinkedHashMap?

4. ¿Go tiene un equivalente a LinkedHashMap?

5. ¿Rust tiene un equivalente? (Pista: `indexmap` crate)

---

### Ejercicio 5.5.2 — Implementar: LRU cache con LinkedHashMap

**Tipo: Implementar**

```java
import java.util.*;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {

    private final int maxSize;

    public LRUCache(int maxSize) {
        super(maxSize * 4 / 3 + 1, 0.75f, true); // accessOrder = true
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }

    public static void main(String[] args) {
        var cache = new LRUCache<String, String>(3);

        cache.put("a", "Alice");
        cache.put("b", "Bob");
        cache.put("c", "Charlie");
        System.out.println("After 3 puts: " + cache.keySet()); // [a, b, c]

        cache.get("a"); // "a" se mueve al final (más reciente)
        System.out.println("After get(a): " + cache.keySet()); // [b, c, a]

        cache.put("d", "Diana"); // evicts "b" (least recently used)
        System.out.println("After put(d): " + cache.keySet()); // [c, a, d]

        System.out.println("get(b) = " + cache.get("b")); // null (evicted)
        System.out.println("get(a) = " + cache.get("a")); // Alice (still here)

        // Benchmark
        int n = 10_000_000;
        var perf = new LRUCache<Integer, Integer>(100_000);
        var rng = new Random(42);

        // Warmup
        for (int i = 0; i < 200_000; i++) perf.put(rng.nextInt(500_000), i);

        long start = System.nanoTime();
        int hits = 0;
        for (int i = 0; i < n; i++) {
            int key = rng.nextInt(500_000);
            if (perf.get(key) != null) {
                hits++;
            } else {
                perf.put(key, i);
            }
        }
        long elapsed = System.nanoTime() - start;

        System.out.printf("%nBenchmark: %d ops in %d ms (%.1f ns/op)%n",
            n, elapsed / 1_000_000, (double) elapsed / n);
        System.out.printf("Hit rate: %.1f%% (cache size=%d, key space=500K)%n",
            (double) hits / n * 100, perf.size());
    }
}
// Output:
// After 3 puts: [a, b, c]
// After get(a): [b, c, a]
// After put(d): [c, a, d]
// get(b) = null
// get(a) = Alice
//
// Benchmark: 10M ops in ~1200 ms (~120 ns/op)
// Hit rate: ~18% (100K cache, 500K key space → ~20% expected)
```

**Preguntas:**

1. ¿Este LRU cache es thread-safe? ¿Cómo lo harías thread-safe?

2. ¿120 ns/op — ¿es aceptable para un cache de producción?

3. ¿Redis usa un LRU cache con qué algoritmo internamente?

4. ¿Un LRU cache de 100K entries con 500K key space
   tiene hit rate ~20%. ¿Cómo mejorarlo?

5. ¿Caffeine (Java) es un LRU cache más sofisticado.
   ¿Qué ventajas tiene sobre LinkedHashMap?

---

### Ejercicio 5.5.3 — Implementar: LRU cache desde cero en Go

**Tipo: Implementar**

```go
package main

import (
    "container/list"
    "fmt"
)

type entry[K comparable, V any] struct {
    key   K
    value V
}

type LRUCache[K comparable, V any] struct {
    capacity int
    items    map[K]*list.Element
    order    *list.List // front = most recent, back = least recent
}

func NewLRUCache[K comparable, V any](capacity int) *LRUCache[K, V] {
    return &LRUCache[K, V]{
        capacity: capacity,
        items:    make(map[K]*list.Element, capacity),
        order:    list.New(),
    }
}

func (c *LRUCache[K, V]) Get(key K) (V, bool) {
    if elem, ok := c.items[key]; ok {
        c.order.MoveToFront(elem) // mark as recently used
        return elem.Value.(*entry[K, V]).value, true
    }
    var zero V
    return zero, false
}

func (c *LRUCache[K, V]) Put(key K, value V) {
    if elem, ok := c.items[key]; ok {
        c.order.MoveToFront(elem)
        elem.Value.(*entry[K, V]).value = value
        return
    }

    // Evict if full
    if c.order.Len() >= c.capacity {
        oldest := c.order.Back()
        if oldest != nil {
            c.order.Remove(oldest)
            delete(c.items, oldest.Value.(*entry[K, V]).key)
        }
    }

    elem := c.order.PushFront(&entry[K, V]{key, value})
    c.items[key] = elem
}

func (c *LRUCache[K, V]) Len() int { return c.order.Len() }

func main() {
    cache := NewLRUCache[string, int](3)

    cache.Put("a", 1)
    cache.Put("b", 2)
    cache.Put("c", 3)

    if v, ok := cache.Get("a"); ok { fmt.Printf("get(a) = %d\n", v) }

    cache.Put("d", 4) // evicts "b"

    _, ok := cache.Get("b")
    fmt.Printf("get(b) found: %v\n", ok) // false

    fmt.Printf("cache size: %d\n", cache.Len())
}
```

**Preguntas:**

1. ¿`container/list` de Go es un doubly-linked list?
   ¿Es eficiente?

2. ¿El `MoveToFront` es O(1)?

3. ¿Este cache es thread-safe? ¿Qué agregarías?

4. ¿Para un LRU cache en Rust, ¿existe un crate popular?

5. ¿Un LRU cache necesita la linked list?
   ¿Hay alternativas (ej: clock algorithm)?

---

## Sección 5.6 — Persistent Hash Maps: HAMT y Structural Sharing

### Ejercicio 5.6.1 — Leer: el problema de la inmutabilidad

**Tipo: Leer**

```
En programación funcional, los datos son INMUTABLES.
"Modificar" un hash map retorna un NUEVO hash map.
El original no cambia.

  val map1 = Map("a" -> 1, "b" -> 2)
  val map2 = map1 + ("c" -> 3)    // map2 es un NUEVO map
  // map1 sigue siendo Map("a" -> 1, "b" -> 2) — no cambió

  Enfoque naive: copiar todo el map.
    map1 tiene n entries. Crear map2 copia n entries + agrega 1.
    Costo: O(n) por operación. Inaceptable.

  Enfoque inteligente: STRUCTURAL SHARING.
    map2 comparte el 99% de la estructura con map1.
    Solo se crean los nodos que cambian.
    Costo: O(log n) por operación.

  La estructura que hace esto posible:
  HAMT — Hash Array Mapped Trie.

  Un HAMT es un trie (árbol de prefijos) sobre los bits del hash.
  Cada nivel del trie consume 5 bits del hash (32 children por nodo).
  Con hashes de 32 bits: máximo 7 niveles (32/5 ≈ 7).

  Nivel 0: bits 0-4 del hash   → 1 de 32 children
  Nivel 1: bits 5-9 del hash   → 1 de 32 children
  Nivel 2: bits 10-14 del hash → 1 de 32 children
  ...
  Nivel 6: bits 30-31 del hash → 1 de 4 children

  Para un map con 1M entries:
    Profundidad promedio: ~4 niveles.
    Lookup: 4 array accesses = O(log₃₂ n) ≈ O(4).
    Insert: crear 4 nuevos nodos (path copying) = O(log₃₂ n).
    Los demás ~999,996 entries se COMPARTEN con el map original.

  Esto es lo que usan:
  - Scala: immutable.HashMap (HAMT desde Scala 2.13)
  - Clojure: PersistentHashMap
  - Haskell: Data.HashMap.Strict (similar)
  - Immer (C++): persistent data structures
```

**Preguntas:**

1. ¿O(log₃₂ n) para 1 billón de entries = cuántos niveles?

2. ¿"Structural sharing" significa que map1 y map2 apuntan
   a los mismos nodos excepto en el path modificado?

3. ¿Si modificas 1000 keys, ¿creas 1000 paths?

4. ¿El HAMT tiene buena cache locality? (nodos de 32 children,
   ¿caben en una cache line?)

5. ¿Python y Go tienen estructuras persistentes?
   ¿Por qué no?

---

### Ejercicio 5.6.2 — Analizar: path copying paso a paso

**Tipo: Analizar**

```
Path copying: crear nuevas copias SOLO de los nodos en el path
desde la raíz hasta el nodo modificado.

  HAMT antes de insert("x", 42):
  (hash("x") = 0b_01010_11000_00101 → nivel 0 = 5, nivel 1 = 24, nivel 2 = 10)

       Root
      /    \     ...
     [5]   [...]
      |
    Node_A
   /     \     ...
  [24]   [...]
   |
  Node_B
   |
  [10] → ("y", 99)

  Después de insert("x", 42):
  Crear NUEVO Node_B' con [10] → ("x", 42) AND ("y", 99)
  Crear NUEVO Node_A' que apunta a Node_B' en [24] y comparte todo lo demás.
  Crear NUEVA Root' que apunta a Node_A' en [5] y comparte todo lo demás.

       Root'                       Root (original, no cambió)
      /    \     ...              /    \     ...
    [5]   [... shared ...]      [5]   [... shared ...]
      |                           |
    Node_A'                     Node_A
   /     \     ...             /     \     ...
  [24]  [... shared ...]     [24]  [... shared ...]
   |                           |
  Node_B'                    Node_B
   |                           |
  [10] → collision:           [10] → ("y", 99)
         ("x",42), ("y",99)

  Nodos creados: 3 (Root', Node_A', Node_B').
  Nodos compartidos: TODOS los demás (potencialmente miles).
  → Eficiente en tiempo Y espacio.
```

**Preguntas:**

1. ¿3 nodos nuevos para un insert. ¿Cuánta memoria es eso?

2. ¿Si dos threads hacen inserts simultáneos en distintas ramas,
   ¿se pueden paralelizar sin locks?

3. ¿El garbage collector limpia las versiones antiguas
   cuando nadie las referencia?

4. ¿Path copying es como copy-on-write de filesystems (ZFS, Btrfs)?

5. ¿Para un map de 10M entries, ¿la ventaja de structural sharing
   vs copiar todo es cuántas veces?

---

### Ejercicio 5.6.3 — Implementar: HAMT simplificado en Scala

**Tipo: Implementar**

```scala
// HAMT simplificado — branching factor 4 (2 bits por nivel) para claridad
// Un HAMT real usa branching factor 32 (5 bits por nivel)

sealed trait HAMTNode[+V]
case object Empty extends HAMTNode[Nothing]
case class Leaf[V](key: String, value: V, hash: Int) extends HAMTNode[V]
case class Branch[V](children: Vector[HAMTNode[V]]) extends HAMTNode[V]

class SimpleHAMT[V](root: HAMTNode[V] = Empty, val size: Int = 0) {

  private val BITS_PER_LEVEL = 2   // 4 children per node
  private val BRANCH_FACTOR = 1 << BITS_PER_LEVEL  // 4
  private val MASK = BRANCH_FACTOR - 1  // 0b11

  private def hashKey(key: String): Int = {
    var h = key.hashCode
    h ^ (h >>> 16)
  }

  private def indexAt(hash: Int, level: Int): Int =
    (hash >>> (level * BITS_PER_LEVEL)) & MASK

  def get(key: String): Option[V] = {
    val h = hashKey(key)
    get(root, key, h, 0)
  }

  private def get(node: HAMTNode[V], key: String, hash: Int, level: Int): Option[V] =
    node match {
      case Empty => None
      case Leaf(k, v, _) if k == key => Some(v)
      case Leaf(_, _, _) => None
      case Branch(children) =>
        val idx = indexAt(hash, level)
        get(children(idx), key, hash, level + 1)
    }

  def put(key: String, value: V): SimpleHAMT[V] = {
    val h = hashKey(key)
    val (newRoot, added) = put(root, key, value, h, 0)
    new SimpleHAMT(newRoot, if (added) size + 1 else size)
  }

  private def put(node: HAMTNode[V], key: String, value: V, hash: Int, level: Int): (HAMTNode[V], Boolean) =
    node match {
      case Empty =>
        (Leaf(key, value, hash), true)

      case Leaf(k, _, existingHash) if k == key =>
        (Leaf(key, value, hash), false) // update, no new entry

      case leaf @ Leaf(existingKey, existingValue, existingHash) =>
        // Collision at this level: create a Branch and insert both
        val emptyChildren = Vector.fill(BRANCH_FACTOR)(Empty: HAMTNode[V])
        var branch = Branch(emptyChildren)
        // Re-insert existing leaf
        val (b1, _) = put(branch, existingKey, existingValue, existingHash, level)
        // Insert new leaf
        val (b2, added) = put(b1, key, value, hash, level)
        (b2, added)

      case Branch(children) =>
        val idx = indexAt(hash, level)
        val (newChild, added) = put(children(idx), key, value, hash, level + 1)
        // PATH COPYING: create new Branch with updated child
        (Branch(children.updated(idx, newChild)), added)
    }

  override def toString: String = s"SimpleHAMT(size=$size)"
}

object HAMTDemo extends App {
  var map = new SimpleHAMT[Int]()

  // Insert
  for (i <- 0 until 10000) {
    map = map.put(s"key_$i", i)
  }
  println(s"After 10K inserts: $map")
  println(s"get(key_42) = ${map.get("key_42")}")
  println(s"get(nonexistent) = ${map.get("nonexistent")}")

  // Immutability demo
  val map1 = map
  val map2 = map.put("new_key", 999)
  println(s"map1.size = ${map1.size}") // 10000 (unchanged)
  println(s"map2.size = ${map2.size}") // 10001
  println(s"map1.get(new_key) = ${map1.get("new_key")}") // None
  println(s"map2.get(new_key) = ${map2.get("new_key")}") // Some(999)

  // Benchmark
  val n = 100_000
  val start = System.nanoTime()
  var bench = new SimpleHAMT[Int]()
  for (i <- 0 until n) bench = bench.put(s"k$i", i)
  val elapsed = (System.nanoTime() - start) / 1_000_000
  println(s"\n$n inserts: $elapsed ms (${elapsed * 1000 / n} μs/op)")

  // Compare with Scala immutable.HashMap
  val start2 = System.nanoTime()
  var scalaMap = scala.collection.immutable.HashMap.empty[String, Int]
  for (i <- 0 until n) scalaMap = scalaMap + (s"k$i" -> i)
  val elapsed2 = (System.nanoTime() - start2) / 1_000_000
  println(s"Scala HashMap: $elapsed2 ms (${elapsed2 * 1000 / n} μs/op)")
}
```

**Preguntas:**

1. ¿Branching factor 4 vs 32: ¿cuántos niveles para 1M entries?

2. ¿`children.updated(idx, newChild)` crea un nuevo Vector?
   ¿Es O(1) o O(32)?

3. ¿La inmutabilidad es visible: map1 no cambió después de map2.
   ¿Es gratis?

4. ¿Nuestro HAMT es más lento que `scala.collection.immutable.HashMap`?
   ¿Cuánto?

5. ¿Un HAMT con branching factor 32 usa bitmap + popcount
   para comprimir nodos sparse. ¿Cómo funciona?

---

### Ejercicio 5.6.4 — Analizar: bitmap compression en HAMT real

**Tipo: Analizar**

```
Un nodo HAMT con branching factor 32 tiene 32 slots.
Pero la mayoría están vacíos (especialmente cerca de la raíz).

  Nodo naive: array de 32 punteros = 256 bytes (64-bit).
  Si solo 3 de 32 slots están ocupados → 29 slots desperdiciados.

  BITMAP COMPRESSION:
  En vez de un array de 32, usar:
  1. Un bitmap de 32 bits donde bit i = 1 si el child i existe.
  2. Un array compacto con solo los children que existen.

  Ejemplo: children en posiciones 3, 7, 21
  Bitmap: 00000000_00100000_00000000_10001000 = 0x00200088
  Array:  [child_3, child_7, child_21]  (solo 3 entries, no 32)

  Para encontrar child_7:
  1. Verificar que bit 7 está encendido: bitmap & (1 << 7) != 0 ✓
  2. Contar bits encendidos antes de la posición 7:
     popcount(bitmap & ((1 << 7) - 1)) = popcount(0x00000088) = 1
     → El child_7 está en array[1].

  popcount (population count) es una instrucción del CPU:
  x86: POPCNT. ARM: CNT.
  Costo: 1 ciclo = ~0.3 ns.

  Ahorro de memoria:
  Nodo naive: 32 × 8 bytes = 256 bytes.
  Nodo comprimido: 4 bytes (bitmap) + 3 × 8 bytes (children) = 28 bytes.
  → 9× menos memoria para un nodo con 3 children.

  Esto es lo que hace Scala immutable.HashMap, Clojure PersistentHashMap,
  y la librería immer de C++.
```

**Preguntas:**

1. ¿POPCNT como instrucción del CPU — ¿todos los CPUs la tienen?

2. ¿El bitmap compression es lo que hace al HAMT real
   más eficiente en memoria que nuestro HAMT simplificado?

3. ¿El lookup con bitmap + popcount es más lento que un array access directo?

4. ¿Para un nodo lleno (32 children), ¿el bitmap ayuda?

5. ¿Roaring Bitmaps (Cap.03) y HAMT bitmap compression
   usan el mismo principio?

---

## Sección 5.7 — El Benchmark Final: Cuándo Usar Cada Variante

### Ejercicio 5.7.1 — Implementar: benchmark de todas las variantes en Java

**Tipo: Implementar**

```java
import java.util.*;
import java.util.concurrent.*;

public class AllMapsBenchmark {

    static final int N = 1_000_000;

    static long benchPut(Runnable setup, java.util.function.IntConsumer put) {
        setup.run();
        long start = System.nanoTime();
        for (int i = 0; i < N; i++) put.accept(i);
        return System.nanoTime() - start;
    }

    public static void main(String[] args) {
        // Single-threaded benchmarks
        System.out.println("═══ SINGLE-THREADED INSERT (1M entries) ═══");

        var hm = new HashMap<Integer, Integer>(N * 2);
        long t1 = benchPut(() -> hm.clear(), i -> hm.put(i, i));

        var lhm = new LinkedHashMap<Integer, Integer>(N * 2);
        long t2 = benchPut(() -> lhm.clear(), i -> lhm.put(i, i));

        var tm = new TreeMap<Integer, Integer>();
        long t3 = benchPut(() -> tm.clear(), i -> tm.put(i, i));

        var chm = new ConcurrentHashMap<Integer, Integer>(N * 2);
        long t4 = benchPut(() -> chm.clear(), i -> chm.put(i, i));

        System.out.printf("HashMap:            %4d ms  (%.0f ns/op)%n",
            t1/1_000_000, (double)t1/N);
        System.out.printf("LinkedHashMap:      %4d ms  (%.0f ns/op)%n",
            t2/1_000_000, (double)t2/N);
        System.out.printf("TreeMap:            %4d ms  (%.0f ns/op)%n",
            t3/1_000_000, (double)t3/N);
        System.out.printf("ConcurrentHashMap:  %4d ms  (%.0f ns/op)%n",
            t4/1_000_000, (double)t4/N);

        // Memory
        System.out.println("\n═══ MEMORY (approximate, 1M Integer→Integer) ═══");
        System.out.println("HashMap:            ~80 bytes/entry");
        System.out.println("LinkedHashMap:      ~96 bytes/entry (+16 for linked list)");
        System.out.println("TreeMap:            ~64 bytes/entry (tree node)");
        System.out.println("ConcurrentHashMap:  ~80 bytes/entry (similar to HashMap)");

        // Decision matrix
        System.out.println("\n═══ DECISION MATRIX ═══");
        System.out.println("Need                        → Use");
        System.out.println("─────────────────────────────────────────────────────");
        System.out.println("Basic key-value, single thread → HashMap");
        System.out.println("Maintain insertion order        → LinkedHashMap");
        System.out.println("LRU cache                      → LinkedHashMap(accessOrder)");
        System.out.println("Sorted by key                  → TreeMap");
        System.out.println("Thread-safe access              → ConcurrentHashMap");
        System.out.println("Immutable snapshots             → Scala immutable.HashMap (HAMT)");
    }
}
```

**Preguntas:**

1. ¿ConcurrentHashMap single-threaded es cuánto más lento que HashMap?

2. ¿LinkedHashMap insert es comparable a HashMap?

3. ¿TreeMap es significativamente más lento. ¿Esperado?

4. ¿La decision matrix debería ser una tabla de referencia
   como la del Cap.02?

5. ¿En Scala, ¿usarías `mutable.HashMap` o `immutable.HashMap`
   por defecto?

---

### Ejercicio 5.7.2 — Analizar: hash maps en la arquitectura de Spark

**Tipo: Analizar**

```
Spark usa hash maps en CADA fase de su ejecución:

  1. SHUFFLE (hash partitioning):
     Cada executor calcula hash(key) % numPartitions
     para decidir a qué reducer enviar cada record.
     → Murmur3 hash function.
     → El resultado determina la localidad de datos en el cluster.

  2. AGGREGATION (groupBy, reduceByKey):
     Dentro de cada executor, Spark mantiene un hash map
     en memoria para agregar valores por key.
     → ExternalAppendOnlyMap: hash map que puede derramar a disco
       si la memoria es insuficiente.
     → Usa ConcurrentHashMap internamente para concurrencia.

  3. BROADCAST JOIN:
     La tabla pequeña se convierte en un hash map.
     Se envía (broadcast) a todos los executors.
     Cada executor hace lookup en el hash map para cada row
     de la tabla grande.
     → O(n + m) en vez de O(n × m) de nested loop join.

  4. TUNGSTEN (off-heap):
     Spark descubrió que Java HashMap usaba 5× más memoria
     de la necesaria (object headers, boxing, pointer chasing).
     Solución: implementar su propio hash map off-heap
     con datos en formato binario (no objetos Java).
     → UnsafeHashedRelation: hash map con datos serializados.
     → Elimina GC, boxing, y object headers.

  El hash map no es solo una estructura de datos para Spark.
  Es la INFRAESTRUCTURA FUNDAMENTAL de todo el motor de ejecución.
  Optimizar el hash map de Spark optimiza todo el cluster.
```

**Preguntas:**

1. ¿Tungsten off-heap hash map — ¿es como lo que implementamos
   en Rust (sin GC, sin headers)?

2. ¿ExternalAppendOnlyMap derrama a disco. ¿Cómo funciona?

3. ¿Broadcast join con un HashMap de 100M entries
   — ¿cabe en memoria?

4. ¿Spark 3.x mejoró algo respecto al uso de hash maps?

5. ¿Si reescribieras Spark en Rust, ¿los hash maps de Tungsten
   serían innecesarios?

---

### Ejercicio 5.7.3 — Resumen: las reglas del hash map avanzado

**Tipo: Leer**

```
Reglas nuevas del Cap.05:

  Regla 17: Nunca uses HashMap con múltiples threads sin protección.
    Java HashMap se corrompe silenciosamente.
    Go map panics en runtime.
    Rust previene data races en compilación.
    Usa ConcurrentHashMap (Java), DashMap (Rust), sharded map (Go).

  Regla 18: ConcurrentHashMap escala, synchronized no.
    Un lock global serializa todo.
    ConcurrentHashMap (per-bucket locking) permite
    paralelismo proporcional al número de buckets.
    Bajo contención real (8+ threads): 5-20× de diferencia.

  Regla 19: LinkedHashMap cuando necesitas orden + O(1) lookup.
    +16 bytes por entry. Misma velocidad que HashMap.
    LRU cache en 5 líneas de código.

  Regla 20: HAMT cuando necesitas inmutabilidad eficiente.
    Structural sharing: O(log₃₂ n) por operación.
    Cada "modificación" comparte 99.99% de la estructura.
    Thread-safe por naturaleza (inmutable → no hay race conditions).
    Costo: ~4-10× más lento que HashMap mutable por operación.

  Regla 21: El hash map es la base de la infraestructura de datos.
    Spark shuffles, joins, aggregations.
    Kafka partitioning. Flink keyed state.
    Bases de datos hash joins.
    Redis y Memcached son hash maps distribuidos.
    Entender el hash map es entender la infraestructura.

  Mapa completo de decisión:

  ¿Single-threaded? → HashMap (Java), HashMap (Rust), map (Go)
  ¿Multi-threaded?  → ConcurrentHashMap / DashMap / sharded map
  ¿Necesitas orden?  → LinkedHashMap / indexmap
  ¿Necesitas sorted? → TreeMap / BTreeMap (Cap.09)
  ¿Necesitas inmutable? → HAMT (Scala) / im (Rust)
  ¿Necesitas LRU?    → LinkedHashMap(accessOrder) / Caffeine
```

**Preguntas:**

1. ¿Las 21 reglas cubren todo lo necesario hasta ahora?

2. ¿La Regla 20 aplica fuera de Scala y Clojure?

3. ¿La Regla 21 ("el hash map es la base de la infraestructura")
   se puede defender con datos concretos?

4. ¿Falta alguna variante importante de hash map?

5. ¿El próximo capítulo (Bloom filters) extiende el hash map
   o es una estructura completamente distinta?
