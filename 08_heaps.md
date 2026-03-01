# Guía de Ejercicios — Cap.08: Heaps y Priority Queues: Orden Parcial

> No siempre necesitas todos los datos ordenados.
> A veces solo necesitas saber cuál es el MÍNIMO.
>
> "¿Cuál es el siguiente evento que debe ejecutarse?"
> "¿Cuál es la tarea con mayor prioridad?"
> "¿Cuál es el K-ésimo elemento más grande?"
> "¿Cuáles son los N archivos más pequeños para merge?"
>
> Un sorted array responde estas preguntas, pero insertar cuesta O(n).
> Un Red-Black tree las responde en O(log n), pero es overkill:
> paga el costo de mantener TODOS los datos ordenados
> cuando solo necesitas acceder al extremo.
>
> El heap es la solución: orden PARCIAL.
> No mantiene todo ordenado — solo garantiza que la raíz
> es el mínimo (o máximo). Insert y extract: O(log n).
> Peek (ver el mínimo sin extraer): O(1).
>
> Y el truco genial: el heap vive en un ARRAY.
> Sin punteros. Sin nodos. Sin cache misses.
> Un árbol conceptual empacado en memoria contigua.
>
> Spark usa heaps para merge sort externo durante shuffles.
> Kafka usa heaps para DelayedOperationPurgatory.
> Airflow ordena tareas por prioridad con una priority queue.
> Los schedulers de sistemas operativos usan heaps para timers.

---

## El modelo mental: el array como árbol

```
Un binary heap es un array interpretado como un árbol binario COMPLETO.

  Array:  [10, 20, 15, 30, 40, 25, 18]
  Índice:   0   1   2   3   4   5   6

  Árbol:
              10          (index 0)
           /      \
         20        15     (index 1, 2)
        /  \      /  \
      30    40  25    18  (index 3, 4, 5, 6)

  Relación padre-hijo (0-indexed):
    parent(i) = (i - 1) / 2
    left(i)   = 2*i + 1
    right(i)  = 2*i + 2

  No hay punteros. El "árbol" son solo multiplicaciones y divisiones
  sobre el índice del array. Las relaciones son implícitas.

  HEAP PROPERTY (min-heap):
    Para todo nodo i: arr[parent(i)] ≤ arr[i]
    La raíz (arr[0]) siempre es el mínimo.

  Operaciones:

  PEEK: return arr[0]. O(1).

  INSERT (sift-up):
    1. Agregar el elemento al final del array.
    2. Comparar con el padre. Si es menor, swap.
    3. Repetir hasta que el padre sea menor o llegue a la raíz.
    O(log n): sube como máximo log₂(n) niveles.

  EXTRACT-MIN (sift-down):
    1. Guardar arr[0] (el mínimo).
    2. Mover el último elemento a arr[0].
    3. Comparar con el menor hijo. Si es mayor, swap.
    4. Repetir hasta que ambos hijos sean mayores o sea hoja.
    O(log n): baja como máximo log₂(n) niveles.

  HEAPIFY (construir heap desde array desordenado):
    Hacer sift-down desde el último nodo interno hasta la raíz.
    O(n) — NO O(n log n). La demostración es hermosa (Cap.02).

  ¿Por qué array y no un tree con nodos?
    - Sin punteros: 0 bytes de overhead por nodo.
    - Memoria contigua: excelente cache locality.
    - Sin allocation por insert: el array crece (dynamic array, Cap.03).
    - parent/child son aritmética, no pointer chasing.
```

---

## Tabla de contenidos

