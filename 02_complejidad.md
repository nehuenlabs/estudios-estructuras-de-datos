# Guía de Ejercicios — Cap.02: Complejidad: Más Allá de Big-O

> El capítulo anterior demostró que recorrer un array es 10-50× más rápido
> que recorrer una linked list, a pesar de que ambos son O(n).
>
> Eso debería hacerte desconfiar de Big-O.
>
> Big-O es una herramienta imprescindible — te dice cómo escala
> un algoritmo cuando n crece. Pero no te dice si tu código
> es rápido o lento en tu hardware, con tus datos, hoy.
>
> Big-O ignora las constantes.
> Big-O ignora las cache lines.
> Big-O ignora el garbage collector.
> Big-O ignora la auto-vectorización.
> Big-O ignora el tamaño del dataset real.
>
> Este capítulo enseña dos habilidades:
> 1. Usar Big-O correctamente (saber qué dice y qué NO dice).
> 2. Medir correctamente (benchmarking sin engañarte a ti mismo).
>
> Después de este capítulo, no vas a decir
> "este algoritmo es O(n log n), así que es rápido".
> Vas a decir "este algoritmo es O(n log n),
> y mi benchmark muestra que procesa 10M elementos en 340 ms
> con 2.1% de cache misses en este hardware".

---

## El modelo mental: Big-O es un mapa, no el territorio

```
Big-O describe el CRECIMIENTO de una función,
no su VALOR en un punto específico.

  O(1)       → "no crece con n"
  O(log n)   → "crece lentamente"
  O(n)       → "crece linealmente"
  O(n log n) → "crece un poco más que linealmente"
  O(n²)      → "crece rápido"
  O(2ⁿ)      → "explota"

  Pero O(1) puede significar 1 nanosegundo o 1 segundo.
  Y O(n) con constante pequeña puede ser más rápido que O(1) con constante grande.

  Ejemplo concreto:

    Algoritmo A: f(n) = 1,000,000  (constante — O(1))
    Algoritmo B: f(n) = 10n        (lineal — O(n))

    Para n = 1:       A = 1,000,000   B = 10        → B gana
    Para n = 1,000:   A = 1,000,000   B = 10,000    → B gana
    Para n = 100,000: A = 1,000,000   B = 1,000,000 → empate
    Para n = 1,000,000: A = 1,000,000 B = 10,000,000 → A gana

    El crossover está en n = 100,000.
    Para datasets menores a 100K, el algoritmo O(n) es más rápido
    que el algoritmo O(1).

  Esto no es un caso artificial. Sucede en la práctica:

    Hash map lookup: O(1), pero involucra hash function + posible collision
    Binary search en sorted array: O(log n), pero excelente cache locality
    Para arrays pequeños (<100 elementos), binary search
    puede ser más rápido que hash map lookup.
```

```
Las tres notaciones que importan:

  Big-O (O):  cota SUPERIOR — "como máximo crece así"
              f(n) = O(g(n)) significa f(n) ≤ c·g(n) para n grande
              Es lo que usamos el 99% del tiempo.

  Big-Ω (Omega):  cota INFERIOR — "como mínimo crece así"
                   f(n) = Ω(g(n)) significa f(n) ≥ c·g(n) para n grande
                   "Este algoritmo NO puede ser más rápido que Ω(n log n)"

  Big-Θ (Theta):  cota EXACTA — "crece exactamente así"
                   f(n) = Θ(g(n)) significa f(n) está acotada por arriba
                   y por abajo por g(n)
                   Es lo más preciso, pero rara vez lo usamos
                   porque es más difícil de demostrar.

  En la práctica:
  - Cuando decimos "sort es O(n log n)" queremos decir Θ(n log n).
  - Cuando decimos "hash lookup es O(1)" queremos decir O(1) amortizado
    en el caso promedio, pero O(n) en el peor caso.
  - Big-O es el estándar de la industria, pero es impreciso
    porque no distingue caso promedio de peor caso.
```

---

## Tabla de contenidos

