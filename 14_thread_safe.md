# Guía de Ejercicios — Cap.14: Estructuras Thread-Safe: Patrones y Tradeoffs

> Hacer una estructura thread-safe NO es "ponerle un mutex".
>
> Un mutex resuelve correctness. Pero destruye performance.
> Un HashMap con un mutex global serializa TODOS los accesos:
> 8 threads accediendo = 1 thread trabajando + 7 esperando.
>
> Los sistemas reales necesitan estrategias más inteligentes:
> - Striped locking: dividir el lock en N segmentos.
> - Read-write locks: múltiples readers, un writer.
> - Copy-on-write: readers sin lock, writers copian todo.
> - Lock-free con CAS: sin locks, sin deadlocks.
> - Ownership (Rust): el compilador previene data races.
>
> Cada patrón tiene un tradeoff.
> Striped locking escala con N segmentos pero complica el resize.
> RWLock beneficia read-heavy pero causa writer starvation.
> Copy-on-write es perfecto para reads pero prohibitivo para writes.
> Lock-free elimina deadlocks pero introduce ABA problem.
>
> Java ConcurrentHashMap usa striped locking (Java 7) y CAS (Java 8+).
> Rust previene data races en tiempo de compilación con Send + Sync.
> Go promueve "share memory by communicating" (channels).
>
> Este capítulo enseña los patrones para que cuando veas
> un ConcurrentHashMap o un Arc<Mutex<T>>, entiendas POR QUÉ
> está diseñado así y cuáles son sus límites.

---

## El modelo mental: el espectro de concurrencia

```
ESPECTRO DE ESTRATEGIAS (de simple a complejo):

  Mutex global     →  Simple, seguro, lento bajo contención.
  RWLock           →  Múltiples readers, un writer. Writer starvation.
  Striped locking  →  N locks independientes. Escala con N.
  Copy-on-write    →  Readers sin lock. Writers copian. Read-heavy.
  Lock-free (CAS)  →  Sin locks. Sin deadlocks. Complejo.
  Wait-free        →  Cada operación termina en N pasos. Rarísimo.

  Throughput bajo contención:

  Mutex:           ████░░░░░░░░░░░░░░░░  (1 thread a la vez)
  RWLock (reads):  ████████████░░░░░░░░  (N readers, pero writers bloquean)
  Striped (16):    ████████████████░░░░  (16 threads en paralelo)
  Lock-free:       ██████████████████░░  (todos progresan, CAS retries)
  Partitioned:     ████████████████████  (sin compartir, como sharding)

  La regla de oro:
    La concurrencia más rápida es NO compartir datos.
    Si no puedes evitar compartir, reduce el SCOPE del sharing.
    Si no puedes reducir el scope, usa el lock más fino posible.
```

---

## Tabla de contenidos

