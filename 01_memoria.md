# Guía de Ejercicios — Cap.01: Cómo Vive un Dato en Memoria

> Antes de implementar un hash map, un B-Tree, o una skip list,
> necesitas entender una verdad incómoda:
>
> La complejidad algorítmica miente.
>
> Recorrer un array de 1 millón de elementos es O(n).
> Recorrer una linked list de 1 millón de elementos también es O(n).
> Pero el array termina en 1 milisegundo
> y la linked list tarda 50 milisegundos.
>
> La diferencia no está en el algoritmo.
> Está en cómo vive cada dato en la memoria física,
> cómo el CPU accede a esa memoria,
> y cuánto tiempo pasa esperando
> a que los datos lleguen desde la RAM.
>
> Si no entiendes esto, vas a elegir estructuras de datos
> basándote en tablas de Big-O que no reflejan la realidad.
>
> Este capítulo te enseña a ver la memoria como la ve el hardware.

---

## El modelo mental: la jerarquía de memoria

```
El CPU no lee de la RAM directamente.
Lee de una jerarquía de caches, cada nivel más grande y más lento:

  ┌──────────────┐
  │  Registros   │  < 1 ns     ~1 KB      dentro del CPU
  ├──────────────┤
  │  Cache L1    │  ~1 ns      32-64 KB   uno por core
  ├──────────────┤
  │  Cache L2    │  ~4 ns      256 KB-1 MB uno por core
  ├──────────────┤
  │  Cache L3    │  ~10 ns     8-64 MB    compartido entre cores
  ├──────────────┤
  │  RAM (DRAM)  │  ~100 ns    8-512 GB   la "memoria principal"
  ├──────────────┤
  │  SSD (NVMe)  │  ~10 μs     1-8 TB     almacenamiento persistente
  ├──────────────┤
  │  HDD         │  ~10 ms     1-20 TB    almacenamiento mecánico
  └──────────────┘

La diferencia entre L1 y RAM es 100×.
La diferencia entre RAM y SSD es 100×.
La diferencia entre L1 y HDD es 10,000,000×.

Cuando tu programa accede a un dato que está en L1: 1 nanosegundo.
Cuando ese dato no está en ningún cache (cache miss): 100 nanosegundos.
Tu programa esperó 99 nanosegundos sin hacer nada.

  → Una estructura de datos que causa muchos cache misses
    es lenta aunque su complejidad algorítmica sea óptima.

  → Una estructura de datos que mantiene los datos juntos en memoria
    (buena "localidad de referencia") aprovecha los caches
    y es rápida en la práctica.
```

```
Regla fundamental de este libro:

  La estructura de datos más rápida
  es la que causa menos cache misses.

  No la que tiene mejor Big-O.
  No la que es más elegante.
  La que mantiene los datos que el CPU necesita
  cerca unos de otros en memoria.

  Array:       datos contiguos → cache-friendly → rápido
  Linked list: datos dispersos → cache-hostile  → lento

  Esta regla explica el 80% de las diferencias
  de rendimiento entre estructuras de datos
  que la teoría dice que deberían ser iguales.
```

---

## Tabla de contenidos