- [Sección 2.1 — Lo que Big-O dice y lo que no dice](#sección-21--lo-que-big-o-dice-y-lo-que-no-dice)
- [Sección 2.2 — Constantes ocultas: cuando O(n) vence a O(1)](#sección-22--constantes-ocultas-cuando-on-vence-a-o1)
- [Sección 2.3 — Complejidad amortizada: el truco del dynamic array](#sección-23--complejidad-amortizada-el-truco-del-dynamic-array)
- [Sección 2.4 — Complejidad en espacio: la memoria que nadie cuenta](#sección-24--complejidad-en-espacio-la-memoria-que-nadie-cuenta)
- [Sección 2.5 — Benchmarking correcto: cómo no engañarte](#sección-25--benchmarking-correcto-cómo-no-engañarte)
- [Sección 2.6 — Los frameworks de benchmarking: JMH, criterion, y más](#sección-26--los-frameworks-de-benchmarking-jmh-criterion-y-más)
- [Sección 2.7 — La tabla de Big-O vs la realidad: el benchmark final](#sección-27--la-tabla-de-big-o-vs-la-realidad-el-benchmark-final)

---

## Sección 2.1 — Lo que Big-O Dice y lo que No Dice

### Ejercicio 2.1.1 — Leer: las trampas de Big-O

**Tipo: Leer**

```
Big-O es una herramienta matemática que describe
el comportamiento asintótico de una función.
Asintótico significa "cuando n tiende a infinito".

Esto implica limitaciones concretas:

  TRAMPA 1: Big-O ignora constantes
    O(n) con constante 1,000 es 1,000× más lento que O(n) con constante 1.
    Pero ambos son "O(n)".
    En la práctica: una suma de array es O(n).
    Una linked list traversal es O(n).
    Pero la suma de array es 10-50× más rápida (Cap.01).

  TRAMPA 2: Big-O ignora n pequeños
    Para n = 10, O(n²) ejecuta 100 operaciones.
    O(n log n) ejecuta ~33 operaciones.
    La diferencia es 67 operaciones.
    Si cada operación tarda 1 ns, la diferencia es 67 ns.
    El overhead de llamar a una función en Java es ~5 ns.
    → Para n = 10, la diferencia algorítmica es irrelevante.

  TRAMPA 3: Big-O no distingue operaciones
    O(n) comparaciones + O(n) cache misses + O(n) asignaciones de heap
    vs
    O(n) comparaciones + 0 cache misses + 0 asignaciones de heap
    Ambos son "O(n)". Pero el segundo es 10× más rápido.

  TRAMPA 4: Big-O no distingue caso promedio de peor caso
    HashMap.get() es O(1) amortizado en el caso promedio.
    En el peor caso (todas las claves colisionan) es O(n).
    Decir "HashMap es O(1)" sin calificar es incompleto.

  TRAMPA 5: Big-O no considera el hardware
    Un algoritmo diseñado en los 70s para cintas magnéticas
    puede tener Big-O óptimo pero rendimiento terrible en SSDs.
    B-Trees fueron diseñados para minimizar accesos a disco.
    En RAM, un sorted array + binary search puede ser más rápido
    a pesar de tener la misma O(log n).
```

**Preguntas:**

1. ¿Insertion sort es O(n²) y merge sort es O(n log n).
   ¿Hay algún valor de n donde insertion sort sea más rápido?

2. ¿Decir "HashMap es O(1)" es correcto o incorrecto?

3. ¿Big-O puede comparar dos algoritmos
   que operan sobre hardware distinto?

4. ¿Existe un algoritmo de sorting mejor que O(n log n)?
   (Pista: comparison-based vs non-comparison-based)

5. ¿Si Big-O tiene tantas limitaciones,
   ¿por qué seguimos usándolo?

---

### Ejercicio 2.1.2 — Analizar: la tabla clásica de Big-O — cuánto miente

**Tipo: Analizar**

```
La tabla que aparece en todos los libros de texto:

  Estructura       Search    Insert    Delete    Notas
  ──────────       ──────    ──────    ──────    ─────
  Array (sorted)   O(log n)  O(n)      O(n)      binary search
  Array (unsorted) O(n)      O(1)*     O(n)      *amortized, append
  Linked List      O(n)      O(1)*     O(1)*     *dado el puntero al nodo
  Hash Table       O(1)*     O(1)*     O(1)*     *amortized, average case
  BST (balanced)   O(log n)  O(log n)  O(log n)  AVL, Red-Black
  B-Tree           O(log n)  O(log n)  O(log n)  branching factor B

  Lo que esta tabla NO te dice:

  1. "Insert O(1) en Linked List"
     Es O(1) si ya tienes el puntero al nodo anterior.
     Si necesitas buscar la posición primero, es O(n) búsqueda + O(1) insert = O(n).
     La tabla omite el costo de ENCONTRAR dónde insertar.

  2. "Search O(1) en Hash Table"
     Es O(1) amortizado en el caso promedio con un buen hash function.
     Peor caso: O(n) si todas las claves colisionan.
     Caso real: O(1) con 2-3 probes en promedio (load factor 0.75).
     Pero cada probe puede ser un cache miss (5-100 ns).
     → O(1) no significa "instantáneo".

  3. "Search O(log n) en BST"
     Un BST balanceado con 1M elementos: log₂(1M) ≈ 20 comparaciones.
     Cada comparación sigue un puntero → posible cache miss.
     20 cache misses × 100 ns = 2,000 ns por búsqueda.
     Un sorted array con binary search: mismas 20 comparaciones,
     pero los datos están contiguos → muchos menos cache misses.
     → Misma O(log n), rendimiento distinto.

  4. "Insert O(n) en Sorted Array"
     Insertar requiere mover todos los elementos posteriores.
     Pero System.arraycopy() / memmove() es extremadamente optimizado
     (usa instrucciones SIMD, copia 64+ bytes por ciclo).
     Para n = 10,000, mover 40 KB tarda ~5 μs.
     Para n = 10,000, insertar en un Red-Black tree: ~20 comparaciones
     × 100 ns (cache misses) = ~2 μs... pero con GC overhead.
     → El O(n) del array puede ser comparable al O(log n) del árbol
       para datasets de tamaño mediano.

  5. "O(log n) en B-Tree"
     Un B-Tree con branching factor 1,000 y 1 billón de claves:
     log₁₀₀₀(1,000,000,000) = 3 accesos.
     Un Red-Black tree con las mismas claves:
     log₂(1,000,000,000) ≈ 30 accesos.
     Ambos son "O(log n)" pero con bases distintas.
     → La base del logaritmo importa en la práctica.
```

**Preguntas:**

1. ¿La tabla debería incluir una columna de "constante típica"
   o de "cache misses por operación"?

2. ¿Red-Black tree alguna vez supera a un sorted array + binary search
   para búsqueda en RAM?

3. ¿La nota "amortized" en hash table es importante?
   ¿Cuándo el peor caso O(n) se manifiesta?

4. ¿Un B-Tree con branching factor 2 es idéntico a un BST?

5. ¿Si reescribieras esta tabla de forma honesta,
   ¿qué columnas agregarías?

---

### Ejercicio 2.1.3 — Leer: caso promedio, peor caso, y caso amortizado

**Tipo: Leer**

```
Tres conceptos que Big-O mezcla y que necesitas separar:

  PEOR CASO (worst-case):
    El máximo tiempo que puede tomar la operación
    para CUALQUIER input posible.

    QuickSort: peor caso O(n²) (si el pivot es siempre el mínimo/máximo).
    HashMap.get(): peor caso O(n) (todas las claves en un bucket).
    Binary search: peor caso O(log n) (siempre lo mismo).

    El peor caso es una garantía: "nunca será peor que esto".
    Es útil para sistemas real-time donde necesitas predecibilidad.

  CASO PROMEDIO (average-case):
    El tiempo esperado sobre TODOS los inputs posibles,
    asumiendo una distribución uniforme (o especificada).

    QuickSort: caso promedio O(n log n) (con pivot aleatorio).
    HashMap.get(): caso promedio O(1) (con buen hash function).
    Linear search: caso promedio O(n/2) = O(n) (elemento en posición aleatoria).

    El caso promedio es más realista pero depende de asunciones
    sobre la distribución de los inputs.
    Si tus datos no son uniformes, el caso promedio no aplica.

  CASO AMORTIZADO (amortized):
    El costo promedio por operación sobre una SECUENCIA de operaciones,
    incluso si una operación individual es costosa.

    ArrayList.add(): la mayoría de los adds son O(1).
    Pero cuando el array está lleno, se duplica la capacidad → O(n).
    Si haces n adds: (n-1) adds son O(1) + 1 resize es O(n).
    Costo total: O(n) + O(n) = O(2n) = O(n).
    Costo amortizado por add: O(n) / n = O(1).

    → "O(1) amortizado" significa que en promedio cada operación
      es O(1), pero ocasionalmente una operación es O(n).

    CUIDADO: amortizado NO es lo mismo que caso promedio.
    - Caso promedio: depende de la distribución del input.
    - Amortizado: se cumple para CUALQUIER secuencia de operaciones.
    - Amortizado es una garantía más fuerte que caso promedio.

    El análisis amortizado es determinístico.
    No asume nada sobre el input — solo sobre la secuencia de operaciones.
```

**Preguntas:**

1. ¿QuickSort con pivot aleatorio: su O(n log n) es caso promedio
   o amortizado?

2. ¿HashMap.get() es O(1) amortizado o caso promedio?

3. ¿Un sistema real-time (ej: ABS de un auto) debería basar
   sus decisiones en peor caso, caso promedio, o amortizado?

4. ¿El análisis amortizado es útil para sistemas de streaming
   donde la latencia de cada evento importa?

5. ¿Existe una estructura con O(1) amortizado para insert
   pero O(n) para una sola inserción sin ser un dynamic array?

---

### Ejercicio 2.1.4 — Analizar: Big-O para operaciones de I/O

**Tipo: Analizar**

```
Big-O fue diseñado para contar OPERACIONES en RAM.
Cuando los datos están en disco o red, las reglas cambian.

  Modelo de computación clásico (RAM model):
    Cada operación (comparación, asignación, aritmética) cuesta O(1).
    No distingue entre acceder a arr[0] y arr[1000000].
    → Asume que la memoria es plana y todo acceso es igual.

  Modelo de I/O (external memory model):
    El costo dominante es el número de TRANSFERS entre disco y RAM.
    Una transferencia mueve un BLOQUE de B elementos.
    El costo de leer n elementos secuencialmente: O(n/B) transfers.
    El costo de leer n elementos aleatoriamente: O(n) transfers.

    Sorting externo:
      En RAM model: O(n log n) comparaciones.
      En I/O model: O((n/B) × log_{M/B}(n/B)) transfers,
        donde M es el tamaño de RAM y B es el tamaño de bloque.
      → La complejidad de I/O depende del hardware (M y B).

  Esto explica decisiones de diseño:

    B-Tree: cada nodo es del tamaño de un bloque de disco (4-16 KB).
    → Minimiza el número de I/O transfers.
    → Un B-Tree con branching factor 1000 y 1G claves:
       3 I/Os para un lookup. Un BST: 30 I/Os.
       Misma O(log n), pero 10× menos I/Os.

    LSM-Tree: las escrituras son secuenciales (append-only).
    → Escritura secuencial en disco es 100× más rápida que aleatoria.
    → Un LSM-Tree convierte escrituras aleatorias en secuenciales
       al costo de lecturas más lentas (compaction).

    Parquet: formato columnar con row groups de 128 MB.
    → Leer una columna de un row group = 1 I/O secuencial.
    → Leer una columna de un row store = n I/Os aleatorios.

  → Para datos en disco, Big-O en el modelo de RAM
    puede ser completamente irrelevante.
    Lo que importa es: ¿cuántos I/Os se hacen?
    ¿Son secuenciales o aleatorios?
```

**Preguntas:**

1. ¿Un sorted array en disco tiene el mismo rendimiento
   de binary search que en RAM?

2. ¿Un hash map en disco necesita cuántos I/Os por lookup?

3. ¿El modelo de I/O explica por qué Cassandra usa LSM-Trees
   en vez de B-Trees?

4. ¿Parquet con columnar layout tiene mejor O(n/B)
   que un row store para un full scan de una columna?

5. ¿En la era de NVMe SSDs donde la latencia de acceso aleatorio
   es ~10 μs (no 10 ms como HDDs), ¿el modelo de I/O sigue importando?

---

### Ejercicio 2.1.5 — Analizar: recalculando Big-O con constantes reales

**Tipo: Analizar**

```
Asignar costos reales a cada operación
convierte Big-O en predicciones de tiempo.

  Costos típicos (hardware moderno, circa 2024):

    Operación                  Costo aproximado
    ─────────────────────────  ────────────────
    Suma/comparación           0.3 ns
    L1 cache hit               1 ns
    L2 cache hit               4 ns
    L3 cache hit               10 ns
    RAM access (cache miss)    100 ns
    Branch misprediction       5-15 ns
    JVM method call            2-5 ns
    Python function call       50-100 ns
    malloc (small)             20-50 ns
    Java new Object()          10-20 ns
    Rust Box::new              10-20 ns
    Go heap alloc (escape)     20-40 ns
    SSD random read (4KB)      10,000 ns (10 μs)
    SSD sequential read (1MB)  200,000 ns (200 μs)
    HDD random read            10,000,000 ns (10 ms)
    Network round-trip (LAN)   500,000 ns (0.5 ms)

  Ahora calculemos con datos reales:

  CASO 1: buscar en un sorted array de 1M ints (4 MB)
    Binary search: log₂(1M) = 20 comparaciones.
    Primeros accesos: cache miss (100 ns). Últimos: cache hit (4 ns).
    Estimación: ~5 cache misses × 100 ns + 15 cache hits × 4 ns
              = 500 + 60 = 560 ns.

  CASO 2: buscar en un Red-Black tree de 1M nodos
    Depth: log₂(1M) = 20 comparaciones.
    Cada nodo está en una dirección aleatoria del heap.
    Estimación: ~15 cache misses × 100 ns + 5 cache hits × 10 ns
              = 1500 + 50 = 1,550 ns.

  CASO 3: buscar en un HashMap de 1M entries (Java)
    hashCode(): ~10 ns
    Array lookup (bucket): 1 L1/L2 hit: ~4 ns
    1-2 comparaciones (chaining, load factor 0.75): ~200 ns (cache misses)
    Estimación: ~214 ns

  CASO 4: buscar en un B-Tree con branching factor 512 y 1M claves
    Depth: log₅₁₂(1M) ≈ 3.3 → 4 nodos visitados.
    Cada nodo: binary search dentro del nodo (log₂(512) = 9 comparaciones).
    Nodo cabe en ~10 cache lines → 2-3 cache misses por nodo.
    Estimación: 4 nodos × (9 comparaciones × 0.3 ns + 3 misses × 100 ns)
              = 4 × (2.7 + 300) = ~1,211 ns.

  Resumen:
    HashMap:           ~214 ns    (O(1))
    Sorted array:      ~560 ns    (O(log n))
    B-Tree:            ~1,211 ns  (O(log n))
    Red-Black tree:    ~1,550 ns  (O(log n))

  Big-O dice: HashMap > sorted array = B-Tree = Red-Black tree.
  Realidad:   HashMap > sorted array > B-Tree > Red-Black tree.
  Las constantes reordenan los resultados DENTRO de la misma clase.
```

**Preguntas:**

1. ¿El B-Tree es 2× más lento que el sorted array a pesar de que
   ambos son O(log n). ¿El B-Tree se justifica en RAM?

2. ¿Para 10 elementos, ¿el HashMap sigue siendo más rápido
   que binary search?

3. ¿Si los nodos del Red-Black tree se asignaran contiguos en memoria
   (arena allocator), ¿cerraría la brecha con el sorted array?

4. ¿Estas estimaciones son suficientemente precisas
   para tomar decisiones de diseño?

5. ¿Un B-Tree en disco con cache de nodos internos en RAM
   tendría cuántos I/Os para 1M claves?

---

## Sección 2.2 — Constantes Ocultas: Cuando O(n) Vence a O(1)

### Ejercicio 2.2.1 — Leer: los factores que Big-O esconde

**Tipo: Leer**

```
Cuando decimos f(n) = O(g(n)), estamos diciendo:
"Existe una constante c tal que f(n) ≤ c × g(n) para n suficientemente grande."

Esa constante c contiene TODO lo que Big-O esconde:

  1. COSTO POR OPERACIÓN
     Un hash function que hace 50 operaciones aritméticas
     vs uno que hace 5.
     Ambos son O(1). La constante difiere 10×.

  2. CACHE BEHAVIOR
     Linear scan de un array: cada cache miss carga 16 ints.
     Costo real por elemento: ~7 ns (amortizado por cache lines).
     Linear scan de una linked list: cada nodo es un cache miss.
     Costo real por nodo: ~50-100 ns.
     Ambos son O(n). La constante difiere 7-14×.

  3. BRANCH PREDICTION
     Un if/else que el CPU predice correctamente: ~0 ns extra.
     Un if/else impredecible (50/50): ~5-15 ns extra.
     Sorting: el inner loop de quicksort tiene branches impredecibles.
     Branchless algorithms eliminan esto al costo de más instrucciones.

  4. SIMD / VECTORIZACIÓN
     Sumar un array de ints:
     - Escalar: 1 suma por ciclo.
     - SIMD (AVX-256): 8 sumas por ciclo.
     El compilador puede auto-vectorizar el loop.
     Mismo O(n), pero 8× más rápido.
     → Vec<i64>::iter().sum() en Rust se auto-vectoriza.
     → LinkedList no se puede vectorizar (datos no contiguos).

  5. OVERHEAD DEL LENGUAJE
     Python function call: ~100 ns.
     Java method call: ~5 ns.
     Rust function call: ~0.3 ns (inline).
     Un algoritmo O(n) que llama una función por elemento:
     Python: O(n × 100 ns). Rust: O(n × 0.3 ns). Ratio: 300×.

  Estas constantes NO son ruido académico.
  Son la razón por la cual un programa "optimizado" con el mejor Big-O
  puede ser más lento que uno "naive" con mejor localidad de cache.
```

**Preguntas:**

1. ¿La auto-vectorización es una optimización del compilador o del hardware?

2. ¿Los branches impredecibles afectan más a los pipelines
   superescalares modernos que a los procesadores antiguos?

3. ¿Un sorting algorithm sin branches (branchless) es siempre
   más rápido que uno con branches?

4. ¿Python con NumPy elimina el overhead de 100 ns por operación?
   ¿Cómo?

5. ¿Es posible que un O(n²) con SIMD sea más rápido
   que un O(n log n) sin SIMD para n < 10,000?

---

### Ejercicio 2.2.2 — Implementar: linear search vs hash lookup para n pequeño

**Tipo: Implementar**

```java
// Demostrar que linear search en un array pequeño
// puede ser más rápido que HashMap.get()

import java.util.*;

public class SmallNBenchmark {

    static int linearSearch(int[] arr, int target) {
        for (int i = 0; i < arr.length; i++) {
            if (arr[i] == target) return i;
        }
        return -1;
    }

    public static void main(String[] args) {
        int[] sizes = {4, 8, 16, 32, 64, 128, 256, 512, 1024, 4096};
        var rng = new Random(42);
        int lookups = 10_000_000;

        System.out.printf("%-8s  %-14s  %-14s  %-10s%n",
            "Size", "Linear (ns)", "HashMap (ns)", "Winner");
        System.out.println("─".repeat(56));

        for (int size : sizes) {
            // Crear array y HashMap con los mismos datos
            int[] arr = new int[size];
            HashMap<Integer, Integer> map = new HashMap<>(size * 2);
            for (int i = 0; i < size; i++) {
                arr[i] = i;
                map.put(i, i);
            }

            // Generar targets aleatorios (dentro del rango)
            int[] targets = new int[lookups];
            for (int i = 0; i < lookups; i++) {
                targets[i] = rng.nextInt(size);
            }

            // Warmup
            for (int t : targets) { linearSearch(arr, t); map.get(t); }

            // Benchmark linear search
            long start = System.nanoTime();
            long dummy = 0;
            for (int t : targets) dummy += linearSearch(arr, t);
            long linearTime = System.nanoTime() - start;

            // Benchmark HashMap
            start = System.nanoTime();
            for (int t : targets) dummy += map.get(t);
            long hashTime = System.nanoTime() - start;

            double linearNs = (double) linearTime / lookups;
            double hashNs = (double) hashTime / lookups;
            String winner = linearNs < hashNs ? "LINEAR" : "HASH";

            System.out.printf("%-8d  %-14.1f  %-14.1f  %-10s%n",
                size, linearNs, hashNs, winner);
        }
    }
}
// Resultado esperado (varía según hardware):
//
// Size      Linear (ns)     HashMap (ns)    Winner
// ────────────────────────────────────────────────────
// 4         3.5             25.0            LINEAR
// 8         5.0             25.0            LINEAR
// 16        8.0             26.0            LINEAR
// 32        14.0            27.0            LINEAR
// 64        28.0            28.0            EMPATE
// 128       55.0            29.0            HASH
// 256       110.0           30.0            HASH
// 512       220.0           31.0            HASH
// 1024      440.0           33.0            HASH
// 4096      1750.0          38.0            HASH
//
// Crossover: ~50-80 elementos.
// Para arrays pequeños, O(n) linear search es más rápido que O(1) HashMap.
// Razón: linear search es cache-friendly y no tiene overhead de hashing.
// HashMap tiene overhead fijo: hashCode(), modulo, posible chaining.
```

**Preguntas:**

1. ¿El crossover de ~64 elementos es estable entre lenguajes?

2. ¿Si usamos `int[]` para keys (sin boxing), ¿el crossover cambia?

3. ¿Rust `HashMap` tiene un crossover diferente que Java `HashMap`?

4. ¿Databases usan linear search para tablas muy pequeñas
   en vez de B-Tree? (Pista: PostgreSQL sequential scan vs index scan)

5. ¿Un Bloom filter con k=3 tiene overhead comparable
   al de un HashMap.get()?

---

### Ejercicio 2.2.3 — Implementar: insertion sort vs merge sort para n pequeño

**Tipo: Implementar**

```python
# Insertion sort es O(n²). Merge sort es O(n log n).
# ¿Para qué valor de n merge sort empieza a ganar?

import time
import random
import sys

sys.setrecursionlimit(100_000)

def insertion_sort(arr):
    for i in range(1, len(arr)):
        key = arr[i]
        j = i - 1
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key

def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result

def benchmark(sort_func, data, runs=50):
    total = 0
    for _ in range(runs):
        copy = data[:]
        start = time.perf_counter_ns()
        sort_func(copy)
        total += time.perf_counter_ns() - start
    return total / runs

sizes = [5, 10, 20, 50, 100, 200, 500, 1000, 2000, 5000]
print(f"{'Size':<8}  {'Insertion (μs)':<16}  {'Merge (μs)':<16}  {'Winner':<10}")
print("─" * 56)

for n in sizes:
    data = list(range(n))
    random.shuffle(data)

    ins_ns = benchmark(insertion_sort, data)
    merge_ns = benchmark(merge_sort, data)

    ins_us = ins_ns / 1000
    merge_us = merge_ns / 1000
    winner = "INSERTION" if ins_ns < merge_ns else "MERGE"

    print(f"{n:<8}  {ins_us:<16.1f}  {merge_us:<16.1f}  {winner:<10}")

# Resultado esperado:
# Size      Insertion (μs)    Merge (μs)        Winner
# ────────────────────────────────────────────────────────
# 5         0.5               2.5               INSERTION
# 10        1.2               6.0               INSERTION
# 20        3.5               14.0              INSERTION
# 50        18.0              42.0              INSERTION
# 100       65.0              95.0              INSERTION
# 200       250.0             210.0             MERGE
# 500       1500.0            570.0             MERGE
# 1000      5800.0            1250.0            MERGE
# 2000      23000.0           2700.0            MERGE
# 5000      145000.0          7500.0            MERGE
#
# Crossover en Python: ~100-200 elementos.
# En Java/Rust (sin overhead de intérprete): ~20-50 elementos.
```

**Preguntas:**

1. ¿Java Arrays.sort() usa insertion sort para arrays pequeños?
   ¿Con qué threshold?

2. ¿Timsort (Python default) combina insertion sort y merge sort?
   ¿Cómo?

3. ¿El crossover depende del patrón de los datos
   (casi ordenados vs completamente aleatorios)?

4. ¿Insertion sort en un array casi ordenado es O(n).
   ¿Eso cambia el crossover?

5. ¿Si implementaras este benchmark en Rust,
   ¿el crossover se movería a un n más pequeño?

---

### Ejercicio 2.2.4 — Analizar: branch prediction y por qué importa

**Tipo: Analizar**

```
El CPU moderno tiene un pipeline de 15-25 etapas.
Cuando encuentra un if/else, tiene dos opciones:
  1. Esperar a saber el resultado (perder 15-25 ciclos).
  2. PREDECIR el resultado y ejecutar especulativamente.
     Si acierta: 0 ciclos perdidos.
     Si falla: 15-25 ciclos perdidos (flush del pipeline).

El predictor de branches del CPU es sofisticado:
  - Patrones simples (siempre true, siempre false): 99%+ acierto.
  - Patrones repetitivos (true, true, false, repeat): 95%+ acierto.
  - Patrones aleatorios (50/50): ~50% acierto → 50% misprediction.

Impacto en estructuras de datos:

  Binary search: cada comparación es un branch.
    Comparación con el middle: ¿ir a la izquierda o a la derecha?
    El branch es impredecible (50/50 para datos uniformes).
    20 comparaciones × 50% misprediction × 15 ciclos = ~150 ciclos extra.
    A 3 GHz: ~50 ns de overhead por búsqueda solo en mispredictions.

  Linear search: el branch "¿encontré el elemento?" es predecible.
    El predictor asume "no encontrado" (true la mayoría del tiempo).
    Para un array de 100 elementos buscando la posición 50:
    49 predicciones correctas ("no encontrado") + 1 misprediction ("encontrado").
    Overhead: ~5 ns (1 misprediction).

  Sorting: quicksort particiona alrededor de un pivot.
    El branch "¿elemento < pivot?" es impredecible para datos aleatorios.
    Cada comparación en la partición: ~50% misprediction.
    → Branchless partition (CMOV, conditional move): elimina el problema.
    → pdqsort (Rust default) usa branchless partition.

  Implicación: para n < 100, el overhead de branch misprediction
  en binary search puede superar el costo de linear scan.
  Linear search tiene branches predecibles + mejor cache locality.
```

**Preguntas:**

1. ¿El branch predictor puede aprender el patrón de acceso
   a un hash map?

2. ¿Branchless programming es siempre más rápido?

3. ¿Un if que se ejecuta 99% como true y 1% como false
   es un problema para el branch predictor?

4. ¿`pdqsort` de Rust es más rápido que `Arrays.sort()` de Java
   por la partición branchless?

5. ¿En Python, ¿las branch mispredictions del CPU
   son relevantes o el overhead del intérprete las eclipsa?

---

### Ejercicio 2.2.5 — Implementar: demostrar el impacto de branch prediction

**Tipo: Implementar**

```java
// Filtrar elementos de un array: con branches vs sorted
import java.util.Arrays;
import java.util.Random;

public class BranchPredictionDemo {

    // Sumar solo los elementos mayores a un threshold
    static long sumAboveThreshold(int[] data, int threshold) {
        long total = 0;
        for (int val : data) {
            if (val > threshold) { // branch: ¿predecible o no?
                total += val;
            }
        }
        return total;
    }

    public static void main(String[] args) {
        int n = 10_000_000;
        int[] data = new int[n];
        var rng = new Random(42);
        for (int i = 0; i < n; i++) data[i] = rng.nextInt(256);

        int threshold = 128; // ~50% de elementos pasan el filtro

        // Caso 1: datos aleatorios (branch impredecible)
        int runs = 20;

        // Warmup
        for (int w = 0; w < 10; w++) sumAboveThreshold(data, threshold);

        long start = System.nanoTime();
        for (int r = 0; r < runs; r++) sumAboveThreshold(data, threshold);
        long unsortedTime = (System.nanoTime() - start) / runs;

        // Caso 2: datos ordenados (branch predecible)
        int[] sorted = data.clone();
        Arrays.sort(sorted);

        // Warmup
        for (int w = 0; w < 10; w++) sumAboveThreshold(sorted, threshold);

        start = System.nanoTime();
        for (int r = 0; r < runs; r++) sumAboveThreshold(sorted, threshold);
        long sortedTime = (System.nanoTime() - start) / runs;

        System.out.printf("Unsorted: %d ms (branches impredecibles)%n",
            unsortedTime / 1_000_000);
        System.out.printf("Sorted:   %d ms (branches predecibles)%n",
            sortedTime / 1_000_000);
        System.out.printf("Ratio:    %.1f×%n",
            (double) unsortedTime / sortedTime);
    }
}
// Resultado esperado:
// Unsorted: ~55 ms (50% branch misprediction)
// Sorted:   ~20 ms (branch predecible: primero todos false, luego todos true)
// Ratio:    ~2.5×
//
// El mismo algoritmo, los mismos datos, la misma O(n).
// Solo cambia el orden → cambia la predicción → cambia 2.5× el tiempo.
```

**Preguntas:**

1. ¿Si el threshold fuera 250 (solo ~2% de elementos pasan),
   ¿la diferencia se reduce?

2. ¿Este efecto aplica a Spark cuando filtra un DataFrame?

3. ¿Una versión branchless (con aritmética en vez de if)
   sería igual de rápida para datos sorted y unsorted?

4. ¿`perf stat -e branch-misses` confirmaría la diferencia?

5. ¿Existe un threshold de "porcentaje de branches true"
   donde la predicción empieza a funcionar bien?

---

## Sección 2.3 — Complejidad Amortizada: el Truco del Dynamic Array

### Ejercicio 2.3.1 — Leer: por qué ArrayList.add() es O(1) si a veces es O(n)

**Tipo: Leer**

```
Un dynamic array (ArrayList, Vec, slice) tiene un truco:
cuando se llena, duplica su capacidad y copia todos los elementos.

  Secuencia de operaciones (capacidad inicial = 1):

    add(a)  → capacidad 1, size 1.   Costo: 1 (solo asignar).
    add(b)  → LLENO. Duplicar a 2.   Costo: 1 (copiar a) + 1 (asignar b) = 2.
    add(c)  → LLENO. Duplicar a 4.   Costo: 2 (copiar a,b) + 1 (asignar c) = 3.
    add(d)  → capacidad 4, size 4.   Costo: 1.
    add(e)  → LLENO. Duplicar a 8.   Costo: 4 (copiar a,b,c,d) + 1 (asignar e) = 5.
    add(f)  → capacidad 8, size 6.   Costo: 1.
    add(g)  → capacidad 8, size 7.   Costo: 1.
    add(h)  → capacidad 8, size 8.   Costo: 1.

  Costos: 1, 2, 3, 1, 5, 1, 1, 1 = 15 operaciones para 8 adds.
  Costo promedio: 15 / 8 = 1.875 por add.

  En general, para n adds:
  Resizes ocurren en: 1, 2, 4, 8, 16, ..., n/2, n
  Costo total de resizes: 1 + 2 + 4 + ... + n = 2n - 1 (serie geométrica)
  Costo total de asignaciones normales: n
  Costo total: 3n - 1 = O(n)
  Costo amortizado por add: O(n) / n = O(1)

  La clave: los resizes costosos son cada vez más INFRECUENTES.
  El resize de n/2 → n ocurre solo una vez.
  Las n/2 operaciones baratas "pagan" por ese resize.
```

**Preguntas:**

1. ¿Si en vez de duplicar, incrementáramos la capacidad en +10,
   ¿el costo amortizado seguiría siendo O(1)?

2. ¿Factor de crecimiento 1.5 (Java ArrayList) vs 2.0 (Rust Vec):
   ¿cuál es mejor?

3. ¿El resize causa un spike de latencia.
   ¿Es un problema para sistemas de baja latencia?

4. ¿Puedes pre-asignar la capacidad si conoces n de antemano?
   ¿Eso elimina todos los resizes?

5. ¿El análisis amortizado garantiza que NINGÚN add individual
   es O(n)? ¿O solo garantiza el promedio?

---

### Ejercicio 2.3.2 — Implementar: dynamic array desde cero con métricas

**Tipo: Implementar**

```java
// Implementar un dynamic array que registra cada resize
public class MiDynamicArray<T> {

    private Object[] data;
    private int size;
    private int resizeCount;
    private long totalCopies;

    public MiDynamicArray() {
        this.data = new Object[1];
        this.size = 0;
        this.resizeCount = 0;
        this.totalCopies = 0;
    }

    public void add(T element) {
        if (size == data.length) {
            resize();
        }
        data[size++] = element;
    }

    private void resize() {
        int newCapacity = data.length * 2; // growth factor = 2
        Object[] newData = new Object[newCapacity];
        System.arraycopy(data, 0, newData, 0, size);
        data = newData;
        resizeCount++;
        totalCopies += size;
    }

    @SuppressWarnings("unchecked")
    public T get(int index) {
        if (index < 0 || index >= size) throw new IndexOutOfBoundsException();
        return (T) data[index];
    }

    public int size() { return size; }
    public int capacity() { return data.length; }
    public int getResizeCount() { return resizeCount; }
    public long getTotalCopies() { return totalCopies; }

    public static void main(String[] args) {
        int[] ns = {100, 1_000, 10_000, 100_000, 1_000_000, 10_000_000};

        System.out.printf("%-12s  %-8s  %-12s  %-14s  %-10s%n",
            "N", "Resizes", "Total copies", "Amort. copies", "Capacity");
        System.out.println("─".repeat(65));

        for (int n : ns) {
            var arr = new MiDynamicArray<Integer>();
            for (int i = 0; i < n; i++) {
                arr.add(i);
            }
            System.out.printf("%-12d  %-8d  %-12d  %-14.2f  %-10d%n",
                n, arr.getResizeCount(), arr.getTotalCopies(),
                (double) arr.getTotalCopies() / n, arr.capacity());
        }
    }
}
// Resultado esperado:
// N             Resizes   Total copies  Amort. copies   Capacity
// ─────────────────────────────────────────────────────────────
// 100           7         127           1.27            128
// 1,000         10        1,023         1.02            1,024
// 10,000        14        16,383        1.64            16,384
// 100,000       17        131,071       1.31            131,072
// 1,000,000     20        1,048,575     1.05            1,048,576
// 10,000,000    24        16,777,215    1.68            16,777,216
//
// Observar: amortized copies/element ≈ 1-2, nunca crece.
// Resizes = log₂(n), crece muy lentamente.
```

**Preguntas:**

1. ¿Cambiar el growth factor a 1.5 reduce las copias totales?

2. ¿La capacidad final siempre es la potencia de 2 más cercana a n?

3. ¿El desperdicio de memoria (capacity - size) es aceptable?

4. ¿Si implementaras shrinking (reducir capacidad cuando size < capacity/4),
   ¿eso afecta el análisis amortizado?

5. ¿Este dynamic array es thread-safe?

---

### Ejercicio 2.3.3 — Implementar: growth factor 1.5 vs 2.0 en Rust

**Tipo: Implementar**

```rust
// Comparar estrategias de crecimiento en un dynamic array
use std::time::Instant;

struct DynArray {
    data: Vec<i64>,    // usamos Vec para la memoria, pero controlamos el growth
    size: usize,
    capacity: usize,
    growth_factor: f64,
    resize_count: usize,
    total_copies: usize,
}

impl DynArray {
    fn new(growth_factor: f64) -> Self {
        DynArray {
            data: Vec::with_capacity(1),
            size: 0,
            capacity: 1,
            growth_factor,
            resize_count: 0,
            total_copies: 0,
        }
    }

    fn push(&mut self, val: i64) {
        if self.size == self.capacity {
            let new_cap = ((self.capacity as f64) * self.growth_factor).ceil() as usize;
            let new_cap = new_cap.max(self.capacity + 1); // al menos +1
            self.data.reserve(new_cap - self.capacity);
            self.capacity = new_cap;
            self.resize_count += 1;
            self.total_copies += self.size;
        }
        if self.size < self.data.len() {
            self.data[self.size] = val;
        } else {
            self.data.push(val);
        }
        self.size += 1;
    }
}

fn benchmark_growth(n: usize, factor: f64) -> (usize, usize, std::time::Duration) {
    let start = Instant::now();
    let mut arr = DynArray::new(factor);
    for i in 0..n {
        arr.push(i as i64);
    }
    let elapsed = start.elapsed();
    (arr.resize_count, arr.total_copies, elapsed)
}

fn main() {
    let n = 10_000_000;
    let factors = [1.25, 1.5, 2.0, 3.0, 4.0];

    println!("N = {}\n", n);
    println!("{:<8}  {:<10}  {:<14}  {:<10}  {:<12}",
        "Factor", "Resizes", "Total copies", "Wasted %", "Time (ms)");
    println!("{}", "─".repeat(60));

    for &factor in &factors {
        let (resizes, copies, elapsed) = benchmark_growth(n, factor);
        // Estimar capacidad final
        let mut cap = 1usize;
        for _ in 0..resizes {
            cap = ((cap as f64) * factor).ceil() as usize;
        }
        let wasted_pct = ((cap - n) as f64 / cap as f64) * 100.0;
        println!("{:<8.2}  {:<10}  {:<14}  {:<10.1}  {:<12}",
            factor, resizes, copies, wasted_pct, elapsed.as_millis());
    }
}
// Resultado esperado:
// Factor    Resizes     Total copies    Wasted %    Time (ms)
// ────────────────────────────────────────────────────────────
// 1.25      75          ~44M            ~20%        ~150
// 1.50      39          ~20M            ~33%        ~80
// 2.00      24          ~10M            ~50%        ~60
// 3.00      15          ~5M             ~66%        ~50
// 4.00      12          ~3M             ~75%        ~45
//
// Tradeoff: mayor growth factor = menos resizes y copias,
//           pero más memoria desperdiciada.
// Factor 1.5 (Java) es conservador en memoria.
// Factor 2.0 (Rust Vec) prioriza rendimiento.
```

**Preguntas:**

1. ¿Factor 1.25 hace 75 resizes vs factor 2.0 con 24. ¿La diferencia
   de copias totales se nota en el tiempo?

2. ¿Por qué Java elige 1.5 y Rust elige 2.0?

3. ¿Factor 3.0 desperdicia 66% de memoria. ¿Cuándo es aceptable?

4. ¿Go slices usan qué factor de crecimiento?

5. ¿Un growth factor de 1.0 (incremento fijo) convierte add
   en O(n) amortizado?

---

### Ejercicio 2.3.4 — Leer: el método del banquero — demostración formal

**Tipo: Leer**

```
El método del banquero es una forma intuitiva de demostrar
costos amortizados. La idea: cada operación "deposita créditos"
que pagan por las operaciones costosas futuras.

  Dynamic array con factor de crecimiento 2:

  Regla: cada add deposita 3 créditos.
    - 1 crédito: paga por la inserción actual.
    - 2 créditos: se guardan para el próximo resize.

  ¿Por qué 2 créditos extra?
  Cuando se hace resize de capacidad N a 2N,
  necesitamos copiar N elementos.
  Esos N elementos fueron insertados desde el último resize.
  Cada uno depositó 2 créditos → tenemos 2N créditos.
  Copiar N elementos cuesta N créditos.
  Tenemos 2N, gastamos N → sobran N créditos.
  El balance nunca es negativo → la estrategia funciona.

  Formalmente:
    Sea aᵢ el costo amortizado de la i-ésima operación.
    Sea cᵢ el costo real de la i-ésima operación.
    Sea Φᵢ el "potencial" (créditos acumulados) después de la i-ésima operación.

    aᵢ = cᵢ + Φᵢ - Φᵢ₋₁

    Para el dynamic array:
    Φ(n) = 2 × size - capacity
    (mide cuántos créditos tenemos guardados)

    Cuando no hay resize: cᵢ = 1, Φ crece en 2 - 0 = 2.
      aᵢ = 1 + 2 = 3.

    Cuando hay resize (size = capacity = N → 2N):
      cᵢ = N + 1 (copiar N + insertar 1)
      Φᵢ - Φᵢ₋₁ = (2(N+1) - 2N) - (2N - N) = (2) - (N) = 2 - N
      aᵢ = (N + 1) + (2 - N) = 3.

    → El costo amortizado es SIEMPRE 3, independiente de si hay resize.
    → O(1) amortizado demostrado.

  ¿Por qué esto importa para este libro?
  Cada vez que decimos "O(1) amortizado" para un dynamic array,
  hash map (rehashing), o splay tree, estamos usando este argumento.
  No es magia — es contabilidad.
```

**Preguntas:**

1. ¿Si el growth factor fuera 1.5, ¿cuántos créditos
   necesita depositar cada add?

2. ¿El método del banquero funciona para CUALQUIER secuencia
   de operaciones o solo para la peor?

3. ¿El método del potencial es una alternativa al del banquero?
   ¿Son equivalentes?

4. ¿Delete de un dynamic array (con shrinking) también es O(1) amortizado?

5. ¿El análisis amortizado de HashMap.put() (con rehashing)
   es idéntico al del dynamic array?

---

### Ejercicio 2.3.5 — Analizar: amortizado vs latencia — el tradeoff real

**Tipo: Analizar**

```
O(1) amortizado NO significa que cada operación es O(1).
Significa que el promedio es O(1) sobre una secuencia.
Una operación individual puede ser O(n).

  Ejemplo: ArrayList con 10M elementos.
  add() normal: ~10 ns.
  add() con resize (copiar 10M elementos): ~30 ms.
  30 ms = 3,000,000× más lento que el add normal.

  Para un web server que responde en <50 ms:
    Un resize de 30 ms durante un request es inaceptable.
    El request se dispara a 80 ms. El usuario percibe latencia.

  Para un batch pipeline que procesa 10M registros:
    Un resize de 30 ms cada 10M inserts es irrelevante.
    30 ms en un pipeline de 30 segundos es 0.1%.

  → Amortizado es perfecto para THROUGHPUT (batch).
  → Amortizado es peligroso para LATENCIA (real-time).

  Soluciones para sistemas de baja latencia:

  1. Pre-allocate: new ArrayList<>(expectedSize)
     Elimina todos los resizes. Requiere conocer el tamaño.

  2. Incremental resize: durante cada add, copiar K elementos
     del viejo array al nuevo. El resize se distribuye en K adds.
     → Cada add es O(1) en el peor caso (no solo amortizado).
     → Más complejo de implementar. Usado en real-time systems.

  3. Ring buffer con tamaño fijo: ArrayBlockingQueue (Java).
     No crece. Si se llena, bloquea o descarta.
     → O(1) siempre. Sin sorpresas.
     → Requiere conocer el tamaño máximo.

  Sistemas reales:
  - Java ConcurrentHashMap: rehash incremental (no para todo de golpe).
  - Redis dict: rehash progresivo (un poco con cada operación).
  - Go map: incremental evacuation during resize.
  - LMAX Disruptor: ring buffer de tamaño fijo (potencia de 2).
```

**Preguntas:**

1. ¿Redis rehash progresivo es O(1) en peor caso o O(1) amortizado?

2. ¿Un sistema de trading puede usar un ArrayList sin pre-allocate?

3. ¿El resize incremental complica el código. ¿Cuándo vale la pena?

4. ¿Kafka usa ring buffers o dynamic arrays internamente?

5. ¿Para un pipeline de Spark, ¿el resize de un array interno
   es relevante comparado con el shuffle?

---

## Sección 2.4 — Complejidad en Espacio: la Memoria que Nadie Cuenta

### Ejercicio 2.4.1 — Leer: espacio no es solo el tamaño de los datos

**Tipo: Leer**

```
La complejidad en espacio describe cuánta MEMORIA EXTRA
usa un algoritmo más allá del input.

  Sorting:
    Merge sort: O(n) espacio extra (necesita un array auxiliar para merge).
    Quick sort: O(log n) espacio extra (solo el stack de recursión).
    Heap sort: O(1) espacio extra (in-place).
    → Si la memoria es limitada, heap sort gana a pesar de ser
      más lento que quick sort por cache locality.

  Búsqueda:
    Binary search iterativo: O(1) espacio extra.
    Binary search recursivo: O(log n) espacio (stack frames).
    → Siempre usa la versión iterativa.

  Estructuras de datos — espacio por elemento:

    Estructura            Datos    Overhead     Total    Overhead %
    ──────────────────    ──────   ─────────    ──────   ──────────
    int[] (Java)          4 B      0            4 B      0%
    Integer[] (Java)      4 B      12 B         16 B     75%
    ArrayList<Integer>    4 B      ~28 B        ~32 B    87%
    HashMap<Int, Int>     8 B      ~72 B        ~80 B    90%
    TreeMap<Int, Int>     8 B      ~40 B        ~48 B    83%
    LinkedList<Integer>   4 B      ~36 B        ~40 B    90%
    Vec<i64> (Rust)       8 B      0            8 B      0%
    HashMap<i64,i64> Rust 16 B     ~34 B        ~50 B    68%
    []int64 (Go)          8 B      0            8 B      0%

  Para 10M elementos:

    int[] (Java):           40 MB   (solo datos)
    HashMap<Int,Int> (Java): 800 MB (90% overhead)
    Vec<i64> (Rust):         80 MB  (solo datos)
    HashMap<i64,i64> (Rust): 500 MB (68% overhead)

  → La elección de estructura puede significar
    40 MB vs 800 MB para los mismos datos.
    20× de diferencia solo por overhead de estructura.
```

**Preguntas:**

1. ¿90% de overhead en un HashMap de Java es aceptable en producción?

2. ¿El overhead de HashMap se reduce con load factors mayores (ej: 0.95)?

3. ¿Eclipse Collections IntIntHashMap reduce el overhead
   a niveles comparables a Rust?

4. ¿El espacio extra de merge sort (O(n)) es un problema real
   para datasets que caben en RAM?

5. ¿En un pipeline de Spark, ¿el overhead de estructuras Java
   es parte del motivo de Tungsten off-heap?

---

### Ejercicio 2.4.2 — Implementar: medir el espacio real de las colecciones Java

**Tipo: Implementar**

```java
// Medir el footprint de memoria de diferentes colecciones Java
// usando Runtime.getRuntime() y forzando GC

public class SpaceBenchmark {

    static long usedMemory() {
        Runtime rt = Runtime.getRuntime();
        // Forzar GC múltiples veces para limpiar
        for (int i = 0; i < 5; i++) {
            System.gc();
            try { Thread.sleep(50); } catch (Exception e) {}
        }
        return rt.totalMemory() - rt.freeMemory();
    }

    public static void main(String[] args) throws Exception {
        int n = 1_000_000;

        System.out.printf("%-30s  %-12s  %-12s%n",
            "Structure", "Memory (MB)", "Bytes/elem");
        System.out.println("─".repeat(60));

        // Baseline
        long baseline = usedMemory();

        // int[]
        int[] intArr = new int[n];
        for (int i = 0; i < n; i++) intArr[i] = i;
        long intArrMem = usedMemory() - baseline;
        System.out.printf("%-30s  %-12.1f  %-12.1f%n",
            "int[]", intArrMem / 1e6, (double) intArrMem / n);
        intArr = null;

        // ArrayList<Integer>
        System.gc(); Thread.sleep(100);
        baseline = usedMemory();
        var arrayList = new java.util.ArrayList<Integer>(n);
        for (int i = 0; i < n; i++) arrayList.add(i + 200); // fuera del cache
        long alMem = usedMemory() - baseline;
        System.out.printf("%-30s  %-12.1f  %-12.1f%n",
            "ArrayList<Integer>", alMem / 1e6, (double) alMem / n);
        arrayList = null;

        // HashMap<Integer, Integer>
        System.gc(); Thread.sleep(100);
        baseline = usedMemory();
        var hashMap = new java.util.HashMap<Integer, Integer>(n * 2);
        for (int i = 0; i < n; i++) hashMap.put(i + 200, i + 300);
        long hmMem = usedMemory() - baseline;
        System.out.printf("%-30s  %-12.1f  %-12.1f%n",
            "HashMap<Integer, Integer>", hmMem / 1e6, (double) hmMem / n);
        hashMap = null;

        // TreeMap<Integer, Integer>
        System.gc(); Thread.sleep(100);
        baseline = usedMemory();
        var treeMap = new java.util.TreeMap<Integer, Integer>();
        for (int i = 0; i < n; i++) treeMap.put(i + 200, i + 300);
        long tmMem = usedMemory() - baseline;
        System.out.printf("%-30s  %-12.1f  %-12.1f%n",
            "TreeMap<Integer, Integer>", tmMem / 1e6, (double) tmMem / n);
        treeMap = null;

        // LinkedList<Integer>
        System.gc(); Thread.sleep(100);
        baseline = usedMemory();
        var linkedList = new java.util.LinkedList<Integer>();
        for (int i = 0; i < n; i++) linkedList.add(i + 200);
        long llMem = usedMemory() - baseline;
        System.out.printf("%-30s  %-12.1f  %-12.1f%n",
            "LinkedList<Integer>", llMem / 1e6, (double) llMem / n);
    }
}
// Resultado esperado (1M elementos):
//
// Structure                       Memory (MB)   Bytes/elem
// ────────────────────────────────────────────────────────
// int[]                           4.0           4.0
// ArrayList<Integer>              20.0          20.0
// HashMap<Integer, Integer>       80.0          80.0
// TreeMap<Integer, Integer>       64.0          64.0
// LinkedList<Integer>             40.0          40.0
```

**Preguntas:**

1. ¿HashMap usa 80 bytes/elem y int[] usa 4 bytes/elem. ¿20× es esperado?

2. ¿LinkedList usa menos memoria que HashMap? ¿Por qué?

3. ¿El method `usedMemory()` es preciso? ¿Qué errores puede tener?

4. ¿Cómo sería este benchmark para colecciones de Scala?

5. ¿Un benchmark equivalente en Rust mostraría qué diferencias?

---

### Ejercicio 2.4.3 — Analizar: el space-time tradeoff

**Tipo: Analizar**

```
Casi toda decisión en estructuras de datos es un tradeoff
entre tiempo y espacio:

  MÁS ESPACIO → MENOS TIEMPO:
    Hash map: usa O(n) espacio extra para el array de buckets
    → búsqueda en O(1) en vez de O(n) (linear search) o O(log n) (sorted).

    Bloom filter: usa O(n) bits para "recordar" n elementos
    → verificar existencia en O(k) en vez de O(n) o O(log n).

    Cache (memoization): almacena resultados previos
    → evita recalcular. Fibonacci recursivo de O(2ⁿ) a O(n) con O(n) espacio.

    Índices de base de datos: un B-Tree index ocupa espacio en disco
    → búsqueda en O(log n) en vez de O(n) (full table scan).

  MENOS ESPACIO → MÁS TIEMPO:
    Sorted array + binary search: O(0) espacio extra
    → búsqueda en O(log n) en vez de O(1) (hash map).

    Heap sort (in-place): O(1) espacio extra
    → más lento que merge sort por cache locality.

    Compresión: datos comprimidos usan menos espacio
    → pero necesitan descomprimirse para cada acceso.

    Streaming algorithms: procesan datos en una pasada con O(1) espacio
    → solo pueden responder preguntas limitadas (ej: HyperLogLog).

  El tradeoff fundamental:
    ESPACIO ↔ TIEMPO ↔ COMPLEJIDAD DE CÓDIGO

    Hash map:       mucho espacio, poco tiempo, código simple.
    Sorted array:   poco espacio, medio tiempo, código simple.
    Bloom filter:   poco espacio, poco tiempo, código medio.
    Skip list:      medio espacio, medio tiempo, código medio.
    Trie:           mucho espacio, poco tiempo (prefixes), código complejo.
```

**Preguntas:**

1. ¿La compresión de Parquet (Snappy, Zstd) es un space-time tradeoff?
   ¿Cuál es el costo en tiempo?

2. ¿HyperLogLog con O(12 KB) de espacio puede estimar
   la cardinalidad de mil millones de elementos. ¿Es magia?

3. ¿Si la RAM es barata, ¿el space-time tradeoff todavía importa?

4. ¿Bloom filters sacrifican EXACTITUD por espacio. ¿Es un tercer eje?

5. ¿En data engineering, ¿qué es más caro: RAM o compute time?

---

### Ejercicio 2.4.4 — Implementar: Fibonacci — el ejemplo canónico del tradeoff

**Tipo: Implementar**

```python
# Tres implementaciones de Fibonacci con distinto space-time tradeoff
import time
import sys

# O(2^n) tiempo, O(n) espacio (stack)
def fib_recursive(n):
    if n <= 1:
        return n
    return fib_recursive(n - 1) + fib_recursive(n - 2)

# O(n) tiempo, O(n) espacio (memoización)
def fib_memo(n, cache=None):
    if cache is None:
        cache = {}
    if n in cache:
        return cache[n]
    if n <= 1:
        return n
    cache[n] = fib_memo(n - 1, cache) + fib_memo(n - 2, cache)
    return cache[n]

# O(n) tiempo, O(1) espacio (iterativo)
def fib_iterative(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b

# Benchmark
for n in [10, 20, 30, 35, 40]:
    results = {}
    for name, func in [("recursive", fib_recursive),
                        ("memoized", fib_memo),
                        ("iterative", fib_iterative)]:
        if name == "recursive" and n > 35:
            results[name] = "too slow"
            continue
        start = time.perf_counter_ns()
        result = func(n)
        elapsed = time.perf_counter_ns() - start
        results[name] = f"{elapsed / 1000:.1f} μs"

    print(f"fib({n:2d}):  recursive={results['recursive']:>12s}  "
          f"memoized={results['memoized']:>10s}  "
          f"iterative={results['iterative']:>10s}")

# Resultado esperado:
# fib(10):  recursive=       5.0 μs  memoized=    3.0 μs  iterative=    0.5 μs
# fib(20):  recursive=     350 μs    memoized=    5.0 μs  iterative=    0.8 μs
# fib(30):  recursive=   40000 μs    memoized=    8.0 μs  iterative=    1.2 μs
# fib(35):  recursive=  440000 μs    memoized=   10.0 μs  iterative=    1.5 μs
# fib(40):  recursive=  too slow     memoized=   12.0 μs  iterative=    1.8 μs
```

**Preguntas:**

1. ¿Memoización convierte O(2ⁿ) en O(n) sacrificando O(n) espacio.
   ¿Es siempre un buen tradeoff?

2. ¿La versión iterativa es O(n) tiempo y O(1) espacio.
   ¿Es estrictamente mejor que memoización?

3. ¿Existe una versión O(log n) de Fibonacci? (Pista: matrix exponentiation)

4. ¿Dynamic programming es esencialmente memoización + tabla?

5. ¿El tradeoff tiempo-espacio de Fibonacci aplica
   a problemas de data engineering?

---

### Ejercicio 2.4.5 — Analizar: espacio en sistemas distribuidos

**Tipo: Analizar**

```
En un solo proceso, espacio = RAM consumida.
En sistemas distribuidos, espacio tiene dimensiones extra:

  1. ESPACIO EN RED (bandwidth):
     Un hash join en Spark envía datos entre nodos (shuffle).
     Un broadcast join envía la tabla pequeña a todos los nodos.
     Broadcast join: más espacio en red (copia en todos los nodos),
                     menos tiempo (no hay shuffle).
     Hash join: menos espacio, más tiempo.
     → Space-time tradeoff a nivel de cluster.

  2. ESPACIO EN DISCO:
     LSM-Tree compaction: merge de SSTables crea archivos nuevos
     antes de borrar los viejos → necesita ~2× el espacio en disco.
     Space amplification: datos reales / espacio en disco.
     LSM-Tree (size-tiered): ~3× space amplification.
     B-Tree: ~1.5× space amplification.

  3. ESPACIO EN ESTADO (streaming):
     Flink con windowed aggregation: mantiene estado por ventana.
     Ventanas de 1 hora con 1M eventos/minuto = 60M registros en estado.
     Más ventanas = más espacio = más resultados.
     Menos ventanas = menos espacio = menos granularidad.

  4. ESPACIO EN REPLICACIÓN:
     Kafka con replication factor 3: cada mensaje ocupa 3× el espacio.
     Más réplicas = más durabilidad, menos espacio.
     RF=1: sin redundancia. RF=3: tolerancia a 2 fallos.
     → Space-reliability tradeoff.

  La complejidad de espacio en sistemas distribuidos
  no es solo RAM de un proceso —
  es RAM + disco + red + replicación + estado.
  Y cada dimensión tiene su propio tradeoff.
```

**Preguntas:**

1. ¿Spark broadcast join tiene un límite de tamaño? ¿Cuál?

2. ¿La space amplification de un LSM-Tree es un problema real?

3. ¿Kafka con compresión reduce el espacio de replicación?

4. ¿Flink state backend (RocksDB) tiene space amplification?

5. ¿El costo en $ de espacio (storage) vs tiempo (compute)
   es el tradeoff final que importa en la nube?

---

## Sección 2.5 — Benchmarking Correcto: Cómo No Engañarte

### Ejercicio 2.5.1 — Leer: los 7 errores mortales del benchmarking

**Tipo: Leer**

```
Medir mal es peor que no medir.
Un benchmark incorrecto te da confianza falsa.

  ERROR 1: No hacer warmup
    La JVM compila bytecode a código nativo (JIT) durante la ejecución.
    Las primeras iteraciones son 10-100× más lentas que las finales.
    Si mides las primeras 1000 iteraciones, mides el intérprete,
    no tu código optimizado.
    Fix: ejecutar 10,000+ iteraciones antes de empezar a medir.

  ERROR 2: Medir solo una vez
    El sistema operativo programa otros procesos.
    El GC puede ejecutarse durante tu medición.
    La cache del CPU está fría o caliente dependiendo del run anterior.
    Un solo run no es representativo.
    Fix: ejecutar el benchmark 20-50 veces. Reportar mediana y percentiles.

  ERROR 3: Dead code elimination
    El compilador detecta que no usas el resultado y ELIMINA tu código.
    
    long start = System.nanoTime();
    for (int i = 0; i < n; i++) {
        Math.sqrt(i);  // resultado no usado → JIT elimina esto
    }
    long elapsed = System.nanoTime() - start;
    // elapsed ≈ 0. No mediste Math.sqrt. Mediste nada.
    
    Fix: acumular el resultado y usarlo (print o return).

  ERROR 4: Medir System.nanoTime() en vez de tu código
    System.nanoTime() tiene overhead de ~20-30 ns.
    Si tu operación tarda 5 ns, el overhead es 6× tu medición.
    Fix: medir MUCHAS operaciones juntas y dividir.

  ERROR 5: Ignorar el GC
    En Java/Go, el GC puede ejecutarse en cualquier momento.
    Si el GC corre durante tu benchmark, el tiempo incluye GC.
    Fix: forzar GC antes del benchmark. Usar JMH con fork.

  ERROR 6: Comparar manzanas con naranjas
    Benchmark de HashMap vs TreeMap donde el HashMap
    tiene 10M entries y el TreeMap tiene 100K entries.
    O donde el HashMap está pre-sized y el TreeMap no.
    Fix: mismos datos, misma cantidad, mismas condiciones.

  ERROR 7: No reportar el hardware y la configuración
    "Mi benchmark tardó 50 ms" no significa nada.
    ¿En qué CPU? ¿Cuánta RAM? ¿Qué JVM version?
    ¿Con qué flags? ¿En qué sistema operativo?
    Fix: reportar siempre: CPU, RAM, OS, runtime version, JVM flags.
```

**Preguntas:**

1. ¿El JIT de Java puede optimizar un benchmark
   de forma que invalide los resultados?

2. ¿Los benchmarks de Python necesitan warmup?

3. ¿`System.nanoTime()` es la forma más precisa
   de medir tiempo en Java?

4. ¿En Rust, ¿el dead code elimination puede eliminar
   un benchmark si usas `--release`?

5. ¿Un benchmark dentro de un Docker container
   es representativo del bare metal?

---

### Ejercicio 2.5.2 — Implementar: un benchmark correcto vs uno incorrecto

**Tipo: Implementar**

```java
// Demostrar las diferencias entre un benchmark malo y uno correcto

public class BenchmarkPitfalls {

    // ═══ BENCHMARK MALO ═══
    static void badBenchmark() {
        int n = 1_000_000;
        int[] data = new int[n];
        for (int i = 0; i < n; i++) data[i] = i;

        // Error 1: sin warmup
        // Error 2: solo un run
        // Error 3: no reporta percentiles
        long start = System.nanoTime();
        long total = 0;
        for (int i = 0; i < n; i++) {
            total += data[i];
        }
        long elapsed = System.nanoTime() - start;
        System.out.printf("BAD: %d ns total, %.1f ns/elem%n",
            elapsed, (double) elapsed / n);
        // Este número NO es confiable.
    }

    // ═══ BENCHMARK CORRECTO (sin JMH) ═══
    static void goodBenchmark() {
        int n = 1_000_000;
        int[] data = new int[n];
        for (int i = 0; i < n; i++) data[i] = i;

        int warmupRuns = 100;
        int measuredRuns = 200;

        // Warmup: dejar que el JIT optimice
        long dummy = 0;
        for (int w = 0; w < warmupRuns; w++) {
            for (int i = 0; i < n; i++) dummy += data[i];
        }

        // Medir múltiples runs
        long[] times = new long[measuredRuns];
        for (int r = 0; r < measuredRuns; r++) {
            long start = System.nanoTime();
            for (int i = 0; i < n; i++) dummy += data[i];
            times[r] = System.nanoTime() - start;
        }

        // Evitar dead code elimination
        if (dummy == Long.MIN_VALUE) System.out.println(dummy);

        // Estadísticas
        java.util.Arrays.sort(times);
        double median = times[measuredRuns / 2];
        double p5 = times[(int)(measuredRuns * 0.05)];
        double p95 = times[(int)(measuredRuns * 0.95)];
        double mean = 0;
        for (long t : times) mean += t;
        mean /= measuredRuns;

        System.out.printf("GOOD: median=%.1f ns/elem, mean=%.1f, p5=%.1f, p95=%.1f%n",
            median / n, mean / n, p5 / n, p95 / n);
    }

    public static void main(String[] args) {
        System.out.println("=== Bad benchmark ===");
        badBenchmark();
        System.out.println("\n=== Good benchmark ===");
        goodBenchmark();
    }
}
```

**Preguntas:**

1. ¿100 warmup runs son suficientes para que el JIT optimice?

2. ¿Por qué reportamos mediana y no media?

3. ¿p5 y p95 muestran qué?

4. ¿El truco `if (dummy == Long.MIN_VALUE)` evita dead code?

5. ¿200 measured runs son suficientes para significancia estadística?

---

### Ejercicio 2.5.3 — Analizar: cuándo confiar en un benchmark

**Tipo: Analizar**

```
Checklist para evaluar si un benchmark es confiable:

  □ ¿Reporta el hardware? (CPU model, cache sizes, RAM)
  □ ¿Reporta el software? (OS, runtime version, compiler flags)
  □ ¿Hizo warmup? (JIT compilation, cache warming)
  □ ¿Ejecutó múltiples runs? (≥20, idealmente ≥100)
  □ ¿Reporta mediana y percentiles? (no solo la media)
  □ ¿Previene dead code elimination? (resultado usado)
  □ ¿Controla el GC? (GC antes de medir, o medición fuera del GC)
  □ ¿Los datasets son comparables? (mismo tamaño, misma distribución)
  □ ¿El benchmark es reproducible? (código disponible, semilla fija)
  □ ¿Mide lo que dice medir? (no mide overhead de setup)

  Red flags — NO confiar en el benchmark si:
  - "Nuestro framework es 100× más rápido que X"
    (Probablemente midieron cosas distintas o en condiciones distintas.)
  - Solo reporta un número sin varianza ni percentiles.
  - No muestra el código del benchmark.
  - Mide operaciones que tardan menos de 100 ns sin loop.
  - Compara un release build contra un debug build.
  - No menciona warmup.

  Herramientas que hacen esto bien automáticamente:
  - JMH (Java): el gold standard. Maneja warmup, fork, dead code, GC.
  - Criterion (Rust): análisis estadístico, detección de outliers, HTML reports.
  - Go testing.B: warmup automático, reporta ns/op.
  - pytest-benchmark (Python): estadísticas, comparación de runs.
  - hyperfine (CLI): para benchmarkear programas completos.
```

**Preguntas:**

1. ¿Un benchmark publicado por un vendor de software
   es confiable por defecto?

2. ¿TPC-H y TPC-DS son benchmarks confiables para bases de datos?

3. ¿Criterion de Rust puede detectar si un resultado
   es estadísticamente distinto de otro?

4. ¿Un benchmark que muestra "±3%" de varianza es bueno?

5. ¿Correr benchmarks en CI/CD para detectar regresiones
   de rendimiento es una buena práctica?

---

### Ejercicio 2.5.4 — Implementar: benchmark correcto en Go y Rust

**Tipo: Implementar**

```go
// Go: usar testing.B (el framework de benchmarking nativo)
// Archivo: bench_test.go

package main

import "testing"

func sumSlice(data []int64) int64 {
    var total int64
    for _, v := range data {
        total += v
    }
    return total
}

var sink int64 // evitar dead code elimination

func BenchmarkSumSlice_1K(b *testing.B) {
    data := make([]int64, 1_000)
    for i := range data { data[i] = int64(i) }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sink = sumSlice(data)
    }
}

func BenchmarkSumSlice_1M(b *testing.B) {
    data := make([]int64, 1_000_000)
    for i := range data { data[i] = int64(i) }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sink = sumSlice(data)
    }
}

func BenchmarkSumSlice_10M(b *testing.B) {
    data := make([]int64, 10_000_000)
    for i := range data { data[i] = int64(i) }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sink = sumSlice(data)
    }
}

// Ejecutar: go test -bench=. -benchtime=5s -count=5
// Output:
// BenchmarkSumSlice_1K-8     5000000     230 ns/op   (0.23 ns/elem)
// BenchmarkSumSlice_1M-8        5000  280000 ns/op   (0.28 ns/elem)
// BenchmarkSumSlice_10M-8        500 2900000 ns/op   (0.29 ns/elem)
```

```rust
// Rust: usar criterion para benchmarks estadísticos
// Cargo.toml: [dev-dependencies] criterion = { version = "0.5", features = ["html_reports"] }
// Archivo: benches/sum_bench.rs

use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};

fn sum_vec(data: &[i64]) -> i64 {
    data.iter().sum()
}

fn bench_sum(c: &mut Criterion) {
    let mut group = c.benchmark_group("sum");

    for size in [1_000, 10_000, 100_000, 1_000_000, 10_000_000] {
        let data: Vec<i64> = (0..size as i64).collect();
        group.bench_with_input(
            BenchmarkId::new("vec", size),
            &data,
            |b, data| {
                b.iter(|| sum_vec(black_box(data)))
            }
        );
    }

    group.finish();
}

criterion_group!(benches, bench_sum);
criterion_main!(benches);

// Ejecutar: cargo bench
// Output (con análisis estadístico):
// sum/vec/1000       time: [119.23 ns 119.45 ns 119.69 ns]
// sum/vec/10000      time: [1.1891 μs 1.1903 μs 1.1915 μs]
// sum/vec/100000     time: [11.842 μs 11.856 μs 11.871 μs]
// sum/vec/1000000    time: [271.42 μs 271.89 μs 272.38 μs]
// sum/vec/10000000   time: [2.7142 ms 2.7189 ms 2.7238 ms]
//
// Criterion reporta: tiempo medio, intervalo de confianza, y
// detecta si el rendimiento cambió respecto al benchmark anterior.
```

**Preguntas:**

1. ¿`black_box` de Criterion es equivalente al truco
   `if (dummy == Long.MIN_VALUE)` de Java?

2. ¿`b.N` de Go se ajusta automáticamente?
   ¿Cómo decide cuántas iteraciones hacer?

3. ¿Criterion genera reportes HTML?
   ¿Son útiles para documentar rendimiento?

4. ¿`go test -count=5` ejecuta el benchmark 5 veces.
   ¿Es suficiente para significancia estadística?

5. ¿pytest-benchmark de Python es comparable
   en calidad a JMH o Criterion?

---

### Ejercicio 2.5.5 — Analizar: qué métricas reportar además del tiempo

**Tipo: Analizar**

```
El tiempo (ns/op) es la métrica más importante
pero no la única que necesitas:

  1. THROUGHPUT (ops/sec o elementos/sec)
     Más intuitivo para pipelines: "procesa 5M eventos/segundo".
     Throughput = 1 / latencia (para operaciones secuenciales).
     Para operaciones paralelas: throughput ≠ 1/latencia.

  2. LATENCIA POR PERCENTIL
     p50 (mediana): la experiencia "típica".
     p99: 1 de cada 100 operaciones tarda al menos esto.
     p99.9: 1 de cada 1000.
     Para un web server con 1000 req/s:
       p99 = 100 ms significa 10 requests/s tardan >100 ms.
     Los percentiles altos revelan tail latency (GC pauses, resizes).

  3. MEMORY USAGE
     Peak memory: el máximo de RAM usado.
     Resident memory: RAM realmente usada (no solo asignada).
     GC pressure: bytes asignados por operación.
     Para Java: usar -verbose:gc o JFR (Java Flight Recorder).

  4. ALLOCATIONS
     Número de asignaciones de heap por operación.
     En Go: benchmem flag: "3 allocs/op".
     Más allocations = más GC pressure = más latencia tail.

  5. CACHE MISSES
     L1 miss rate: el % de accesos que no encuentran datos en L1.
     LLC (Last Level Cache) miss rate: accesos que van a RAM.
     perf stat: "2.5% of all L1-dcache loads resulted in misses".

  6. INSTRUCTIONS PER CYCLE (IPC)
     Mide cuán eficientemente el CPU ejecuta tu código.
     IPC < 1: CPU esperando (memory bound, branch misses).
     IPC > 2: CPU eficiente (compute bound).
     perf stat: "1.85 insn per cycle".

  Un benchmark completo reporta:
    "HashMap.get() para 1M entries:
     Mediana: 45 ns/op, p99: 120 ns/op (GC tail)
     Throughput: 22M ops/sec
     Memory: 80 MB resident, 12 bytes/alloc
     L1 miss rate: 15%, IPC: 0.8 (memory bound)"

  Con eso puedes tomar decisiones informadas.
  Sin eso, solo tienes un número sin contexto.
```

**Preguntas:**

1. ¿Un throughput de 22M ops/sec es bueno para un hash map?

2. ¿Un IPC de 0.8 es bajo? ¿Qué indica?

3. ¿Tail latency (p99) importa más que mediana para servicios web?

4. ¿Go `testing.B` con `-benchmem` reporta allocations?

5. ¿JFR (Java Flight Recorder) puede capturar todas estas métricas
   en un solo run?

---

## Sección 2.6 — Los Frameworks de Benchmarking: JMH, Criterion, y Más

### Ejercicio 2.6.1 — Implementar: JMH — el gold standard en Java

**Tipo: Implementar**

```java
// JMH (Java Microbenchmark Harness) — el framework de OpenJDK
// Dependency: org.openjdk.jmh:jmh-core:1.37
//             org.openjdk.jmh:jmh-generator-annprocess:1.37

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.options.OptionsBuilder;
import java.util.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 2)      // 5 warmup iterations de 2s
@Measurement(iterations = 10, time = 2) // 10 measurement iterations de 2s
@Fork(3)                                 // 3 JVM forks (aísla de JIT state)
public class CollectionBenchmark {

    @Param({"100", "10000", "1000000"})
    int size;

    int[] intArray;
    ArrayList<Integer> arrayList;
    HashMap<Integer, Integer> hashMap;

    @Setup
    public void setup() {
        intArray = new int[size];
        arrayList = new ArrayList<>(size);
        hashMap = new HashMap<>(size * 2);
        for (int i = 0; i < size; i++) {
            intArray[i] = i;
            arrayList.add(i + 200);
            hashMap.put(i + 200, i);
        }
    }

    @Benchmark
    public long sumIntArray() {
        long total = 0;
        for (int v : intArray) total += v;
        return total; // return evita dead code elimination
    }

    @Benchmark
    public long sumArrayList() {
        long total = 0;
        for (int v : arrayList) total += v;
        return total;
    }

    @Benchmark
    public Integer hashMapGet() {
        return hashMap.get(size / 2 + 200);
    }

    public static void main(String[] args) throws Exception {
        var opt = new OptionsBuilder()
            .include(CollectionBenchmark.class.getSimpleName())
            .build();
        new Runner(opt).run();
    }
}
// JMH maneja automáticamente:
// - Warmup (JIT compilation)
// - Fork (JVM fresh start, sin estado previo)
// - Dead code elimination (@Benchmark return value)
// - Estadísticas (media, error, percentiles)
// - GC noise (entre forks)
```

**Preguntas:**

1. ¿`@Fork(3)` ejecuta 3 JVMs separadas? ¿Por qué?

2. ¿`@Param` permite benchmarkear con diferentes tamaños
   en un solo run?

3. ¿JMH detecta si el JIT eliminó tu benchmark?

4. ¿`Mode.AverageTime` vs `Mode.Throughput`: ¿cuándo usar cuál?

5. ¿JMH puede medir allocations/op?
   (Pista: `-prof gc`)

---

### Ejercicio 2.6.2 — Implementar: benchmark completo en Go con benchstat

**Tipo: Implementar**

```go
// Go: benchmarks con testing.B + benchstat para análisis estadístico
// Archivo: collections_bench_test.go

package main

import (
    "testing"
)

var sinkInt64 int64
var sinkBool bool

func BenchmarkSliceIterate(b *testing.B) {
    sizes := []int{100, 10_000, 1_000_000}
    for _, size := range sizes {
        data := make([]int64, size)
        for i := range data { data[i] = int64(i) }

        b.Run(formatSize(size), func(b *testing.B) {
            b.ReportAllocs()  // reportar allocations
            for i := 0; i < b.N; i++ {
                var total int64
                for _, v := range data {
                    total += v
                }
                sinkInt64 = total
            }
        })
    }
}

func BenchmarkMapLookup(b *testing.B) {
    sizes := []int{100, 10_000, 1_000_000}
    for _, size := range sizes {
        m := make(map[int64]int64, size)
        for i := 0; i < size; i++ { m[int64(i)] = int64(i * 2) }
        target := int64(size / 2)

        b.Run(formatSize(size), func(b *testing.B) {
            b.ReportAllocs()
            for i := 0; i < b.N; i++ {
                v, ok := m[target]
                sinkInt64 = v
                sinkBool = ok
            }
        })
    }
}

func formatSize(n int) string {
    switch {
    case n >= 1_000_000: return fmt.Sprintf("%dM", n/1_000_000)
    case n >= 1_000:     return fmt.Sprintf("%dK", n/1_000)
    default:             return fmt.Sprintf("%d", n)
    }
}

// Ejecutar:
//   go test -bench=. -benchmem -count=10 | tee old.txt
//   # (hacer cambios)
//   go test -bench=. -benchmem -count=10 | tee new.txt
//   benchstat old.txt new.txt
//
// benchstat compara dos sets de benchmarks y reporta
// si la diferencia es estadísticamente significativa.
```

**Preguntas:**

1. ¿`b.ReportAllocs()` reporta allocations por operación o totales?

2. ¿`benchstat` usa qué test estadístico para comparar?

3. ¿`-count=10` es suficiente para benchstat?

4. ¿Los sub-benchmarks con `b.Run()` comparten warmup?

5. ¿Go benchmarks son comparables en rigor a JMH?

---

### Ejercicio 2.6.3 — Analizar: cuándo NO usar un microbenchmark

**Tipo: Analizar**

```
Los microbenchmarks miden operaciones aisladas.
Pero en producción, las operaciones no están aisladas.

  PROBLEMAS CON MICROBENCHMARKS:

  1. Cache caliente vs cache fría
     Un microbenchmark ejecuta la misma operación millones de veces.
     Los datos están en L1 cache después de la primera iteración.
     En producción, los datos pueden NO estar en cache.
     → El microbenchmark es más rápido que la realidad.

  2. Branch predictor entrenado
     Después de 1000 iteraciones, el predictor conoce el patrón.
     En producción, los patrones son impredecibles.
     → El microbenchmark tiene menos mispredictions que la realidad.

  3. Sin contención
     Un microbenchmark single-threaded no tiene lock contention.
     En producción, 50 threads compiten por el mismo HashMap.
     → El microbenchmark no refleja la latencia bajo carga.

  4. Sin GC pressure
     Un microbenchmark con poco heap no activa el GC.
     En producción, el heap está lleno y el GC corre frecuentemente.
     → El microbenchmark no captura GC pauses.

  CUÁNDO USAR MICROBENCHMARKS:
  - Comparar dos implementaciones de la misma operación.
  - Medir el costo de una optimización específica.
  - Verificar que una change no introduce una regresión.
  - Entender el rendimiento teórico (upper bound).

  CUÁNDO NO USAR MICROBENCHMARKS:
  - Predecir el rendimiento en producción.
  - Decidir la arquitectura de un sistema.
  - Comparar lenguajes (demasiadas variables confusas).

  ALTERNATIVAS:
  - Load testing: simular tráfico realista (wrk, k6, gatling).
  - Profiling: medir un sistema real en staging (async-profiler, pprof).
  - Tracing: medir latencia end-to-end en producción (Jaeger, Zipkin).
  - Macro-benchmarks: medir pipelines completos con datos reales.
```

**Preguntas:**

1. ¿Los benchmarks de TPC-H son micro o macro benchmarks?

2. ¿Un benchmark de Spark que procesa 1TB de datos
   es un microbenchmark?

3. ¿Async-profiler puede medir allocations y cache misses
   de un servicio Java en producción?

4. ¿Los benchmarks de este libro son microbenchmarks?
   ¿Son útiles igualmente?

5. ¿Cómo reconcilias un microbenchmark que dice "HashMap.get() = 45 ns"
   con producción donde HashMap.get() tarda 500 ns?

---

### Ejercicio 2.6.4 — Implementar: Python benchmark correcto con pytest-benchmark

**Tipo: Implementar**

```python
# pip install pytest-benchmark

import pytest
import array
import random

def sum_list(data):
    return sum(data)

def sum_array(data):
    total = 0
    for v in data:
        total += v
    return total

def sum_array_module(data):
    """Usar array.array (datos contiguos, pero iteración Python)"""
    total = 0
    for v in data:
        total += v
    return total

@pytest.fixture(params=[1_000, 100_000, 1_000_000])
def dataset(request):
    n = request.param
    return {
        'list': list(range(n)),
        'array': array.array('l', range(n)),
        'size': n,
    }

def test_sum_list(benchmark, dataset):
    benchmark.name = f"list[{dataset['size']}]"
    result = benchmark(sum_list, dataset['list'])
    assert result == sum(range(dataset['size']))

def test_sum_array_module(benchmark, dataset):
    benchmark.name = f"array[{dataset['size']}]"
    result = benchmark(sum_array_module, dataset['array'])
    assert result == sum(range(dataset['size']))

def test_sum_builtin(benchmark, dataset):
    """sum() builtin — implementado en C"""
    benchmark.name = f"sum_builtin[{dataset['size']}]"
    result = benchmark(sum, dataset['list'])
    assert result == sum(range(dataset['size']))

# Ejecutar: pytest bench_test.py --benchmark-sort=mean --benchmark-columns=mean,stddev,rounds
#
# pytest-benchmark maneja automáticamente:
# - Warmup (calibración automática)
# - Múltiples rounds
# - Estadísticas (mean, stddev, min, max, rounds)
# - Comparación entre benchmarks
```

**Preguntas:**

1. ¿`sum()` builtin es más rápido que un loop Python?
   ¿Por cuánto?

2. ¿`array.array` es más rápido que `list` para iteración en Python?

3. ¿pytest-benchmark calibra automáticamente las iteraciones?

4. ¿NumPy `np.sum()` sería cuántas veces más rápido que `sum(list)`?

5. ¿Los benchmarks de Python son útiles o el overhead del intérprete
   domina todos los resultados?

---

### Ejercicio 2.6.5 — Analizar: reproducibilidad — el benchmark que no puedes repetir

**Tipo: Analizar**

```
Un benchmark no reproducible es inútil.

  FACTORES QUE AFECTAN LA REPRODUCIBILIDAD:

  1. Thermal throttling
     Después de 30 segundos de benchmark, el CPU se calienta.
     La frecuencia baja de 4.5 GHz a 3.8 GHz.
     Los últimos runs son 15% más lentos que los primeros.
     Fix: monitorear la frecuencia del CPU durante el benchmark.

  2. Turbo boost
     El primer core se ejecuta a 5.0 GHz (turbo).
     Cuando todos los cores están activos, baja a 4.0 GHz.
     Un benchmark single-threaded es 25% más rápido que multi-threaded
     solo por la frecuencia.
     Fix: fijar la frecuencia: cpupower frequency-set -g performance.

  3. NUMA (Non-Uniform Memory Access)
     En servidores con 2+ sockets, la RAM está dividida por socket.
     Acceder a RAM del otro socket es 2× más lento.
     Si tu benchmark se migra al otro socket, el rendimiento cambia.
     Fix: pinear el proceso a un socket: numactl --cpunodebind=0.

  4. Otros procesos
     Un navegador con 100 tabs consume RAM y CPU.
     Un servicio de indexación de archivos compite por cache.
     Fix: cerrar todo. O mejor: benchmark en un servidor dedicado.

  5. OS page cache
     La primera lectura de un archivo va a disco.
     La segunda lectura va al page cache (RAM) → 100× más rápida.
     Fix: limpiar page cache: echo 3 > /proc/sys/vm/drop_caches.

  CHECKLIST DE REPRODUCIBILIDAD:
  □ CPU frequency fija (no turbo, no throttling)
  □ Proceso pineado a core(s) específico(s)
  □ Sin otros procesos significativos
  □ Page cache limpio (si benchmarkeas I/O)
  □ Mismo hardware entre runs comparados
  □ Semilla aleatoria fija (para datos generados)
  □ Código del benchmark versionado (git commit hash)
```

**Preguntas:**

1. ¿Thermal throttling afecta más a laptops que a servidores?

2. ¿Los benchmarks en instancias de cloud (EC2) son reproducibles?

3. ¿`taskset` en Linux puede pinear un proceso a un core específico?

4. ¿Un benchmark que corre 10 minutos es más o menos confiable
   que uno que corre 10 segundos?

5. ¿Publicar el commit hash del benchmark y el hardware
   debería ser obligatorio en papers académicos?

---

## Sección 2.7 — La Tabla de Big-O vs la Realidad: el Benchmark Final

### Ejercicio 2.7.1 — Implementar: medir las 5 operaciones clave en 4 estructuras

**Tipo: Implementar**

```java
// El benchmark definitivo: ArrayList vs LinkedList vs HashMap vs TreeMap
// Operaciones: iterate, search, insert, delete, sort

import java.util.*;

public class RealityVsBigO {

    static final int N = 1_000_000;
    static final int OPS = 100_000;
    static Random rng = new Random(42);

    // ═══ ITERATE ═══
    static long iterateArray(ArrayList<Integer> list) {
        long total = 0;
        for (int v : list) total += v;
        return total;
    }
    static long iterateLinked(LinkedList<Integer> list) {
        long total = 0;
        for (int v : list) total += v;
        return total;
    }

    // ═══ SEARCH (contains) ═══
    static boolean searchArray(ArrayList<Integer> list, int target) {
        return list.contains(target);
    }
    static boolean searchHash(HashMap<Integer, Integer> map, int target) {
        return map.containsKey(target);
    }
    static boolean searchTree(TreeMap<Integer, Integer> map, int target) {
        return map.containsKey(target);
    }

    // ═══ INSERT (random position) ═══
    static void insertArray(ArrayList<Integer> list, int pos, int val) {
        list.add(pos, val);
    }
    static void insertHash(HashMap<Integer, Integer> map, int key, int val) {
        map.put(key, val);
    }
    static void insertTree(TreeMap<Integer, Integer> map, int key, int val) {
        map.put(key, val);
    }

    static void bench(String name, Runnable op, int ops) {
        // Warmup
        for (int i = 0; i < Math.min(ops, 1000); i++) op.run();
        // Measure
        long start = System.nanoTime();
        for (int i = 0; i < ops; i++) op.run();
        long elapsed = System.nanoTime() - start;
        System.out.printf("  %-35s %8.0f ns/op%n", name, (double) elapsed / ops);
    }

    public static void main(String[] args) {
        // Setup
        var arrayList = new ArrayList<Integer>(N);
        var linkedList = new LinkedList<Integer>();
        var hashMap = new HashMap<Integer, Integer>(N * 2);
        var treeMap = new TreeMap<Integer, Integer>();

        for (int i = 0; i < N; i++) {
            arrayList.add(i);
            linkedList.add(i);
            hashMap.put(i, i);
            treeMap.put(i, i);
        }

        int[] targets = new int[OPS];
        for (int i = 0; i < OPS; i++) targets[i] = rng.nextInt(N);

        System.out.println("═══ ITERATE (full scan) ═══");
        bench("ArrayList  O(n)", () -> iterateArray(arrayList), 20);
        bench("LinkedList O(n)", () -> iterateLinked(linkedList), 20);

        System.out.println("\n═══ SEARCH (random key) ═══");
        int[] ti = {0}; // counter para los targets
        bench("ArrayList.contains  O(n)",
            () -> searchArray(arrayList, targets[ti[0]++ % OPS]), OPS);
        ti[0] = 0;
        bench("HashMap.containsKey O(1)",
            () -> searchHash(hashMap, targets[ti[0]++ % OPS]), OPS);
        ti[0] = 0;
        bench("TreeMap.containsKey O(log n)",
            () -> searchTree(treeMap, targets[ti[0]++ % OPS]), OPS);

        System.out.println("\n═══ INSERT (new elements) ═══");
        var freshHash = new HashMap<Integer, Integer>(N * 2);
        var freshTree = new TreeMap<Integer, Integer>();
        bench("HashMap.put  O(1) amort.",
            () -> { freshHash.put(rng.nextInt(), rng.nextInt()); }, OPS);
        bench("TreeMap.put  O(log n)",
            () -> { freshTree.put(rng.nextInt(), rng.nextInt()); }, OPS);
    }
}
// Resultado esperado:
// ═══ ITERATE (full scan) ═══
//   ArrayList  O(n)                        8,000,000 ns/op
//   LinkedList O(n)                       25,000,000 ns/op
//
// ═══ SEARCH (random key) ═══
//   ArrayList.contains  O(n)                  5,000 ns/op
//   HashMap.containsKey O(1)                     28 ns/op
//   TreeMap.containsKey O(log n)                 85 ns/op
//
// ═══ INSERT (new elements) ═══
//   HashMap.put  O(1) amort.                     35 ns/op
//   TreeMap.put  O(log n)                       120 ns/op
```

**Preguntas:**

1. ¿LinkedList iterate es 3× más lento que ArrayList.
   Big-O dice que ambos son O(n). ¿Esperabas este ratio?

2. ¿HashMap search es 180× más rápido que ArrayList search.
   ¿Big-O predice correctamente este gap?

3. ¿TreeMap search (85 ns) es 3× más lento que HashMap (28 ns).
   ¿Es significativo en producción?

4. ¿ArrayList insert (random position) no se midió. ¿Por qué
   sería problemático medirlo?

5. ¿Estos resultados cambiarían significativamente en Rust?

---

### Ejercicio 2.7.2 — Analizar: construir tu tabla de Big-O honesta

**Tipo: Analizar**

```
Con los benchmarks de este capítulo, podemos construir
una tabla que refleja la realidad, no solo la teoría.

  Estructura       Operación        Big-O       ns/op real   Cache misses
  ──────────       ─────────        ─────       ──────────   ────────────
  int[]            iterate/elem     O(n)        0.5          bajo (2.5%)
  ArrayList<Int>   iterate/elem     O(n)        8            medio (15%)
  LinkedList<Int>  iterate/elem     O(n)        25           alto (45%)
  HashMap<Int,Int> get              O(1) avg    28           medio
  TreeMap<Int,Int> get              O(log n)    85           alto
  ArrayList<Int>   contains         O(n)        5,000        medio
  int[] sorted     binary search    O(log n)    40           bajo
  HashMap<Int,Int> put              O(1) amort  35           medio
  TreeMap<Int,Int> put              O(log n)    120          alto

  Insights que la tabla clásica de Big-O NO te da:

  1. Iterate: int[] es 50× más rápido que LinkedList
     a pesar de que ambos son O(n).
     Causa: cache locality (Cap.01).

  2. Search: HashMap es 6× más rápido que TreeMap.
     HashMap es O(1), TreeMap es O(log n).
     La diferencia es real pero no tan grande como sugiere
     O(1) vs O(log 1M) = O(20).
     Causa: las constantes del hash function y los cache misses
     del tree se compensan parcialmente.

  3. Search: sorted array binary search (40 ns) es comparable
     a HashMap (28 ns) y más rápido que TreeMap (85 ns).
     Si tus datos están ordenados y no cambian,
     un array + binary search puede ser mejor que un TreeMap
     y comparable a un HashMap, con mucha menos memoria.

  4. Insert: HashMap put (35 ns) vs TreeMap put (120 ns).
     3.4× de diferencia. Para inserts masivos, HashMap gana.
     Pero TreeMap mantiene el orden — HashMap no.
     → Si necesitas orden, TreeMap. Si no, HashMap.

  Esta tabla es TU referencia para los próximos 16 capítulos.
  Cada estructura nueva que implementemos se medirá
  y se comparará contra esta tabla.
```

**Preguntas:**

1. ¿Sorted array binary search (40 ns) compite con HashMap (28 ns).
   ¿Por qué la industria no usa más sorted arrays?

2. ¿Si agregaras una columna "memoria por elemento",
   ¿qué estructura ganaría?

3. ¿Las constantes de esta tabla aplican a otros CPUs?
   ¿A ARM (Apple M1)?

4. ¿Cómo cambiaría esta tabla si los datos estuvieran en disco?

5. ¿Esta tabla debería estar en la primera página
   de todo libro de estructuras de datos?

---

### Ejercicio 2.7.3 — Implementar: generar tu propia tabla de referencia

**Tipo: Implementar**

```rust
// Rust: generar una tabla de referencia para tu hardware
use std::collections::{HashMap, BTreeMap, LinkedList};
use std::time::Instant;

fn bench<F: FnMut() -> u64>(name: &str, ops: usize, mut f: F) {
    // Warmup
    for _ in 0..ops.min(10_000) { let _ = f(); }

    let start = Instant::now();
    let mut sink = 0u64;
    for _ in 0..ops {
        sink = sink.wrapping_add(f());
    }
    let elapsed = start.elapsed();
    let ns_per_op = elapsed.as_nanos() as f64 / ops as f64;
    println!("  {:<40} {:>8.1} ns/op", name, ns_per_op);
    // prevent optimization
    std::hint::black_box(sink);
}

fn main() {
    let n = 1_000_000usize;
    let ops = 100_000;

    // Setup
    let vec: Vec<i64> = (0..n as i64).collect();
    let linked: LinkedList<i64> = (0..n as i64).collect();
    let hash: HashMap<i64, i64> = (0..n as i64).map(|i| (i, i * 2)).collect();
    let btree: BTreeMap<i64, i64> = (0..n as i64).map(|i| (i, i * 2)).collect();

    println!("N = {}\n", n);

    println!("═══ ITERATE ═══");
    bench("Vec<i64> iterate", 20, || vec.iter().sum::<i64>() as u64);
    bench("LinkedList<i64> iterate", 20, || linked.iter().sum::<i64>() as u64);

    println!("\n═══ SEARCH ═══");
    let mut idx = 0usize;
    bench("Vec binary_search O(log n)", ops, || {
        idx = (idx + 7919) % n; // pseudo-random
        vec.binary_search(&(idx as i64)).unwrap_or(0) as u64
    });
    idx = 0;
    bench("HashMap get O(1)", ops, || {
        idx = (idx + 7919) % n;
        *hash.get(&(idx as i64)).unwrap() as u64
    });
    idx = 0;
    bench("BTreeMap get O(log n)", ops, || {
        idx = (idx + 7919) % n;
        *btree.get(&(idx as i64)).unwrap() as u64
    });

    println!("\n═══ INSERT (fresh collections) ═══");
    bench("HashMap insert", ops, || {
        let mut m = HashMap::with_capacity(100);
        for i in 0..100 { m.insert(i as i64, i as i64); }
        m.len() as u64
    });
    bench("BTreeMap insert", ops, || {
        let mut m = BTreeMap::new();
        for i in 0..100 { m.insert(i as i64, i as i64); }
        m.len() as u64
    });
}
// cargo run --release
// Resultado (varía por hardware):
// Vec iterate ~0.3 ns/elem (SIMD)
// LinkedList iterate ~8 ns/elem
// Vec binary_search ~15 ns
// HashMap get ~20 ns
// BTreeMap get ~45 ns
```

**Preguntas:**

1. ¿Rust HashMap (20 ns) es más rápido que Java HashMap (28 ns)?
   ¿Por cuánto y por qué?

2. ¿BTreeMap de Rust es un B-Tree o un B+ Tree?

3. ¿Vec binary_search (15 ns) supera a HashMap (20 ns) en Rust?
   ¿Es el mismo resultado que en Java?

4. ¿Estos números son tu baseline personal para tu hardware?

5. ¿Deberías ejecutar este benchmark cada vez que cambias de laptop?

---

### Ejercicio 2.7.4 — Analizar: las reglas post-benchmark

**Tipo: Analizar**

```
Después de este capítulo, estas son las reglas actualizadas:

  Regla 1 (del Cap.01, actualizada):
    La estructura más rápida es la que causa menos cache misses.
    → CONFIRMADO por los benchmarks de este capítulo.

  Regla 2 (nueva):
    Big-O predice el comportamiento asintótico,
    no el rendimiento para tu n.
    → CONFIRMADO: linear search vence a hash lookup para n < 64.

  Regla 3 (nueva):
    Las constantes importan tanto como la clase de complejidad.
    → CONFIRMADO: sorted array binary search (~40 ns) compite
      con HashMap (~28 ns), a pesar de O(log n) vs O(1).

  Regla 4 (nueva):
    "O(1) amortizado" no es "O(1) siempre".
    → La diferencia importa para latencia, no para throughput.

  Regla 5 (nueva):
    Mide antes de decidir. Mide correctamente.
    → JMH, Criterion, go test -bench, pytest-benchmark.
    → Warmup, múltiples runs, mediana, percentiles.

  Regla 6 (nueva):
    El espacio es el tradeoff olvidado.
    → HashMap usa 80 bytes/elem en Java.
    → int[] usa 4 bytes/elem.
    → 20× de diferencia = la diferencia entre caber en RAM o no.

  Regla 7 (del Cap.01, confirmada):
    La elección de estructura importa más que la elección de lenguaje.
    → Array vs linked list = 10-50× diferencia en cualquier lenguaje.
    → Java vs Rust para la misma estructura = 2-3× diferencia.
```

**Preguntas:**

1. ¿Alguna de estas reglas es controversial?

2. ¿Un senior engineer debería saber todas estas reglas?

3. ¿Estas reglas aplican a Python con NumPy/Polars?

4. ¿Un sistema en la nube donde el costo de RAM es $X/GB
   cambia la importancia de la Regla 6?

5. ¿Cuál de estas reglas te habría ahorrado más tiempo
   si la hubieras sabido antes?

---

### Ejercicio 2.7.5 — Resumen: el framework mental para el resto del libro

**Tipo: Leer**

```
Después de los Capítulos 01 y 02, tienes un framework
para evaluar cualquier estructura de datos:

  PASO 1: ¿Cuál es la complejidad Big-O?
    Necesario pero insuficiente.
    Te da el orden de magnitud.

  PASO 2: ¿Cuáles son las constantes ocultas?
    Cache locality, branch prediction, vectorización, overhead del lenguaje.
    Pueden cambiar el rendimiento 10-50× dentro de la misma clase O.

  PASO 3: ¿Cuál es el costo en memoria?
    Overhead de la estructura por elemento.
    Para millones de elementos, puede ser la diferencia
    entre caber en RAM o necesitar disco.

  PASO 4: ¿Cuál es el comportamiento bajo carga real?
    Amortizado vs peor caso.
    GC pauses, lock contention, tail latency.
    Un microbenchmark no captura esto.

  PASO 5: ¿Los números confirman la teoría?
    Mide con un framework serio (JMH, Criterion, go test -bench).
    Si los números contradicen la teoría, confía en los números.

  A partir del Capítulo 03, cada estructura de datos
  se evaluará con estos 5 pasos.
  No solo "este es un hash map y es O(1)".
  Sino "este hash map usa Robin Hood hashing (constante baja),
  ocupa 50 bytes/entry (overhead alto), es O(1) amortizado
  (resize cada 2× entries), y mi benchmark muestra 20 ns/op
  con 3% de L1 cache misses en mi hardware".

  Eso es entender una estructura de datos desde las tripas.
```

**Preguntas:**

1. ¿Este framework de 5 pasos es completo o le falta algo?

2. ¿Cuánto tiempo toma ejecutar estos 5 pasos para una estructura nueva?

3. ¿Un junior developer necesita los 5 pasos
   o puede empezar con los pasos 1 y 5?

4. ¿La industria evalúa estructuras de datos con este rigor?

5. ¿Si tuvieras que explicar este framework en 30 segundos,
   ¿qué dirías?