- [Sección 14.1 — Mutex y RWLock: lo básico y sus límites](#sección-141--mutex-y-rwlock-lo-básico-y-sus-límites)
- [Sección 14.2 — Striped locking: ConcurrentHashMap por dentro](#sección-142--striped-locking-concurrenthashmap-por-dentro)
- [Sección 14.3 — Copy-on-write: inmutabilidad como estrategia](#sección-143--copy-on-write-inmutabilidad-como-estrategia)
- [Sección 14.4 — Lock-free con CAS: Treiber stack y contadores](#sección-144--lock-free-con-cas-treiber-stack-y-contadores)
- [Sección 14.5 — ABA problem y memory reclamation](#sección-145--aba-problem-y-memory-reclamation)
- [Sección 14.6 — Rust ownership como solución de concurrencia](#sección-146--rust-ownership-como-solución-de-concurrencia)
- [Sección 14.7 — Patrones en producción: de Java a Go a Rust](#sección-147--patrones-en-producción-de-java-a-go-a-rust)

---

## Sección 14.1 — Mutex y RWLock: lo Básico y sus Límites

### Ejercicio 14.1.1 — Implementar: contador concurrente en Java, Go, Rust

**Tipo: Implementar**

```java
// Java: Mutex vs AtomicInteger vs LongAdder
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class CounterBenchmark {
    static final int THREADS = 8;
    static final int OPS_PER_THREAD = 5_000_000;

    interface Counter { void increment(); long get(); }

    static class MutexCounter implements Counter {
        private long count = 0;
        public synchronized void increment() { count++; }
        public synchronized long get() { return count; }
    }

    static class AtomicCounter implements Counter {
        private final AtomicLong count = new AtomicLong();
        public void increment() { count.incrementAndGet(); }
        public long get() { return count.get(); }
    }

    static class LongAdderCounter implements Counter {
        private final LongAdder adder = new LongAdder();
        public void increment() { adder.increment(); }
        public long get() { return adder.sum(); }
    }

    static long benchmark(Counter counter) throws Exception {
        var latch = new CountDownLatch(THREADS);
        long start = System.nanoTime();
        for (int t = 0; t < THREADS; t++) {
            new Thread(() -> {
                for (int i = 0; i < OPS_PER_THREAD; i++) counter.increment();
                latch.countDown();
            }).start();
        }
        latch.await();
        return System.nanoTime() - start;
    }

    public static void main(String[] args) throws Exception {
        long total = (long) THREADS * OPS_PER_THREAD;
        System.out.printf("%d threads × %,d ops = %,d total%n%n", THREADS, OPS_PER_THREAD, total);

        for (var entry : new Object[][]{
            {"synchronized", new MutexCounter()},
            {"AtomicLong", new AtomicCounter()},
            {"LongAdder", new LongAdderCounter()}
        }) {
            String name = (String) entry[0];
            Counter c = (Counter) entry[1];
            long elapsed = benchmark(c);
            System.out.printf("%-15s: %5d ms  (%3.0f ns/op)  count=%,d%n",
                name, elapsed / 1_000_000, (double) elapsed / total, c.get());
        }
    }
}
// Expected:
// synchronized:    ~2000 ms  (~50 ns/op)    ← lock contention
// AtomicLong:      ~800 ms   (~20 ns/op)    ← CAS, still contention
// LongAdder:       ~200 ms   (~5 ns/op)     ← per-core cells, minimal contention
```

**Preguntas:**

1. ¿LongAdder es 10× más rápido que synchronized. ¿Cómo?

2. ¿AtomicLong usa CAS. ¿Sigue teniendo contención?

3. ¿LongAdder tiene un Cell por core. ¿`sum()` es exacto?

4. ¿Go `sync.Mutex` vs `atomic.Int64` — ¿misma diferencia?

5. ¿Rust `Mutex<i64>` vs `AtomicI64` — ¿cuál es idiomático?

---

### Ejercicio 14.1.2 — Analizar: RWLock y el problema de writer starvation

**Tipo: Analizar**

```
Read-Write Lock: múltiples readers O un writer.

  Regla:
    Si no hay writers: N readers pueden leer en paralelo.
    Si un writer quiere escribir: espera a que todos los readers terminen.
    Mientras escribe: ningún reader ni writer puede acceder.

  Problema: WRITER STARVATION.
    Si los readers llegan constantemente, siempre hay al menos uno leyendo.
    El writer nunca encuentra un momento para escribir.
    → El writer espera indefinidamente.

  Soluciones:
    - Fair RWLock: writers tienen prioridad. Los nuevos readers
      esperan si hay un writer esperando.
      Java: ReentrantReadWriteLock(fair=true).
    - Writer-preferring: el default en muchas implementaciones.
    - Epoch-based: readers registran su "epoch". Writers esperan
      a que todos los readers del epoch actual terminen.

  ¿Cuándo usar RWLock?
    Solo si reads >> writes (90%+ reads) Y los reads son LENTOS.
    Si los reads son rápidos, el overhead del RWLock (acquire/release)
    puede ser mayor que simplemente usar un Mutex.

  Benchmark típico:
    Workload          Mutex    RWLock   Ratio
    ──────────        ─────    ──────   ─────
    100% reads        800 ms   200 ms   4×    ← RWLock gana
    50% reads/writes  800 ms   900 ms   0.9×  ← Mutex gana (RWLock overhead)
    100% writes       800 ms   1200 ms  0.7×  ← Mutex gana

  RWLock solo gana con workloads muy read-heavy.
```

**Preguntas:**

1. ¿Writer starvation con fair=true — ¿ahora los readers pueden starve?

2. ¿Go `sync.RWMutex` — ¿es writer-preferring?

3. ¿Rust `RwLock` — ¿es fair o writer-preferring?

4. ¿Para un cache read-heavy, ¿RWLock es la mejor opción?

5. ¿StampedLock de Java — ¿es una alternativa mejor al RWLock?

---

## Sección 14.2 — Striped Locking: ConcurrentHashMap por Dentro

### Ejercicio 14.2.1 — Implementar: hash map con striped locking en Java

**Tipo: Implementar**

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

public class StripedHashMap<K, V> {

    private static final int NUM_STRIPES = 16;
    private final ReentrantLock[] locks;
    private final List<Map.Entry<K, V>>[] buckets;
    private int size;

    @SuppressWarnings("unchecked")
    public StripedHashMap(int numBuckets) {
        locks = new ReentrantLock[NUM_STRIPES];
        for (int i = 0; i < NUM_STRIPES; i++) locks[i] = new ReentrantLock();
        buckets = new List[numBuckets];
        for (int i = 0; i < numBuckets; i++) buckets[i] = new ArrayList<>();
    }

    private int bucketIndex(K key) {
        return (key.hashCode() & 0x7FFFFFFF) % buckets.length;
    }

    private ReentrantLock lockFor(K key) {
        return locks[(key.hashCode() & 0x7FFFFFFF) % NUM_STRIPES];
    }

    public void put(K key, V value) {
        var lock = lockFor(key);
        lock.lock();
        try {
            int idx = bucketIndex(key);
            for (var entry : buckets[idx]) {
                if (entry.getKey().equals(key)) {
                    entry.setValue(value);
                    return;
                }
            }
            buckets[idx].add(new AbstractMap.SimpleEntry<>(key, value));
            size++;
        } finally { lock.unlock(); }
    }

    public V get(K key) {
        var lock = lockFor(key);
        lock.lock();
        try {
            int idx = bucketIndex(key);
            for (var entry : buckets[idx]) {
                if (entry.getKey().equals(key)) return entry.getValue();
            }
            return null;
        } finally { lock.unlock(); }
    }

    public int size() { return size; }

    public static void main(String[] args) throws Exception {
        int n = 2_000_000;
        int threads = 8;
        int opsPerThread = n / threads;

        // Striped
        var striped = new StripedHashMap<Integer, Integer>(n);
        long start = System.nanoTime();
        var latch1 = new CountDownLatch(threads);
        for (int t = 0; t < threads; t++) {
            final int tid = t;
            new Thread(() -> {
                for (int i = tid; i < n; i += threads) striped.put(i, i);
                latch1.countDown();
            }).start();
        }
        latch1.await();
        long stripedTime = System.nanoTime() - start;

        // Single mutex
        var mutexMap = Collections.synchronizedMap(new HashMap<Integer, Integer>(n));
        start = System.nanoTime();
        var latch2 = new CountDownLatch(threads);
        for (int t = 0; t < threads; t++) {
            final int tid = t;
            new Thread(() -> {
                for (int i = tid; i < n; i += threads) mutexMap.put(i, i);
                latch2.countDown();
            }).start();
        }
        latch2.await();
        long mutexTime = System.nanoTime() - start;

        // ConcurrentHashMap
        var chm = new ConcurrentHashMap<Integer, Integer>(n);
        start = System.nanoTime();
        var latch3 = new CountDownLatch(threads);
        for (int t = 0; t < threads; t++) {
            final int tid = t;
            new Thread(() -> {
                for (int i = tid; i < n; i += threads) chm.put(i, i);
                latch3.countDown();
            }).start();
        }
        latch3.await();
        long chmTime = System.nanoTime() - start;

        System.out.printf("%d threads, %,d inserts:%n", threads, n);
        System.out.printf("  Striped (16 locks):   %5d ms%n", stripedTime / 1_000_000);
        System.out.printf("  SynchronizedMap:      %5d ms%n", mutexTime / 1_000_000);
        System.out.printf("  ConcurrentHashMap:    %5d ms%n", chmTime / 1_000_000);
        System.out.printf("  Striped/Mutex:        %.1f× faster%n", (double) mutexTime / stripedTime);
    }
}
// Expected:
// Striped (16 locks):    ~300-600 ms
// SynchronizedMap:       ~1500-3000 ms
// ConcurrentHashMap:     ~200-400 ms
// Striped ~3-5× faster than single mutex.
// ConcurrentHashMap slightly faster (Java 8+ uses CAS, not stripes).
```

**Preguntas:**

1. ¿16 stripes = 16 locks independientes. ¿Por qué 16?

2. ¿ConcurrentHashMap Java 8+ no usa striped locking. ¿Qué usa?

3. ¿Si dos keys mapean al mismo stripe, ¿hay contención?

4. ¿Resize con striped locking — ¿necesitas tomar TODOS los locks?

5. ¿Python `threading.Lock` vs Go `sync.Mutex` — ¿cuál es más eficiente?

---

## Sección 14.3 — Copy-on-Write: Inmutabilidad como Estrategia

### Ejercicio 14.3.1 — Analizar: CopyOnWriteArrayList de Java

**Tipo: Analizar**

```
CopyOnWriteArrayList: reads sin lock, writes copian todo el array.

  READ: leer directamente del array actual. Sin lock. O(1).
  WRITE: crear una COPIA del array, modificar la copia,
         reemplazar el array atómicamente. O(n).

  Implementación interna simplificada:
    volatile Object[] array;  // el array actual

    T get(int index) {
        return array[index];  // no lock needed!
    }

    void add(T element) {
        synchronized(this) {
            Object[] old = array;
            Object[] copy = Arrays.copyOf(old, old.length + 1);
            copy[old.length] = element;
            array = copy;  // volatile write: visible to all threads
        }
    }

  ¿Por qué funciona sin locks para reads?
    El array es `volatile`: la escritura es visible inmediatamente.
    Los readers ven una versión CONSISTENTE (el array viejo o el nuevo).
    Nunca ven un estado parcialmente actualizado.
    → Es un SNAPSHOT read.

  ¿Cuándo usar?
    READ >> WRITE (99%+ reads):
      - Listener lists (event handlers): se registran una vez, se notifican miles.
      - Configuration: se actualiza raramente, se lee constantemente.
      - Cache: se regenera periódicamente, se consulta continuamente.

    NUNCA para write-heavy:
      Cada write copia todo el array. O(n) por write.
      100 writes en un array de 10K = 1M copias de elementos.

  Equivalente en otros lenguajes:
    Rust: no existe directamente. Arc<RwLock<Vec<T>>> o inmutable clones.
    Go: no existe. Patrón manual con atomic.Value.
    Scala: las colecciones inmutables SON copy-on-write por naturaleza.
```

**Preguntas:**

1. ¿Readers sin lock = zero overhead. ¿Es siempre mejor que RWLock?

2. ¿Writers copian TODO el array. ¿Para 1M elementos?

3. ¿`volatile` en Java garantiza visibilidad. ¿Es lo mismo que `atomic` en Go?

4. ¿Scala colecciones inmutables son COW. ¿El persistent HAMT del Cap.05?

5. ¿Para un cache compartido con 99% reads, ¿COW o ConcurrentHashMap?

---

## Sección 14.4 — Lock-Free con CAS: Treiber Stack y Contadores

### Ejercicio 14.4.1 — Implementar: Treiber stack lock-free en Java

**Tipo: Implementar**

```java
import java.util.concurrent.atomic.*;

public class TreiberStack<T> {

    private static class Node<T> {
        final T value;
        Node<T> next;
        Node(T value) { this.value = value; }
    }

    private final AtomicReference<Node<T>> top = new AtomicReference<>(null);

    public void push(T value) {
        Node<T> newNode = new Node<>(value);
        while (true) {
            Node<T> current = top.get();
            newNode.next = current;
            if (top.compareAndSet(current, newNode)) return;
            // CAS failed: another thread pushed first. Retry.
        }
    }

    public T pop() {
        while (true) {
            Node<T> current = top.get();
            if (current == null) return null;
            if (top.compareAndSet(current, current.next)) return current.value;
            // CAS failed: another thread popped first. Retry.
        }
    }

    public static void main(String[] args) throws Exception {
        var stack = new TreiberStack<Integer>();
        int threads = 8;
        int opsPerThread = 1_000_000;
        var latch = new java.util.concurrent.CountDownLatch(threads);

        long start = System.nanoTime();
        for (int t = 0; t < threads; t++) {
            new Thread(() -> {
                for (int i = 0; i < opsPerThread; i++) {
                    stack.push(i);
                    stack.pop();
                }
                latch.countDown();
            }).start();
        }
        latch.await();
        long elapsed = System.nanoTime() - start;

        long totalOps = (long) threads * opsPerThread * 2;
        System.out.printf("Treiber Stack: %,d ops in %d ms (%.0f ns/op)%n",
            totalOps, elapsed / 1_000_000, (double) elapsed / totalOps);
    }
}
// Expected: ~500-1500 ms depending on contention
```

**Preguntas:**

1. ¿`compareAndSet(current, newNode)` — ¿qué pasa si falla?

2. ¿El loop `while(true)` con CAS — ¿es un spin lock?

3. ¿Bajo alta contención, ¿cuántos retries por operación?

4. ¿Un stack lock-free es útil para qué en la práctica?

5. ¿Java `ConcurrentLinkedDeque` usa un enfoque similar?

---

## Sección 14.5 — ABA Problem y Memory Reclamation

### Ejercicio 14.5.1 — Leer: el ABA problem explicado

**Tipo: Leer**

```
ABA Problem: el talón de Aquiles de CAS.

  CAS(expected, new): "si el valor actual es expected, cámbialo a new".
  Pero: ¿qué pasa si el valor era A, cambió a B, y volvió a A?

  Ejemplo con Treiber stack:
    Estado: top → [A] → [B] → [C]

    Thread 1: quiere pop(). Lee top=A, prepara CAS(A, B).
    Thread 1: SUSPENDIDO por el scheduler.

    Thread 2: pop() → saca A.     top → [B] → [C]
    Thread 2: pop() → saca B.     top → [C]
    Thread 2: push(A).            top → [A] → [C]   (A reutilizado!)

    Thread 1: RESUMIDO. CAS(A, B) → ¡ÉXITO! (top sigue siendo A)
    Pero top ahora apunta a [B] → [C], y B ya fue liberado.
    → CORRUPCIÓN DE MEMORIA.

  El CAS solo verifica que el PUNTERO es el mismo.
  No verifica que el ESTADO no cambió.

  Soluciones:

  1. TAGGED POINTERS (counter + pointer):
     En vez de CAS(pointer), usar CAS(pointer + version_counter).
     Cada modificación incrementa el counter.
     CAS falla si el counter cambió, incluso si el pointer es el mismo.
     → Java AtomicStampedReference.

  2. EPOCH-BASED RECLAMATION:
     No reutilizar memoria inmediatamente.
     Mantener "epochs" (generaciones). Los nodos viejos se liberan
     solo cuando NINGÚN thread puede estar referenciándolos.
     → Rust crossbeam-epoch usa esto.

  3. HAZARD POINTERS:
     Cada thread publica los punteros que está usando.
     Antes de liberar un nodo, verificar que ningún thread lo usa.
     → Más preciso que epochs, pero más overhead.

  4. GARBAGE COLLECTION:
     Java/Go/Scala: el GC se encarga. Los nodos no se reutilizan
     mientras alguien los referencia.
     → El ABA problem es MENOS relevante en lenguajes con GC.
     (Pero no inexistente: el CAS puede ver el mismo objeto
      con estado diferente.)
```

**Preguntas:**

1. ¿En Java con GC, ¿el ABA problem es un problema real?

2. ¿Rust sin GC necesita epoch-based reclamation. ¿crossbeam?

3. ¿Tagged pointers: ¿el counter se desborda?

4. ¿Go con GC: ¿el ABA problem existe para `atomic.Pointer`?

5. ¿Para la mayoría de programadores, ¿el ABA problem es relevante?

---

## Sección 14.6 — Rust Ownership como Solución de Concurrencia

### Ejercicio 14.6.1 — Analizar: Send + Sync como prevención de data races

**Tipo: Analizar**

```
Rust previene data races EN TIEMPO DE COMPILACIÓN.

  Reglas del borrow checker:
    1. Un valor tiene UN owner.
    2. Puedes tener N references inmutables (&T) O 1 reference mutable (&mut T).
    3. NUNCA ambas al mismo tiempo.

  Para concurrencia, Rust agrega:
    Send: el tipo puede ser ENVIADO a otro thread.
    Sync: el tipo puede ser COMPARTIDO entre threads (via &T).

    Tipo             Send    Sync    Uso
    ────             ────    ────    ───
    i32              ✓       ✓       Cualquier thread
    String           ✓       ✓       Cualquier thread
    Rc<T>            ✗       ✗       Solo un thread (no atómico)
    Arc<T>           ✓       ✓       Shared ownership entre threads
    Mutex<T>         ✓       ✓       Exclusive access entre threads
    RwLock<T>        ✓       ✓       Multiple readers / single writer
    Cell<T>          ✓       ✗       Interior mutability, single thread
    AtomicI64        ✓       ✓       Lock-free counter

  Patrones comunes:

  Arc<Mutex<T>>:
    Shared ownership (Arc) + exclusive access (Mutex).
    Equivalente a synchronized en Java, pero enforced por el compilador.

    let data = Arc::new(Mutex::new(Vec::new()));
    let data_clone = Arc::clone(&data);
    thread::spawn(move || {
        let mut guard = data_clone.lock().unwrap();
        guard.push(42);  // ← safe: Mutex guarantees exclusive access
    });

  Arc<RwLock<T>>:
    Shared ownership + read-write lock.
    Múltiples readers O un writer.

  Sin Arc ni Mutex:
    let data = vec![1, 2, 3];
    thread::spawn(move || {  // move ownership to new thread
        println!("{:?}", data);  // data is now OWNED by this thread
    });
    // println!("{:?}", data);  // ← COMPILE ERROR: data was moved

  Rust NO PUEDE tener data races:
    El compilador rechaza código que comparte datos mutables
    entre threads sin protección adecuada.
    → "Fearless concurrency".
```

**Preguntas:**

1. ¿`Rc<T>` no es `Send`. ¿Porque no es atómico?

2. ¿`Arc<Mutex<T>>` es el patrón default en Rust.
   ¿Es más lento que ConcurrentHashMap de Java?

3. ¿Rust previene data races pero no deadlocks.
   ¿Por qué no puede prevenir deadlocks?

4. ¿Go "share memory by communicating". ¿Es más seguro que Rust?

5. ¿Para un data engineer escribiendo en Python/Java,
   ¿importa la concurrencia de Rust?

---

## Sección 14.7 — Patrones en Producción: de Java a Go a Rust

### Ejercicio 14.7.1 — Analizar: qué patrón usar cuándo

**Tipo: Analizar**

```
Guía de decisión por lenguaje:

  JAVA:
    ¿HashMap concurrent?           → ConcurrentHashMap (CAS + synchronized)
    ¿Sorted map concurrent?        → ConcurrentSkipListMap (lock-free)
    ¿List read-heavy?              → CopyOnWriteArrayList
    ¿Counter concurrent?           → LongAdder (per-core cells)
    ¿Queue producer-consumer?      → ArrayBlockingQueue / LinkedBlockingQueue
    ¿Queue lock-free?              → ConcurrentLinkedQueue (Michael-Scott)

  GO:
    ¿Map concurrent?               → sync.Map (read-heavy) o Mutex + map
    ¿Producer-consumer?            → channels (buffered)
    ¿Counter concurrent?           → atomic.Int64
    ¿Read-heavy shared state?      → atomic.Value (copy-on-write manual)
    ¿Complex concurrent logic?     → channels + goroutines

  RUST:
    ¿Shared mutable state?         → Arc<Mutex<T>> o Arc<RwLock<T>>
    ¿Lock-free?                    → crossbeam crate
    ¿Map concurrent?               → dashmap crate
    ¿Counter concurrent?           → AtomicI64
    ¿Async producer-consumer?      → tokio::sync::mpsc channel
    ¿No compartir datos?           → move ownership to thread

  SCALA:
    ¿Colecciones inmutables?       → Default (persistent data structures)
    ¿Mutable concurrent?           → TrieMap (concurrent trie)
    ¿Actor model?                  → Akka actors (message passing)

  Regla general:
    1. Evita compartir estado mutable si puedes.
    2. Si compartes: usa las estructuras concurrent del stdlib.
    3. Si necesitas más rendimiento: CAS / lock-free.
    4. Si necesitas ultra-rendimiento: LMAX Disruptor / partitioning.
```

**Preguntas:**

1. ¿`sync.Map` de Go — ¿cuándo es mejor que `Mutex + map`?

2. ¿DashMap de Rust — ¿es como ConcurrentHashMap de Java?

3. ¿Scala TrieMap — ¿es el concurrent trie mencionado en Cap.11?

4. ¿Akka actors vs channels de Go — ¿cuál es mejor?

5. ¿Para un data engineer, ¿cuánto de esto necesita saber?

---

### Ejercicio 14.7.2 — Resumen: las reglas de concurrencia

**Tipo: Leer**

```
Reglas nuevas del Cap.14:

  Regla 59: No compartir es la mejor concurrencia.
    Si cada thread tiene sus propios datos, no hay contención.
    Partitioning, message passing, move ownership.
    Compartir datos mutables es la fuente de TODOS los bugs de concurrencia.

  Regla 60: Striped locking escala con N segmentos.
    En vez de 1 lock global, dividir en N locks independientes.
    ConcurrentHashMap Java 7: 16 segments.
    Contención se reduce N veces. Simple y efectivo.

  Regla 61: Copy-on-write para read-heavy workloads.
    Readers sin lock (zero overhead). Writers copian todo.
    Solo viable si writes son raros (1% o menos).
    CopyOnWriteArrayList, atomic.Value (Go), colecciones inmutables Scala.

  Regla 62: CAS es la primitiva de lock-free.
    Compare-And-Swap: "si el valor es X, cámbialo a Y".
    Treiber stack, Michael-Scott queue, ConcurrentHashMap Java 8+.
    Sin deadlocks, pero introduce ABA problem y complejidad.

  Regla 63: Rust previene data races en compilación.
    Send + Sync traits. Borrow checker.
    Arc<Mutex<T>> como patrón default.
    El compilador rechaza código con data races potenciales.
    → "Fearless concurrency" — no es marketing, es real.

  La Parte 5 continúa:
    Cap.15: Consistent Hashing, Merkle Trees, CRDTs.
    → Las estructuras que hacen posible la distribución de datos.
```

**Preguntas:**

1. ¿63 reglas en 14 capítulos. ¿Es un ritmo sostenible?

2. ¿La Regla 59 (no compartir) es la más importante del capítulo?

3. ¿Estas reglas aplican directamente al trabajo de un data engineer?

4. ¿El Cap.15 conecta con Cassandra, Kafka, y sistemas distribuidos?

5. ¿El proyecto final (Cap.18) necesitará estructuras concurrent?
