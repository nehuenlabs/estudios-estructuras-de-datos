# Guía de Ejercicios — Cap.12: Skip Lists: Simplicidad que Compite con Árboles

> El Red-Black tree del Cap.07 garantiza O(log n).
> Pero implementar insert con fix-up son 50+ líneas de código,
> 6 cases + mirror, rotaciones, recolorings.
> Delete es peor: 100+ líneas, 10+ cases.
>
> ¿Y si pudieras conseguir O(log n) ESPERADO
> con una estructura 10× más simple?
> ¿Sin rotaciones? ¿Sin rebalanceo? ¿Sin colores?
>
> Tira una moneda. Cara: el nodo sube un nivel. Cruz: se queda.
> Eso es un skip list.
>
> Una linked list con "express lanes" — niveles de atajos
> que permiten saltar sobre nodos durante la búsqueda.
> La altura de cada nodo se decide ALEATORIAMENTE.
> No hay invariante de balance que mantener.
>
> El resultado: O(log n) esperado para search, insert, delete.
> Con una implementación que cabe en 30 líneas.
>
> Pero la verdadera razón por la que el skip list importa
> no es la simplicidad. Es la CONCURRENCIA.
> Un Red-Black tree requiere un lock global para rotaciones.
> Un skip list permite inserts concurrentes sin lock global:
> cada insert solo modifica punteros locales.
>
> Por eso Redis eligió skip list para sorted sets (ZSET).
> Por eso LevelDB/RocksDB usan skip list como MemTable.
> Por eso Java ofrece ConcurrentSkipListMap
> pero NO ConcurrentTreeMap.

---

## El modelo mental: una linked list con atajos

```
Una linked list ordenada — search es O(n):

  HEAD → 3 → 7 → 12 → 18 → 25 → 33 → 42 → 55 → 67 → 80 → NIL

  Buscar 42: recorrer 7 nodos. O(n).

¿Y si pudieras SALTAR?

  Nivel 3: HEAD ──────────────────── 25 ──────────────────── 67 → NIL
  Nivel 2: HEAD ────── 12 ────────── 25 ────── 42 ────────── 67 → NIL
  Nivel 1: HEAD → 7 → 12 → 18 → 25 → 33 → 42 → 55 → 67 → 80 → NIL
  Nivel 0: HEAD → 3 → 7 → 12 → 18 → 25 → 33 → 42 → 55 → 67 → 80 → NIL

  Buscar 42:
    Nivel 3: HEAD → 25 (25 < 42, avanzar) → 67 (67 > 42, bajar)
    Nivel 2: 25 → 42 (¡encontrado!)
    → 3 comparaciones en vez de 7.

  ¿Cómo se decide la altura de cada nodo?
    LANZAR UNA MONEDA.
    Insertar un nodo: nivel 0 siempre (linked list base).
    ¿Sube a nivel 1? Flip coin: 50% probabilidad.
    ¿Sube a nivel 2? Flip coin: 50% probabilidad.
    ¿Sube a nivel 3? Flip coin: 50% probabilidad.
    ...
    Altura esperada de un nodo: 1 / (1 - p) = 2 (para p=0.5).

    Resultado:
      ~n nodos en nivel 0 (todos).
      ~n/2 nodos en nivel 1.
      ~n/4 nodos en nivel 2.
      ~n/8 nodos en nivel 3.
      ...
      ~1 nodo en nivel log₂(n).
      → Número esperado de niveles: O(log n).
      → Búsqueda esperada: O(log n) comparaciones.

  ¿Es peor caso O(n)?
    Sí, si la moneda siempre cae cruz.
    Probabilidad de eso: (1/2)^n → astronómicamente improbable.
    En la práctica: O(log n) con probabilidad abrumadora.
```

---

## Tabla de contenidos