- [Sección 1.1 — Stack vs Heap: dónde viven tus datos](#sección-11--stack-vs-heap-dónde-viven-tus-datos)
- [Sección 1.2 — Localidad de referencia: el secreto del rendimiento](#sección-12--localidad-de-referencia-el-secreto-del-rendimiento)
- [Sección 1.3 — Cache lines: la unidad real de transferencia](#sección-13--cache-lines-la-unidad-real-de-transferencia)
- [Sección 1.4 — Alineación, padding, y el costo oculto de los structs](#sección-14--alineación-padding-y-el-costo-oculto-de-los-structs)
- [Sección 1.5 — Java: object headers, boxing, y el precio de la abstracción](#sección-15--java-object-headers-boxing-y-el-precio-de-la-abstracción)
- [Sección 1.6 — Go y Rust: dos filosofías de memoria](#sección-16--go-y-rust-dos-filosofías-de-memoria)
- [Sección 1.7 — El benchmark que cambia todo: array vs linked list](#sección-17--el-benchmark-que-cambia-todo-array-vs-linked-list)

---

## Sección 1.1 — Stack vs Heap: Dónde Viven tus Datos

### Ejercicio 1.1.1 — Leer: los dos mundos de la memoria

**Tipo: Leer**

```
Todo programa tiene dos regiones de memoria:

  STACK                              HEAP
  ─────                              ────
  Rápido (ya en cache)               Lento (puede no estar en cache)
  Tamaño fijo (1-8 MB por thread)    Tamaño variable (toda la RAM)
  Asignación: mover un puntero       Asignación: buscar espacio libre
  Liberación: automática (al salir   Liberación: manual (C/C++) o
    de la función)                     garbage collector (Java/Go/Python)
  Datos: variables locales,           Datos: objetos, strings,
    parámetros, return address          arrays grandes, cualquier cosa
                                        con tamaño dinámico

El stack es como una pila de platos:
  - Cada función pone sus platos (variables locales) arriba.
  - Cuando la función termina, saca sus platos.
  - Nunca puedes sacar un plato del medio.
  - Es rápido porque el CPU siempre sabe dónde está el tope.

El heap es como un almacén:
  - Puedes pedir espacio de cualquier tamaño, en cualquier momento.
  - Puedes liberar espacio en cualquier orden.
  - Pero alguien tiene que llevar la cuenta de qué espacio está libre.
  - Esa "cuenta" es el allocator, y es lento comparado con el stack.

  Ejemplo en Java:

    void procesarVenta(long montoEnCentavos) {
        // montoEnCentavos vive en el STACK (primitivo, tamaño fijo)
        // Costo de "asignación": ~0 ns (mover stack pointer)

        String descripcion = "Venta #" + montoEnCentavos;
        // descripcion vive en el HEAP (String es un objeto, tamaño variable)
        // Costo de asignación: ~10-50 ns (buscar espacio, inicializar headers)

        double[] precios = new double[1000];
        // precios vive en el HEAP (array, tamaño dinámico)
        // Costo de asignación: ~100 ns (8000 bytes + array header)
    }
    // Al salir de procesarVenta:
    // - montoEnCentavos se libera del stack instantáneamente
    // - descripcion y precios siguen en el heap
    //   hasta que el GC los encuentre y los libere

  Contemos: en una sola función, el stack costó 0 ns
  y el heap costó ~150 ns en asignaciones.
  Si esta función se llama 1 millón de veces por segundo,
  esas asignaciones de heap suman 150 milisegundos de overhead.
```

**Preguntas:**

1. ¿Por qué el stack tiene tamaño fijo y el heap no?

2. ¿Un StackOverflowError en Java se produce por recursión infinita,
   por variables locales demasiado grandes, o por ambas?

3. ¿Python tiene la misma distinción stack/heap que Java?
   ¿Dónde vive un `int` en Python?

4. ¿Rust tiene garbage collector? Si no, ¿cómo se libera la memoria del heap?

5. ¿Cuántas llamadas recursivas puedes hacer en Java
   antes de un StackOverflowError con el stack default de 1 MB?

---

### Ejercicio 1.1.2 — Leer: por qué el heap necesita un allocator

**Tipo: Leer**

```
Asignar memoria en el stack es trivial:
  stack_pointer -= tamaño;
  return stack_pointer;
  // Eso es todo. Un decremento. Una instrucción.

Asignar memoria en el heap es un PROBLEMA:
  1. Buscar un bloque libre de tamaño suficiente
  2. Decidir cuál bloque usar si hay varios (first-fit, best-fit, worst-fit)
  3. Actualizar la lista de bloques libres
  4. Manejar la fragmentación (bloques pequeños inutilizables)

Estrategias de los allocators reales:

  malloc (glibc):
    Usa "bins" de tamaños fijos para asignaciones pequeñas
    y mmap() para asignaciones grandes (>128 KB).
    Rápido para el caso común, pero puede fragmentar.

  jemalloc (FreeBSD, Firefox, Redis):
    Usa "arenas" por thread para evitar lock contention.
    Mejor para programas multi-threaded que malloc.
    Redis lo usa por esta razón.

  tcmalloc (Google):
    Thread-caching malloc. Cada thread tiene su propia cache
    de bloques pequeños. Asignaciones pequeñas no necesitan lock.

  mimalloc (Microsoft):
    Optimizado para cache locality: agrupa objetos del mismo
    tamaño en las mismas páginas.

  El allocator de Go:
    Basado en tcmalloc. Usa "mcache" por goroutine (P).
    Asignaciones pequeñas (<32 KB) van al mcache sin lock.
    Asignaciones grandes van al heap global con lock.

  Rust (por defecto usa el system allocator):
    No tiene GC. Usa RAII: la memoria se libera automáticamente
    cuando el owner sale del scope. El allocator solo necesita
    malloc/free, no necesita "encontrar basura".
```

**Preguntas:**

1. ¿Por qué Redis usa jemalloc en vez del malloc estándar?

2. ¿Un allocator por thread (como tcmalloc) elimina la contención?
   ¿O solo la reduce?

3. ¿La fragmentación del heap es un problema real en producción?
   ¿En qué tipo de programas?

4. ¿El GC de Java reemplaza al allocator o lo complementa?

5. ¿Un programa que solo usa el stack (sin heap)
   es posible en la práctica?

---

### Ejercicio 1.1.3 — Comparar: dónde viven los datos en 5 lenguajes

**Tipo: Analizar**

```python
# PYTHON
# En CPython, TODOS los valores viven en el heap.
# Incluso un int es un objeto en el heap con un header de 28 bytes.
# El "stack" de Python solo tiene referencias (punteros) a objetos del heap.

x = 42
# x no es 42 en el stack.
# x es un puntero en el stack que apunta a un objeto PyLongObject
# en el heap que contiene 42, un refcount, y un type pointer.
# Costo total del int 42: ~28 bytes en el heap.

# Un int en C ocupa 4 bytes en el stack.
# Un int en Python ocupa 28 bytes en el heap + 8 bytes de referencia.
# Ratio: 9× más memoria para el mismo dato.
```

```java
// JAVA
// Primitivos (int, long, double, boolean) viven en el stack.
// Objetos (String, Integer, arrays, cualquier clase) viven en el heap.

int x = 42;              // stack: 4 bytes, sin overhead
Integer y = 42;           // heap: 16 bytes (12 header + 4 valor)
                          // ratio: 4× más memoria por el boxing

int[] arr = new int[100]; // heap: 16 (header) + 400 (datos) = 416 bytes
                          // los datos del array están contiguos en el heap
                          // → buena cache locality

Integer[] boxed = new Integer[100];
                          // heap: 16 (array header) + 800 (100 punteros)
                          //       + 100 × 16 (100 objetos Integer)
                          //     = 2416 bytes (5.8× más que int[])
                          // los Integer están dispersos en el heap
                          // → mala cache locality
```

```scala
// SCALA
// Scala compila a JVM bytecode → mismas reglas que Java para memory layout.
// Pero el modelo de programación favorece la inmutabilidad:

val x: Int = 42           // primitivo JVM, stack, 4 bytes (como Java int)
val y: Option[Int] = Some(42)
                          // Some es un objeto en el heap
                          // contiene un int boxeado (Integer)
                          // costo: ~32 bytes (Some header + Integer header + valor)

// Las colecciones inmutables de Scala (List, Vector, Map)
// viven completamente en el heap y usan structural sharing
// para evitar copiar todo en cada "modificación".
// El costo de memoria es mayor que las colecciones mutables de Java,
// pero la seguridad en concurrencia lo compensa.
```

```go
// GO
// Go tiene una distinción interesante: el compilador DECIDE
// dónde vive cada variable mediante "escape analysis".

func procesar() int {
    x := 42       // el compilador analiza: ¿x escapa de esta función?
    return x       // no escapa → x vive en el STACK
}

func crear() *Venta {
    v := Venta{Monto: 100}
    return &v      // v escapa (devolvemos un puntero a v)
                   // → v vive en el HEAP
                   // el GC de Go se encarga de liberarla después
}

// Diferencia clave con Java:
// En Java, TODO objeto va al heap. Siempre.
// En Go, un struct puede vivir en el stack si no escapa.
// Esto reduce la presión sobre el GC significativamente.

// Para ver las decisiones del compilador:
// go build -gcflags='-m' tu_programa.go
// Output: "v escapes to heap" o "v does not escape"
```

```rust
// RUST
// Rust da el control total al programador. Sin GC.
// Por defecto, TODO vive en el stack.
// Solo va al heap si tú lo pides explícitamente.

fn procesar() {
    let x: i64 = 42;           // stack, 8 bytes, sin overhead
    let arr = [0i32; 1000];     // stack, 4000 bytes, sin overhead
                                // (cuidado: arrays muy grandes
                                //  pueden desbordar el stack)

    let vec = Vec::new();       // Vec internamente usa el heap para los datos
                                // pero el struct Vec (ptr, len, cap) está en el stack
                                // los datos del Vec están contiguos en el heap
                                // → buena cache locality (como int[] de Java)

    let boxed = Box::new(42);   // Box: asigna explícitamente en el heap
                                // 8 bytes en heap + 8 bytes de puntero en stack
}
// Al salir de procesar():
// - x y arr se liberan del stack automáticamente (RAII)
// - vec se destruye: el Drop trait libera la memoria del heap
// - boxed se destruye: Box libera su memoria del heap
// No hay GC. No hay "encontrar basura". El compilador sabe
// exactamente cuándo cada valor deja de ser usado.
```

**Preguntas:**

1. ¿Un `int` de Python ocupa 28 bytes y un `int` de Java ocupa 4 bytes.
   ¿Qué impacto tiene esto para un array de 10 millones de enteros?

2. ¿El escape analysis de Go es un optimization pass del compilador
   o una feature del runtime?

3. ¿Rust puede tener memory leaks si no tiene GC?

4. ¿`Integer[]` de Java y `list[int]` de Python tienen el mismo problema
   de cache locality? ¿Por la misma razón?

5. ¿Scala paga el mismo "impuesto de objetos" que Java
   dado que ambos corren en la JVM?

---

### Ejercicio 1.1.4 — Medir: cuánto cuesta una asignación en el heap

**Tipo: Implementar**

```java
// Java: medir el costo de asignaciones de heap
// Ejecutar con JMH (Java Microbenchmark Harness) para resultados confiables.
// Sin JMH: ejecutar manualmente, ignorar el primer millón de iteraciones (warmup).

public class AllocBenchmark {

    // Caso 1: solo stack (primitivos)
    static long soloStack() {
        long a = 1;
        long b = 2;
        long c = a + b;
        return c;
    }

    // Caso 2: heap (un objeto)
    static Object unObjeto() {
        return new Object();  // 16 bytes en el heap
    }

    // Caso 3: heap (un array pequeño)
    static int[] arrayPequeno() {
        return new int[10];   // 56 bytes en el heap (16 header + 40 datos)
    }

    // Caso 4: heap (un array grande)
    static int[] arrayGrande() {
        return new int[10_000]; // ~40 KB en el heap
    }

    // Caso 5: heap con boxing
    static Integer boxing() {
        return Integer.valueOf(42);
        // ¿Es heap o cache? Integer caches -128 a 127.
    }

    public static void main(String[] args) {
        int iterations = 100_000_000;

        // Warmup
        for (int i = 0; i < 1_000_000; i++) {
            soloStack(); unObjeto(); arrayPequeno();
        }

        long start, end;

        start = System.nanoTime();
        for (int i = 0; i < iterations; i++) soloStack();
        end = System.nanoTime();
        System.out.printf("soloStack:     %.1f ns/op%n",
            (double)(end - start) / iterations);

        start = System.nanoTime();
        for (int i = 0; i < iterations; i++) unObjeto();
        end = System.nanoTime();
        System.out.printf("unObjeto:      %.1f ns/op%n",
            (double)(end - start) / iterations);

        start = System.nanoTime();
        for (int i = 0; i < iterations; i++) arrayPequeno();
        end = System.nanoTime();
        System.out.printf("arrayPequeno:  %.1f ns/op%n",
            (double)(end - start) / iterations);

        start = System.nanoTime();
        for (int i = 0; i < iterations; i++) arrayGrande();
        end = System.nanoTime();
        System.out.printf("arrayGrande:   %.1f ns/op%n",
            (double)(end - start) / iterations);

        // NOTA: este benchmark es simplificado.
        // Un benchmark riguroso requiere JMH para manejar:
        // - Dead code elimination (JIT puede eliminar código sin efecto)
        // - GC noise (el GC puede ejecutarse durante la medición)
        // - Warmup adecuado (JIT tarda en optimizar)
    }
}
```

```go
// Go: medir el costo de asignación + escape analysis
package main

import (
    "fmt"
    "runtime"
    "time"
)

type Venta struct {
    OrderID   int64
    Monto     float64
    Region    [32]byte  // string fijo para evitar puntero
}

// noEscape: el compilador puede poner Venta en el stack
func noEscape() Venta {
    return Venta{OrderID: 1, Monto: 100.0}
}

// escape: el compilador DEBE poner Venta en el heap
func escape() *Venta {
    v := Venta{OrderID: 1, Monto: 100.0}
    return &v
}

func benchmark(name string, iterations int, f func()) {
    runtime.GC() // limpiar antes de medir
    start := time.Now()
    for i := 0; i < iterations; i++ {
        f()
    }
    elapsed := time.Since(start)
    fmt.Printf("%s: %.1f ns/op\n", name, float64(elapsed.Nanoseconds())/float64(iterations))
}

func main() {
    n := 100_000_000
    benchmark("noEscape (stack)", n, func() { _ = noEscape() })
    benchmark("escape (heap)",    n, func() { _ = escape() })
}

// Compilar con: go build -gcflags='-m' para ver decisiones de escape analysis
// Esperar: noEscape ~1-2 ns, escape ~10-30 ns
```

**Preguntas:**

1. ¿Cuántas veces más lento es `arrayGrande` vs `soloStack`?

2. ¿`Integer.valueOf(42)` es una asignación de heap o no?
   ¿Qué pasa con `Integer.valueOf(200)`?

3. ¿Go `noEscape` es comparable en velocidad a Java `soloStack`?

4. ¿El GC ejecutándose durante el benchmark afecta los resultados?
   ¿Cómo lo mitigas?

5. ¿Cómo escribirías este benchmark en Rust?
   ¿Necesitarías medir asignación de heap vs stack de la misma forma?

---

### Ejercicio 1.1.5 — Analizar: el garbage collector como costo oculto

**Tipo: Analizar**

```
El GC no es gratis. Es un tradeoff:

  SIN GC (Rust, C, C++):
    + No hay pausas impredecibles
    + Memoria se libera inmediatamente cuando el owner termina
    - El programador debe pensar en lifetimes y ownership
    - Use-after-free y double-free son posibles (en C/C++, no en Rust)

  CON GC (Java, Go, Python, Scala):
    + El programador no piensa en cuándo liberar memoria
    + No hay use-after-free ni double-free
    - El GC consume CPU (típicamente 5-15% del tiempo total)
    - El GC causa pausas (latency spikes)
    - El GC necesita memoria extra (para poder trabajar)

Impacto real del GC en estructuras de datos:

  Java HashMap con 10 millones de entries:
  - 10M objetos Entry en el heap
  - Cada Entry tiene: key (objeto), value (objeto), hash (int), next (puntero)
  - El GC debe RECORRER esos 10M objetos para determinar cuáles están vivos
  - GC pause: 50-500 ms dependiendo del collector y heap size
  - Durante esa pausa, tu programa está DETENIDO

  Rust HashMap con 10 millones de entries:
  - Los datos están en arrays contiguos internamente
  - No hay GC. Cuando el HashMap sale del scope, se libera instantáneamente.
  - No hay pausas. Nunca.

  Go map con 10 millones de entries:
  - El GC de Go es concurrent (corre en paralelo con tu programa)
  - Pausas típicas: < 1 ms (Go prioriza baja latencia sobre throughput)
  - Pero el GC consume ~25% más CPU que un programa sin GC

GC collectors en Java:
  G1 GC (default desde Java 9):
    Divide el heap en regiones. GC incremental.
    Pausa target: 200 ms (configurable).
    Bueno para heaps de 2-32 GB.

  ZGC (Java 15+):
    Pausas < 1 ms independientemente del heap size.
    Bueno para heaps de 8 GB - 16 TB.
    Costo: ~15% más throughput que G1.

  El tradeoff del GC no es "GC bueno" o "GC malo".
  Es "latencia predecible" vs "facilidad de desarrollo".
  La respuesta depende de tu aplicación.
```

**Preguntas:**

1. ¿Un hash map de 10M entries en Java con ZGC
   tiene pausas comparables a Rust (sin GC)?

2. ¿El GC de Go es mejor que el de Java para latencia?
   ¿Mejor para throughput?

3. ¿Scala puede usar ZGC dado que corre en la JVM?

4. ¿Un pool de objetos (reutilizar en vez de crear y destruir)
   reduce la presión del GC? ¿Es una buena práctica?

5. ¿Python con reference counting + cycle collector
   tiene pausas de GC comparables a Java?

---

## Sección 1.2 — Localidad de Referencia: el Secreto del Rendimiento

### Ejercicio 1.2.1 — Leer: spatial y temporal locality

**Tipo: Leer**

```
El hardware predice qué datos vas a necesitar.
Si predice bien, los datos ya están en cache cuando los pides.
Si predice mal, tu programa espera 100 ns por cada acceso.

La predicción se basa en dos patrones:

  SPATIAL LOCALITY (localidad espacial):
  "Si accediste al byte N, probablemente accedas al byte N+1 pronto."

    Cuando lees una posición de memoria, el hardware no trae solo ese byte.
    Trae una CACHE LINE completa (64 bytes en x86).
    Si los datos que necesitas están juntos (array), los 64 bytes
    contienen tus próximos ~8 longs o ~16 ints.
    → Un solo acceso a RAM carga datos para las próximas 8-16 iteraciones.

    Si los datos están dispersos (linked list), los 64 bytes
    contienen un nodo y basura irrelevante.
    → Cada iteración necesita un nuevo acceso a RAM.

  TEMPORAL LOCALITY (localidad temporal):
  "Si accediste al dato X, probablemente lo accedas otra vez pronto."

    Los datos recientemente accedidos se quedan en cache.
    Un bucle que opera sobre el mismo array repetidamente
    encuentra los datos en cache en la segunda iteración.
    Una estructura que accede a datos aleatorios (hash map con chaining)
    tiene poca temporal locality porque cada lookup va a una dirección distinta.

  Ejemplo numérico:

    Array de 1M ints (4 MB):
      Primer acceso: L1 miss → L2 miss → L3 miss → RAM (100 ns)
      Pero la cache line de 64 bytes trae 16 ints de golpe.
      Los siguientes 15 accesos: L1 hit (1 ns cada uno).
      Costo amortizado por elemento: 100/16 + 15×1/16 ≈ 7 ns

    Linked list de 1M nodos (cada nodo en una dirección aleatoria):
      Cada acceso: seguir un puntero a una dirección impredecible.
      L1 miss → L2 miss → posible L3 miss → RAM (100 ns)
      Costo por elemento: ~50-100 ns

    Ratio: la linked list es 7-14× más lenta para un recorrido secuencial.
    Big-O dice que ambas son O(n). La realidad dice lo contrario.
```

**Preguntas:**

1. ¿Un hash map con open addressing (datos en un array)
   tiene mejor spatial locality que uno con chaining (linked lists)?

2. ¿Un B-Tree con nodos grandes (1000 claves por nodo)
   tiene mejor spatial locality que un Red-Black tree (1 clave por nodo)?

3. ¿La temporal locality explica por qué acceder al mismo
   hash map repetidamente es más rápido que la primera vez?

4. ¿Qué tipo de locality tiene un recorrido en BFS de un grafo?

5. ¿El prefetcher del hardware puede ayudar
   con los accesos de una linked list?

---

### Ejercicio 1.2.2 — Implementar: visualizar el impacto de la localidad

**Tipo: Implementar**

```python
# Python: medir el impacto de acceso secuencial vs aleatorio
import array
import random
import time

def sequential_access(data, iterations):
    """Recorrer el array secuencialmente."""
    total = 0
    for _ in range(iterations):
        for i in range(len(data)):
            total += data[i]
    return total

def random_access(data, indices, iterations):
    """Acceder al array en orden aleatorio."""
    total = 0
    for _ in range(iterations):
        for i in indices:
            total += data[i]
    return total

def benchmark():
    n = 1_000_000
    iterations = 5

    # Usar array.array para datos contiguos (no list que es array de punteros)
    data = array.array('l', range(n))
    indices = list(range(n))
    random.shuffle(indices)

    # Warmup
    sequential_access(data, 1)
    random_access(data, indices, 1)

    # Medir secuencial
    start = time.perf_counter_ns()
    sequential_access(data, iterations)
    seq_time = (time.perf_counter_ns() - start) / (n * iterations)

    # Medir aleatorio
    start = time.perf_counter_ns()
    random_access(data, indices, iterations)
    rand_time = (time.perf_counter_ns() - start) / (n * iterations)

    print(f"Secuencial:  {seq_time:.1f} ns/elemento")
    print(f"Aleatorio:   {rand_time:.1f} ns/elemento")
    print(f"Ratio:       {rand_time/seq_time:.1f}×")

benchmark()
# Resultado esperado (varía según hardware):
# Secuencial:  ~15-30 ns/elemento (limitado por Python, no por cache)
# Aleatorio:   ~40-100 ns/elemento
# Ratio:       ~2-4× (en Python el overhead del intérprete domina;
#              en C/Java/Rust el ratio sería 5-15×)
```

```java
// Java: el mismo benchmark, donde el efecto es más visible
public class LocalityBenchmark {

    public static long sequential(int[] data) {
        long total = 0;
        for (int i = 0; i < data.length; i++) {
            total += data[i];
        }
        return total;
    }

    public static long randomAccess(int[] data, int[] indices) {
        long total = 0;
        for (int i = 0; i < indices.length; i++) {
            total += data[indices[i]];
        }
        return total;
    }

    public static void main(String[] args) {
        int n = 10_000_000;  // 40 MB — no cabe en L3
        int[] data = new int[n];
        int[] indices = new int[n];

        for (int i = 0; i < n; i++) {
            data[i] = i;
            indices[i] = i;
        }
        // Fisher-Yates shuffle
        var rng = new java.util.Random(42);
        for (int i = n - 1; i > 0; i--) {
            int j = rng.nextInt(i + 1);
            int tmp = indices[i]; indices[i] = indices[j]; indices[j] = tmp;
        }

        // Warmup
        for (int w = 0; w < 10; w++) {
            sequential(data);
            randomAccess(data, indices);
        }

        int runs = 20;
        long start, elapsed;

        start = System.nanoTime();
        for (int r = 0; r < runs; r++) sequential(data);
        elapsed = System.nanoTime() - start;
        System.out.printf("Sequential: %.1f ns/elem%n",
            (double) elapsed / (n * (long) runs));

        start = System.nanoTime();
        for (int r = 0; r < runs; r++) randomAccess(data, indices);
        elapsed = System.nanoTime() - start;
        System.out.printf("Random:     %.1f ns/elem%n",
            (double) elapsed / (n * (long) runs));
    }
}
// Resultado esperado:
// Sequential: ~0.5-1.0 ns/elem (datos en cache, auto-vectorización)
// Random:     ~5-20 ns/elem (cache misses constantes)
// Ratio:      10-20×
```

**Preguntas:**

1. ¿Por qué el benchmark de Java muestra un ratio de 10-20×
   pero el de Python solo muestra 2-4×?

2. ¿Si el array cabe en L3 cache (~16 MB), el ratio disminuye?

3. ¿El JIT de Java puede auto-vectorizar el acceso secuencial?
   ¿Qué efecto tiene eso?

4. ¿Este benchmark tiene sentido ejecutarlo en Go y Rust?
   ¿Esperarías resultados más parecidos a Java o a Python?

5. ¿En un array de 100 elementos (400 bytes), ¿hay diferencia
   entre acceso secuencial y aleatorio? ¿Por qué?

---

### Ejercicio 1.2.3 — Analizar: por qué importa para las estructuras de datos

**Tipo: Analizar**

```
Mapa de localidad de las estructuras de datos más comunes:

  Estructura           Spatial    Temporal    Por qué
  ─────────────────    ────────   ────────    ──────────────────────────
  Array                ★★★★★     ★★★★★       Datos contiguos en memoria
  Dynamic array        ★★★★★     ★★★★★       Igual que array (más resize)
  Hash map (open addr) ★★★★☆     ★★★☆☆       Array + probes cercanos
  Hash map (chaining)  ★★☆☆☆     ★★☆☆☆       Cada chain es una linked list
  Binary heap          ★★★★★     ★★★★☆       Almacenado como array
  B-Tree               ★★★★☆     ★★★☆☆       Nodos grandes (caben en cache)
  Red-Black tree       ★★☆☆☆     ★★☆☆☆       Nodos dispersos en heap
  Linked list          ★☆☆☆☆     ★☆☆☆☆       Cada nodo en dirección aleatoria
  Skip list            ★★☆☆☆     ★★☆☆☆       Nodos dispersos + multinivel

  Esta tabla explica decisiones de la industria:

  1. Java ArrayList es el default, no LinkedList.
     A pesar de que LinkedList tiene O(1) insert al inicio,
     ArrayList es más rápido para casi todo por cache locality.

  2. Cassandra usa un B-Tree/SSTable (buena locality)
     no un Red-Black tree (mala locality) para datos en disco.

  3. Redis sorted sets usan skip list, no Red-Black tree.
     Pero Redis está in-memory (RAM es rápida),
     así que la diferencia de locality importa menos
     que la simplicidad de implementación de la skip list.

  4. Rust HashMap usa Robin Hood hashing (open addressing)
     no separate chaining — por cache locality.

  5. Google's SwissTable (abseil, también en Rust hashbrown)
     almacena metadatos de control en un array de bytes
     que cabe en una cache line (16 bytes = 16 slots de metadata).
     Un lookup verifica 16 slots con una sola lectura de cache.
```

**Preguntas:**

1. ¿El heap de un binary heap (la estructura) y el heap de la memoria
   son el mismo concepto? ¿Por qué comparten nombre?

2. ¿Un hash map con open addressing que tiene load factor 0.9
   pierde su ventaja de locality?

3. ¿Una linked list donde los nodos se asignan secuencialmente
   (uno tras otro en el heap) tiene buena spatial locality?

4. ¿El B-Tree tiene mejor locality que el Red-Black tree
   por diseño o por coincidencia?

5. ¿SwissTable sacrifica algo para conseguir esa locality?

---

### Ejercicio 1.2.4 — Implementar: linked list vs array — el benchmark definitivo

**Tipo: Implementar**

```rust
// Rust: la comparación más limpia entre array y linked list
// porque Rust no tiene GC que interfiera con la medición.

use std::collections::LinkedList;
use std::time::Instant;

fn sum_vec(data: &Vec<i64>) -> i64 {
    let mut total: i64 = 0;
    for val in data.iter() {
        total = total.wrapping_add(*val);
    }
    total
}

fn sum_linked_list(data: &LinkedList<i64>) -> i64 {
    let mut total: i64 = 0;
    for val in data.iter() {
        total = total.wrapping_add(*val);
    }
    total
}

fn main() {
    let n: usize = 10_000_000;
    let runs = 20;

    // Crear Vec (datos contiguos)
    let vec_data: Vec<i64> = (0..n as i64).collect();

    // Crear LinkedList (datos dispersos)
    let list_data: LinkedList<i64> = (0..n as i64).collect();

    // Warmup
    for _ in 0..3 {
        let _ = sum_vec(&vec_data);
        let _ = sum_linked_list(&list_data);
    }

    // Benchmark Vec
    let start = Instant::now();
    for _ in 0..runs {
        let _ = sum_vec(&vec_data);
    }
    let vec_elapsed = start.elapsed();
    let vec_ns = vec_elapsed.as_nanos() as f64 / (n as f64 * runs as f64);

    // Benchmark LinkedList
    let start = Instant::now();
    for _ in 0..runs {
        let _ = sum_linked_list(&list_data);
    }
    let list_elapsed = start.elapsed();
    let list_ns = list_elapsed.as_nanos() as f64 / (n as f64 * runs as f64);

    println!("Vec<i64>:        {:.2} ns/elem", vec_ns);
    println!("LinkedList<i64>: {:.2} ns/elem", list_ns);
    println!("Ratio:           {:.1}×", list_ns / vec_ns);

    // Resultado esperado:
    // Vec<i64>:        ~0.3-0.5 ns/elem (SIMD auto-vectorización)
    // LinkedList<i64>: ~5-15 ns/elem (cache misses en cada nodo)
    // Ratio:           15-40×
}
// Compilar con: cargo run --release
// (--release activa optimizaciones, incluyendo auto-vectorización)
```

```go
// Go: misma comparación, con container/list como linked list
package main

import (
    "container/list"
    "fmt"
    "runtime"
    "time"
)

func sumSlice(data []int64) int64 {
    var total int64
    for _, v := range data {
        total += v
    }
    return total
}

func sumLinkedList(l *list.List) int64 {
    var total int64
    for e := l.Front(); e != nil; e = e.Next() {
        total += e.Value.(int64) // type assertion — costo extra
    }
    return total
}

func main() {
    n := 10_000_000
    runs := 20

    // Slice (datos contiguos)
    sliceData := make([]int64, n)
    for i := range sliceData {
        sliceData[i] = int64(i)
    }

    // LinkedList (datos dispersos)
    listData := list.New()
    for i := 0; i < n; i++ {
        listData.PushBack(int64(i))
    }

    runtime.GC()

    // Benchmark slice
    start := time.Now()
    for r := 0; r < runs; r++ {
        _ = sumSlice(sliceData)
    }
    sliceTime := time.Since(start)
    sliceNs := float64(sliceTime.Nanoseconds()) / float64(n*runs)

    // Benchmark LinkedList
    start = time.Now()
    for r := 0; r < runs; r++ {
        _ = sumLinkedList(listData)
    }
    listTime := time.Since(start)
    listNs := float64(listTime.Nanoseconds()) / float64(n*runs)

    fmt.Printf("[]int64:     %.2f ns/elem\n", sliceNs)
    fmt.Printf("list.List:   %.2f ns/elem\n", listNs)
    fmt.Printf("Ratio:       %.1f×\n", listNs/sliceNs)
}
```

**Preguntas:**

1. ¿Por qué el ratio en Rust (~15-40×) es mayor que en Go (~5-15×)?
   Pista: auto-vectorización de SIMD.

2. ¿`LinkedList` de Rust está deprecada de facto?
   ¿La documentación de Rust recomienda usarla?

3. ¿En Go, la type assertion `e.Value.(int64)` añade overhead
   al benchmark de la linked list?

4. ¿Si ambas estructuras caben en L3 cache (ej: 1000 elementos),
   ¿el ratio se reduce?

5. ¿Hay algún caso de uso real donde linked list
   sea mejor que un array? ¿Cuál?

---

### Ejercicio 1.2.5 — Analizar: el mantra "usar siempre arrays" tiene excepciones

**Tipo: Analizar**

```
Después de ver los benchmarks, la conclusión obvia es:
"siempre usa arrays, nunca uses linked lists."

Pero la realidad tiene matices:

  Caso 1: insertar en el medio de una colección ordenada
    Array: O(n) — mover todos los elementos posteriores
    Linked list: O(1) — solo cambiar dos punteros
    PERO: para encontrar la posición de inserción,
    la linked list necesita O(n) recorrido secuencial.
    → El array con memmove/System.arraycopy es frecuentemente
      más rápido que la linked list por locality,
      incluso con la copia.

  Caso 2: cola con insert/remove en ambos extremos (deque)
    Array (ring buffer): O(1) amortizado, excelente locality
    Linked list: O(1) pero mala locality
    → Ring buffer gana. Java ArrayDeque > LinkedList.

  Caso 3: cola de prioridad con merge de dos colas
    Array (heap): merge es O(n) — hay que re-heapify
    Linked list (mergeable heap): merge es O(log n)
    → La linked list gana si merges son frecuentes.
      Fibonacci heap usa linked lists internamente por esto.

  Caso 4: OS kernel — lista de procesos, lista de timers
    El kernel de Linux usa doubly linked lists extensivamente.
    ¿Por qué? Porque los nodos están embedded en el struct
    (no asignados por separado), y el kernel necesita
    O(1) remove dado un puntero al nodo.
    → No hay búsqueda, no hay recorrido, solo unlink.

  Caso 5: undo history, immutable lists (funcional)
    En Scala/Haskell, las listas inmutables comparten estructura:
      val a = List(1, 2, 3)
      val b = 0 :: a     // b = List(0, 1, 2, 3)
    b reutiliza toda la memoria de a (structural sharing).
    Un array necesitaría copiar todo.
    → En el paradigma funcional, las linked lists son naturales.

  Regla revisada:
    Usa arrays por defecto.
    Usa linked lists cuando:
    (a) necesitas O(1) insert/remove dado un puntero al nodo, O
    (b) necesitas structural sharing (programación funcional), O
    (c) merge de colecciones es una operación frecuente.
    En todos los demás casos, el array gana.
```

**Preguntas:**

1. ¿Java `ArrayDeque` es siempre mejor que `LinkedList` como deque?

2. ¿La "intrusive linked list" del kernel de Linux
   tiene mejor locality que una linked list estándar?

3. ¿`List` de Scala (linked list inmutable) paga un costo de locality
   por el structural sharing?

4. ¿`VecDeque` de Rust (ring buffer) tiene el mismo rendimiento
   que un `Vec` para acceso secuencial?

5. ¿Si alguien te dice "necesito una linked list", ¿cuál es
   la primera pregunta que le harías?

---

## Sección 1.3 — Cache Lines: la Unidad Real de Transferencia

### Ejercicio 1.3.1 — Leer: qué es una cache line y por qué mide 64 bytes

**Tipo: Leer**

```
El CPU nunca lee un solo byte de la RAM.
Lee una CACHE LINE completa: 64 bytes en x86 (Intel/AMD).

  ¿Por qué 64 bytes?
  - Demasiado pequeño (8 bytes): necesitas muchas transferencias → lento.
  - Demasiado grande (4 KB): traes datos que no necesitas → desperdicio.
  - 64 bytes es el tradeoff óptimo para las cargas de trabajo típicas.
  - ARM puede usar 128 bytes (Apple M1/M2 usa 128).

  Implicaciones prácticas:

  1. Un int de 4 bytes lee 64 bytes → 60 bytes "gratis"
     Si los datos están contiguos, esos 60 bytes son útiles.
     Si no, son desperdicio.

  2. Dos variables en la misma cache line se leen juntas.
     Esto es bueno si ambas se usan juntas.
     Esto es MALO si dos threads escriben en variables distintas
     que comparten cache line → "false sharing".

  3. Un nodo de linked list de 24 bytes desperdicia 40 bytes
     de la cache line (si el siguiente nodo está en otra línea).

Visualización de una cache line (64 bytes):

  Cache line 0:  |int[0]|int[1]|int[2]|int[3]|int[4]|...|int[15]|
                  4 bytes × 16 = 64 bytes
                  → Un acceso a int[0] carga int[0] a int[15]

  Cache line 1:  |int[16]|int[17]|...|int[31]|
                  → Un acceso a int[16] carga int[16] a int[31]

  Para un int[], un cache miss cada 16 elementos.
  Para un long[], un cache miss cada 8 elementos.
  Para un Object[] (8 bytes por referencia + 64+ bytes por objeto):
    un cache miss por cada referencia + un cache miss por cada objeto.
    → El DOBLE de cache misses.
```

**Preguntas:**

1. ¿Cuántos cache misses ocurren al recorrer un `int[]` de 10,000 elementos?

2. ¿Y para un `long[]` de 10,000 elementos?

3. ¿Un `Object[]` de 10,000 elementos causa cuántos cache misses?

4. ¿El Apple M1 con cache lines de 128 bytes
   beneficia más a los arrays que un Intel con 64 bytes?

5. ¿Un struct de 65 bytes siempre ocupa 2 cache lines?

---

### Ejercicio 1.3.2 — Implementar: el stride benchmark

**Tipo: Implementar**

```java
// Recorrer un array con diferentes "strides" (saltos entre accesos)
// revela directamente el efecto de las cache lines.

public class StrideBenchmark {

    public static long strideAccess(int[] data, int stride) {
        long total = 0;
        for (int i = 0; i < data.length; i += stride) {
            total += data[i];
        }
        return total;
    }

    public static void main(String[] args) {
        int n = 64 * 1024 * 1024; // 256 MB — no cabe en L3
        int[] data = new int[n];
        for (int i = 0; i < n; i++) data[i] = i;

        int[] strides = {1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024};

        // Warmup
        for (int s : strides) strideAccess(data, s);

        System.out.println("Stride  Accesos      ns/acceso");
        System.out.println("──────  ────────     ─────────");

        for (int stride : strides) {
            int accesses = n / stride;
            long start = System.nanoTime();
            for (int r = 0; r < 5; r++) {
                strideAccess(data, stride);
            }
            long elapsed = System.nanoTime() - start;
            double nsPerAccess = (double) elapsed / (accesses * 5L);
            System.out.printf("%5d   %,10d    %.1f%n", stride, accesses, nsPerAccess);
        }
    }
}
// Resultado esperado:
//
// Stride  Accesos      ns/acceso
// ──────  ────────     ─────────
//     1   67,108,864    0.5      ← 16 accesos por cache line
//     2   33,554,432    0.6      ← 8 accesos por cache line
//     4   16,777,216    0.8      ← 4 accesos por cache line
//     8    8,388,608    1.2      ← 2 accesos por cache line
//    16    4,194,304    3.5      ← 1 acceso por cache line (64/4=16 ints)
//    32    2,097,152    3.5      ← 1 acceso por cache line (salta una línea)
//    64    1,048,576    3.5      ← 1 acceso por cache line
//   128      524,288    5.0      ← prefetcher empieza a fallar
//   256      262,144    8.0      ← TLB misses empiezan a aparecer
//   512      131,072   12.0      ← acceso casi aleatorio
//  1024       65,536   15.0      ← acceso completamente aleatorio
//
// Observar: el ns/acceso sube abruptamente en stride=16
// porque ahí es donde cada acceso es un cache miss nuevo.
// Stride 1-8: múltiples accesos aprovechan la misma cache line.
// Stride 16+: cada acceso es un cache miss.
// Stride 128+: el prefetcher no puede predecir el patrón.
```

**Preguntas:**

1. ¿Por qué el salto abrupto ocurre en stride=16 y no en stride=64?

2. ¿Si las cache lines fueran de 128 bytes (Apple M1),
   ¿en qué stride ocurriría el salto?

3. ¿El hardware prefetcher ayuda con stride=2 pero no con stride=256?

4. ¿Qué son los TLB misses que aparecen en strides grandes?

5. ¿Este benchmark es válido en Python?
   ¿O el overhead del intérprete domina?

---

### Ejercicio 1.3.3 — Leer: false sharing — cuando la cache line es el enemigo

**Tipo: Leer**

```
False sharing ocurre cuando dos threads escriben
en variables DISTINTAS que están en la MISMA cache line.

  Thread A escribe variable X (en cache line N).
  Thread B escribe variable Y (también en cache line N).

  El CPU de Thread A invalida la cache line N del CPU de Thread B.
  El CPU de Thread B invalida la cache line N del CPU de Thread A.
  Ambos CPUs pasan más tiempo invalidando que trabajando.

  Esto se llama "ping-pong de cache line" y puede
  hacer que un programa multi-threaded sea MÁS LENTO
  que uno single-threaded.

Ejemplo:

  // MAL: contadores en posiciones adyacentes de un array
  long[] counters = new long[NUM_THREADS];
  // counters[0] y counters[1] están en la misma cache line (64 bytes = 8 longs)
  // Thread 0 incrementa counters[0], Thread 1 incrementa counters[1]
  // → false sharing. Rendimiento destruido.

  // BIEN: padding para separar los contadores en cache lines distintas
  long[] counters = new long[NUM_THREADS * 8]; // 8 longs = 64 bytes entre cada counter
  // Thread 0 incrementa counters[0], Thread 1 incrementa counters[8]
  // → cada counter en su propia cache line. Sin false sharing.

  // MEJOR (Java 8+): @Contended annotation
  @jdk.internal.vm.annotation.Contended
  static class PaddedCounter {
      volatile long value;
  }
  // La JVM inserta padding automáticamente.

  // En Rust: alinear a 64 bytes
  #[repr(align(64))]
  struct PaddedCounter {
      value: AtomicI64,
  }

False sharing en sistemas reales:
  - LMAX Disruptor: los sequence counters están padded a cache lines
  - java.util.concurrent: ForkJoinPool usa @Contended internamente
  - Go runtime: la estructura P (processor) tiene padding entre campos
```

**Preguntas:**

1. ¿False sharing es un problema de correctitud o de rendimiento?

2. ¿Cuántos `long` caben en una cache line de 64 bytes?

3. ¿El padding desperdicia memoria. ¿Cuándo ese costo se justifica?

4. ¿False sharing puede ocurrir entre campos de un mismo struct?

5. ¿Un single-threaded program puede sufrir false sharing? ¿Cómo?

---

### Ejercicio 1.3.4 — Implementar: demostrar false sharing

**Tipo: Implementar**

```java
// Demostrar el impacto de false sharing en Java

public class FalseSharingDemo {

    static final int NUM_THREADS = 4;
    static final long ITERATIONS = 500_000_000L;

    // Versión 1: false sharing (contadores adyacentes)
    static volatile long[] sharedCounters = new long[NUM_THREADS];

    // Versión 2: sin false sharing (contadores separados por padding)
    static volatile long[] paddedCounters = new long[NUM_THREADS * 16]; // 16 longs = 128 bytes

    static void runBenchmark(String label, int counterStride) throws Exception {
        // Reset
        java.util.Arrays.fill(sharedCounters, 0);
        java.util.Arrays.fill(paddedCounters, 0);

        long[] targetArray = (counterStride == 1) ? sharedCounters : paddedCounters;

        Thread[] threads = new Thread[NUM_THREADS];
        long start = System.nanoTime();

        for (int t = 0; t < NUM_THREADS; t++) {
            final int idx = t * counterStride;
            threads[t] = new Thread(() -> {
                for (long i = 0; i < ITERATIONS; i++) {
                    targetArray[idx]++;
                }
            });
            threads[t].start();
        }

        for (Thread t : threads) t.join();
        long elapsed = System.nanoTime() - start;

        System.out.printf("%s: %,d ms  (%.1f ns/op)%n",
            label, elapsed / 1_000_000,
            (double) elapsed / ITERATIONS);
    }

    public static void main(String[] args) throws Exception {
        System.out.println("Threads: " + NUM_THREADS);
        System.out.println("Iterations per thread: " + ITERATIONS);
        System.out.println();

        runBenchmark("Con false sharing (stride=1) ", 1);
        runBenchmark("Sin false sharing (stride=16)", 16);
    }
}
// Resultado esperado:
// Con false sharing (stride=1):  ~8000-15000 ms
// Sin false sharing (stride=16): ~1500-3000 ms
// Ratio: 3-8× más lento con false sharing
```

```go
// Go: demostrar false sharing
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

const numGoroutines = 4
const iterations = 500_000_000

// Con false sharing: contadores adyacentes
type SharedCounters struct {
    counters [numGoroutines]int64
}

// Sin false sharing: cada contador en su propia cache line
type PaddedCounter struct {
    value int64
    _pad  [7]int64 // padding: 7 * 8 = 56 bytes + 8 bytes (value) = 64 bytes
}
type PaddedCounters struct {
    counters [numGoroutines]PaddedCounter
}

func benchmarkShared() time.Duration {
    var counters SharedCounters
    var wg sync.WaitGroup
    start := time.Now()
    for g := 0; g < numGoroutines; g++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            for i := 0; i < iterations; i++ {
                atomic.AddInt64(&counters.counters[idx], 1)
            }
        }(g)
    }
    wg.Wait()
    return time.Since(start)
}

func benchmarkPadded() time.Duration {
    var counters PaddedCounters
    var wg sync.WaitGroup
    start := time.Now()
    for g := 0; g < numGoroutines; g++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            for i := 0; i < iterations; i++ {
                atomic.AddInt64(&counters.counters[idx].value, 1)
            }
        }(g)
    }
    wg.Wait()
    return time.Since(start)
}

func main() {
    fmt.Printf("Con false sharing: %v\n", benchmarkShared())
    fmt.Printf("Sin false sharing: %v\n", benchmarkPadded())
}
```

**Preguntas:**

1. ¿El ratio de false sharing empeora con más threads? ¿Por qué?

2. ¿`volatile` en Java y `atomic.AddInt64` en Go son necesarios
   para que el benchmark sea correcto?

3. ¿El padding de 128 bytes (en vez de 64) sería más seguro
   considerando procesadores con cache lines de 128 bytes?

4. ¿En Rust, `#[repr(align(64))]` es la forma idiomática
   de prevenir false sharing?

5. ¿Cómo detectarías false sharing en un programa existente
   sin modificar el código? (Pista: `perf c2c` en Linux)

---

### Ejercicio 1.3.5 — Analizar: las cache lines en las estructuras de datos

**Tipo: Analizar**

```
Ahora que entiendes cache lines, reconsidera cada estructura:

  Array de structs vs struct de arrays:

    // Array of Structs (AoS):
    struct Venta { int userId; double monto; long timestamp; }
    Venta[] ventas = new Venta[1_000_000];

    // Si solo necesitas sumar todos los montos:
    // Cada Venta ocupa ~24 bytes. Una cache line trae ~2.6 ventas.
    // Pero solo necesitas monto (8 bytes de cada 24).
    // Eficiencia: 8/24 = 33% de los datos cargados son útiles.

    // Struct of Arrays (SoA):
    int[] userIds       = new int[1_000_000];
    double[] montos     = new double[1_000_000];
    long[] timestamps   = new long[1_000_000];

    // Para sumar montos: recorrer el array montos[].
    // Una cache line de 64 bytes trae 8 doubles.
    // Eficiencia: 100% de los datos cargados son útiles.
    // → SoA es ~3× más rápido para operaciones columnar.

  Esto es EXACTAMENTE lo que hace un columnar store (Parquet, Arrow):
    Almacena cada columna como un array independiente.
    → SoA a nivel de almacenamiento.

  Y es lo que hace un row store (PostgreSQL, MySQL):
    Almacena cada fila como un struct contiguo.
    → AoS a nivel de almacenamiento.

  La decisión AoS vs SoA depende del access pattern:
    - Leo filas completas (OLTP): AoS / row store
    - Leo columnas completas (OLAP): SoA / columnar store

  Esto no es coincidencia. Es cache lines.
```

**Preguntas:**

1. ¿ECS (Entity Component System) en game engines usa SoA?
   ¿Por la misma razón que los columnar stores?

2. ¿Arrow (Apache Arrow) es fundamentalmente un layout SoA?

3. ¿Puedes tener AoS y SoA en el mismo programa?
   ¿Cuándo elegirías cada uno?

4. ¿Un compilador puede transformar AoS a SoA automáticamente?

5. ¿El concepto de SoA aplica a estructuras de datos
   más allá de arrays (por ejemplo, un B-Tree)?

---

## Sección 1.4 — Alineación, Padding, y el Costo Oculto de los Structs

### Ejercicio 1.4.1 — Leer: por qué un struct no ocupa lo que piensas

**Tipo: Leer**

```
El CPU accede a la memoria más eficientemente cuando los datos
están alineados a su tamaño natural:
  - Un int32 debe estar en una dirección múltiplo de 4
  - Un int64 debe estar en una dirección múltiplo de 8
  - Un double debe estar en una dirección múltiplo de 8

Si un dato NO está alineado, el CPU puede necesitar dos lecturas
en vez de una (o directamente fallar en algunas arquitecturas).

Para garantizar alineación, el compilador inserta PADDING (bytes vacíos)
entre los campos del struct.

  Ejemplo en C/Rust/Go:

    struct MalOrdenado {
        char  a;     // 1 byte  (offset 0)
                     // 7 bytes de padding (para alinear b a múltiplo de 8)
        long  b;     // 8 bytes (offset 8)
        char  c;     // 1 byte  (offset 16)
                     // 7 bytes de padding (para alinear el struct a 8 bytes)
    };
    // sizeof: 24 bytes
    // Datos útiles: 10 bytes (1 + 8 + 1)
    // Desperdicio: 14 bytes (58%)

    struct BienOrdenado {
        long  b;     // 8 bytes (offset 0)
        char  a;     // 1 byte  (offset 8)
        char  c;     // 1 byte  (offset 9)
                     // 6 bytes de padding (para alinear el struct a 8 bytes)
    };
    // sizeof: 16 bytes
    // Datos útiles: 10 bytes
    // Desperdicio: 6 bytes (37%)

  Solo reordenando los campos, el struct pasó de 24 a 16 bytes.
  Para un array de 1 millón de structs:
    MalOrdenado: 24 MB
    BienOrdenado: 16 MB
    → 33% menos memoria y 33% más structs por cache line.

  Regla de oro: ordenar los campos de mayor a menor tamaño.
```

**Preguntas:**

1. ¿Java tiene el mismo problema de padding en sus objetos?

2. ¿Python se ve afectado por alignment de structs?

3. ¿El compilador de Go reordena los campos automáticamente?

4. ¿Rust con `#[repr(C)]` respeta el orden? ¿Y sin `#[repr(C)]`?

5. ¿Para un array de 100 millones de structs,
   la diferencia de 8 bytes por struct importa?

---

### Ejercicio 1.4.2 — Implementar: medir el tamaño real de un struct

**Tipo: Implementar**

```rust
// Rust: verificar el tamaño y alignment de structs
use std::mem;

struct MalOrdenado {
    a: u8,      // 1 byte
    b: u64,     // 8 bytes
    c: u8,      // 1 byte
}

struct BienOrdenado {
    b: u64,     // 8 bytes
    a: u8,      // 1 byte
    c: u8,      // 1 byte
}

#[repr(C)]  // forzar layout de C (sin reordenamiento)
struct CLayout {
    a: u8,
    b: u64,
    c: u8,
}

#[repr(packed)]  // sin padding (CUIDADO: puede causar accesos desalineados)
struct Packed {
    a: u8,
    b: u64,
    c: u8,
}

fn main() {
    println!("MalOrdenado:  size={}, align={}",
        mem::size_of::<MalOrdenado>(), mem::align_of::<MalOrdenado>());
    println!("BienOrdenado: size={}, align={}",
        mem::size_of::<BienOrdenado>(), mem::align_of::<BienOrdenado>());
    println!("CLayout:      size={}, align={}",
        mem::size_of::<CLayout>(), mem::align_of::<CLayout>());
    println!("Packed:       size={}, align={}",
        mem::size_of::<Packed>(), mem::align_of::<Packed>());

    // Resultado esperado:
    // MalOrdenado:  size=16, align=8  (Rust reordena los campos!)
    // BienOrdenado: size=16, align=8
    // CLayout:      size=24, align=8  (C no reordena → padding)
    // Packed:       size=10, align=1  (sin padding, posible desalineación)

    // NOTA: Rust (sin repr(C)) puede reordenar los campos libremente.
    // MalOrdenado y BienOrdenado pueden tener el MISMO tamaño
    // porque el compilador optimizó el layout.
}
```

```go
// Go: verificar tamaños con unsafe.Sizeof
package main

import (
    "fmt"
    "unsafe"
)

type MalOrdenado struct {
    A byte    // 1 byte
    B int64   // 8 bytes
    C byte    // 1 byte
}

type BienOrdenado struct {
    B int64   // 8 bytes
    A byte    // 1 byte
    C byte    // 1 byte
}

func main() {
    fmt.Printf("MalOrdenado:  %d bytes\n", unsafe.Sizeof(MalOrdenado{}))
    fmt.Printf("BienOrdenado: %d bytes\n", unsafe.Sizeof(BienOrdenado{}))

    // Resultado:
    // MalOrdenado:  24 bytes (Go NO reordena campos)
    // BienOrdenado: 16 bytes
    // → En Go, el programador es responsable de ordenar los campos.
}
```

**Preguntas:**

1. ¿Rust reordena campos por defecto pero Go no. ¿Cuál es más seguro?

2. ¿`#[repr(packed)]` en Rust puede causar crashes?
   ¿En qué arquitecturas?

3. ¿`fieldalignment` de Go (linter) detecta structs mal ordenados?

4. ¿Para interoperar con C (FFI), ¿necesitas `#[repr(C)]` en Rust?

5. ¿Java te da control sobre el layout de los campos de una clase?

---

### Ejercicio 1.4.3 — Implementar: el costo del padding en arrays grandes

**Tipo: Implementar**

```go
// Go: benchmark del impacto del padding en un array de 10M structs
package main

import (
    "fmt"
    "runtime"
    "time"
    "unsafe"
)

type MalOrdenado struct {
    Flag    bool     // 1 byte
    Amount  float64  // 8 bytes
    ID      int32    // 4 bytes
    Balance float64  // 8 bytes
    Status  byte     // 1 byte
}

type BienOrdenado struct {
    Amount  float64  // 8 bytes
    Balance float64  // 8 bytes
    ID      int32    // 4 bytes
    Flag    bool     // 1 byte
    Status  byte     // 1 byte
}

func main() {
    n := 10_000_000

    fmt.Printf("MalOrdenado:  %d bytes/struct\n", unsafe.Sizeof(MalOrdenado{}))
    fmt.Printf("BienOrdenado: %d bytes/struct\n", unsafe.Sizeof(BienOrdenado{}))

    // Medir asignación y recorrido de MalOrdenado
    runtime.GC()
    start := time.Now()
    mal := make([]MalOrdenado, n)
    for i := range mal {
        mal[i] = MalOrdenado{Amount: float64(i), Balance: float64(i) * 1.1}
    }
    var totalMal float64
    for i := range mal {
        totalMal += mal[i].Amount + mal[i].Balance
    }
    malTime := time.Since(start)

    // Medir asignación y recorrido de BienOrdenado
    runtime.GC()
    start = time.Now()
    bien := make([]BienOrdenado, n)
    for i := range bien {
        bien[i] = BienOrdenado{Amount: float64(i), Balance: float64(i) * 1.1}
    }
    var totalBien float64
    for i := range bien {
        totalBien += bien[i].Amount + bien[i].Balance
    }
    bienTime := time.Since(start)

    fmt.Printf("\nMalOrdenado:  %d MB, %v\n",
        int(unsafe.Sizeof(MalOrdenado{}))*n/1024/1024, malTime)
    fmt.Printf("BienOrdenado: %d MB, %v\n",
        int(unsafe.Sizeof(BienOrdenado{}))*n/1024/1024, bienTime)
}
```

**Preguntas:**

1. ¿Cuántos MB de diferencia hay entre las dos versiones para 10M structs?

2. ¿La diferencia de tiempo es proporcional a la diferencia de tamaño?

3. ¿Para structs con muchos campos, ¿la optimización de orden es más significativa?

4. ¿Un linter que detecte structs mal ordenados debería ser obligatorio en CI?

5. ¿En un sistema de data engineering que procesa millones de registros,
   ¿el padding de structs impacta el rendimiento del pipeline?

---

### Ejercicio 1.4.4 — Leer: object headers en la JVM — el costo de ser un objeto

**Tipo: Leer**

```
En Java y Scala, cada objeto en el heap tiene un header
ANTES de los datos del usuario.

  Object header en HotSpot JVM (64-bit, compressed oops):
  ┌────────────────────┬────────────────────┐
  │  Mark Word (8 bytes)│  Class Pointer (4B) │
  └────────────────────┴────────────────────┘
  Total: 12 bytes, alineado a 8 → 16 bytes de header

  Mark Word contiene:
  - Hash code (si se llamó hashCode())
  - GC age (cuántas veces sobrevivió el GC)
  - Lock state (biased, thin, fat lock)
  - GC forwarding pointer (durante GC)

  Arrays tienen un header extra:
  ┌────────────────────┬────────────────────┬──────────────┐
  │  Mark Word (8 bytes)│  Class Pointer (4B) │  Length (4B)  │
  └────────────────────┴────────────────────┴──────────────┘
  Total: 16 bytes de header para cada array.

  Impacto práctico:
    new Object()        →  16 bytes (header only, sin datos)
    new Integer(42)     →  16 bytes (12 header + 4 int, aligned to 16)
    new int[0]          →  16 bytes (header + length, sin datos)
    new int[1]          →  24 bytes (16 header + 4 datos, aligned to 24)
    new int[10]         →  56 bytes (16 header + 40 datos)
    new String("hello") → ~56 bytes (header + char[] referencia + hash + ...)

  Para un HashMap<Integer, Integer> con 1M entries:
    1M Entry objects:   32 bytes × 1M = 32 MB
    1M Integer keys:    16 bytes × 1M = 16 MB
    1M Integer values:  16 bytes × 1M = 16 MB
    Backing array:      ~8 bytes × 2M = 16 MB (load factor 0.5)
    Total: ~80 MB

  Si fuera int → int (sin boxing):
    ~16 bytes × 1M = 16 MB (array de pairs)
    Ratio: 5× más memoria con objetos.

  → Este es el costo de la abstracción en la JVM.
  → Java 21+ tiene Project Valhalla (value types)
    para reducir este overhead.
```

**Preguntas:**

1. ¿16 bytes de header por objeto es mucho o poco?

2. ¿Compressed oops reduce el header de 16 a 12 bytes. ¿Cómo funciona?

3. ¿Project Valhalla de Java eliminará los headers para value types?

4. ¿Scala sufre el mismo overhead que Java para sus case classes?

5. ¿Un ArrayList<Integer> de 1M elementos usa más o menos memoria
   que un HashMap<Integer, Integer> de 500K entries?

---

### Ejercicio 1.4.5 — Analizar: el presupuesto de memoria de una estructura

**Tipo: Analizar**

```
Antes de implementar cualquier estructura de datos,
calcula su "presupuesto de memoria" — cuántos bytes
ocupa por elemento, incluyendo todos los overheads.

  Ejemplo: almacenar 10M pares (userId: long, monto: double)

  Opción 1: Java HashMap<Long, Double>
    Entry:   32 bytes (header + key ref + value ref + hash + next)
    Long:    24 bytes (16 header + 8 dato)
    Double:  24 bytes (16 header + 8 dato)
    Per entry: 80 bytes
    Total: 800 MB + backing array (~160 MB) ≈ 960 MB

  Opción 2: Dos arrays paralelos (long[] + double[])
    long[]:  16 header + 80M datos = ~80 MB
    double[]:16 header + 80M datos = ~80 MB
    Total: ~160 MB

  Opción 3: Un solo array de structs (si tuviéramos value types)
    16 bytes por par (8 + 8, sin headers)
    Total: ~160 MB

  Opción 4: Rust HashMap<i64, f64>
    ~50 bytes por entry (SwissTable, open addressing)
    Total: ~500 MB (mejor que Java, peor que arrays)

  Opción 5: Rust Vec<(i64, f64)> ordenado + binary search
    16 bytes per element (sin overhead)
    Total: ~160 MB
    Tradeoff: lookup O(log n) en vez de O(1)

  La opción más elegante (HashMap) usa 6× más memoria
  que la opción más simple (arrays paralelos).
  Para 10M elementos, la diferencia es 800 MB.
  Esa memoria extra no solo cuesta RAM — cuesta cache misses,
  GC pressure, y tiempo de inicialización.
```

**Preguntas:**

1. ¿960 MB para 10M pares key-value es aceptable en producción?

2. ¿Cuándo el HashMap justifica su overhead de memoria
   sobre los arrays paralelos?

3. ¿Los arrays paralelos mantienen la asociación key→value?
   ¿Cómo haces un lookup eficiente?

4. ¿En Scala, un `Map[Long, Double]` inmutable usa más o menos
   memoria que un `HashMap<Long, Double>` mutable de Java?

5. ¿Calcular el presupuesto de memoria antes de elegir
   la estructura debería ser una práctica estándar?

---

## Sección 1.5 — Java: Object Headers, Boxing, y el Precio de la Abstracción

### Ejercicio 1.5.1 — Implementar: medir el overhead real de Integer vs int

**Tipo: Implementar**

```java
import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.info.GraphLayout;

// Ejecutar con JOL (Java Object Layout):
// mvn dependency: org.openjdk.jol:jol-core:0.17

public class MemoryLayoutDemo {

    public static void main(String[] args) {
        // Layout de un Integer
        System.out.println(ClassLayout.parseInstance(Integer.valueOf(42)).toPrintable());
        // Esperado:
        // java.lang.Integer object internals:
        //  OFFSET  SIZE  TYPE   DESCRIPTION
        //       0    12        (object header: mark + class)
        //      12     4  int   Integer.value
        // Instance size: 16 bytes

        // Layout de un int[]
        int[] arr = new int[10];
        System.out.println(ClassLayout.parseInstance(arr).toPrintable());
        // Esperado:
        // OFFSET  SIZE  TYPE   DESCRIPTION
        //      0    16        (array header: mark + class + length)
        //     16    40  int[]  [10 ints × 4 bytes]
        // Instance size: 56 bytes

        // Layout de Integer[]
        Integer[] boxed = new Integer[10];
        for (int i = 0; i < 10; i++) boxed[i] = i + 200; // fuera del cache
        System.out.println(GraphLayout.parseInstance((Object) boxed).toFootprint());
        // Esperado: ~216 bytes
        // 16 (array header) + 80 (10 refs) + 10 × 16 (10 Integer objects) = 256 bytes
        // vs int[10] = 56 bytes → 4.6× más memoria

        // HashMap<Integer, Integer> con 100 entries
        var map = new java.util.HashMap<Integer, Integer>();
        for (int i = 0; i < 100; i++) map.put(i + 200, i + 300);
        System.out.println(GraphLayout.parseInstance(map).toFootprint());
        // Observar: el footprint total incluye Entry objects, Integer keys/values,
        // y el backing array. Mucho más que 100 × 8 bytes.
    }
}
```

**Preguntas:**

1. ¿JOL es la única forma de medir el layout de objetos en Java?

2. ¿`Integer.valueOf(42)` devuelve un objeto del cache. ¿Qué pasa
   con `Integer.valueOf(200)` que está fuera del rango -128 a 127?

3. ¿El footprint de un `HashMap<Integer, Integer>` de 100 entries
   es mayor o menor que el de un `int[]` de 200 elementos?

4. ¿`HashMap<String, List<Integer>>` tiene cuántos niveles
   de indirección (pointer chasing)?

5. ¿Las versiones especializadas como `IntIntHashMap` de Eclipse Collections
   eliminan el overhead de boxing?

---

### Ejercicio 1.5.2 — Analizar: autoboxing — la trampa silenciosa de Java

**Tipo: Analizar**

```java
// Autoboxing: Java convierte automáticamente int ↔ Integer.
// Esto genera asignaciones de heap INVISIBLES en el código.

// Caso 1: suma "inocente"
Long total = 0L;
for (int i = 0; i < 1_000_000; i++) {
    total += i;  // total = Long.valueOf(total.longValue() + i)
                 // Cada iteración crea un NUEVO Long object en el heap.
                 // 1M objetos Long = 24 MB de basura para el GC.
}
// Fix: usar long (primitivo) en vez de Long (objeto).

// Caso 2: HashMap con primitivos
Map<Integer, Integer> map = new HashMap<>();
for (int i = 0; i < 1_000_000; i++) {
    map.put(i, i * 2);
    // put(int, int) → put(Integer, Integer)
    // 2 autoboxing operations por iteración
    // 2M objetos Integer creados
}
// Fix: usar IntIntHashMap de Eclipse Collections o fastutil.

// Caso 3: comparación con ==
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true (Integer cache: -128 a 127)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false (fuera del cache, objetos distintos)
System.out.println(c.equals(d));  // true (comparación por valor)
// Este bug es real y ha causado incidentes en producción.
```

**Preguntas:**

1. ¿El compilador de Java puede optimizar el autoboxing
   en el caso de `Long total += i`?

2. ¿`IntStream.range(0, 1_000_000).sum()` evita el autoboxing?

3. ¿Kotlin resuelve el problema de autoboxing?

4. ¿Scala `Int` se compila como `int` (primitivo) o `Integer` (objeto)?

5. ¿Cuántas asignaciones de heap genera
   `list.stream().map(x -> x + 1).collect(toList())`
   para una lista de 1M Integers?

---

### Ejercicio 1.5.3 — Implementar: int[] vs Integer[] vs ArrayList<Integer>

**Tipo: Implementar**

```java
public class PrimitiveVsBoxedBenchmark {

    static final int N = 10_000_000;

    static long sumIntArray(int[] data) {
        long total = 0;
        for (int v : data) total += v;
        return total;
    }

    static long sumIntegerArray(Integer[] data) {
        long total = 0;
        for (Integer v : data) total += v; // unboxing en cada iteración
        return total;
    }

    static long sumArrayList(java.util.ArrayList<Integer> data) {
        long total = 0;
        for (Integer v : data) total += v; // unboxing + iterator overhead
        return total;
    }

    public static void main(String[] args) {
        int[] primitiveArr = new int[N];
        Integer[] boxedArr = new Integer[N];
        var list = new java.util.ArrayList<Integer>(N);

        for (int i = 0; i < N; i++) {
            primitiveArr[i] = i;
            boxedArr[i] = i; // autoboxing
            list.add(i);     // autoboxing
        }

        // Warmup
        for (int w = 0; w < 10; w++) {
            sumIntArray(primitiveArr);
            sumIntegerArray(boxedArr);
            sumArrayList(list);
        }

        int runs = 50;
        long start, elapsed;

        start = System.nanoTime();
        for (int r = 0; r < runs; r++) sumIntArray(primitiveArr);
        elapsed = System.nanoTime() - start;
        System.out.printf("int[]:             %.1f ns/elem%n", (double) elapsed / (N * (long) runs));

        start = System.nanoTime();
        for (int r = 0; r < runs; r++) sumIntegerArray(boxedArr);
        elapsed = System.nanoTime() - start;
        System.out.printf("Integer[]:         %.1f ns/elem%n", (double) elapsed / (N * (long) runs));

        start = System.nanoTime();
        for (int r = 0; r < runs; r++) sumArrayList(list);
        elapsed = System.nanoTime() - start;
        System.out.printf("ArrayList<Integer>: %.1f ns/elem%n", (double) elapsed / (N * (long) runs));
    }
}
// Resultado esperado:
// int[]:              ~0.3-0.5 ns/elem (SIMD auto-vectorización posible)
// Integer[]:          ~3-5 ns/elem (pointer chasing, unboxing)
// ArrayList<Integer>: ~4-6 ns/elem (pointer chasing, unboxing, iterator)
// Ratio: int[] es 8-15× más rápido que ArrayList<Integer>
```

**Preguntas:**

1. ¿El JIT puede eliminar el unboxing de Integer a int
   en el loop de `sumIntegerArray`?

2. ¿`Integer[]` tiene mejor o peor locality que `ArrayList<Integer>`?

3. ¿Las primitive-specialized collections (fastutil, Eclipse Collections)
   alcanzan el rendimiento de `int[]`?

4. ¿En un pipeline de Spark, los datos internos usan
   arrays primitivos o boxed? (Pista: Tungsten)

5. ¿Valhalla (JEP 401) eliminará la diferencia entre int[] e Integer[]?

---

### Ejercicio 1.5.4 — Analizar: pointer chasing — el enemigo invisible

**Tipo: Analizar**

```
Pointer chasing: acceder a un dato que requiere seguir
una cadena de punteros, cada uno en una dirección impredecible.

  Ejemplo: HashMap.get("key")
    1. hash("key") → bucket index
    2. array[bucket] → puntero al primer Entry (cache miss #1)
    3. Entry.key → puntero al String key (cache miss #2)
    4. Comparar key: String.value → puntero al char[] (cache miss #3)
    5. Si no coincide: Entry.next → puntero al siguiente Entry (cache miss #4)
    6. Repetir 3-5 hasta encontrar

  Cada "→" es un puntero que puede apuntar a cualquier parte del heap.
  Cada puntero es un posible cache miss (~5-100 ns).

  En el peor caso, un solo HashMap.get() puede causar 4-8 cache misses.
  A 100 ns cada uno = 400-800 ns por lookup.

  vs un array lookup:
    array[index]: 1 acceso directo. 0-1 cache misses. ~1-5 ns.

  Pointer chasing es la razón por la cual:
  - HashMap es más lento de lo que sugiere O(1)
  - LinkedList es inutilizable para datasets grandes
  - TreeMap pierde contra un sorted array + binary search
  - Java es más lento que Rust para las mismas estructuras
    (Rust almacena datos inline, Java almacena punteros a objetos)
```

**Preguntas:**

1. ¿Open addressing (sin linked lists) reduce el pointer chasing
   de un hash map? ¿A cuántos cache misses?

2. ¿Un B-Tree reduce el pointer chasing vs un Red-Black tree? ¿Cuánto?

3. ¿"Flat" data structures (datos inline, sin punteros)
   son siempre más rápidas que las "indirected"?

4. ¿Java puede eliminar la indirección de String
   (puntero a char[]) con compact strings?

5. ¿El concepto de pointer chasing existe en Python?
   ¿Todo acceso en Python es pointer chasing?

---

### Ejercicio 1.5.5 — Analizar: Scala en la JVM — ¿paga el mismo precio?

**Tipo: Analizar**

```scala
// Scala compila a JVM bytecode.
// ¿Sufre los mismos costos que Java?

// Primitivos:
val x: Int = 42
// Compilado a: int x = 42 (primitivo JVM, stack, 4 bytes)
// → SIN overhead. Scala Int = Java int en bytecode.

// Pero:
val y: Option[Int] = Some(42)
// Some es un objeto. Int se boxea a Integer.
// → 32+ bytes en heap (Some header + Integer header + valor)

// Colecciones inmutables:
val list = List(1, 2, 3, 4, 5)
// List es una linked list. Cada elemento es un nodo Cons en el heap.
// Cons: header (16) + head ref (8) + tail ref (8) = ~32 bytes por nodo
// + Integer boxing: ~16 bytes por elemento
// Total: ~48 bytes por elemento vs 4 bytes en int[]
// Ratio: 12× más memoria.

// Vector (Scala's default "array-like" immutable collection):
val vec = Vector(1, 2, 3, 4, 5)
// Implementado como un trie con branching factor 32.
// Mejor que List para acceso por índice (O(log32 n) ≈ O(1) práctico).
// Pero todavía boxea los Int a Integer.
// → Para datos numéricos grandes, usar Array[Int] (mutable, primitivo).

// El tradeoff de Scala:
// Inmutabilidad + seguridad + expresividad
// vs
// Overhead de boxing + más objetos en heap + más GC pressure.

// La decisión pragmática:
// - Colecciones pequeñas (<10K) o lógica compleja: inmutables (List, Vector, Map)
// - Procesamiento numérico masivo: Array[Int/Long/Double] (primitivos JVM)
// - Hot paths: cuidar el boxing (usar @specialized o manual specialization)
```

**Preguntas:**

1. ¿`@specialized` de Scala genera versiones primitivas de las colecciones?

2. ¿Scala 3 mejora la situación de boxing con opaque types?

3. ¿Un `Vector[Int]` de Scala es más rápido que un `List[Int]`
   para recorrido secuencial? ¿Por la locality?

4. ¿`Array[Int]` de Scala es literalmente un `int[]` de Java?

5. ¿Las colecciones inmutables de Scala se justifican
   en programas single-threaded?

---

## Sección 1.6 — Go y Rust: Dos Filosofías de Memoria

### Ejercicio 1.6.1 — Leer: Go — structs por valor y escape analysis

**Tipo: Leer**

```go
// Go tiene una filosofía distinta a Java:
// Los structs son valores, no referencias.
// Un struct se copia cuando se pasa a una función (a menos que uses puntero).

type Venta struct {
    OrderID   int64
    UserID    int64
    Monto     float64
    Region    [16]byte  // string fijo — no puntero, no heap
}
// sizeof(Venta) = 8 + 8 + 8 + 16 = 40 bytes
// Sin headers. Sin punteros internos (si usas [16]byte en vez de string).
// Un slice de 1M Ventas = 40 MB de datos contiguos.
// → Excelente cache locality.

// Si usas string en vez de [16]byte:
type VentaConString struct {
    OrderID int64
    UserID  int64
    Monto   float64
    Region  string    // string en Go = (puntero, largo) = 16 bytes
}
// sizeof(VentaConString) = 8 + 8 + 8 + 16 = 40 bytes (mismo tamaño)
// PERO: Region.puntero apunta a datos en el heap.
// El GC necesita rastrear esos punteros. Más GC pressure.
// Y recorrer Region tiene un cache miss extra (pointer chasing).

// Escape analysis decide si un valor va al stack o al heap:
// go build -gcflags='-m' muestra las decisiones.

func crearEnStack() Venta {
    v := Venta{OrderID: 1, Monto: 100.0}
    return v  // COPIA al caller. v no escapa. Stack.
}

func crearEnHeap() *Venta {
    v := Venta{OrderID: 1, Monto: 100.0}
    return &v  // &v escapa. v se asigna en el heap.
}

// Regla de Go:
// Si puedes retornar por valor (sin puntero), el compilador usa el stack.
// Si retornas un puntero, el valor escapa al heap.
// → Preferir retorno por valor para structs pequeños (< 256 bytes).
// → Usar punteros para structs grandes (evitar copias costosas).
```

**Preguntas:**

1. ¿Cuál es el tamaño límite para retornar un struct por valor vs por puntero en Go?

2. ¿`[16]byte` es mejor que `string` para campos de tamaño fijo? ¿Siempre?

3. ¿El GC de Go es más rápido si hay menos punteros en los structs?

4. ¿Un slice `[]Venta` almacena los structs contiguos en memoria? ¿Siempre?

5. ¿Go tiene algún equivalente a Project Valhalla de Java?

---

### Ejercicio 1.6.2 — Leer: Rust — ownership como modelo de memoria

**Tipo: Leer**

```rust
// Rust no tiene GC. En vez de eso, tiene OWNERSHIP:
// Cada valor tiene exactamente un dueño.
// Cuando el dueño sale del scope, el valor se libera.

fn main() {
    // s1 es el owner del String "hello"
    let s1 = String::from("hello");
    
    // MOVE: s1 transfiere el ownership a s2
    let s2 = s1;
    // s1 ya no es válido. Usar s1 aquí es un ERROR DE COMPILACIÓN.
    // println!("{}", s1); // ERROR: value borrowed after move
    
    // s2 es el owner ahora
    println!("{}", s2);  // OK
    
    // Cuando s2 sale del scope, la memoria se libera automáticamente.
    // No hay GC. No hay reference counting (excepto con Rc/Arc).
    // El compilador sabe exactamente cuándo se libera cada byte.
}

// Para estructuras de datos, ownership tiene consecuencias:
struct NodoLista {
    valor: i64,
    siguiente: Option<Box<NodoLista>>,  // Box = puntero a heap con ownership
}
// Cuando un NodoLista se libera, su Box<siguiente> se libera
// recursivamente. Toda la lista se libera en cadena.
// Sin GC. Sin leaks (excepto con Rc + ciclos).

// BORROWING: acceder sin transferir ownership
fn sumar_slice(data: &[i64]) -> i64 {
    // &[i64]: referencia prestada (borrow) a un slice
    // Esta función puede LEER data pero no modificarla ni liberarla.
    data.iter().sum()
}

fn main2() {
    let datos = vec![1, 2, 3, 4, 5];
    let total = sumar_slice(&datos);  // préstamo inmutable
    println!("{}", datos[0]);          // datos sigue siendo válido
}

// Implicación para estructuras de datos:
// En Rust, el HashMap es DUEÑO de sus keys y values.
// No puedes tener un puntero a una key del HashMap
// y al mismo tiempo modificar el HashMap.
// El compilador lo prohíbe. Esto previene toda una clase de bugs.
```

**Preguntas:**

1. ¿El modelo de ownership de Rust elimina use-after-free?

2. ¿Box en Rust es equivalente a `new` en Java?

3. ¿Un HashMap de Rust puede tener references como values?
   ¿Qué complicaciones trae?

4. ¿`Arc<Mutex<T>>` en Rust es comparable al GC de Java
   en términos de overhead?

5. ¿El borrow checker de Rust impide construir ciertas estructuras
   (ej: doubly linked list)? ¿Cómo se resuelve?

---

### Ejercicio 1.6.3 — Comparar: el mismo struct en 5 lenguajes

**Tipo: Implementar**

```
Tarea: definir una estructura "Transaccion" con:
  - id: entero de 64 bits
  - monto: decimal de 64 bits
  - activa: booleano
  - region: string de hasta 16 caracteres

Calcular el tamaño en memoria en cada lenguaje.
```

```python
# Python
from dataclasses import dataclass
import sys

@dataclass
class Transaccion:
    id: int          # ~28 bytes (PyLongObject)
    monto: float     # ~24 bytes (PyFloatObject)
    activa: bool     # ~28 bytes (PyBoolObject — sí, 28 bytes para un bool)
    region: str      # ~65+ bytes (PyUnicodeObject para "norte")

t = Transaccion(id=1, monto=100.0, activa=True, region="norte")
print(sys.getsizeof(t))  # ~56 bytes (solo el dataclass object, sin los campos)
# Tamaño total real: ~200+ bytes (incluyendo todos los objetos referenciados)
```

```java
// Java
public class Transaccion {
    long id;          // 8 bytes (primitivo)
    double monto;     // 8 bytes (primitivo)
    boolean activa;   // 1 byte + 7 padding
    String region;    // 8 bytes (referencia) + ~56 bytes (String object en heap)
}
// Object header: 16 bytes
// Campos: 8 + 8 + 8 + 8 = 32 bytes
// Total del objeto: 48 bytes + ~56 bytes del String = ~104 bytes
```

```scala
// Scala
case class Transaccion(
    id: Long,         // primitivo JVM: 8 bytes
    monto: Double,    // primitivo JVM: 8 bytes
    activa: Boolean,  // primitivo JVM: 1 byte + padding
    region: String    // referencia: 8 bytes + String object en heap
)
// Mismo layout que Java (misma JVM).
// Total: ~104 bytes (idéntico a Java)
// case class agrega: toString, equals, hashCode, copy generados.
// Pero el layout de memoria es el mismo.
```

```go
// Go
type Transaccion struct {
    ID     int64     // 8 bytes
    Monto  float64   // 8 bytes
    Activa bool      // 1 byte
    Region string    // 16 bytes (puntero + largo)
}
// Sin object header.
// sizeof: 8 + 8 + 1 + 7(pad) + 16 = 40 bytes
// + string data en heap: ~5 bytes para "norte"
// Total: ~45 bytes
```

```rust
// Rust
struct Transaccion {
    id: i64,            // 8 bytes
    monto: f64,         // 8 bytes
    activa: bool,       // 1 byte
    region: String,     // 24 bytes (puntero + largo + capacidad)
}
// Sin object header.
// size_of: 8 + 8 + 1 + 7(pad) + 24 = 48 bytes (stack)
// + string data en heap: ~5 bytes para "norte"
// Total: ~53 bytes

// Alternativa con string fijo (sin heap):
struct TransaccionInline {
    id: i64,
    monto: f64,
    activa: bool,
    region: [u8; 16],   // 16 bytes inline, sin heap
}
// size_of: 8 + 8 + 1 + 16 + 7(pad) = 40 bytes
// Total: 40 bytes, todo en stack. Sin heap. Sin GC.
```

```
Resumen:

  Lenguaje    Tamaño por Transaccion     Heap allocs    GC
  ────────    ──────────────────────      ──────────     ──
  Python      ~200 bytes                 5+ objetos     sí (refcount + cycle)
  Java        ~104 bytes                 2 (obj + str)  sí (G1/ZGC)
  Scala       ~104 bytes                 2 (obj + str)  sí (JVM GC)
  Go          ~45 bytes                  1 (str data)   sí (concurrent)
  Rust        ~53 bytes (String)         1 (str data)   no
  Rust        ~40 bytes ([u8; 16])       0              no

  Para 10 millones de transacciones:
  Python:  ~2.0 GB
  Java:    ~1.0 GB
  Scala:   ~1.0 GB
  Go:      ~0.45 GB
  Rust:    ~0.40 GB

  Ratio Python/Rust: 5×
  Ratio Java/Rust: 2.5×
```

**Preguntas:**

1. ¿La diferencia de 5× entre Python y Rust importa en producción?

2. ¿Go y Rust tienen tamaños casi iguales. ¿La diferencia real está en el GC?

3. ¿Java con Valhalla cerraría la brecha con Go/Rust?

4. ¿Para un pipeline de data engineering que procesa 10M registros,
   ¿el lenguaje cambia el costo de infraestructura?

5. ¿Cuándo el overhead de Python es aceptable
   y cuándo necesitas Go o Rust?

---

### Ejercicio 1.6.4 — Implementar: escape analysis de Go en acción

**Tipo: Implementar**

```go
// Demostrar escape analysis con go build -gcflags='-m'
package main

import "fmt"

type Punto struct {
    X, Y float64
}

// Caso 1: no escapa → stack
func sumar(a, b Punto) Punto {
    return Punto{X: a.X + b.X, Y: a.Y + b.Y}
}

// Caso 2: escapa → heap (retorna puntero)
func nuevo(x, y float64) *Punto {
    p := Punto{X: x, Y: y}
    return &p // &p escapa → heap
}

// Caso 3: escapa → heap (asignado a variable externa)
var global *Punto
func asignarGlobal() {
    p := Punto{X: 1, Y: 2}
    global = &p // &p escapa a variable global → heap
}

// Caso 4: escapa → heap (interface{})
func imprimir(p Punto) {
    fmt.Println(p) // fmt.Println acepta interface{}
                   // p se boxea → escapa al heap
}

// Caso 5: no escapa → stack (slice local)
func sumarSlice() float64 {
    puntos := make([]Punto, 100) // ¿escapa?
    // Si el compilador determina que puntos no escapa,
    // lo asigna en el stack (¡100 Puntos × 16 bytes = 1600 bytes en stack!)
    var total float64
    for _, p := range puntos {
        total += p.X + p.Y
    }
    return total
}

func main() {
    a := Punto{1, 2}
    b := Punto{3, 4}
    c := sumar(a, b)
    fmt.Println(c)

    d := nuevo(5, 6)
    fmt.Println(d)

    asignarGlobal()
    imprimir(Punto{7, 8})
    fmt.Println(sumarSlice())
}

// Compilar con: go build -gcflags='-m -m' main.go
// El output mostrará las decisiones de escape analysis para cada caso.
```

**Preguntas:**

1. ¿`fmt.Println` causa escape porque acepta `interface{}`?
   ¿Cómo evitarlo?

2. ¿El caso 5 (slice de 100 Puntos) se asigna en el stack? ¿Siempre?

3. ¿Hay un tamaño máximo para asignaciones en el stack en Go?

4. ¿El escape analysis de Go puede cambiar entre versiones del compilador?

5. ¿Monitorear las asignaciones de heap (`go tool pprof -alloc_space`)
   es una práctica habitual en Go?

---

### Ejercicio 1.6.5 — Analizar: eligiendo el lenguaje por el modelo de memoria

**Tipo: Analizar**

```
Cada lenguaje tiene un tradeoff de memoria:

  Python:  Máxima facilidad, máximo overhead.
           Bueno para: prototipos, scripts, glue code.
           Malo para: procesar millones de registros en memoria.
           → Usa NumPy/Polars/PyArrow para hot paths.

  Java:    Abstracciones seguras, overhead moderado.
           Bueno para: sistemas de larga duración, grandes equipos.
           Malo para: baja latencia predecible (GC pauses).
           → Usa primitive arrays y collections especializadas para hot paths.

  Scala:   Misma JVM que Java + inmutabilidad + expresividad.
           Bueno para: lógica compleja, pipelines de datos (Spark).
           Malo para: code sections donde boxing es inaceptable.
           → Usa Array[Int/Long/Double] para hot paths numéricos.

  Go:      Structs sin headers, escape analysis, GC de baja latencia.
           Bueno para: servicios de red, herramientas CLI, infraestructura.
           Malo para: cuando necesitas cero pausas de GC.
           → Diseña structs para minimizar punteros (reduce GC pressure).

  Rust:    Control total, cero overhead, sin GC.
           Bueno para: bases de datos, motores de búsqueda, sistemas embebidos.
           Malo para: prototipado rápido, equipos sin experiencia en ownership.
           → El lenguaje ideal para implementar estructuras de datos.

  En este libro:
  - Python para entender la lógica (claridad)
  - Java para la implementación de referencia (ecosistema de datos)
  - Scala para el enfoque funcional (inmutabilidad, pattern matching)
  - Go para concurrencia (goroutines, channels)
  - Rust para rendimiento y control de memoria (benchmarks definitivos)
```

**Preguntas:**

1. ¿Un data engineer necesita saber Rust?

2. ¿Spark (Scala/JVM) paga el overhead de boxing
   para sus DataFrames internos? (Pista: Tungsten off-heap)

3. ¿Un servicio en Go que procesa 1M requests/segundo
   tiene problemas de GC?

4. ¿Python con PyArrow evita el overhead de objetos?

5. ¿Si solo pudieras elegir dos lenguajes de esta lista
   para implementar estructuras de datos, ¿cuáles serían y por qué?

---

## Sección 1.7 — El Benchmark que Cambia Todo: Array vs Linked List

### Ejercicio 1.7.1 — Implementar: el benchmark completo en Java

**Tipo: Implementar**

```java
// El benchmark definitivo: ArrayList vs LinkedList en Java
// Operaciones: iterate, get(random), add(middle), remove(middle)

import java.util.*;

public class ArrayVsLinkedBenchmark {

    static final int N = 1_000_000;
    static final int LOOKUPS = 100_000;

    static void benchIterate(List<Integer> list, String name) {
        long start = System.nanoTime();
        long total = 0;
        for (int v : list) total += v;
        long elapsed = System.nanoTime() - start;
        System.out.printf("%-25s iterate:      %6d ms%n", name, elapsed / 1_000_000);
    }

    static void benchRandomGet(List<Integer> list, int[] indices, String name) {
        long start = System.nanoTime();
        long total = 0;
        for (int idx : indices) total += list.get(idx);
        long elapsed = System.nanoTime() - start;
        System.out.printf("%-25s random get:    %6d ms  (%d lookups)%n",
            name, elapsed / 1_000_000, indices.length);
    }

    static void benchAddMiddle(String name, int size) {
        // Crear lista fresca para cada test
        List<Integer> arrayList = new ArrayList<>();
        List<Integer> linkedList = new LinkedList<>();
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
            linkedList.add(i);
        }

        int inserts = 10_000;

        long start = System.nanoTime();
        for (int i = 0; i < inserts; i++) {
            arrayList.add(size / 2, i); // insert en el medio
        }
        long arrayTime = System.nanoTime() - start;

        start = System.nanoTime();
        // LinkedList: primero hay que LLEGAR al medio (O(n))
        ListIterator<Integer> it = linkedList.listIterator(size / 2);
        for (int i = 0; i < inserts; i++) {
            it.add(i); // insert en la posición actual (O(1))
        }
        long linkedTime = System.nanoTime() - start;

        System.out.printf("add(middle) × %d:  ArrayList=%d ms, LinkedList=%d ms%n",
            inserts, arrayTime / 1_000_000, linkedTime / 1_000_000);
    }

    public static void main(String[] args) {
        ArrayList<Integer> arrayList = new ArrayList<>();
        LinkedList<Integer> linkedList = new LinkedList<>();

        for (int i = 0; i < N; i++) {
            arrayList.add(i);
            linkedList.add(i);
        }

        var rng = new Random(42);
        int[] indices = new int[LOOKUPS];
        for (int i = 0; i < LOOKUPS; i++) indices[i] = rng.nextInt(N);

        // Warmup
        benchIterate(arrayList, "warmup");
        benchIterate(linkedList, "warmup");

        System.out.println("\n=== N = " + N + " ===\n");
        benchIterate(arrayList, "ArrayList<Integer>");
        benchIterate(linkedList, "LinkedList<Integer>");
        System.out.println();
        benchRandomGet(arrayList, indices, "ArrayList<Integer>");
        benchRandomGet(linkedList, indices, "LinkedList<Integer>");
        System.out.println();
        benchAddMiddle("N=" + N, N);
    }
}
// Resultados esperados (N=1,000,000):
//
// ArrayList<Integer>    iterate:         8 ms
// LinkedList<Integer>   iterate:        25 ms  (~3×)
//
// ArrayList<Integer>    random get:      2 ms
// LinkedList<Integer>   random get:  28000 ms  (~14,000×)
//
// add(middle) × 10000:  ArrayList=15 ms, LinkedList=3 ms
//   (LinkedList gana en add, PERO solo porque usamos ListIterator
//    que ya está en la posición. Sin ListIterator, LinkedList.add(n/2, x)
//    necesita O(n) para llegar al medio primero.)
```

**Preguntas:**

1. ¿LinkedList random get es 14,000× más lento. ¿Es un error de benchmark?

2. ¿LinkedList gana en add(middle) solo porque usamos ListIterator?

3. ¿Hay alguna operación donde LinkedList sea claramente mejor
   que ArrayList en este benchmark?

4. ¿Joshua Bloch (autor de Java Collections) recomienda
   nunca usar LinkedList?

5. ¿Este benchmark sería diferente con `int[]` vs `int`-linked-list
   (sin boxing)?

---

### Ejercicio 1.7.2 — Implementar: el mismo benchmark en Rust y Go

**Tipo: Implementar**

```rust
// Rust: Vec vs LinkedList — el benchmark más limpio (sin GC)
use std::collections::LinkedList;
use std::time::Instant;

fn bench_iterate_vec(data: &[i64]) -> (i64, std::time::Duration) {
    let start = Instant::now();
    let total: i64 = data.iter().sum();
    (total, start.elapsed())
}

fn bench_iterate_list(data: &LinkedList<i64>) -> (i64, std::time::Duration) {
    let start = Instant::now();
    let total: i64 = data.iter().sum();
    (total, start.elapsed())
}

fn main() {
    let n: usize = 10_000_000;

    let vec_data: Vec<i64> = (0..n as i64).collect();
    let list_data: LinkedList<i64> = (0..n as i64).collect();

    // Warmup
    let _ = bench_iterate_vec(&vec_data);
    let _ = bench_iterate_list(&list_data);

    let runs = 10;
    let mut vec_total = std::time::Duration::ZERO;
    let mut list_total = std::time::Duration::ZERO;

    for _ in 0..runs {
        vec_total += bench_iterate_vec(&vec_data).1;
        list_total += bench_iterate_list(&list_data).1;
    }

    let vec_ns = vec_total.as_nanos() as f64 / (n as f64 * runs as f64);
    let list_ns = list_total.as_nanos() as f64 / (n as f64 * runs as f64);

    println!("Vec<i64>:        {:.2} ns/elem  ({:.0} ms total)",
        vec_ns, vec_total.as_millis() as f64 / runs as f64);
    println!("LinkedList<i64>: {:.2} ns/elem  ({:.0} ms total)",
        list_ns, list_total.as_millis() as f64 / runs as f64);
    println!("Ratio:           {:.0}×", list_ns / vec_ns);
}
// cargo run --release
// Esperado:
// Vec<i64>:        ~0.3 ns/elem  (SIMD auto-vectorización)
// LinkedList<i64>: ~8-15 ns/elem (cache misses)
// Ratio:           25-50×
```

**Preguntas:**

1. ¿El ratio en Rust es mayor que en Java? ¿Por qué?

2. ¿`cargo run --release` es esencial para este benchmark?
   ¿Qué pasa sin `--release`?

3. ¿La documentación de Rust recomienda `VecDeque` sobre `LinkedList`?

4. ¿En Go, `container/list` almacena `interface{}` — ¿eso añade overhead?

5. ¿En qué momento de la carrera de un programador
   debería ejecutar este benchmark y entender los resultados?

---

### Ejercicio 1.7.3 — Analizar: por qué esto importa para data engineering

**Tipo: Analizar**

```
Las decisiones de memoria de este capítulo aparecen en cada sistema
de datos que usarás:

  Apache Spark (Tungsten):
    Spark descubrió que los DataFrames basados en objetos Java
    usaban 5× más memoria que los datos raw.
    Solución: Tungsten almacena datos OFF-HEAP en formato binario,
    evitando object headers, boxing, y GC.
    → Es exactamente el principio de SoA + arrays primitivos.

  Apache Kafka (log segments):
    Kafka almacena mensajes en archivos append-only (log segments).
    Cada segmento es un archivo mapeado en memoria (mmap).
    Los datos son bytes contiguos — no objetos Java en el heap.
    → Kafka evita el GC almacenando datos fuera del heap.

  Apache Flink (state backend):
    Flink puede usar RocksDB como state backend.
    RocksDB almacena datos en formato serializado (bytes),
    no como objetos Java.
    → Misma estrategia: evitar el overhead de objetos.

  Parquet / Arrow:
    Ambos usan layout columnar (SoA).
    Los datos de una columna son un array contiguo de valores.
    → Cache lines traen 8-16 valores de la misma columna en un acceso.
    → Es la diferencia entre escanear 1 GB en 0.5 segundos vs 5 segundos.

  La lección:
    Los sistemas de datos de alto rendimiento NO usan las abstracciones
    de alto nivel del lenguaje para almacenar datos.
    Usan arrays de bytes, off-heap memory, o formatos binarios.
    Entender POR QUÉ lo hacen es entender este capítulo.
```

**Preguntas:**

1. ¿Spark Tungsten es esencialmente "evitar la JVM para datos"?

2. ¿Kafka usando mmap es una forma de evitar el heap de Java?

3. ¿Si Java tuviera value types (Valhalla), ¿Spark necesitaría Tungsten?

4. ¿Arrow como formato in-memory es la solución universal
   al problema de los object headers?

5. ¿Un data engineer necesita saber sobre cache lines
   para usar Spark correctamente? ¿O solo para entender por qué
   ciertas configuraciones son más rápidas?

---

### Ejercicio 1.7.4 — Implementar: medir cache misses con perf

**Tipo: Implementar**

```bash
# Linux: usar perf stat para medir cache misses reales del hardware
# Requiere: Linux, perf instalado, permisos (o perf_event_paranoid=1)

# Compilar los benchmarks:
# Java: javac ArrayVsLinkedBenchmark.java
# Rust: cargo build --release

# Medir cache misses del benchmark de Java:
perf stat -e cache-references,cache-misses,L1-dcache-loads,L1-dcache-load-misses \
    java ArrayVsLinkedBenchmark

# Medir cache misses del benchmark de Rust:
perf stat -e cache-references,cache-misses,L1-dcache-loads,L1-dcache-load-misses \
    ./target/release/benchmark

# Ejemplo de output para Vec (secuencial):
#   100,000,000  L1-dcache-loads
#     2,500,000  L1-dcache-load-misses   (2.5% de todos los loads)

# Ejemplo de output para LinkedList:
#   100,000,000  L1-dcache-loads
#    45,000,000  L1-dcache-load-misses   (45% de todos los loads)

# La diferencia: 2.5% vs 45% de cache misses.
# Eso es lo que explica la diferencia de rendimiento.
```

**Preguntas:**

1. ¿2.5% de cache misses para un array es normal o alto?

2. ¿45% de cache misses para una linked list es bajo o alto?

3. ¿`perf` funciona dentro de un container Docker?

4. ¿Hay una forma de medir cache misses en macOS?

5. ¿Si ejecutas este benchmark en una instancia cloud (AWS EC2),
   ¿los resultados serían comparables a bare metal?

---

### Ejercicio 1.7.5 — Resumen: las reglas del juego

**Tipo: Leer**

```
Reglas para todo el resto de este libro:

  Regla 1: La memoria no es plana.
    Hay una jerarquía (L1 → L2 → L3 → RAM) con 100× de diferencia
    entre el nivel más rápido y el más lento.
    Las estructuras de datos que respetan esta jerarquía son rápidas.

  Regla 2: Los arrays son la estructura de datos más importante.
    Datos contiguos → cache-friendly → rápidos.
    Todas las demás estructuras se miden contra un array.

  Regla 3: Los punteros son costosos.
    Cada puntero es un posible cache miss.
    Minimizar punteros = minimizar cache misses.
    Esto explica por qué open addressing > chaining,
    B-Tree > Red-Black tree, y Vec > LinkedList.

  Regla 4: Los object headers son impuesto.
    En Java/Scala: 16 bytes por objeto, siempre.
    En Go: 0 bytes por struct.
    En Rust: 0 bytes por struct.
    Para millones de objetos, este impuesto domina la memoria.

  Regla 5: El GC no es gratis.
    Más objetos en el heap = más trabajo para el GC.
    Menos punteros = menos trabajo para el GC.
    Los sistemas de alto rendimiento evitan el heap.

  Regla 6: Big-O es necesario pero insuficiente.
    La complejidad algorítmica ignora las constantes,
    las cache lines, el prefetcher, la vectorización, y el GC.
    Mide siempre. Confía en los números, no en la teoría.

  Regla 7: El lenguaje importa, pero menos de lo que crees.
    La diferencia entre un Vec de Rust y un ArrayList de Java
    es ~2-3× para operaciones simples.
    La diferencia entre un array y una linked list
    es 10-50× en cualquier lenguaje.
    La elección de estructura importa más que la elección de lenguaje.

Estas reglas son la base para los próximos 17 capítulos.
Cada estructura de datos que implementemos se evaluará
contra estas reglas. No contra la tabla de Big-O del textbook.
```

**Preguntas:**

1. ¿La Regla 7 contradice las benchmarks de la Sección 1.6
   donde Python era 5× más lento que Rust?

2. ¿Las reglas aplican igual para datos que viven en disco (SSD)
   que para datos en RAM?

3. ¿Si solo pudieras recordar una regla de las siete, cuál sería?

4. ¿Estas reglas cambian con hardware nuevo
   (memoria HBM, CXL, persistent memory)?

5. ¿Cuál de las siete reglas es la más contraintuitiva
   para alguien que aprendió estructuras de datos
   en un curso universitario tradicional?
