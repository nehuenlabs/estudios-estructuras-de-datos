# Guía de Ejercicios — Cap.13: Queues: de FIFO a Lock-Free a Distributed

> La queue es la estructura más subestimada de la informática.
>
> Kafka es una queue persistente distribuida.
> Cada goroutine de Go se comunica a través de channels (queues).
> Java ForkJoinPool (que Spark usa) tiene work-stealing deques.
> LMAX Disruptor procesa 6 millones de mensajes/segundo con un ring buffer.
> Cada web server tiene una queue de requests pendientes.
> Cada thread pool tiene una queue de tareas.
>
> La queue es simple: FIFO. El primero que entra, el primero que sale.
> Pero la implementación importa enormemente:
>
> Array-based (ring buffer): O(1) amortizado, cache-friendly.
> Linked-list: O(1) siempre, pero peor cache locality.
> Blocking: los consumers esperan cuando está vacía.
> Lock-free: millones de ops/sec sin locks.
>
> Este capítulo va de lo simple (array queue) a lo sofisticado
> (lock-free ring buffer del LMAX Disruptor), pasando por
> blocking queues y work-stealing deques.

---

## El modelo mental: de buffer a canal de comunicación

```
La queue es un BUFFER entre productores y consumidores.

  PRODUCER → [───────── QUEUE ─────────] → CONSUMER
                FIFO: primero in, primero out

  Variantes según el contexto:

  1. UNBOUNDED QUEUE:
     Crece sin límite. Simple. Riesgo: OOM si el consumer es lento.
     Ejemplo: LinkedList como queue.

  2. BOUNDED QUEUE (ring buffer):
     Tamaño fijo. Si está llena: el producer espera o descarta.
     Ejemplo: ArrayBlockingQueue, Go channels con buffer.
     → BACKPRESSURE: el sistema se auto-regula.

  3. BLOCKING QUEUE:
     Consumer se bloquea (duerme) si la queue está vacía.
     Producer se bloquea si está llena.
     → Coordinación entre threads sin busy-waiting.

  4. LOCK-FREE QUEUE:
     Sin locks. Usa CAS (Compare-And-Swap) para coordinación.
     → Máximo throughput bajo alta contención.

  5. WORK-STEALING DEQUE:
     Cada thread tiene su propia deque (double-ended queue).
     Si un thread se queda sin trabajo, "roba" del otro extremo
     de la deque de otro thread.
     → Java ForkJoinPool, Go runtime, Tokio (Rust).

  Ring buffer (array circular):
    Array de tamaño fijo. head y tail avanzan circularmente.

    [_, _, _, x, y, z, _, _]
             ^        ^
             head     tail

    enqueue(w):  arr[tail] = w; tail = (tail + 1) % capacity;
    dequeue():   val = arr[head]; head = (head + 1) % capacity;
    O(1) siempre. Sin allocation. Cache-friendly.
```

---

## Tabla de contenidos