- [Sección 8.1 — Binary heap: el array como árbol](#sección-81--binary-heap-el-array-como-árbol)
- [Sección 8.2 — Implementar min-heap desde cero](#sección-82--implementar-min-heap-desde-cero)
- [Sección 8.3 — Heapify: construir un heap en O(n)](#sección-83--heapify-construir-un-heap-en-on)
- [Sección 8.4 — Priority queues: la abstracción sobre el heap](#sección-84--priority-queues-la-abstracción-sobre-el-heap)
- [Sección 8.5 — D-ary heaps: más hijos, menos profundidad](#sección-85--d-ary-heaps-más-hijos-menos-profundidad)
- [Sección 8.6 — Indexed priority queue y decrease-key](#sección-86--indexed-priority-queue-y-decrease-key)
- [Sección 8.7 — Heaps en producción: schedulers, merges, y timers](#sección-87--heaps-en-producción-schedulers-merges-y-timers)

---

## Sección 8.1 — Binary Heap: el Array como Árbol

### Ejercicio 8.1.1 — Leer: anatomía del binary heap

**Tipo: Leer**

```
El binary heap tiene dos invariantes:

  1. SHAPE PROPERTY (forma):
     El árbol es un árbol binario COMPLETO.
     Todos los niveles están llenos excepto posiblemente el último,
     que se llena de izquierda a derecha.
     → Esto garantiza que el árbol cabe perfectamente en un array.
     → La altura es siempre ⌊log₂(n)⌋.

  2. HEAP PROPERTY (orden):
     Min-heap: arr[parent(i)] ≤ arr[i] para todo i.
     Max-heap: arr[parent(i)] ≥ arr[i] para todo i.
     → La raíz es siempre el mínimo (o máximo).

  ¿Qué NO garantiza?
    NO garantiza orden entre siblings.
    En [10, 20, 15, 30, 40]:
      20 y 15 son hijos de 10. ¿20 < 15? No importa.
      30 y 40 son hijos de 20. ¿30 < 40? No importa.
    Solo importa que cada padre ≤ cada hijo.
    → El heap NO da datos "ordenados". Solo da acceso al extremo.

  Comparación de overhead por elemento:

    Estructura         Overhead/elem   Cache behavior
    ─────────          ─────────────   ──────────────
    Binary heap (array) 0 bytes        Excelente (contiguo)
    BST node           24-32 bytes     Malo (nodos dispersos)
    Red-Black node     25-40 bytes     Malo (nodos dispersos)
    Linked list node   8-16 bytes      Terrible (nodos dispersos)

  El heap es la estructura más cache-friendly de las que hemos visto
  para acceder al extremo, porque ES un array.
```

**Preguntas:**

1. ¿Un heap de 1M elementos tiene altura exactamente ⌊log₂(1M)⌋ = 19?

2. ¿El heap NO ordena siblings. ¿Eso hace la iteración en orden imposible?

3. ¿El overhead de 0 bytes por elemento — ¿es literalmente cero?
   ¿Y para Java con boxed Integer?

4. ¿Un array completo sin "huecos" — ¿por qué eso es importante
   para el cache?

5. ¿Un heap de `int` ocupa exactamente `n × 4` bytes. ¿Un TreeMap?

---

### Ejercicio 8.1.2 — Analizar: sift-up y sift-down paso a paso

**Tipo: Analizar**

```
SIFT-UP (después de insert):
  Insertar 5 en min-heap [10, 20, 15, 30, 40, 25, 18]:

  Paso 0: agregar 5 al final.
  [10, 20, 15, 30, 40, 25, 18, 5]
                                ^  index 7

  Paso 1: parent(7) = 3. arr[3]=30, arr[7]=5. 5 < 30 → swap.
  [10, 20, 15, 5, 40, 25, 18, 30]
               ^                    index 3

  Paso 2: parent(3) = 1. arr[1]=20, arr[3]=5. 5 < 20 → swap.
  [10, 5, 15, 20, 40, 25, 18, 30]
       ^                           index 1

  Paso 3: parent(1) = 0. arr[0]=10, arr[1]=5. 5 < 10 → swap.
  [5, 10, 15, 20, 40, 25, 18, 30]
   ^                                index 0 (raíz, terminado)

  Resultado: 5 es ahora la raíz. 3 swaps. Altura = 3 = ⌊log₂(8)⌋.


SIFT-DOWN (después de extract-min):
  Extract min de [5, 10, 15, 20, 40, 25, 18, 30]:
  Resultado a retornar: 5.

  Paso 0: mover último al tope.
  [30, 10, 15, 20, 40, 25, 18]
   ^                             index 0

  Paso 1: hijos de 0 son 1 (10) y 2 (15). Menor: 10 (index 1). 30 > 10 → swap.
  [10, 30, 15, 20, 40, 25, 18]
       ^                         index 1

  Paso 2: hijos de 1 son 3 (20) y 4 (40). Menor: 20 (index 3). 30 > 20 → swap.
  [10, 20, 15, 30, 40, 25, 18]
               ^                 index 3

  Paso 3: hijos de 3 son 7 (fuera de rango). Nodo 3 es hoja. Terminado.
  [10, 20, 15, 30, 40, 25, 18]

  Resultado: heap restaurado. 2 swaps. Altura = 3.
```

**Preguntas:**

1. ¿Sift-up siempre compara con UN nodo (el padre).
   Sift-down compara con DOS nodos (ambos hijos).
   ¿Eso hace sift-down más costoso?

2. ¿En sift-down, ¿por qué elegimos el hijo MENOR para swap
   (en un min-heap)?

3. ¿El peor caso de sift-up es que el elemento suba hasta la raíz.
   ¿Cuándo pasa?

4. ¿El peor caso de sift-down es que baje hasta una hoja.
   ¿Cuándo pasa?

5. ¿Swap en un array es O(1). ¿En un tree con nodos
   sería más costoso?

---

## Sección 8.2 — Implementar Min-Heap Desde Cero

### Ejercicio 8.2.1 — Implementar: min-heap en Python

**Tipo: Implementar**

```python
class MinHeap:
    def __init__(self):
        self._data = []

    def _parent(self, i): return (i - 1) // 2
    def _left(self, i):   return 2 * i + 1
    def _right(self, i):  return 2 * i + 2

    def _swap(self, i, j):
        self._data[i], self._data[j] = self._data[j], self._data[i]

    def _sift_up(self, i):
        while i > 0:
            p = self._parent(i)
            if self._data[i] < self._data[p]:
                self._swap(i, p)
                i = p
            else:
                break

    def _sift_down(self, i):
        n = len(self._data)
        while True:
            smallest = i
            l, r = self._left(i), self._right(i)
            if l < n and self._data[l] < self._data[smallest]:
                smallest = l
            if r < n and self._data[r] < self._data[smallest]:
                smallest = r
            if smallest != i:
                self._swap(i, smallest)
                i = smallest
            else:
                break

    def push(self, value):
        self._data.append(value)
        self._sift_up(len(self._data) - 1)

    def pop(self) -> object:
        if not self._data:
            raise IndexError("pop from empty heap")
        if len(self._data) == 1:
            return self._data.pop()
        root = self._data[0]
        self._data[0] = self._data.pop()  # move last to root
        self._sift_down(0)
        return root

    def peek(self):
        if not self._data:
            raise IndexError("peek at empty heap")
        return self._data[0]

    def __len__(self):
        return len(self._data)

    def __bool__(self):
        return len(self._data) > 0


# ═══ Test ═══
import random

heap = MinHeap()
data = list(range(1000))
random.shuffle(data)
for d in data:
    heap.push(d)

# Extract all: should come out sorted
extracted = []
while heap:
    extracted.append(heap.pop())

assert extracted == list(range(1000)), "Heap sort failed!"
print(f"Heap sort of 1000 elements: OK")
print(f"First 10: {extracted[:10]}")

# Benchmark
import time
n = 1_000_000
heap = MinHeap()
start = time.perf_counter_ns()
for i in range(n):
    heap.push(random.randint(0, n * 10))
push_time = time.perf_counter_ns() - start

start = time.perf_counter_ns()
for _ in range(n):
    heap.pop()
pop_time = time.perf_counter_ns() - start

print(f"\nBenchmark ({n:,} elements):")
print(f"  push: {push_time // 1_000_000} ms ({push_time // n} ns/op)")
print(f"  pop:  {pop_time // 1_000_000} ms ({pop_time // n} ns/op)")
```

**Preguntas:**

1. ¿Extractar todos los elementos produce datos ordenados.
   ¿Es heap sort?

2. ¿`_sift_down` compara con el hijo MÁS PEQUEÑO, no con ambos
   secuencialmente. ¿Es una optimización?

3. ¿Python `heapq` del stdlib es más rápido. ¿Por qué?

4. ¿Un max-heap se implementa invirtiendo las comparaciones.
   ¿O negando los valores?

5. ¿`self._data.pop()` al final del array es O(1).
   ¿Si fuera al inicio?

---

### Ejercicio 8.2.2 — Implementar: min-heap genérico en Java

**Tipo: Implementar**

```java
import java.util.Arrays;

public class MinHeap<T extends Comparable<T>> {

    private Object[] data;
    private int size;

    public MinHeap() { this(16); }
    public MinHeap(int capacity) { data = new Object[capacity]; }

    @SuppressWarnings("unchecked")
    private T get(int i) { return (T) data[i]; }

    private void swap(int i, int j) {
        Object tmp = data[i]; data[i] = data[j]; data[j] = tmp;
    }

    private void siftUp(int i) {
        while (i > 0) {
            int parent = (i - 1) / 2;
            if (get(i).compareTo(get(parent)) < 0) {
                swap(i, parent);
                i = parent;
            } else break;
        }
    }

    private void siftDown(int i) {
        while (true) {
            int smallest = i;
            int l = 2 * i + 1, r = 2 * i + 2;
            if (l < size && get(l).compareTo(get(smallest)) < 0) smallest = l;
            if (r < size && get(r).compareTo(get(smallest)) < 0) smallest = r;
            if (smallest != i) { swap(i, smallest); i = smallest; }
            else break;
        }
    }

    public void push(T value) {
        if (size == data.length) data = Arrays.copyOf(data, data.length * 2);
        data[size] = value;
        siftUp(size);
        size++;
    }

    @SuppressWarnings("unchecked")
    public T pop() {
        if (size == 0) throw new java.util.NoSuchElementException();
        T root = get(0);
        data[0] = data[--size];
        data[size] = null; // help GC
        siftDown(0);
        return root;
    }

    public T peek() {
        if (size == 0) throw new java.util.NoSuchElementException();
        return get(0);
    }

    public int size() { return size; }
    public boolean isEmpty() { return size == 0; }

    public static void main(String[] args) {
        var heap = new MinHeap<Integer>();
        var rng = new java.util.Random(42);
        int n = 1_000_000;

        // Benchmark push
        long start = System.nanoTime();
        for (int i = 0; i < n; i++) heap.push(rng.nextInt(n * 10));
        long pushTime = System.nanoTime() - start;

        // Benchmark pop
        start = System.nanoTime();
        int prev = Integer.MIN_VALUE;
        boolean sorted = true;
        for (int i = 0; i < n; i++) {
            int val = heap.pop();
            if (val < prev) sorted = false;
            prev = val;
        }
        long popTime = System.nanoTime() - start;

        System.out.printf("MinHeap (%,d elements):%n", n);
        System.out.printf("  push: %d ms (%.0f ns/op)%n", pushTime / 1_000_000, (double) pushTime / n);
        System.out.printf("  pop:  %d ms (%.0f ns/op)%n", popTime / 1_000_000, (double) popTime / n);
        System.out.printf("  sorted: %s%n", sorted);

        // Compare with PriorityQueue
        var pq = new java.util.PriorityQueue<Integer>(n);
        rng = new java.util.Random(42);
        start = System.nanoTime();
        for (int i = 0; i < n; i++) pq.add(rng.nextInt(n * 10));
        long pqPush = System.nanoTime() - start;

        start = System.nanoTime();
        while (!pq.isEmpty()) pq.poll();
        long pqPop = System.nanoTime() - start;

        System.out.printf("%nPriorityQueue (%,d elements):%n", n);
        System.out.printf("  push: %d ms (%.0f ns/op)%n", pqPush / 1_000_000, (double) pqPush / n);
        System.out.printf("  pop:  %d ms (%.0f ns/op)%n", pqPop / 1_000_000, (double) pqPop / n);
    }
}
// Resultado esperado:
// MinHeap: push ~300-400 ms (~300-400 ns/op), pop ~800-1200 ms (~800-1200 ns/op)
// PriorityQueue: push ~250-350 ms, pop ~700-1000 ms
// Nuestro heap es competitivo con PriorityQueue de JDK.
```

**Preguntas:**

1. ¿`data[size] = null` después de pop — ¿es necesario para el GC?

2. ¿Pop es 2-3× más lento que push. ¿Por qué?

3. ¿Nuestro heap vs `PriorityQueue` de JDK — ¿qué diferencias hay?

4. ¿`Object[]` con casting — ¿hay una alternativa con generics reales?

5. ¿Autoboxing de `int` a `Integer` — ¿cuánto cuesta en el benchmark?

---

### Ejercicio 8.2.3 — Implementar: min-heap y max-heap genéricos en Rust

**Tipo: Implementar**

```rust
use std::cmp::Ordering;

pub struct BinaryHeap<T> {
    data: Vec<T>,
    compare: fn(&T, &T) -> Ordering, // custom comparator
}

impl<T> BinaryHeap<T> {
    pub fn new_min() -> Self where T: Ord {
        BinaryHeap { data: Vec::new(), compare: |a, b| a.cmp(b) }
    }

    pub fn new_max() -> Self where T: Ord {
        BinaryHeap { data: Vec::new(), compare: |a, b| b.cmp(a) }
    }

    pub fn with_comparator(compare: fn(&T, &T) -> Ordering) -> Self {
        BinaryHeap { data: Vec::new(), compare }
    }

    fn is_less(&self, i: usize, j: usize) -> bool {
        (self.compare)(&self.data[i], &self.data[j]) == Ordering::Less
    }

    fn sift_up(&mut self, mut i: usize) {
        while i > 0 {
            let parent = (i - 1) / 2;
            if self.is_less(i, parent) {
                self.data.swap(i, parent);
                i = parent;
            } else { break; }
        }
    }

    fn sift_down(&mut self, mut i: usize) {
        let n = self.data.len();
        loop {
            let mut smallest = i;
            let l = 2 * i + 1;
            let r = 2 * i + 2;
            if l < n && self.is_less(l, smallest) { smallest = l; }
            if r < n && self.is_less(r, smallest) { smallest = r; }
            if smallest != i {
                self.data.swap(i, smallest);
                i = smallest;
            } else { break; }
        }
    }

    pub fn push(&mut self, value: T) {
        self.data.push(value);
        let last = self.data.len() - 1;
        self.sift_up(last);
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.data.is_empty() { return None; }
        let last = self.data.len() - 1;
        self.data.swap(0, last);
        let result = self.data.pop();
        if !self.data.is_empty() { self.sift_down(0); }
        result
    }

    pub fn peek(&self) -> Option<&T> { self.data.first() }
    pub fn len(&self) -> usize { self.data.len() }
    pub fn is_empty(&self) -> bool { self.data.is_empty() }
}

fn main() {
    // Min-heap
    let mut min_heap: BinaryHeap<i64> = BinaryHeap::new_min();
    for &v in &[30, 10, 50, 5, 20, 40, 15] {
        min_heap.push(v);
    }
    print!("Min-heap extract order: ");
    while let Some(v) = min_heap.pop() { print!("{} ", v); }
    println!(); // 5 10 15 20 30 40 50

    // Max-heap
    let mut max_heap: BinaryHeap<i64> = BinaryHeap::new_max();
    for &v in &[30, 10, 50, 5, 20, 40, 15] {
        max_heap.push(v);
    }
    print!("Max-heap extract order: ");
    while let Some(v) = max_heap.pop() { print!("{} ", v); }
    println!(); // 50 40 30 20 15 10 5

    // Benchmark
    let n = 1_000_000usize;
    let mut heap = BinaryHeap::new_min();
    let start = std::time::Instant::now();
    for i in 0..n { heap.push((i as i64 * 7919) % (n as i64 * 10)); }
    let push_time = start.elapsed();
    let start = std::time::Instant::now();
    while heap.pop().is_some() {}
    let pop_time = start.elapsed();

    println!("\nBenchmark ({} elements):", n);
    println!("  push: {:?} ({} ns/op)", push_time, push_time.as_nanos() / n as u128);
    println!("  pop:  {:?} ({} ns/op)", pop_time, pop_time.as_nanos() / n as u128);
}
```

**Preguntas:**

1. ¿`fn(&T, &T) -> Ordering` como comparador — ¿Rust `std::collections::BinaryHeap`
   usa el mismo enfoque?

2. ¿Rust `BinaryHeap` de std es un MAX-heap. ¿Por qué no min-heap?

3. ¿`self.data.swap(0, last)` + `pop()` — ¿es más eficiente
   que copiar la raíz y mover el último?

4. ¿Sin GC, ¿el `pop()` libera memoria inmediatamente?

5. ¿Nuestro heap debería ser más rápido que la versión Java
   por ausencia de boxing y GC?

---

### Ejercicio 8.2.4 — Implementar: min-heap en Go con generics

**Tipo: Implementar**

```go
package main

import (
    "cmp"
    "fmt"
    "math/rand"
    "time"
)

type Heap[T cmp.Ordered] struct {
    data []T
}

func NewHeap[T cmp.Ordered]() *Heap[T] {
    return &Heap[T]{}
}

func (h *Heap[T]) siftUp(i int) {
    for i > 0 {
        parent := (i - 1) / 2
        if h.data[i] < h.data[parent] {
            h.data[i], h.data[parent] = h.data[parent], h.data[i]
            i = parent
        } else {
            break
        }
    }
}

func (h *Heap[T]) siftDown(i int) {
    n := len(h.data)
    for {
        smallest := i
        l, r := 2*i+1, 2*i+2
        if l < n && h.data[l] < h.data[smallest] { smallest = l }
        if r < n && h.data[r] < h.data[smallest] { smallest = r }
        if smallest != i {
            h.data[i], h.data[smallest] = h.data[smallest], h.data[i]
            i = smallest
        } else {
            break
        }
    }
}

func (h *Heap[T]) Push(val T) {
    h.data = append(h.data, val)
    h.siftUp(len(h.data) - 1)
}

func (h *Heap[T]) Pop() (T, bool) {
    if len(h.data) == 0 {
        var zero T
        return zero, false
    }
    root := h.data[0]
    last := len(h.data) - 1
    h.data[0] = h.data[last]
    h.data = h.data[:last]
    if len(h.data) > 0 {
        h.siftDown(0)
    }
    return root, true
}

func (h *Heap[T]) Peek() (T, bool) {
    if len(h.data) == 0 {
        var zero T
        return zero, false
    }
    return h.data[0], true
}

func (h *Heap[T]) Len() int { return len(h.data) }

func main() {
    h := NewHeap[int]()
    n := 1_000_000

    start := time.Now()
    for i := 0; i < n; i++ {
        h.Push(rand.Intn(n * 10))
    }
    pushTime := time.Since(start)

    start = time.Now()
    prev := -1
    sorted := true
    for h.Len() > 0 {
        v, _ := h.Pop()
        if v < prev { sorted = false }
        prev = v
    }
    popTime := time.Since(start)

    fmt.Printf("Heap (%d elements):\n", n)
    fmt.Printf("  push: %v (%.0f ns/op)\n", pushTime, float64(pushTime.Nanoseconds())/float64(n))
    fmt.Printf("  pop:  %v (%.0f ns/op)\n", popTime, float64(popTime.Nanoseconds())/float64(n))
    fmt.Printf("  sorted: %v\n", sorted)
}
```

**Preguntas:**

1. ¿`cmp.Ordered` de Go 1.21 — ¿es el constraint correcto para un heap?

2. ¿Go stdlib tiene `container/heap` que opera sobre interfaces.
   ¿Es más flexible pero más lento?

3. ¿`h.data = h.data[:last]` para el pop — ¿shrinkea el backing array?

4. ¿Go generics eliminan el overhead de interfaces para este caso?

5. ¿Para un heap de structs con campo priority,
   ¿cómo adaptarías esta implementación?

---

## Sección 8.3 — Heapify: Construir un Heap en O(n)

### Ejercicio 8.3.1 — Leer: por qué heapify es O(n) y no O(n log n)

**Tipo: Leer**

```
Enfoque naive: insertar elementos uno por uno.
  n inserts × O(log n) por insert = O(n log n).

Heapify (Floyd's algorithm):
  Empezar desde el último nodo INTERNO y hacer sift-down hacia la raíz.
  Parece O(n log n): n nodos × O(log n) sift-down.
  Pero es O(n). ¿Por qué?

  Conteo preciso de swaps:
    Nivel h (contando desde abajo, h=0 son las hojas):
    - Número de nodos en el nivel h: ≤ n / 2^(h+1)
    - Cada nodo puede bajar como máximo h niveles.
    - Trabajo en el nivel h: (n / 2^(h+1)) × h

    Trabajo total: Σ(h=0 a log n) de n × h / 2^(h+1)
                 = n × Σ(h=0 a ∞) h / 2^(h+1)
                 = n × 2   (la serie converge a 2)
                 = O(n)

  Intuición: la MAYORÍA de los nodos están cerca de las hojas.
    n/2 nodos son hojas (h=0): 0 swaps cada uno.
    n/4 nodos están a altura 1: ≤ 1 swap cada uno.
    n/8 nodos están a altura 2: ≤ 2 swaps cada uno.
    ...
    1 nodo (raíz) está a altura log n: ≤ log n swaps.

    Solo 1 nodo hace O(log n) trabajo.
    La mitad de los nodos no hacen nada.
    → El trabajo total es O(n).

  Esto es importante porque:
    1. Construir un heap de un array es O(n), no O(n log n).
    2. Heap sort empieza con heapify O(n) + n extract-min O(log n) = O(n log n).
    3. En Spark, construir un heap para merge de archivos ordenados es O(n).
```

**Preguntas:**

1. ¿La serie Σ h/2^h converge a 2. ¿Es fácil demostrarlo?

2. ¿Heapify de abajo hacia arriba vs de arriba hacia abajo
   — ¿por qué bottom-up es O(n)?

3. ¿Java `PriorityQueue(Collection)` usa heapify?

4. ¿Para un stream de datos (no conoces todos los elementos de antemano),
   ¿puedes usar heapify?

5. ¿Heapify es un ejemplo de "la mayoría de los nodos están
   en los niveles inferiores". ¿Qué otras estructuras explotan esto?

---

### Ejercicio 8.3.2 — Implementar: heapify y heapsort en Python

**Tipo: Implementar**

```python
def heapify(arr):
    """Build a min-heap in-place in O(n)"""
    n = len(arr)
    # Start from last internal node, sift down to root
    for i in range(n // 2 - 1, -1, -1):
        _sift_down(arr, i, n)

def _sift_down(arr, i, n):
    while True:
        smallest = i
        l, r = 2 * i + 1, 2 * i + 2
        if l < n and arr[l] < arr[smallest]: smallest = l
        if r < n and arr[r] < arr[smallest]: smallest = r
        if smallest != i:
            arr[i], arr[smallest] = arr[smallest], arr[i]
            i = smallest
        else:
            break

def heapsort(arr):
    """Sort array in-place using max-heap"""
    n = len(arr)
    # Build max-heap (reverse comparisons)
    def sift_down_max(i, size):
        while True:
            largest = i
            l, r = 2 * i + 1, 2 * i + 2
            if l < size and arr[l] > arr[largest]: largest = l
            if r < size and arr[r] > arr[largest]: largest = r
            if largest != i:
                arr[i], arr[largest] = arr[largest], arr[i]
                i = largest
            else: break

    for i in range(n // 2 - 1, -1, -1):
        sift_down_max(i, n)

    # Extract max repeatedly
    for end in range(n - 1, 0, -1):
        arr[0], arr[end] = arr[end], arr[0]  # move max to end
        sift_down_max(0, end)

# Test
import random, time

data = list(range(1_000_000))
random.shuffle(data)

# Heapify benchmark
copy1 = data[:]
start = time.perf_counter_ns()
heapify(copy1)
heapify_time = time.perf_counter_ns() - start

# Verify heap property
is_heap = all(copy1[(i-1)//2] <= copy1[i] for i in range(1, len(copy1)))
print(f"Heapify {len(data):,}: {heapify_time // 1_000_000} ms, valid={is_heap}")

# Heapsort benchmark
copy2 = data[:]
start = time.perf_counter_ns()
heapsort(copy2)
sort_time = time.perf_counter_ns() - start

is_sorted = all(copy2[i] <= copy2[i+1] for i in range(len(copy2)-1))
print(f"Heapsort {len(data):,}: {sort_time // 1_000_000} ms, sorted={is_sorted}")

# Compare with built-in sort (Timsort)
copy3 = data[:]
start = time.perf_counter_ns()
copy3.sort()
timsort_time = time.perf_counter_ns() - start
print(f"Timsort {len(data):,}: {timsort_time // 1_000_000} ms")
print(f"Heapsort/Timsort ratio: {sort_time / timsort_time:.1f}×")
# Expected: heapsort ~3-5× slower than Timsort (cache misses)
```

**Preguntas:**

1. ¿Heapify tarda significativamente menos que heapsort. ¿Esperado?

2. ¿Heapsort es 3-5× más lento que Timsort. ¿Cache misses?

3. ¿Heapsort es in-place (O(1) extra memory). ¿Timsort?

4. ¿Heapsort no es estable. ¿Eso importa en la práctica?

5. ¿El heapify bottom-up hace cuántos swaps para 1M elementos?

---

## Sección 8.4 — Priority Queues: la Abstracción Sobre el Heap

### Ejercicio 8.4.1 — Analizar: priority queue vs sorted structure

**Tipo: Analizar**

```
Una priority queue ofrece:
  - insert(element, priority): agregar con prioridad.
  - extractMin(): extraer el de mayor prioridad (menor valor).
  - peek(): ver el de mayor prioridad sin extraer.

Implementaciones posibles:

  Implementación          insert     extractMin   peek
  ──────────────          ──────     ──────────   ────
  Unsorted array          O(1)       O(n)         O(n)
  Sorted array            O(n)       O(1)         O(1)
  Binary heap             O(log n)   O(log n)     O(1)
  BST / Red-Black tree    O(log n)   O(log n)     O(log n)*
  Fibonacci heap          O(1) am.   O(log n) am. O(1)

  * O(1) si cachead leftmost.

  El binary heap es la implementación estándar porque:
  1. O(log n) para insert Y extractMin (buen balance).
  2. O(1) para peek (la raíz siempre es el mínimo).
  3. Implementación simple (array, sin punteros).
  4. Excelente cache locality.
  5. No necesita decrease-key para la mayoría de casos de uso.

  APIs en cada lenguaje:
    Java:   PriorityQueue<T> — binary heap, min-heap.
    Python: heapq — funciones sobre lista, min-heap.
    Go:     container/heap — interface, custom order.
    Rust:   BinaryHeap<T> — binary heap, MAX-heap.
    Scala:  mutable.PriorityQueue — MAX-heap.

  Nota: Java y Python son min-heap por defecto.
        Rust y Scala son MAX-heap por defecto.
        → Siempre verificar la documentación.
```

**Preguntas:**

1. ¿Rust `BinaryHeap` es max-heap. ¿Cómo usarlo como min-heap?

2. ¿Python `heapq` opera sobre listas existentes (no es una clase).
   ¿Es un buen diseño?

3. ¿Go `container/heap` usa interfaces. ¿Cuánto overhead agrega?

4. ¿Para un job scheduler que necesita "siguiente tarea más urgente",
   ¿min-heap o max-heap?

5. ¿Un priority queue thread-safe — ¿Java `PriorityBlockingQueue`?

---

### Ejercicio 8.4.2 — Implementar: top-K con un heap

**Tipo: Implementar**

```java
// Encontrar los K elementos más grandes de un stream de N elementos.
// Usar un min-heap de tamaño K.

import java.util.PriorityQueue;
import java.util.Random;

public class TopK {

    public static int[] topK(int[] data, int k) {
        // Min-heap de tamaño K
        var heap = new PriorityQueue<Integer>(k);
        for (int val : data) {
            if (heap.size() < k) {
                heap.add(val);
            } else if (val > heap.peek()) {
                heap.poll(); // remove the smallest of top-K
                heap.add(val);
            }
        }
        return heap.stream().sorted().mapToInt(Integer::intValue).toArray();
    }

    public static void main(String[] args) {
        var rng = new Random(42);
        int n = 10_000_000;
        int k = 100;
        int[] data = new int[n];
        for (int i = 0; i < n; i++) data[i] = rng.nextInt(n);

        // TopK with heap
        long start = System.nanoTime();
        int[] result = topK(data, k);
        long heapTime = System.nanoTime() - start;

        // TopK with full sort
        start = System.nanoTime();
        java.util.Arrays.sort(data);
        int[] sortResult = new int[k];
        System.arraycopy(data, n - k, sortResult, 0, k);
        long sortTime = System.nanoTime() - start;

        System.out.printf("Top-%d from %,d elements:%n", k, n);
        System.out.printf("  Heap:   %d ms%n", heapTime / 1_000_000);
        System.out.printf("  Sort:   %d ms%n", sortTime / 1_000_000);
        System.out.printf("  Ratio:  %.1f× (heap faster)%n", (double) sortTime / heapTime);
        System.out.printf("  Memory: heap=O(%d), sort=O(%,d)%n", k, n);
    }
}
// Resultado esperado:
// Top-100 from 10,000,000 elements:
//   Heap:   ~50-80 ms
//   Sort:   ~800-1200 ms
//   Ratio:  ~10-20× (heap faster)
//   Memory: heap=O(100), sort=O(10,000,000)
```

**Preguntas:**

1. ¿Top-K con min-heap usa O(K) memoria, no O(N). ¿Por qué?

2. ¿El heap approach es O(N log K). ¿Sort es O(N log N)?

3. ¿Para K cercano a N (ej: top 50%), ¿sigue siendo mejor el heap?

4. ¿Quickselect puede encontrar el K-ésimo en O(N) esperado.
   ¿Es mejor que el heap?

5. ¿Spark `takeOrdered(K)` usa un heap internamente?

---

### Ejercicio 8.4.3 — Implementar: merge de K streams ordenados

**Tipo: Implementar**

```java
// K-way merge: combinar K listas ordenadas en una sola lista ordenada.
// Usado en Spark shuffle merge, external sort, merge de SSTables.

import java.util.*;

public class KWayMerge {

    static int[] mergeKSorted(int[][] lists) {
        // Min-heap de (value, listIndex, elementIndex)
        var heap = new PriorityQueue<int[]>(
            Comparator.comparingInt(a -> a[0])
        );

        // Initialize: push first element of each list
        int totalSize = 0;
        for (int i = 0; i < lists.length; i++) {
            if (lists[i].length > 0) {
                heap.add(new int[]{lists[i][0], i, 0});
                totalSize += lists[i].length;
            }
        }

        int[] result = new int[totalSize];
        int idx = 0;

        while (!heap.isEmpty()) {
            int[] top = heap.poll();
            result[idx++] = top[0];

            int listIdx = top[1];
            int elemIdx = top[2] + 1;
            if (elemIdx < lists[listIdx].length) {
                heap.add(new int[]{lists[listIdx][elemIdx], listIdx, elemIdx});
            }
        }

        return result;
    }

    public static void main(String[] args) {
        int k = 100;         // number of sorted lists
        int perList = 100_000; // elements per list
        int[][] lists = new int[k][perList];
        var rng = new Random(42);

        // Create K sorted lists
        for (int i = 0; i < k; i++) {
            for (int j = 0; j < perList; j++)
                lists[i][j] = rng.nextInt(k * perList * 10);
            Arrays.sort(lists[i]);
        }

        long start = System.nanoTime();
        int[] merged = mergeKSorted(lists);
        long elapsed = System.nanoTime() - start;

        // Verify sorted
        boolean sorted = true;
        for (int i = 1; i < merged.length; i++) {
            if (merged[i] < merged[i - 1]) { sorted = false; break; }
        }

        System.out.printf("K-way merge: K=%d, %,d elements/list, total=%,d%n", k, perList, merged.length);
        System.out.printf("Time: %d ms%n", elapsed / 1_000_000);
        System.out.printf("Sorted: %s%n", sorted);
        System.out.printf("Complexity: O(N log K) where N=%,d, K=%d%n", merged.length, k);
    }
}
// Resultado esperado:
// K-way merge: K=100, 100,000 elements/list, total=10,000,000
// Time: ~1500-3000 ms
// Sorted: true
// Complexity: O(N log K) where N=10,000,000, K=100
```

**Preguntas:**

1. ¿K-way merge es O(N log K). ¿Si K=1 es O(N)?

2. ¿El heap siempre tiene exactamente K elementos (uno por lista).
   ¿Eso limita la memoria?

3. ¿Spark usa K-way merge durante shuffle. ¿K es el número de reducers?

4. ¿External sort (datos que no caben en RAM) usa K-way merge.
   ¿Cómo?

5. ¿Cassandra compaction merge de SSTables usa K-way merge?

---

## Sección 8.5 — D-ary Heaps: Más Hijos, Menos Profundidad

### Ejercicio 8.5.1 — Leer: por qué 2 hijos no siempre es óptimo

**Tipo: Leer**

```
Un binary heap (d=2) tiene profundidad log₂(n).
Un d-ary heap tiene profundidad log_d(n).

  d=2:  profundidad = log₂(1M) ≈ 20
  d=4:  profundidad = log₄(1M) ≈ 10
  d=8:  profundidad = log₈(1M) ≈ 7
  d=16: profundidad = log₁₆(1M) ≈ 5

  Menos profundidad → menos swaps en sift-up.
  Pero sift-down compara con TODOS los d hijos en cada nivel.

  Tradeoff:
    sift-up:   O(log_d n) comparaciones. Mejora con d grande.
    sift-down: O(d × log_d n) comparaciones. Empeora con d grande.

  Para INSERT-HEAVY workloads (más sift-up que sift-down):
    → d grande (4, 8) es mejor.

  Para EXTRACT-HEAVY workloads (más sift-down que sift-up):
    → d pequeño (2, 3) es mejor.

  Cache effects:
    d=4: los 4 hijos de un nodo están en posiciones 4i+1 a 4i+4.
    Con int de 4 bytes: 4 hijos = 16 bytes = 1/4 de una cache line.
    → Los 4 hijos probablemente están en la misma cache line.
    → Comparar 4 hijos es casi tan rápido como comparar 2.

  En la práctica, d=4 es frecuentemente óptimo:
    - Profundidad reducida a la mitad vs d=2.
    - Los 4 hijos caben en una cache line.
    - La comparación extra (4 vs 2) se oculta en el cache hit.
```

**Preguntas:**

1. ¿d=4 duplica las comparaciones por nivel pero reduce la profundidad
   a la mitad. ¿Es un empate o gana d=4?

2. ¿Los 4 hijos en la misma cache line — ¿es siempre cierto?

3. ¿Para un Fibonacci heap, d es efectivamente variable. ¿Eso importa?

4. ¿Java `PriorityQueue` usa d=2. ¿Podría beneficiarse de d=4?

5. ¿Los scheduling frameworks (Airflow, Celery) — ¿les importa d?

---

### Ejercicio 8.5.2 — Implementar: benchmark d-ary heap

**Tipo: Implementar**

```java
public class DAryHeapBenchmark {

    static class DAryHeap {
        int[] data;
        int size;
        final int d;

        DAryHeap(int capacity, int d) { this.data = new int[capacity]; this.d = d; }

        void push(int val) {
            data[size] = val;
            siftUp(size);
            size++;
        }

        int pop() {
            int root = data[0];
            data[0] = data[--size];
            siftDown(0);
            return root;
        }

        private void siftUp(int i) {
            while (i > 0) {
                int parent = (i - 1) / d;
                if (data[i] < data[parent]) {
                    int tmp = data[i]; data[i] = data[parent]; data[parent] = tmp;
                    i = parent;
                } else break;
            }
        }

        private void siftDown(int i) {
            while (true) {
                int smallest = i;
                int firstChild = d * i + 1;
                int lastChild = Math.min(firstChild + d, size);
                for (int c = firstChild; c < lastChild; c++) {
                    if (data[c] < data[smallest]) smallest = c;
                }
                if (smallest != i) {
                    int tmp = data[i]; data[i] = data[smallest]; data[smallest] = tmp;
                    i = smallest;
                } else break;
            }
        }
    }

    public static void main(String[] args) {
        int n = 5_000_000;
        var rng = new java.util.Random(42);
        int[] values = new int[n];
        for (int i = 0; i < n; i++) values[i] = rng.nextInt(n * 10);

        System.out.printf("Benchmark: %,d push + %,d pop%n%n", n, n);
        System.out.printf("%-6s  %10s  %10s  %10s%n", "d", "Push (ms)", "Pop (ms)", "Total (ms)");
        System.out.println("─".repeat(42));

        for (int d : new int[]{2, 3, 4, 5, 6, 8, 16}) {
            var heap = new DAryHeap(n, d);
            long start = System.nanoTime();
            for (int v : values) heap.push(v);
            long pushMs = (System.nanoTime() - start) / 1_000_000;

            start = System.nanoTime();
            for (int i = 0; i < n; i++) heap.pop();
            long popMs = (System.nanoTime() - start) / 1_000_000;

            System.out.printf("%-6d  %10d  %10d  %10d%n", d, pushMs, popMs, pushMs + popMs);
        }
    }
}
// Resultado esperado (varía):
// d       Push (ms)    Pop (ms)   Total (ms)
// ──────────────────────────────────────────
// 2            450        1200        1650
// 3            380        1050        1430
// 4            350         950        1300  ← sweet spot
// 5            340        1000        1340
// 6            340        1050        1390
// 8            330        1150        1480
// 16           320        1500        1820
```

**Preguntas:**

1. ¿d=4 es el sweet spot. ¿Es consistente entre diferentes CPUs?

2. ¿Push mejora monotónicamente con d. ¿Pop empeora después de d=4?

3. ¿d=16 tiene push más rápido pero pop mucho más lento. ¿Por qué?

4. ¿Para un workload de 90% push + 10% pop, ¿d=8 sería mejor?

5. ¿Los benchmarks usan `int[]` (primitivos). ¿Con Object[] cambiaría?

---

## Sección 8.6 — Indexed Priority Queue y Decrease-Key

### Ejercicio 8.6.1 — Leer: el problema de decrease-key

**Tipo: Leer**

```
En un heap estándar, no puedes encontrar un elemento por key.
Solo accedes al mínimo (raíz).

  Operación decrease-key(key, newPriority):
    Encontrar el elemento con esta key.
    Reducir su prioridad.
    Restaurar la heap property (sift-up).

  ¿Por qué importa?
  - Dijkstra's shortest path: cuando encuentras un camino más corto
    a un nodo, necesitas REDUCIR su distancia en la priority queue.
  - Prim's MST: similar, reduce el peso de la arista al nodo.
  - Job schedulers: actualizar la prioridad de una tarea en ejecución.

  Sin decrease-key:
    Dijkstra usa "lazy deletion":
    Insertar el nodo con la nueva prioridad (sin eliminar el viejo).
    Al extraer, verificar si ya fue procesado → skip.
    → Funciona, pero el heap crece a O(E) en vez de O(V).

  Con decrease-key eficiente:
    El heap siempre tiene exactamente V elementos.
    → Menos memoria, menos comparaciones.

  Indexed Priority Queue:
    Un heap + un array de posiciones.
    Para cada key K, positions[K] indica dónde está K en el heap.
    decrease-key(K, newPriority):
      1. Buscar K en el heap: positions[K] → O(1).
      2. Actualizar la prioridad.
      3. Sift-up desde positions[K] → O(log n).
    Total: O(log n) en vez de O(n) sin el índice.
```

**Preguntas:**

1. ¿"Lazy deletion" en Dijkstra — ¿funciona correctamente?

2. ¿El array de posiciones añade O(n) de memoria extra. ¿Vale la pena?

3. ¿Fibonacci heap hace decrease-key en O(1) amortizado.
   ¿Por qué no se usa en la práctica?

4. ¿Para Dijkstra con un grafo de 1M nodos y 10M aristas,
   ¿cuántos decrease-keys hay?

5. ¿Un indexed priority queue thread-safe — ¿es difícil de implementar?

---

### Ejercicio 8.6.2 — Implementar: indexed priority queue en Java

**Tipo: Implementar**

```java
// Indexed Priority Queue: min-heap con decrease-key en O(log n)

public class IndexedMinPQ {

    private final int maxN;
    private int size;
    private final int[] heap;     // heap[i] = key at position i in heap
    private final int[] pos;      // pos[key] = position of key in heap (-1 if absent)
    private final double[] prio;  // prio[key] = priority of key

    public IndexedMinPQ(int maxN) {
        this.maxN = maxN;
        heap = new int[maxN];
        pos = new int[maxN];
        prio = new double[maxN];
        java.util.Arrays.fill(pos, -1);
    }

    public boolean contains(int key) { return pos[key] != -1; }
    public boolean isEmpty() { return size == 0; }
    public int size() { return size; }
    public int peekKey() { return heap[0]; }
    public double peekPriority() { return prio[heap[0]]; }

    public void insert(int key, double priority) {
        prio[key] = priority;
        heap[size] = key;
        pos[key] = size;
        siftUp(size);
        size++;
    }

    public int extractMin() {
        int minKey = heap[0];
        swap(0, --size);
        pos[minKey] = -1;
        siftDown(0);
        return minKey;
    }

    public void decreaseKey(int key, double newPriority) {
        prio[key] = newPriority;
        siftUp(pos[key]);
    }

    private void siftUp(int i) {
        while (i > 0) {
            int parent = (i - 1) / 2;
            if (prio[heap[i]] < prio[heap[parent]]) {
                swap(i, parent);
                i = parent;
            } else break;
        }
    }

    private void siftDown(int i) {
        while (true) {
            int smallest = i;
            int l = 2 * i + 1, r = 2 * i + 2;
            if (l < size && prio[heap[l]] < prio[heap[smallest]]) smallest = l;
            if (r < size && prio[heap[r]] < prio[heap[smallest]]) smallest = r;
            if (smallest != i) { swap(i, smallest); i = smallest; }
            else break;
        }
    }

    private void swap(int i, int j) {
        int tmp = heap[i]; heap[i] = heap[j]; heap[j] = tmp;
        pos[heap[i]] = i;
        pos[heap[j]] = j;
    }

    // Dijkstra demo
    public static void main(String[] args) {
        // Small graph: 5 nodes, weighted edges
        //   0 --(4)--> 1
        //   0 --(1)--> 2
        //   2 --(2)--> 1
        //   1 --(5)--> 3
        //   2 --(8)--> 3
        //   3 --(3)--> 4
        //   1 --(1)--> 4

        int V = 5;
        int[][] adj = {
            {1, 4}, {2, 1},           // from 0
            {3, 5}, {4, 1},           // from 1
            {1, 2}, {3, 8},           // from 2
            {4, 3},                   // from 3
        };
        int[][] from = {{0,0}, {0,0}, {1,1}, {2,2}, {3,3}};
        // Simplified adjacency list
        int[][][] graph = {
            {{1,4},{2,1}},
            {{3,5},{4,1}},
            {{1,2},{3,8}},
            {{4,3}},
            {}
        };

        double[] dist = new double[V];
        java.util.Arrays.fill(dist, Double.MAX_VALUE);
        dist[0] = 0;

        var pq = new IndexedMinPQ(V);
        pq.insert(0, 0);

        while (!pq.isEmpty()) {
            int u = pq.extractMin();
            for (int[] edge : graph[u]) {
                int v = edge[0]; double w = edge[1];
                if (dist[u] + w < dist[v]) {
                    dist[v] = dist[u] + w;
                    if (pq.contains(v)) pq.decreaseKey(v, dist[v]);
                    else pq.insert(v, dist[v]);
                }
            }
        }

        System.out.println("Dijkstra shortest paths from node 0:");
        for (int i = 0; i < V; i++)
            System.out.printf("  0 → %d: %.0f%n", i, dist[i]);
        // Expected: 0→0: 0, 0→1: 3, 0→2: 1, 0→3: 8, 0→4: 4
    }
}
```

**Preguntas:**

1. ¿El array `pos[]` mantiene las posiciones sincronizadas con `heap[]`.
   ¿Cada swap actualiza ambos?

2. ¿`decreaseKey` solo hace sift-up. ¿Podría necesitar sift-down
   si la prioridad aumenta?

3. ¿Para Dijkstra con V=1M, el indexed PQ usa cuánta memoria extra?

4. ¿Sin indexed PQ, Dijkstra con lazy deletion funciona.
   ¿Es más lento? ¿Cuánto?

5. ¿A* search usa una priority queue. ¿Necesita decrease-key?

---

## Sección 8.7 — Heaps en Producción: Schedulers, Merges, y Timers

### Ejercicio 8.7.1 — Implementar: event scheduler con heap en Go

**Tipo: Implementar**

```go
package main

import (
    "fmt"
    "time"
)

type Event struct {
    fireAt time.Time
    name   string
}

type EventScheduler struct {
    events []Event
}

func (s *EventScheduler) siftUp(i int) {
    for i > 0 {
        parent := (i - 1) / 2
        if s.events[i].fireAt.Before(s.events[parent].fireAt) {
            s.events[i], s.events[parent] = s.events[parent], s.events[i]
            i = parent
        } else { break }
    }
}

func (s *EventScheduler) siftDown(i int) {
    n := len(s.events)
    for {
        smallest := i
        l, r := 2*i+1, 2*i+2
        if l < n && s.events[l].fireAt.Before(s.events[smallest].fireAt) { smallest = l }
        if r < n && s.events[r].fireAt.Before(s.events[smallest].fireAt) { smallest = r }
        if smallest != i {
            s.events[i], s.events[smallest] = s.events[smallest], s.events[i]
            i = smallest
        } else { break }
    }
}

func (s *EventScheduler) Schedule(name string, delay time.Duration) {
    event := Event{fireAt: time.Now().Add(delay), name: name}
    s.events = append(s.events, event)
    s.siftUp(len(s.events) - 1)
}

func (s *EventScheduler) NextEvent() (Event, bool) {
    if len(s.events) == 0 {
        return Event{}, false
    }
    event := s.events[0]
    last := len(s.events) - 1
    s.events[0] = s.events[last]
    s.events = s.events[:last]
    if len(s.events) > 0 {
        s.siftDown(0)
    }
    return event, true
}

func (s *EventScheduler) PeekNext() (Event, bool) {
    if len(s.events) == 0 { return Event{}, false }
    return s.events[0], true
}

func (s *EventScheduler) Len() int { return len(s.events) }

func main() {
    sched := &EventScheduler{}

    sched.Schedule("send_email", 500*time.Millisecond)
    sched.Schedule("process_payment", 100*time.Millisecond)
    sched.Schedule("generate_report", 2*time.Second)
    sched.Schedule("notify_user", 200*time.Millisecond)
    sched.Schedule("cleanup_temp", 1*time.Second)

    fmt.Println("Event execution order:")
    for sched.Len() > 0 {
        event, _ := sched.NextEvent()
        fmt.Printf("  [%v] %s\n",
            event.fireAt.Format("15:04:05.000"), event.name)
    }
    // Events come out sorted by fireAt time (earliest first)
}
```

**Preguntas:**

1. ¿Este scheduler ejecuta eventos en orden de tiempo.
   ¿Es como un timer wheel?

2. ¿Kafka `DelayedOperationPurgatory` usa un esquema similar?

3. ¿Airflow ordena tareas por prioridad con una PQ. ¿Es lo mismo?

4. ¿Un timer wheel (hashed timing wheel) es más eficiente
   que un heap para timers. ¿Cuándo?

5. ¿Para 10M timers pendientes, ¿el heap degrada?

---

### Ejercicio 8.7.2 — Analizar: heaps en la infraestructura de datos

**Tipo: Analizar**

```
El heap aparece en puntos críticos de la infraestructura:

  SPARK SHUFFLE MERGE:
    Spark shuffle produce N archivos ordenados por reducer.
    El merge phase combina estos N archivos en uno.
    → K-way merge con un min-heap de K elementos.
    → K = número de particiones. Típicamente 200-10,000.
    → El heap tiene K entries = pocos KB. El costo es I/O, no el heap.

  KAFKA DelayedOperationPurgatory:
    Kafka tiene operaciones que deben ejecutarse "después de un delay".
    Ej: un produce request tiene un timeout de 30 segundos.
    Las operaciones delayed se almacenan en una priority queue
    ordenada por tiempo de expiración.
    → Un timer thread extrae la operación con menor timeout → heap.

  AIRFLOW SCHEDULER:
    Airflow decide cuál tarea ejecutar siguiente basándose en:
    - Dependencias satisfechas
    - Prioridad del DAG
    - Prioridad del task
    → Priority queue de tasks listos para ejecutar.

  FLINK WATERMARKS:
    Flink necesita saber el menor timestamp entre todos los input streams.
    → Min-heap de watermarks por input stream.

  OPERATING SYSTEM SCHEDULERS:
    Linux timer subsystem: min-heap de timers pendientes.
    El kernel extrae el timer con el menor deadline → heap.
    Cambió de Red-Black tree a heap en kernel 4.x (más simple, más rápido).

  EXTERNAL SORT:
    Datos que no caben en RAM:
    1. Dividir en chunks que caben en RAM.
    2. Ordenar cada chunk (quicksort/mergesort).
    3. Merge de K chunks con K-way merge → heap.
    → El heap tiene K entries, los datos fluyen del disco.
```

**Preguntas:**

1. ¿Linux cambió de Red-Black tree a heap para timers.
   ¿Por qué?

2. ¿Spark shuffle merge con K=10,000 — ¿el heap de 10K entries
   cabe en L1 cache?

3. ¿External sort con K chunks y K-way merge — ¿K óptimo?

4. ¿Flink watermarks como min-heap — ¿se actualiza frecuentemente?

5. ¿El heap es más simple que un Red-Black tree para timers.
   ¿Es también más rápido?

---

### Ejercicio 8.7.3 — Resumen: las reglas del heap

**Tipo: Leer**

```
Reglas nuevas del Cap.08:

  Regla 31: Heap = orden parcial en un array.
    Si solo necesitas el mínimo (o máximo), no ordenes todo.
    Un heap da peek O(1), insert/extract O(log n).
    Sin punteros, sin nodos, sin overhead. Solo un array.

  Regla 32: Heapify es O(n), no O(n log n).
    Construir un heap de un array desordenado cuesta O(n).
    La mitad de los nodos son hojas y no hacen nada.
    Solo 1 nodo (la raíz) hace O(log n) trabajo.

  Regla 33: d=4 suele ser óptimo para d-ary heaps.
    Menos profundidad que d=2, los 4 hijos caben en una cache line.
    Para workloads insert-heavy, d=4 o d=8. Para balanced, d=4.

  Regla 34: K-way merge es el patrón killer del heap.
    Combinar K streams ordenados: heap de K entries.
    O(N log K) total. Usado en Spark shuffles, external sort,
    SSTable compaction, merge de archivos ordenados.

  Regla 35: Indexed PQ cuando necesitas decrease-key.
    Dijkstra, Prim, job schedulers con prioridades dinámicas.
    O(log n) decrease-key con un array de posiciones.
    Fibonacci heap promete O(1) pero la complejidad no compensa.

  Resumen de la Parte 3 hasta ahora:
    Cap.07: Árboles BST/AVL/Red-Black → datos ordenados en memoria.
    Cap.08: Heaps → acceso al extremo, merge de streams, scheduling.
    Cap.09: B-Trees → datos ordenados en DISCO.
    Cap.10: LSM-Trees → escrituras rápidas en disco.
```

**Preguntas:**

1. ¿Las 35 reglas son un manual de referencia útil?

2. ¿La Regla 34 (K-way merge) es la más práctica del capítulo?

3. ¿Falta una regla sobre heapsort?

4. ¿Un data engineer interactúa directamente con heaps
   o solo las usa a través de PriorityQueue?

5. ¿El Cap.09 (B-Trees) es donde todo converge para storage?