- [Sección 12.1 — Skip list: idea y análisis probabilístico](#sección-121--skip-list-idea-y-análisis-probabilístico)
- [Sección 12.2 — Implementar skip list desde cero](#sección-122--implementar-skip-list-desde-cero)
- [Sección 12.3 — Delete y range scan](#sección-123--delete-y-range-scan)
- [Sección 12.4 — Skip list vs Red-Black tree vs B-Tree](#sección-124--skip-list-vs-red-black-tree-vs-b-tree)
- [Sección 12.5 — Concurrent skip lists: por qué importa](#sección-125--concurrent-skip-lists-por-qué-importa)
- [Sección 12.6 — Benchmark: skip list vs TreeMap vs ConcurrentSkipListMap](#sección-126--benchmark-skip-list-vs-treemap-vs-concurrentskiplistmap)
- [Sección 12.7 — Skip lists en producción: Redis, RocksDB, Kafka](#sección-127--skip-lists-en-producción-redis-rocksdb-kafka)

---

## Sección 12.1 — Skip List: Idea y Análisis Probabilístico

### Ejercicio 12.1.1 — Leer: la matemática del skip list

**Tipo: Leer**

```
Análisis probabilístico del skip list (William Pugh, 1990):

  Parámetro p: probabilidad de que un nodo suba al siguiente nivel.
    p = 0.5: cada nivel tiene ~la mitad de nodos del nivel anterior.
    p = 0.25: cada nivel tiene ~un cuarto de nodos. Menos niveles, más nodos por nivel.

  Número esperado de niveles: log_{1/p}(n)
    p = 0.5:  log₂(n) niveles. Para n = 1M: ~20 niveles.
    p = 0.25: log₄(n) niveles. Para n = 1M: ~10 niveles.

  Espacio esperado: n / (1 - p) punteros totales.
    p = 0.5:  2n punteros (cada nodo tiene 2 punteros en promedio).
    p = 0.25: 1.33n punteros (cada nodo tiene 1.33 punteros en promedio).

  Comparaciones esperadas por búsqueda:
    p = 0.5:  (log₂ n) / 1 ≈ 20 para n = 1M.
    p = 0.25: (log₂ n) / 2 ≈ 10 para n = 1M (MENOS comparaciones).

  ¿p = 0.25 es mejor que p = 0.5?
    p = 0.25: menos niveles, menos punteros, menos comparaciones.
    p = 0.5:  más niveles, más punteros, más comparaciones.
    → p = 0.25 es frecuentemente óptimo.
    Redis usa p = 0.25 con MAX_LEVEL = 32.
    LevelDB usa p = 0.25 con MAX_LEVEL = 12.

  Comparación con árboles balanceados:

    Estructura       Search       Insert       Delete       Space
    ─────────        ──────       ──────       ──────       ─────
    Skip list        O(log n)*    O(log n)*    O(log n)*    O(n) expected
    Red-Black tree   O(log n)     O(log n)     O(log n)     O(n)
    AVL tree         O(log n)     O(log n)     O(log n)     O(n)

    * Esperado, no worst case.

  Ventaja del skip list:
    1. Implementación más simple (sin rotaciones).
    2. Concurrent-friendly (inserts locales, sin restructuración global).
    3. Range scan natural (seguir la linked list del nivel 0).

  Desventaja:
    1. O(log n) esperado, no garantizado (en la práctica irrelevante).
    2. Más memoria por nodo (múltiples punteros de nivel).
    3. Peor cache locality que un array-based B-Tree.
```

**Preguntas:**

1. ¿p = 0.25 da menos comparaciones que p = 0.5. ¿Es contra-intuitivo?

2. ¿Redis usa p = 0.25 y MAX_LEVEL = 32. ¿32 niveles
   soporta cuántos elementos?

3. ¿El peor caso O(n) es teóricamente posible.
   ¿Alguna vez pasa en producción?

4. ¿Skip list tiene mejor cache locality que un Red-Black tree?

5. ¿La implementación es realmente "más simple"?
   ¿Cuántas líneas vs Red-Black?

---

## Sección 12.2 — Implementar Skip List Desde Cero

### Ejercicio 12.2.1 — Implementar: skip list en Python

**Tipo: Implementar**

```python
import random

class SkipNode:
    __slots__ = ('key', 'value', 'forward')

    def __init__(self, key, value, level):
        self.key = key
        self.value = value
        self.forward = [None] * (level + 1)  # forward[i] = next node at level i

class SkipList:
    MAX_LEVEL = 20
    P = 0.25  # probability of promotion

    def __init__(self):
        self.header = SkipNode(None, None, self.MAX_LEVEL)
        self.level = 0   # current max level in use
        self.size = 0

    def _random_level(self):
        lvl = 0
        while random.random() < self.P and lvl < self.MAX_LEVEL:
            lvl += 1
        return lvl

    def search(self, key):
        node = self.header
        for i in range(self.level, -1, -1):
            while node.forward[i] and node.forward[i].key < key:
                node = node.forward[i]
        node = node.forward[0]
        if node and node.key == key:
            return node.value
        return None

    def insert(self, key, value):
        update = [None] * (self.MAX_LEVEL + 1)
        node = self.header
        for i in range(self.level, -1, -1):
            while node.forward[i] and node.forward[i].key < key:
                node = node.forward[i]
            update[i] = node

        node = node.forward[0]
        if node and node.key == key:
            node.value = value  # update existing
            return

        new_level = self._random_level()
        if new_level > self.level:
            for i in range(self.level + 1, new_level + 1):
                update[i] = self.header
            self.level = new_level

        new_node = SkipNode(key, value, new_level)
        for i in range(new_level + 1):
            new_node.forward[i] = update[i].forward[i]
            update[i].forward[i] = new_node
        self.size += 1

    def delete(self, key):
        update = [None] * (self.MAX_LEVEL + 1)
        node = self.header
        for i in range(self.level, -1, -1):
            while node.forward[i] and node.forward[i].key < key:
                node = node.forward[i]
            update[i] = node

        node = node.forward[0]
        if node is None or node.key != key:
            return False

        for i in range(self.level + 1):
            if update[i].forward[i] != node:
                break
            update[i].forward[i] = node.forward[i]

        while self.level > 0 and self.header.forward[self.level] is None:
            self.level -= 1
        self.size -= 1
        return True

    def range_scan(self, lo, hi):
        """Yield (key, value) pairs where lo <= key <= hi"""
        node = self.header
        for i in range(self.level, -1, -1):
            while node.forward[i] and node.forward[i].key < lo:
                node = node.forward[i]
        node = node.forward[0]
        results = []
        while node and node.key <= hi:
            results.append((node.key, node.value))
            node = node.forward[0]
        return results

    def __len__(self):
        return self.size

    def __contains__(self, key):
        return self.search(key) is not None


# ═══ Test ═══
sl = SkipList()
data = list(range(1000))
random.shuffle(data)
for d in data:
    sl.insert(d, d * 10)

print(f"Size: {len(sl)}, Level: {sl.level}")
print(f"search(42) = {sl.search(42)}")
print(f"search(9999) = {sl.search(9999)}")
print(f"range_scan(10, 20) = {sl.range_scan(10, 20)}")

sl.delete(42)
print(f"After delete(42): search(42) = {sl.search(42)}")
print(f"Size: {len(sl)}")

# Verify sorted order
node = sl.header.forward[0]
prev = float('-inf')
sorted_ok = True
while node:
    if node.key < prev: sorted_ok = False
    prev = node.key
    node = node.forward[0]
print(f"Sorted order: {sorted_ok}")

# Benchmark
import time
n = 500_000
sl = SkipList()
start = time.perf_counter_ns()
for i in range(n):
    sl.insert(random.randint(0, n * 10), i)
insert_time = time.perf_counter_ns() - start

start = time.perf_counter_ns()
for i in range(100_000):
    sl.search(random.randint(0, n * 10))
search_time = time.perf_counter_ns() - start

print(f"\nBenchmark ({n:,} elements, p={SkipList.P}):")
print(f"  Insert: {insert_time // 1_000_000} ms ({insert_time // n} ns/op)")
print(f"  Search: {search_time // 1_000_000} ms ({search_time // 100_000} ns/op)")
print(f"  Levels used: {sl.level}")
```

**Preguntas:**

1. ¿`_random_level` con p=0.25 — ¿cuántos nodos llegan a nivel 5+?

2. ¿El array `update[]` almacena los predecesores en cada nivel.
   ¿Es necesario para relinking?

3. ¿`range_scan` simplemente sigue `forward[0]` — ¿es O(k)?

4. ¿La implementación completa (search + insert + delete + range)
   cabe en ~60 líneas. ¿Red-Black tree?

5. ¿El `MAX_LEVEL = 20` limita a cuántos elementos?

---

### Ejercicio 12.2.2 — Implementar: skip list en Java

**Tipo: Implementar**

```java
import java.util.*;
import java.util.concurrent.ThreadLocalRandom;

public class SkipList<K extends Comparable<K>, V> {

    private static final int MAX_LEVEL = 24;
    private static final double P = 0.25;

    static class Node<K, V> {
        K key; V value;
        @SuppressWarnings("unchecked")
        Node<K, V>[] forward = new Node[MAX_LEVEL + 1];
        int level;
        Node(K key, V value, int level) {
            this.key = key; this.value = value; this.level = level;
        }
    }

    private final Node<K, V> header = new Node<>(null, null, MAX_LEVEL);
    private int level = 0;
    private int size = 0;

    private int randomLevel() {
        int lvl = 0;
        while (ThreadLocalRandom.current().nextDouble() < P && lvl < MAX_LEVEL) lvl++;
        return lvl;
    }

    public V get(K key) {
        Node<K, V> node = header;
        for (int i = level; i >= 0; i--) {
            while (node.forward[i] != null && node.forward[i].key.compareTo(key) < 0)
                node = node.forward[i];
        }
        node = node.forward[0];
        return (node != null && node.key.compareTo(key) == 0) ? node.value : null;
    }

    @SuppressWarnings("unchecked")
    public void put(K key, V value) {
        Node<K, V>[] update = new Node[MAX_LEVEL + 1];
        Node<K, V> node = header;
        for (int i = level; i >= 0; i--) {
            while (node.forward[i] != null && node.forward[i].key.compareTo(key) < 0)
                node = node.forward[i];
            update[i] = node;
        }
        node = node.forward[0];
        if (node != null && node.key.compareTo(key) == 0) {
            node.value = value; return;
        }
        int newLevel = randomLevel();
        if (newLevel > level) {
            for (int i = level + 1; i <= newLevel; i++) update[i] = header;
            level = newLevel;
        }
        Node<K, V> newNode = new Node<>(key, value, newLevel);
        for (int i = 0; i <= newLevel; i++) {
            newNode.forward[i] = update[i].forward[i];
            update[i].forward[i] = newNode;
        }
        size++;
    }

    public boolean remove(K key) {
        @SuppressWarnings("unchecked")
        Node<K, V>[] update = new Node[MAX_LEVEL + 1];
        Node<K, V> node = header;
        for (int i = level; i >= 0; i--) {
            while (node.forward[i] != null && node.forward[i].key.compareTo(key) < 0)
                node = node.forward[i];
            update[i] = node;
        }
        node = node.forward[0];
        if (node == null || node.key.compareTo(key) != 0) return false;
        for (int i = 0; i <= level; i++) {
            if (update[i].forward[i] != node) break;
            update[i].forward[i] = node.forward[i];
        }
        while (level > 0 && header.forward[level] == null) level--;
        size--;
        return true;
    }

    public int size() { return size; }

    public static void main(String[] args) {
        int n = 1_000_000;
        var rng = new Random(42);

        // ═══ Skip List ═══
        var sl = new SkipList<Integer, Integer>();
        long start = System.nanoTime();
        for (int i = 0; i < n; i++) sl.put(rng.nextInt(n * 10), i);
        long slInsert = System.nanoTime() - start;

        rng = new Random(42);
        start = System.nanoTime();
        int found1 = 0;
        for (int i = 0; i < n; i++) if (sl.get(rng.nextInt(n * 10)) != null) found1++;
        long slSearch = System.nanoTime() - start;

        // ═══ TreeMap ═══
        var tm = new TreeMap<Integer, Integer>();
        rng = new Random(42);
        start = System.nanoTime();
        for (int i = 0; i < n; i++) tm.put(rng.nextInt(n * 10), i);
        long tmInsert = System.nanoTime() - start;

        rng = new Random(42);
        start = System.nanoTime();
        int found2 = 0;
        for (int i = 0; i < n; i++) if (tm.get(rng.nextInt(n * 10)) != null) found2++;
        long tmSearch = System.nanoTime() - start;

        System.out.printf("n = %,d%n%n", n);
        System.out.printf("%-20s %10s %10s%n", "", "Insert(ms)", "Search(ms)");
        System.out.printf("%-20s %10d %10d%n", "SkipList",
            slInsert / 1_000_000, slSearch / 1_000_000);
        System.out.printf("%-20s %10d %10d%n", "TreeMap (RBT)",
            tmInsert / 1_000_000, tmSearch / 1_000_000);
        System.out.printf("%nSkipList/TreeMap: insert=%.2f×, search=%.2f×%n",
            (double) slInsert / tmInsert, (double) slSearch / tmSearch);
    }
}
// Resultado esperado:
//                      Insert(ms) Search(ms)
// SkipList                  ~800      ~600
// TreeMap (RBT)             ~700      ~500
// SkipList/TreeMap: insert=~1.1×, search=~1.2×
//
// Skip list es ~10-20% más lento que TreeMap en single-threaded.
// La ventaja aparece en concurrencia.
```

**Preguntas:**

1. ¿Skip list ~10-20% más lento que TreeMap. ¿Esperado?

2. ¿`ThreadLocalRandom` en vez de `Random` — ¿para concurrencia futura?

3. ¿Cada nodo tiene un array de MAX_LEVEL+1 punteros. ¿Desperdicio?

4. ¿Con p=0.25, ¿cuántos nodos llegan a nivel 10+ de 1M?

5. ¿Si la ventaja es concurrencia, ¿por qué implementar single-threaded?

---

### Ejercicio 12.2.3 — Implementar: skip list en Rust

**Tipo: Implementar**

```rust
use rand::Rng;
use std::fmt;

const MAX_LEVEL: usize = 16;
const P: f64 = 0.25;

struct Node<K, V> {
    key: Option<K>,
    value: Option<V>,
    forward: Vec<Option<*mut Node<K, V>>>,
}

pub struct SkipList<K: Ord, V> {
    header: *mut Node<K, V>,
    level: usize,
    size: usize,
}

impl<K: Ord, V> SkipList<K, V> {
    pub fn new() -> Self {
        let header = Box::into_raw(Box::new(Node {
            key: None,
            value: None,
            forward: vec![None; MAX_LEVEL + 1],
        }));
        SkipList { header, level: 0, size: 0 }
    }

    fn random_level() -> usize {
        let mut rng = rand::thread_rng();
        let mut lvl = 0;
        while rng.gen::<f64>() < P && lvl < MAX_LEVEL { lvl += 1; }
        lvl
    }

    pub fn get(&self, key: &K) -> Option<&V> {
        unsafe {
            let mut node = self.header;
            for i in (0..=self.level).rev() {
                while let Some(next) = (*node).forward[i] {
                    if (*next).key.as_ref().unwrap() < key {
                        node = next;
                    } else { break; }
                }
            }
            if let Some(next) = (*node).forward[0] {
                if (*next).key.as_ref().unwrap() == key {
                    return (*next).value.as_ref();
                }
            }
            None
        }
    }

    pub fn insert(&mut self, key: K, value: V) {
        unsafe {
            let mut update: Vec<*mut Node<K, V>> = vec![self.header; MAX_LEVEL + 1];
            let mut node = self.header;
            for i in (0..=self.level).rev() {
                while let Some(next) = (*node).forward[i] {
                    if (*next).key.as_ref().unwrap() < &key {
                        node = next;
                    } else { break; }
                }
                update[i] = node;
            }
            // Check if key already exists
            if let Some(next) = (*node).forward[0] {
                if (*next).key.as_ref().unwrap() == &key {
                    (*next).value = Some(value);
                    return;
                }
            }
            let new_level = Self::random_level();
            if new_level > self.level {
                for i in (self.level + 1)..=new_level { update[i] = self.header; }
                self.level = new_level;
            }
            let new_node = Box::into_raw(Box::new(Node {
                key: Some(key), value: Some(value),
                forward: vec![None; new_level + 1],
            }));
            for i in 0..=new_level {
                (*new_node).forward[i] = (*update[i]).forward[i];
                (*update[i]).forward[i] = Some(new_node);
            }
            self.size += 1;
        }
    }

    pub fn len(&self) -> usize { self.size }
}

impl<K: Ord, V> Drop for SkipList<K, V> {
    fn drop(&mut self) {
        unsafe {
            let mut node = self.header;
            while let Some(next) = (*node).forward[0] {
                let _ = Box::from_raw(node);
                node = next;
            }
            let _ = Box::from_raw(node);
        }
    }
}

fn main() {
    let mut sl = SkipList::new();
    let n = 1_000_000;

    let start = std::time::Instant::now();
    for i in 0..n {
        sl.insert(i * 7919 % (n * 10), i);
    }
    let insert_time = start.elapsed();

    let start = std::time::Instant::now();
    let mut found = 0u64;
    for i in 0..n {
        if sl.get(&(i * 7919 % (n * 10))).is_some() { found += 1; }
    }
    let search_time = start.elapsed();

    println!("SkipList ({} elements):", sl.len());
    println!("  Insert: {:?} ({} ns/op)", insert_time, insert_time.as_nanos() / n as u128);
    println!("  Search: {:?} ({} ns/op), found={}", search_time, search_time.as_nanos() / n as u128, found);
}
```

**Preguntas:**

1. ¿`*mut Node` y `unsafe` — ¿es inevitable para linked structures en Rust?

2. ¿`Box::into_raw` para nodos — ¿cómo garantizamos que se liberan?

3. ¿La implementación en Rust es significativamente más rápida
   que Java por ausencia de GC?

4. ¿`Drop` recorre la lista nivel 0 y libera todo.
   ¿Es O(n)?

5. ¿`BTreeMap` de Rust es siempre preferible a un skip list
   para uso general?

---

## Sección 12.3 — Delete y Range Scan

### Ejercicio 12.3.1 — Analizar: por qué delete es simple en skip list

**Tipo: Analizar**

```
Delete en un skip list vs Red-Black tree:

  SKIP LIST DELETE:
    1. Buscar el nodo (como search): O(log n).
    2. Para cada nivel donde aparece el nodo:
       update[i].forward[i] = node.forward[i]   (unlink)
    3. Si el nivel máximo bajó: ajustar self.level.
    4. Liberar el nodo.
    → ~15 líneas de código. Sin rotaciones. Sin recolorings.

  RED-BLACK TREE DELETE:
    1. Buscar el nodo: O(log n).
    2. Si tiene dos hijos: reemplazar con sucesor.
    3. fixAfterDelete: 4+ cases + mirror = 8+ cases.
       - Hermano rojo → rotar + recolorear.
       - Hermano negro, ambos sobrinos negros → recolorear + propagar.
       - Hermano negro, sobrino cercano rojo → rotar + recolorear.
       - Hermano negro, sobrino lejano rojo → rotar + recolorear.
    → 50+ líneas de código. Complejo.

  Esta es la razón #1 por la que el skip list existe:
  delete es trivial.

  RANGE SCAN:
    Skip list: navegar al primer key ≥ lo, luego recorrer forward[0].
    Red-Black tree: inorder traversal parcial.
    Ambos: O(log n + k).
    Pero el skip list es una linked list en nivel 0:
    → seguir punteros linealmente es simple y predecible.
```

**Preguntas:**

1. ¿15 líneas vs 50+ para delete. ¿El código de producción refleja esto?

2. ¿Redis ZREM (delete de sorted set) usa el delete del skip list?

3. ¿Range scan en nivel 0 es exactamente como una linked list.
   ¿Cache locality?

4. ¿Un skip list con delete frecuente mantiene su distribución
   de niveles?

5. ¿Si la distribución de niveles es aleatoria, ¿delete
   no puede "desbalancear" la estructura?

---

## Sección 12.4 — Skip List vs Red-Black Tree vs B-Tree

### Ejercicio 12.4.1 — Analizar: cuándo usar cada estructura

**Tipo: Analizar**

```
Tres estructuras para datos ordenados:

  SKIP LIST:
    Complejidad: O(log n) esperado.
    Implementación: ~60 líneas (insert + search + delete + range).
    Concurrencia: excelente (inserts locales, CAS para lock-free).
    Cache locality: mala (pointer chasing entre nodos dispersos).
    Uso: MemTables (RocksDB), sorted sets (Redis), concurrent maps (Java).

  RED-BLACK TREE:
    Complejidad: O(log n) worst case.
    Implementación: ~200+ líneas (especialmente delete).
    Concurrencia: difícil (rotaciones son globales, requieren locks).
    Cache locality: mala (nodos dispersos, como skip list).
    Uso: TreeMap (Java), std::map (C++), kernel de Linux.

  B-TREE:
    Complejidad: O(log_B n) worst case.
    Implementación: ~300+ líneas (split, merge, redistribute).
    Concurrencia: moderada (latch-per-page, crabbing protocol).
    Cache locality: excelente (nodos = arrays contiguos, cache-friendly).
    Uso: bases de datos (PostgreSQL, MySQL), BTreeMap (Rust), filesystems.

  Decisión:

  ┌────────────────────────────────────────────────────────┐
  │ ¿Necesitas concurrencia multi-thread?                  │
  │   Sí → Skip list (ConcurrentSkipListMap).              │
  │   No ↓                                                 │
  │ ¿Los datos están en disco?                             │
  │   Sí → B-Tree (pages, I/O amortizado).                 │
  │   No ↓                                                 │
  │ ¿Cache locality importa?                               │
  │   Sí → B-Tree in-memory (Rust BTreeMap, array-based).  │
  │   No → Red-Black tree (TreeMap) o skip list.           │
  └────────────────────────────────────────────────────────┘

  Rust eligió BTreeMap sobre TreeMap:
    B-Tree in-memory tiene mejor cache locality que Red-Black tree.
    Los nodos de B-Tree son arrays contiguos → prefetch-friendly.
    Para n > ~100, BTreeMap es más rápido que un hipotético RBTreeMap.
```

**Preguntas:**

1. ¿Rust BTreeMap es más rápido que un Red-Black tree in-memory.
   ¿Cuánto más rápido?

2. ¿Java podría reemplazar TreeMap por BTreeMap.
   ¿Por qué no lo ha hecho?

3. ¿ConcurrentSkipListMap es la ÚNICA estructura concurrent sorted
   en Java stdlib?

4. ¿Para un MemTable de LSM-Tree, ¿por qué skip list y no B-Tree?

5. ¿La elección entre estas tres estructuras importa para
   la mayoría de aplicaciones?

---

## Sección 12.5 — Concurrent Skip Lists: Por Qué Importa

### Ejercicio 12.5.1 — Leer: por qué skip list es concurrent-friendly

**Tipo: Leer**

```
¿Por qué un skip list es más fácil de hacer concurrent que un Red-Black tree?

  RED-BLACK TREE INSERT:
    1. Navegar al punto de inserción.
    2. Insertar el nodo.
    3. fixAfterInsert: rotaciones + recolorings.
       Las rotaciones modifican MULTIPLES nodos (padre, abuelo, tío).
       → Un insert puede afectar nodos en CUALQUIER parte del árbol.
       → Necesitas un lock global (o lock escalation complejo).

  SKIP LIST INSERT:
    1. Navegar al punto de inserción (con update[] array).
    2. Crear nodo con nivel aleatorio.
    3. Para cada nivel: relinkear el puntero forward.
       Cada relinkeo modifica DOS nodos: el predecesor y el nuevo.
       → El impacto es LOCAL: solo los nodos adyacentes.
       → Cada relinkeo puede hacerse con CAS (compare-and-swap).

  CAS (Compare-And-Swap):
    Operación atómica del hardware:
    "Si el puntero actual es X, cámbialo a Y. Si no, falla."
    
    forward[i].CAS(expected=old_next, new=new_node)
    Si otro thread modificó forward[i] antes: CAS falla → retry.
    Sin locks. Sin deadlocks. Lock-free.

  Java ConcurrentSkipListMap (java.util.concurrent):
    - Lock-free search: los readers NUNCA bloquean.
    - Lock-free insert: usa CAS para relinkear punteros.
    - Lock-free delete: marca el nodo como "logically deleted",
      luego lo desvincula físicamente.
    - Concurrent: miles de threads leyendo y escribiendo simultáneamente.
    - Sorted: mantiene orden natural (Comparable) o Comparator.

  No existe ConcurrentTreeMap en Java porque:
    Las rotaciones del Red-Black tree son intrínsecamente globales.
    Hacer un RBT lock-free requiere técnicas extremadamente complejas.
    El skip list es la solución pragmática.
```

**Preguntas:**

1. ¿CAS es una instrucción de hardware? ¿Es realmente O(1)?

2. ¿Lock-free search significa zero overhead para readers?

3. ¿ConcurrentSkipListMap vs ConcurrentHashMap
   — ¿cuándo usar cada uno?

4. ¿El delete "logically deleted" de ConcurrentSkipListMap
   — ¿es como un tombstone?

5. ¿Para un concurrent sorted map en Go, ¿qué opción existe?

---

### Ejercicio 12.5.2 — Implementar: benchmark concurrent en Java

**Tipo: Implementar**

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class ConcurrentBenchmark {

    static final int N = 1_000_000;
    static final int THREADS = 8;

    static long benchmarkConcurrent(String name, Runnable setup,
                                     Runnable task, int threads) throws Exception {
        setup.run();
        var latch = new CountDownLatch(1);
        var done = new CountDownLatch(threads);
        var workers = new Thread[threads];
        for (int t = 0; t < threads; t++) {
            workers[t] = new Thread(() -> {
                try { latch.await(); } catch (Exception e) {}
                task.run();
                done.countDown();
            });
            workers[t].start();
        }
        long start = System.nanoTime();
        latch.countDown(); // start all threads
        done.await();
        return System.nanoTime() - start;
    }

    public static void main(String[] args) throws Exception {
        // Shared data structures
        var skipMap = new ConcurrentSkipListMap<Integer, Integer>();
        var hashMap = new ConcurrentHashMap<Integer, Integer>(N * 2);
        var treeMap = Collections.synchronizedMap(new TreeMap<Integer, Integer>());

        int opsPerThread = N / THREADS;

        System.out.printf("Concurrent benchmark: %d threads, %,d ops/thread%n%n", THREADS, opsPerThread);

        // ═══ Write benchmark ═══
        System.out.println("═══ CONCURRENT WRITES ═══");

        long skipWrite = benchmarkConcurrent("ConcurrentSkipListMap",
            skipMap::clear,
            () -> {
                var rng = ThreadLocalRandom.current();
                for (int i = 0; i < opsPerThread; i++)
                    skipMap.put(rng.nextInt(N * 10), i);
            }, THREADS);

        long hashWrite = benchmarkConcurrent("ConcurrentHashMap",
            hashMap::clear,
            () -> {
                var rng = ThreadLocalRandom.current();
                for (int i = 0; i < opsPerThread; i++)
                    hashMap.put(rng.nextInt(N * 10), i);
            }, THREADS);

        long treeWrite = benchmarkConcurrent("SynchronizedTreeMap",
            () -> { synchronized(treeMap) { treeMap.clear(); } },
            () -> {
                var rng = ThreadLocalRandom.current();
                for (int i = 0; i < opsPerThread; i++)
                    synchronized(treeMap) { treeMap.put(rng.nextInt(N * 10), i); }
            }, THREADS);

        System.out.printf("  ConcurrentSkipListMap:  %5d ms%n", skipWrite / 1_000_000);
        System.out.printf("  ConcurrentHashMap:      %5d ms%n", hashWrite / 1_000_000);
        System.out.printf("  SynchronizedTreeMap:    %5d ms%n", treeWrite / 1_000_000);

        // ═══ Read benchmark ═══
        System.out.println("\n═══ CONCURRENT READS ═══");
        // Pre-populate
        for (int i = 0; i < N; i++) {
            int k = i * 3;
            skipMap.put(k, i); hashMap.put(k, i);
            synchronized(treeMap) { treeMap.put(k, i); }
        }

        long skipRead = benchmarkConcurrent("ConcurrentSkipListMap",
            () -> {},
            () -> {
                var rng = ThreadLocalRandom.current();
                for (int i = 0; i < opsPerThread; i++) skipMap.get(rng.nextInt(N * 10));
            }, THREADS);

        long hashRead = benchmarkConcurrent("ConcurrentHashMap",
            () -> {},
            () -> {
                var rng = ThreadLocalRandom.current();
                for (int i = 0; i < opsPerThread; i++) hashMap.get(rng.nextInt(N * 10));
            }, THREADS);

        long treeRead = benchmarkConcurrent("SynchronizedTreeMap",
            () -> {},
            () -> {
                var rng = ThreadLocalRandom.current();
                for (int i = 0; i < opsPerThread; i++)
                    synchronized(treeMap) { treeMap.get(rng.nextInt(N * 10)); }
            }, THREADS);

        System.out.printf("  ConcurrentSkipListMap:  %5d ms%n", skipRead / 1_000_000);
        System.out.printf("  ConcurrentHashMap:      %5d ms%n", hashRead / 1_000_000);
        System.out.printf("  SynchronizedTreeMap:    %5d ms%n", treeRead / 1_000_000);
        System.out.printf("\nSkip vs Synced Tree (writes): %.1f× faster%n",
            (double) treeWrite / skipWrite);
        System.out.printf("Skip vs Synced Tree (reads):  %.1f× faster%n",
            (double) treeRead / skipRead);
    }
}
// Resultado esperado:
// CONCURRENT WRITES:
//   ConcurrentSkipListMap:   ~300 ms
//   ConcurrentHashMap:       ~150 ms
//   SynchronizedTreeMap:     ~2000 ms  ← lock contention!
//
// CONCURRENT READS:
//   ConcurrentSkipListMap:   ~200 ms
//   ConcurrentHashMap:       ~80 ms
//   SynchronizedTreeMap:     ~800 ms   ← lock contention!
//
// Skip list ~5-10× más rápido que synchronized TreeMap en 8 threads.
// ConcurrentHashMap es más rápido pero no soporta sorted operations.
```

**Preguntas:**

1. ¿ConcurrentSkipListMap es 5-10× más rápido que SynchronizedTreeMap
   con 8 threads. ¿Escala linealmente con más threads?

2. ¿ConcurrentHashMap es más rápido. ¿Cuándo elegir skip list?

3. ¿Con 1 thread, ¿SynchronizedTreeMap sería comparable al skip list?

4. ¿Para un MemTable de LSM-Tree con N writer threads,
   ¿ConcurrentSkipListMap es la elección correcta?

5. ¿Lock-free vs lock-based — ¿cuánta diferencia hay bajo baja contención?

---

## Sección 12.6 — Benchmark: Skip List vs TreeMap vs ConcurrentSkipListMap

### Ejercicio 12.6.1 — Analizar: cuándo el skip list gana

**Tipo: Analizar**

```
Resumen de benchmarks:

  SINGLE-THREADED (1M operations):
    Estructura          Insert    Search    Range(100)
    ──────────          ──────    ──────    ──────────
    TreeMap (RBT)       ~700 ms   ~500 ms   ~50 ms
    SkipList (custom)   ~800 ms   ~600 ms   ~55 ms
    HashMap             ~200 ms   ~150 ms   N/A

    → TreeMap gana single-threaded por ~10-20%.
    → HashMap gana absolutamente si no necesitas orden.

  MULTI-THREADED (8 threads, 1M operations):
    Estructura              Insert    Search
    ──────────              ──────    ──────
    ConcurrentSkipListMap   ~300 ms   ~200 ms
    SynchronizedTreeMap     ~2000 ms  ~800 ms
    ConcurrentHashMap       ~150 ms   ~80 ms

    → ConcurrentSkipListMap gana 5-10× vs synced TreeMap.
    → ConcurrentHashMap gana si no necesitas orden.

  REGLA DE DECISIÓN:
    ¿Single-threaded + sorted? → TreeMap (Java) / BTreeMap (Rust).
    ¿Multi-threaded + sorted? → ConcurrentSkipListMap.
    ¿Single-threaded + no sorted? → HashMap.
    ¿Multi-threaded + no sorted? → ConcurrentHashMap.

  El skip list gana en UN escenario: concurrent + sorted.
  En todos los demás, hay alternativas mejores.
  Pero ese escenario es exactamente el que necesitan:
  - MemTables de LSM-Trees (RocksDB, Cassandra, HBase).
  - Sorted sets con concurrent access (Redis ZSET).
  - Concurrent sorted maps en Java (ConcurrentSkipListMap).
```

**Preguntas:**

1. ¿El skip list solo gana en "concurrent + sorted".
   ¿Es suficiente para justificar su existencia?

2. ¿Redis ZSET es single-threaded. ¿Por qué usa skip list
   en vez de Red-Black tree?

3. ¿RocksDB MemTable es concurrent. ¿Cuántos writer threads?

4. ¿Para un data engineer, ¿cuándo interactúa directamente
   con ConcurrentSkipListMap?

5. ¿Go no tiene un concurrent sorted map en stdlib.
   ¿Qué alternativa hay?

---

## Sección 12.7 — Skip Lists en Producción: Redis, RocksDB, Kafka

### Ejercicio 12.7.1 — Analizar: por qué Redis eligió skip list

**Tipo: Analizar**

```
Redis Sorted Sets (ZSET):

  Cada elemento tiene un score (double) y un member (string).
  Operaciones principales:
    ZADD key score member       — insert.
    ZRANGE key start stop       — range by rank.
    ZRANGEBYSCORE key min max   — range by score.
    ZRANK key member            — rank of member.
    ZREM key member             — delete.

  ¿Por qué skip list y no Red-Black tree?

  Antirez (Salvatore Sanfilippo, creador de Redis) explicó:

  1. RANGE QUERIES:
     El skip list tiene una linked list en nivel 0.
     ZRANGEBYSCORE: buscar el primer score ≥ min,
     luego recorrer la linked list hasta score > max.
     → Natural y eficiente. Un RBT requiere inorder traversal.

  2. SIMPLICIDAD:
     Skip list insert: ~30 líneas en C.
     Red-Black tree insert + delete: ~100+ líneas.
     Menos código = menos bugs en una pieza crítica de infraestructura.

  3. IMPLEMENTACIÓN ESPECÍFICA:
     Redis necesita rank() eficiente: ¿cuántos elementos tienen score < X?
     En el skip list de Redis, cada puntero forward almacena
     el "span" (cuántos nodos salta). Sumando spans durante
     la búsqueda → rank en O(log n). Elegante.

  Estructura de Redis skip list:
    typedef struct zskiplistNode {
        sds ele;              // member string
        double score;         // score
        struct zskiplistNode *backward;  // prev pointer (level 0)
        struct zskiplistLevel {
            struct zskiplistNode *forward;
            unsigned long span;    // ← key addition: nodes skipped
        } level[];
    } zskiplistNode;

  El span por nivel permite calcular rank en O(log n).
  Esto es difícil de implementar eficientemente en un RBT.

  Redis usa skip list + hash table juntos:
    Hash table: O(1) lookup por member (¿existe "alice"?).
    Skip list: sorted operations por score.
    → ZADD verifica en el hash table, luego inserta en el skip list.
```

**Preguntas:**

1. ¿El "span" por nivel es la innovación clave de Redis skip list.
   ¿Cuánta memoria extra cuesta?

2. ¿Redis es single-threaded. ¿No pierde la ventaja de concurrencia?

3. ¿Un RBT augmentado con subtree sizes también da rank en O(log n).
   ¿Es más complejo?

4. ¿Para ZRANGEBYSCORE con millones de resultados,
   ¿el skip list es eficiente?

5. ¿Redis 7+ tiene multi-threading parcial.
   ¿Eso cambia la decisión?

---

### Ejercicio 12.7.2 — Resumen: las reglas del skip list

**Tipo: Leer**

```
Reglas nuevas del Cap.12:

  Regla 50: Skip list = datos ordenados sin rotaciones.
    Mismas operaciones que un Red-Black tree: search, insert, delete,
    range scan en O(log n) esperado. Pero sin rotaciones,
    sin recolorings, sin los 10+ cases de delete.
    La altura se decide con un coin flip. Simple y efectivo.

  Regla 51: Skip list gana en concurrencia.
    Inserts son locales (solo relinkean punteros adyacentes).
    CAS (Compare-And-Swap) permite lock-free inserts.
    ConcurrentSkipListMap de Java: la ÚNICA estructura sorted
    concurrent en el stdlib.
    5-10× más rápido que SynchronizedTreeMap bajo contención.

  Regla 52: p = 0.25 es el sweet spot.
    Menos niveles, menos punteros, menos comparaciones que p = 0.5.
    Redis: p = 0.25, MAX_LEVEL = 32 (~4 billón de elementos).
    RocksDB: p = 0.25, MAX_LEVEL = 12 (~16M elementos por MemTable).

  Regla 53: Skip list + hash table = Redis ZSET.
    Hash table para O(1) membership. Skip list para sorted operations.
    El span por nivel da rank() en O(log n).
    La combinación resuelve un problema que ninguna estructura
    sola resuelve eficientemente.

  La Parte 4 (Búsqueda y Texto) está completa:
    Cap.11: Tries → búsqueda por prefijo, HTTP routers, FST.
    Cap.12: Skip lists → datos ordenados, concurrencia, Redis.

  Parte 5: Concurrencia y Sistemas Distribuidos.
    Cap.13: Queues → FIFO, lock-free, Disruptor, Kafka.
    Cap.14: Estructuras lock-free → CAS, ABA problem, hazard pointers.
```

**Preguntas:**

1. ¿Las 53 reglas cubren 12 capítulos. ¿Cuántas quedan para los 6 restantes?

2. ¿La Regla 51 (concurrencia) es la razón de existir del skip list?

3. ¿Un data engineer debería saber implementar un skip list
   o solo entender cuándo usarlo?

4. ¿La Parte 5 (Concurrencia) es la más relevante para
   sistemas distribuidos?

5. ¿El proyecto final (Cap.18) usará un skip list como MemTable?