- [Sección 13.1 — Queue básica: array vs linked list](#sección-131--queue-básica-array-vs-linked-list)
- [Sección 13.2 — Ring buffer: la queue de tamaño fijo](#sección-132--ring-buffer-la-queue-de-tamaño-fijo)
- [Sección 13.3 — Blocking queue: coordinación entre threads](#sección-133--blocking-queue-coordinación-entre-threads)
- [Sección 13.4 — Work-stealing deque](#sección-134--work-stealing-deque)
- [Sección 13.5 — Lock-free queue: Michael-Scott y LMAX Disruptor](#sección-135--lock-free-queue-michael-scott-y-lmax-disruptor)
- [Sección 13.6 — Benchmark: blocking vs lock-free bajo contención](#sección-136--benchmark-blocking-vs-lock-free-bajo-contención)
- [Sección 13.7 — Queues en producción: de Kafka a ForkJoinPool](#sección-137--queues-en-producción-de-kafka-a-forkjoinpool)

---

## Sección 13.1 — Queue Básica: Array vs Linked List

### Ejercicio 13.1.1 — Implementar: queue con array dinámico en Python

**Tipo: Implementar**

```python
class ArrayQueue:
    def __init__(self, capacity=16):
        self._data = [None] * capacity
        self._head = 0
        self._tail = 0
        self._size = 0

    def enqueue(self, value):
        if self._size == len(self._data):
            self._resize(len(self._data) * 2)
        self._data[self._tail] = value
        self._tail = (self._tail + 1) % len(self._data)
        self._size += 1

    def dequeue(self):
        if self._size == 0:
            raise IndexError("dequeue from empty queue")
        value = self._data[self._head]
        self._data[self._head] = None
        self._head = (self._head + 1) % len(self._data)
        self._size -= 1
        return value

    def peek(self):
        if self._size == 0:
            raise IndexError("peek at empty queue")
        return self._data[self._head]

    def _resize(self, new_cap):
        new_data = [None] * new_cap
        for i in range(self._size):
            new_data[i] = self._data[(self._head + i) % len(self._data)]
        self._data = new_data
        self._head = 0
        self._tail = self._size

    def __len__(self):
        return self._size

    def __bool__(self):
        return self._size > 0


# ═══ Test ═══
import time, random

q = ArrayQueue()
for i in range(10):
    q.enqueue(i)
print(f"Size: {len(q)}, peek: {q.peek()}")
while q:
    print(q.dequeue(), end=' ')
print()

# Benchmark
n = 1_000_000
q = ArrayQueue()
start = time.perf_counter_ns()
for i in range(n):
    q.enqueue(i)
for i in range(n):
    q.dequeue()
elapsed = time.perf_counter_ns() - start
print(f"\nArrayQueue {n:,} enqueue+dequeue: {elapsed // 1_000_000} ms ({elapsed // (n*2)} ns/op)")
```

**Preguntas:**

1. ¿El ring buffer evita mover elementos en dequeue. ¿Sin ring buffer
   sería O(n) por dequeue?

2. ¿`_resize` copia todos los elementos. ¿Amortizado O(1)?

3. ¿`collections.deque` de Python es más rápido. ¿Por qué?

4. ¿Para una queue que nunca crece más de N, ¿es mejor un ring buffer fijo?

5. ¿Un linked list queue no necesita resize. ¿Cuál es la desventaja?

---

### Ejercicio 13.1.2 — Implementar: queue con linked list en Java

**Tipo: Implementar**

```java
public class LinkedQueue<T> {

    private static class Node<T> {
        T value;
        Node<T> next;
        Node(T value) { this.value = value; }
    }

    private Node<T> head;
    private Node<T> tail;
    private int size;

    public void enqueue(T value) {
        Node<T> node = new Node<>(value);
        if (tail == null) { head = tail = node; }
        else { tail.next = node; tail = node; }
        size++;
    }

    public T dequeue() {
        if (head == null) throw new java.util.NoSuchElementException();
        T value = head.value;
        head = head.next;
        if (head == null) tail = null;
        size--;
        return value;
    }

    public T peek() {
        if (head == null) throw new java.util.NoSuchElementException();
        return head.value;
    }

    public int size() { return size; }
    public boolean isEmpty() { return size == 0; }

    public static void main(String[] args) {
        int n = 1_000_000;
        // Linked queue
        var lq = new LinkedQueue<Integer>();
        long start = System.nanoTime();
        for (int i = 0; i < n; i++) lq.enqueue(i);
        for (int i = 0; i < n; i++) lq.dequeue();
        long linkedTime = System.nanoTime() - start;

        // ArrayDeque (JDK ring buffer)
        var aq = new java.util.ArrayDeque<Integer>(n);
        start = System.nanoTime();
        for (int i = 0; i < n; i++) aq.add(i);
        for (int i = 0; i < n; i++) aq.poll();
        long arrayTime = System.nanoTime() - start;

        System.out.printf("LinkedQueue:  %d ms (%.0f ns/op)%n",
            linkedTime / 1_000_000, (double) linkedTime / (n * 2));
        System.out.printf("ArrayDeque:   %d ms (%.0f ns/op)%n",
            arrayTime / 1_000_000, (double) arrayTime / (n * 2));
        System.out.printf("Ratio: %.1f× (ArrayDeque faster)%n",
            (double) linkedTime / arrayTime);
    }
}
// Resultado esperado:
// LinkedQueue:  ~200-400 ms (~100-200 ns/op) — GC pressure, cache misses
// ArrayDeque:   ~50-100 ms  (~25-50 ns/op)   — contiguous memory
// Ratio: ~3-5× (ArrayDeque faster)
```

**Preguntas:**

1. ¿ArrayDeque es 3-5× más rápido. ¿Solo cache locality?

2. ¿LinkedQueue crea un Node por enqueue. ¿GC pressure?

3. ¿Java `LinkedList` implementa `Queue`. ¿Es eficiente?

4. ¿Para una queue con elementos grandes, ¿la diferencia se reduce?

5. ¿Go channels internamente — ¿son ring buffers?

---

## Sección 13.2 — Ring Buffer: la Queue de Tamaño Fijo

### Ejercicio 13.2.1 — Implementar: ring buffer genérico en Rust

**Tipo: Implementar**

```rust
pub struct RingBuffer<T> {
    data: Vec<Option<T>>,
    head: usize,
    tail: usize,
    size: usize,
    capacity: usize,
}

impl<T> RingBuffer<T> {
    pub fn new(capacity: usize) -> Self {
        let mut data = Vec::with_capacity(capacity);
        for _ in 0..capacity { data.push(None); }
        RingBuffer { data, head: 0, tail: 0, size: 0, capacity }
    }

    pub fn push_back(&mut self, value: T) -> Result<(), T> {
        if self.size == self.capacity { return Err(value); }
        self.data[self.tail] = Some(value);
        self.tail = (self.tail + 1) % self.capacity;
        self.size += 1;
        Ok(())
    }

    pub fn pop_front(&mut self) -> Option<T> {
        if self.size == 0 { return None; }
        let value = self.data[self.head].take();
        self.head = (self.head + 1) % self.capacity;
        self.size -= 1;
        value
    }

    pub fn peek(&self) -> Option<&T> {
        if self.size == 0 { return None; }
        self.data[self.head].as_ref()
    }

    pub fn len(&self) -> usize { self.size }
    pub fn is_empty(&self) -> bool { self.size == 0 }
    pub fn is_full(&self) -> bool { self.size == self.capacity }
}

fn main() {
    let mut rb = RingBuffer::new(8);
    for i in 0..8 { rb.push_back(i).unwrap(); }
    assert!(rb.push_back(99).is_err()); // full!

    for _ in 0..4 { rb.pop_front(); }
    for i in 10..14 { rb.push_back(i).unwrap(); }

    print!("Contents: ");
    while let Some(v) = rb.pop_front() { print!("{} ", v); }
    println!(); // 4 5 6 7 10 11 12 13

    // Benchmark
    let n = 10_000_000usize;
    let mut rb = RingBuffer::new(1024);
    let start = std::time::Instant::now();
    for i in 0..n {
        if rb.is_full() { rb.pop_front(); }
        rb.push_back(i).unwrap();
    }
    let elapsed = start.elapsed();
    println!("RingBuffer {}: {:?} ({} ns/op)", n, elapsed, elapsed.as_nanos() / n as u128);
}
```

**Preguntas:**

1. ¿`push_back` retorna `Err(value)` si está lleno. ¿Es el patrón idiomático
   en Rust para bounded structures?

2. ¿`Option<T>` en cada slot — ¿hay overhead?

3. ¿Un ring buffer de 1024 entries — ¿cabe en L1 cache
   si T es `i64`?

4. ¿El LMAX Disruptor es un ring buffer. ¿Qué lo hace especial?

5. ¿Go channels con buffer son ring buffers. ¿Con qué capacidad?

---

## Sección 13.3 — Blocking Queue: Coordinación entre Threads

### Ejercicio 13.3.1 — Implementar: bounded blocking queue en Java

**Tipo: Implementar**

```java
import java.util.concurrent.locks.*;

public class BoundedBlockingQueue<T> {

    private final Object[] data;
    private int head, tail, size;
    private final int capacity;

    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public BoundedBlockingQueue(int capacity) {
        this.capacity = capacity;
        this.data = new Object[capacity];
    }

    public void put(T value) throws InterruptedException {
        lock.lock();
        try {
            while (size == capacity) notFull.await(); // wait if full
            data[tail] = value;
            tail = (tail + 1) % capacity;
            size++;
            notEmpty.signal(); // wake up a consumer
        } finally { lock.unlock(); }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (size == 0) notEmpty.await(); // wait if empty
            T value = (T) data[head];
            data[head] = null;
            head = (head + 1) % capacity;
            size--;
            notFull.signal(); // wake up a producer
            return value;
        } finally { lock.unlock(); }
    }

    public int size() {
        lock.lock();
        try { return size; } finally { lock.unlock(); }
    }

    public static void main(String[] args) throws Exception {
        var queue = new BoundedBlockingQueue<Integer>(1024);
        int total = 1_000_000;
        int producers = 4;
        int consumers = 4;
        int perProducer = total / producers;
        var counter = new java.util.concurrent.atomic.AtomicInteger(0);

        long start = System.nanoTime();

        // Producers
        var prodThreads = new Thread[producers];
        for (int p = 0; p < producers; p++) {
            prodThreads[p] = new Thread(() -> {
                try {
                    for (int i = 0; i < perProducer; i++) queue.put(i);
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
            prodThreads[p].start();
        }

        // Consumers
        var consThreads = new Thread[consumers];
        for (int c = 0; c < consumers; c++) {
            consThreads[c] = new Thread(() -> {
                try {
                    while (counter.get() < total) {
                        queue.take();
                        counter.incrementAndGet();
                    }
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
            consThreads[c].start();
        }

        for (var t : prodThreads) t.join();
        // Signal consumers to stop (simplified: just wait a bit)
        Thread.sleep(100);
        for (var t : consThreads) t.interrupt();
        for (var t : consThreads) t.join();

        long elapsed = System.nanoTime() - start;
        System.out.printf("BoundedBlockingQueue: %d producers, %d consumers%n", producers, consumers);
        System.out.printf("  %,d messages in %d ms%n", counter.get(), elapsed / 1_000_000);
        System.out.printf("  Throughput: %,d msgs/sec%n",
            (long) counter.get() * 1_000_000_000L / elapsed);
    }
}
// Resultado esperado:
// 1,000,000 messages in ~200-500 ms
// Throughput: ~2-5M msgs/sec
```

**Preguntas:**

1. ¿`Condition notFull/notEmpty` — ¿por qué dos conditions separadas?

2. ¿`while (size == capacity)` en vez de `if` — ¿por qué while?

3. ¿`ArrayBlockingQueue` de JDK usa el mismo patrón.
   ¿Es comparable en rendimiento?

4. ¿La capacidad de 1024 — ¿cómo se elige?

5. ¿Go channels implementan la misma semántica. ¿Con goroutines
   en vez de threads?

---

### Ejercicio 13.3.2 — Implementar: blocking queue en Go con channels

**Tipo: Implementar**

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

func main() {
    capacity := 1024
    total := 1_000_000
    producers := 4
    consumers := 4
    perProducer := total / producers

    ch := make(chan int, capacity) // Go channel IS a bounded blocking queue
    var consumed atomic.Int64
    var wg sync.WaitGroup

    start := time.Now()

    // Producers
    for p := 0; p < producers; p++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for i := 0; i < perProducer; i++ {
                ch <- i // blocks if full
            }
        }()
    }

    // Close channel after all producers done
    go func() {
        wg.Wait()
        close(ch)
    }()

    // Consumers
    var cwg sync.WaitGroup
    for c := 0; c < consumers; c++ {
        cwg.Add(1)
        go func() {
            defer cwg.Done()
            for range ch { // blocks if empty, exits when closed
                consumed.Add(1)
            }
        }()
    }
    cwg.Wait()

    elapsed := time.Since(start)
    msgs := consumed.Load()
    fmt.Printf("Go channels: %d producers, %d consumers\n", producers, consumers)
    fmt.Printf("  %d messages in %v\n", msgs, elapsed)
    fmt.Printf("  Throughput: %d msgs/sec\n",
        int64(float64(msgs)/elapsed.Seconds()))
}
// Resultado esperado:
// 1,000,000 messages in ~100-300 ms
// Throughput: ~3-10M msgs/sec
```

**Preguntas:**

1. ¿Un Go channel con buffer ES una bounded blocking queue?

2. ¿`ch <- i` bloquea si el channel está lleno. ¿Es backpressure?

3. ¿Go channels son más rápidos que Java blocking queues?

4. ¿Un channel sin buffer (`make(chan int)`) es una queue de tamaño 0.
   ¿Es útil?

5. ¿Kafka es conceptualmente un channel distribuido y persistente?

---

## Sección 13.4 — Work-Stealing Deque

### Ejercicio 13.4.1 — Leer: el patrón work-stealing

**Tipo: Leer**

```
Work-stealing: cada thread tiene su propia deque de tareas.

  Thread 1: [task_a, task_b, task_c] ← push/pop aquí (LIFO)
  Thread 2: [task_d, task_e]
  Thread 3: []  ← vacía, necesita trabajo

  Thread 3 "roba" del OTRO extremo de la deque de Thread 1:
  Thread 1: [task_a, task_b] ← task_c fue robada
  Thread 3: [task_c] ← ejecuta task_c

  ¿Por qué LIFO para el dueño y FIFO para el ladrón?
    El dueño trabaja en la tarea más reciente (LIFO):
    → Mejor cache locality (datos frescos en cache).
    El ladrón roba la tarea más antigua (FIFO):
    → Tareas antiguas suelen ser las más grandes (divide-and-conquer).
    → Menos contención (acceden a extremos opuestos de la deque).

  Uso real:
    Java ForkJoinPool: cada worker thread tiene un work-stealing deque.
    → Spark usa ForkJoinPool internamente.
    Go runtime: goroutine scheduling con work-stealing.
    Tokio (Rust): task scheduling con work-stealing.
    Intel TBB: parallel_for con work-stealing.

  ¿Por qué no una queue compartida?
    Queue compartida = contención entre TODOS los threads.
    Work-stealing = cada thread trabaja en su propia deque.
    Solo hay contención cuando un thread necesita robar.
    → Mucho menos contención = mucho mejor escalabilidad.
```

**Preguntas:**

1. ¿ForkJoinPool de Java (que Spark usa) tiene work-stealing.
   ¿Spark parallel stages lo aprovechan?

2. ¿Go runtime usa work-stealing para goroutines.
   ¿Cada P (processor) tiene una run queue?

3. ¿LIFO para el dueño, FIFO para el ladrón. ¿Es siempre óptimo?

4. ¿La contención solo ocurre durante el robo.
   ¿Cuándo robar es frecuente?

5. ¿Un thread pool simple (ThreadPoolExecutor) no usa work-stealing.
   ¿Es significativamente peor?

---

### Ejercicio 13.4.2 — Implementar: work-stealing simplificado en Go

**Tipo: Implementar**

```go
package main

import (
    "fmt"
    "math/rand"
    "sync"
    "sync/atomic"
    "time"
)

type WorkStealingPool struct {
    queues   []chan func()
    nWorkers int
    done     atomic.Bool
}

func NewWorkStealingPool(nWorkers, queueSize int) *WorkStealingPool {
    pool := &WorkStealingPool{
        queues:   make([]chan func(), nWorkers),
        nWorkers: nWorkers,
    }
    for i := range pool.queues {
        pool.queues[i] = make(chan func(), queueSize)
    }
    return pool
}

func (p *WorkStealingPool) Start() {
    for i := 0; i < p.nWorkers; i++ {
        go p.worker(i)
    }
}

func (p *WorkStealingPool) worker(id int) {
    myQueue := p.queues[id]
    for !p.done.Load() {
        // Try own queue first
        select {
        case task, ok := <-myQueue:
            if ok { task() }
        default:
            // Own queue empty: try to steal
            stolen := false
            victim := rand.Intn(p.nWorkers)
            if victim != id {
                select {
                case task, ok := <-p.queues[victim]:
                    if ok { task(); stolen = true }
                default:
                }
            }
            if !stolen {
                time.Sleep(time.Microsecond) // avoid busy-wait
            }
        }
    }
}

func (p *WorkStealingPool) Submit(workerHint int, task func()) {
    target := workerHint % p.nWorkers
    select {
    case p.queues[target] <- task:
    default:
        // Queue full: try others
        for i := 0; i < p.nWorkers; i++ {
            select {
            case p.queues[(target+i)%p.nWorkers] <- task:
                return
            default:
            }
        }
        // All full: block on target
        p.queues[target] <- task
    }
}

func (p *WorkStealingPool) Shutdown() {
    p.done.Store(true)
    for _, q := range p.queues { close(q) }
}

func main() {
    nWorkers := 4
    pool := NewWorkStealingPool(nWorkers, 256)
    pool.Start()

    total := 1_000_000
    var completed atomic.Int64
    var wg sync.WaitGroup

    start := time.Now()
    for i := 0; i < total; i++ {
        wg.Add(1)
        idx := i
        pool.Submit(idx, func() {
            _ = idx * idx // trivial work
            completed.Add(1)
            wg.Done()
        })
    }
    wg.Wait()
    elapsed := time.Since(start)
    pool.Shutdown()

    fmt.Printf("Work-stealing pool: %d workers\n", nWorkers)
    fmt.Printf("  %d tasks in %v\n", completed.Load(), elapsed)
    fmt.Printf("  Throughput: %d tasks/sec\n",
        int64(float64(completed.Load())/elapsed.Seconds()))
}
```

**Preguntas:**

1. ¿Cada worker intenta su propia queue primero. ¿Es la clave
   de la performance?

2. ¿El victim se elige al azar. ¿Hay mejores estrategias?

3. ¿`time.Sleep(Microsecond)` para evitar busy-wait. ¿Optimal?

4. ¿Este modelo simplificado vs ForkJoinPool real
   — ¿qué falta?

5. ¿Para tareas heterogéneas (unas pesadas, otras livianas),
   ¿work-stealing es ideal?

---

## Sección 13.5 — Lock-Free Queue: Michael-Scott y LMAX Disruptor

### Ejercicio 13.5.1 — Leer: LMAX Disruptor — el ring buffer que cambió todo

**Tipo: Leer**

```
LMAX Disruptor (2011):
  Un ring buffer lock-free que procesa 6+ millones de mensajes/segundo
  en un solo thread. Usado en sistemas de trading de baja latencia.

  ¿Por qué es tan rápido?

  1. RING BUFFER PRE-ALLOCATED:
     Array fijo de tamaño potencia de 2.
     Los slots se pre-alocan al inicio. Zero GC durante operación.
     index = sequence & (bufferSize - 1)  (bitwise AND en vez de modulo).

  2. MECHANICAL SYMPATHY:
     Diseñado para aprovechar cómo funciona el hardware.
     - Cada slot del ring buffer cabe en una cache line (64 bytes).
     - PADDING entre campos para evitar false sharing.
     - Sequential access patterns → prefetch del CPU funciona.

  3. SECUENCIAS EN VEZ DE LOCKS:
     Cada producer y consumer tiene un SEQUENCE (long atómico).
     Producer: "he escrito hasta la posición 42".
     Consumer: "he leído hasta la posición 38".
     Para escribir: verificar que ningún consumer está leyendo ese slot.
     Para leer: verificar que el producer ha escrito ese slot.
     → Sin locks. Sin CAS en el hot path (single producer).

  4. FALSE SHARING PREVENTION:
     En un CPU multi-core, los cores comparten cache lines.
     Si dos threads escriben variables en la misma cache line,
     ambos invalidan el cache del otro (false sharing).
     El Disruptor usa padding (7 longs de relleno) para que
     cada secuencia ocupe su propia cache line.

  Patrón típico:
    Producer (1) → [Ring Buffer] → Consumer A → Consumer B → Consumer C
                                   (pipeline de procesamiento)

  Benchmark:
    LMAX Disruptor: ~6M msgs/sec (single producer, single consumer).
    ArrayBlockingQueue: ~1M msgs/sec.
    → 6× más throughput. Latencia sub-microsegundo.
```

**Preguntas:**

1. ¿"Mechanical sympathy" — ¿es diseñar software que entiende
   el hardware?

2. ¿False sharing — ¿un problema real que cuesta rendimiento?

3. ¿Bitwise AND en vez de modulo — ¿cuánto más rápido?

4. ¿6M msgs/sec vs 1M — ¿la diferencia es solo por los locks?

5. ¿Para un sistema de streaming (Flink, Kafka Streams),
   ¿el Disruptor es aplicable?

---

### Ejercicio 13.5.2 — Implementar: Disruptor simplificado en Java

**Tipo: Implementar**

```java
import java.util.concurrent.atomic.AtomicLong;

public class SimpleDisruptor {

    private final long[] buffer;
    private final int mask;

    // Padding to prevent false sharing
    private long p1, p2, p3, p4, p5, p6, p7;
    private final AtomicLong writerSequence = new AtomicLong(-1);
    private long p8, p9, p10, p11, p12, p13, p14;
    private final AtomicLong readerSequence = new AtomicLong(-1);
    private long p15, p16, p17, p18, p19, p20, p21;

    public SimpleDisruptor(int bufferSize) {
        // Must be power of 2
        assert (bufferSize & (bufferSize - 1)) == 0 : "Must be power of 2";
        this.buffer = new long[bufferSize];
        this.mask = bufferSize - 1;
    }

    // Single producer
    public void publish(long value) {
        long nextSeq = writerSequence.get() + 1;
        // Wait until consumer has consumed the slot we want to write
        while (nextSeq - readerSequence.get() > mask) {
            Thread.onSpinWait(); // hint to CPU
        }
        buffer[(int)(nextSeq & mask)] = value;
        writerSequence.lazySet(nextSeq); // release semantics
    }

    // Single consumer
    public long consume() {
        long nextSeq = readerSequence.get() + 1;
        // Wait until producer has written the slot we want to read
        while (writerSequence.get() < nextSeq) {
            Thread.onSpinWait();
        }
        long value = buffer[(int)(nextSeq & mask)];
        readerSequence.lazySet(nextSeq);
        return value;
    }

    public static void main(String[] args) throws Exception {
        int bufferSize = 1024 * 64; // 64K slots
        var disruptor = new SimpleDisruptor(bufferSize);
        int n = 50_000_000;

        var producer = new Thread(() -> {
            for (long i = 0; i < n; i++) disruptor.publish(i);
        });
        var sum = new long[]{0};
        var consumer = new Thread(() -> {
            for (int i = 0; i < n; i++) sum[0] += disruptor.consume();
        });

        long start = System.nanoTime();
        consumer.start();
        producer.start();
        producer.join();
        consumer.join();
        long elapsed = System.nanoTime() - start;

        System.out.printf("SimpleDisruptor: %,d messages%n", n);
        System.out.printf("Time: %d ms%n", elapsed / 1_000_000);
        System.out.printf("Throughput: %,d msgs/sec%n",
            (long)((double) n / elapsed * 1_000_000_000));
        System.out.printf("Latency: %.0f ns/msg%n", (double) elapsed / n);

        // Compare with ArrayBlockingQueue
        var abq = new java.util.concurrent.ArrayBlockingQueue<Long>(bufferSize);
        start = System.nanoTime();
        var p2t = new Thread(() -> {
            for (long i = 0; i < n; i++) {
                try { abq.put(i); } catch (InterruptedException e) {}
            }
        });
        var c2t = new Thread(() -> {
            for (int i = 0; i < n; i++) {
                try { abq.take(); } catch (InterruptedException e) {}
            }
        });
        c2t.start(); p2t.start(); p2t.join(); c2t.join();
        long abqTime = System.nanoTime() - start;

        System.out.printf("%nArrayBlockingQueue:%n");
        System.out.printf("Throughput: %,d msgs/sec%n",
            (long)((double) n / abqTime * 1_000_000_000));
        System.out.printf("Disruptor/ABQ: %.1f× faster%n",
            (double) abqTime / elapsed);
    }
}
// Resultado esperado:
// SimpleDisruptor: 50,000,000 messages
// Throughput: ~30-80M msgs/sec
// Latency: ~12-30 ns/msg
// ArrayBlockingQueue: ~5-15M msgs/sec
// Disruptor/ABQ: ~3-6× faster
```

**Preguntas:**

1. ¿`lazySet` en vez de `set` — ¿qué diferencia hay?

2. ¿`Thread.onSpinWait()` — ¿es una instrucción PAUSE del CPU?

3. ¿30-80M msgs/sec — ¿es realista para un sistema de trading?

4. ¿Este Disruptor es single-producer/single-consumer.
   ¿Multi-producer es más complejo?

5. ¿El padding con 7 longs — ¿realmente evita false sharing?

---

## Sección 13.6 — Benchmark: Blocking vs Lock-Free Bajo Contención

### Ejercicio 13.6.1 — Analizar: cuándo usar cada tipo de queue

**Tipo: Analizar**

```
Resumen de throughput (single producer → single consumer):

  Queue                     Throughput    Latency     Complejidad
  ─────                     ──────────    ───────     ───────────
  ArrayBlockingQueue (Java) ~5-15M/sec   ~70-200 ns  Simple
  LinkedBlockingQueue       ~3-8M/sec    ~125-300 ns Simple
  Disruptor (ring buffer)   ~30-80M/sec  ~12-30 ns   Moderada
  Go channel (buffered)     ~10-30M/sec  ~30-100 ns  Simple

  Multi-producer, multi-consumer (4P, 4C):

  Queue                     Throughput    Contention
  ─────                     ──────────    ──────────
  ArrayBlockingQueue        ~2-5M/sec    Alta (single lock)
  LinkedBlockingQueue       ~3-8M/sec    Media (separate locks)
  ConcurrentLinkedQueue     ~5-15M/sec   Baja (CAS, lock-free)
  Disruptor (multi)         ~10-30M/sec  Mínima (sequences)

  Regla de decisión:

  ¿Baja latencia crítica (trading, real-time)?
    → Disruptor o ring buffer lock-free.
  ¿Alto throughput con múltiples threads?
    → ConcurrentLinkedQueue o Disruptor multi-producer.
  ¿Simplicidad + correctness?
    → ArrayBlockingQueue o Go channels.
  ¿Backpressure necesaria?
    → Bounded blocking queue (ABQ, Go buffered channel).
```

**Preguntas:**

1. ¿Disruptor es 3-6× más rápido que ABQ. ¿Justifica la complejidad?

2. ¿Go channels están entre ABQ y Disruptor. ¿Buen balance?

3. ¿LinkedBlockingQueue tiene separate locks para head y tail.
   ¿Eso reduce contención?

4. ¿Para un pipeline de datos (ETL), ¿qué queue?

5. ¿Kafka internamente usa qué tipo de queue?

---

## Sección 13.7 — Queues en Producción: de Kafka a ForkJoinPool

### Ejercicio 13.7.1 — Analizar: queues en la infraestructura real

**Tipo: Analizar**

```
KAFKA:
  Un distributed commit log (queue persistente).
  Cada partition es un append-only log en disco.
  Producers append al final. Consumers leen con un offset.
  → Conceptualmente: una bounded queue donde el "dequeue"
    no elimina el mensaje (lo retiene por tiempo o tamaño).
  → Throughput: millones de msgs/sec por broker.
  → Persistencia: los mensajes sobreviven crashes.

JAVA FORKJOINPOOL (Spark, Akka):
  Cada worker thread tiene un work-stealing deque.
  Fork: dividir una tarea en subtareas → push a la deque local.
  Join: esperar que las subtareas terminen.
  Steal: si un worker está idle, roba del otro extremo de otro worker.
  → Spark usa ForkJoinPool para parallelismo intra-stage.

GO RUNTIME:
  Cada P (processor) tiene una local run queue (ring buffer de 256).
  Goroutines nuevas van a la local queue.
  Si la local queue está llena: mover la mitad a la global queue.
  Si está vacía: steal de otro P, o tomar de la global queue.
  → Work-stealing a nivel del runtime del lenguaje.

LMAX DISRUPTOR (Trading):
  Ring buffer de events para un exchange.
  Produce → Journal → Replicate → Business Logic → Publish
  Pipeline de consumers en cadena sobre el mismo ring buffer.
  → Sub-microsecond latency. 6M+ events/sec.

FLINK:
  Los operadores se comunican vía network buffers (ring buffers).
  Backpressure: si un operador downstream es lento,
  los buffers se llenan → upstream reduce velocidad.
  → Ring buffers como mecanismo de flow control.
```

**Preguntas:**

1. ¿Kafka partition ≈ append-only queue persistente. ¿Es un buen
   modelo mental?

2. ¿Flink usa ring buffers para backpressure entre operadores.
   ¿Es como un bounded blocking queue distribuido?

3. ¿Go runtime tiene work-stealing para goroutines.
   ¿Es transparente para el programador?

4. ¿LMAX Disruptor en un exchange procesa 6M events/sec.
   ¿Con cuántos threads?

5. ¿Para un data engineer, ¿cuáles de estas queues son las más
   relevantes?

---

### Ejercicio 13.7.2 — Resumen: las reglas de las queues

**Tipo: Leer**

```
Reglas nuevas del Cap.13:

  Regla 54: Ring buffer > linked list para queues.
    Array contiguo, cache-friendly, zero allocation.
    ArrayDeque es 3-5× más rápido que LinkedList como queue.
    Go channels, LMAX Disruptor, Flink buffers: todos ring buffers.

  Regla 55: Bounded queue = backpressure automática.
    Si la queue está llena, el producer espera.
    El sistema se auto-regula. Sin OOM.
    Kafka partitions, Go buffered channels, ArrayBlockingQueue.

  Regla 56: Work-stealing escala mejor que una queue compartida.
    Cada thread tiene su propia deque. Solo hay contención al robar.
    ForkJoinPool (Spark), Go runtime, Tokio (Rust).
    Para workloads paralelizables: work-stealing > thread pool simple.

  Regla 57: Lock-free queues para baja latencia extrema.
    LMAX Disruptor: 30-80M msgs/sec, ~20 ns latencia.
    3-6× más rápido que ArrayBlockingQueue.
    Justificable para trading, real-time. Overkill para ETL.

  La Parte 5 continúa:
    Cap.14: Estructuras Thread-Safe — mutex, striped locking,
            CAS, ABA problem, Rust ownership model.
```

**Preguntas:**

1. ¿La Regla 54 (ring buffer > linked list) es una regla universal?

2. ¿La Regla 55 (backpressure) es más un patrón de diseño
   que una propiedad de la estructura?

3. ¿Un data engineer necesita implementar un Disruptor
   o basta con entender el concepto?

4. ¿Las queues del Cap.13 se conectan con Kafka (Cap.17)?

5. ¿57 reglas en 13 capítulos — ¿es un buen ritmo?
