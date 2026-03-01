# Guía de Ejercicios — Cap.07: Árboles Binarios de Búsqueda: de BST a AVL a Red-Black

> Un hash map responde "¿existe esta key?" en O(1).
> Pero no puede responder:
>   "¿Cuál es la key más cercana a X?"
>   "Dame todas las keys entre A y B."
>   "¿Cuál es la key mínima? ¿La máxima?"
>   "Dame las keys en orden."
>
> Para eso necesitas datos ORDENADOS.
> Y para datos ordenados con insert/delete eficiente,
> necesitas un árbol de búsqueda.
>
> Un sorted array hace binary search en O(log n),
> pero insertar es O(n) (hay que mover todo).
> Un BST hace búsqueda, inserción Y eliminación en O(log n)
> — si está balanceado.
>
> "Si está balanceado" es la frase clave.
> Un BST sin balanceo puede degenerar en una linked list: O(n).
> La historia de los árboles de búsqueda es la historia
> de cómo GARANTIZAR el balance.
>
> AVL: balance estricto (altura difiere ≤1). Lecturas rápidas.
> Red-Black: balance relajado (altura ≤ 2× óptima). Escrituras rápidas.
> LLRB: Red-Black simplificado. La mitad de casos.
>
> Este capítulo implementa los tres desde cero.
> Al final, sabrás por qué Java eligió Red-Black para TreeMap,
> por qué Linux eligió Red-Black para el CFS scheduler,
> y por qué casi nadie usa AVL en producción.

---

## El modelo mental: del array ordenado al árbol

```
Datos ordenados: [2, 5, 8, 12, 15, 18, 23, 27, 31]

  SORTED ARRAY:
    Search: O(log n) — binary search. Excelente.
    Insert: O(n) — mover todos los elementos. Terrible.
    Delete: O(n) — mover todos los elementos. Terrible.
    Range query: O(log n + k) — encontrar inicio, luego scan. Excelente.

  BINARY SEARCH TREE (BST):
    Search: O(log n) si balanceado. O(n) si degenerado.
    Insert: O(log n) si balanceado. O(n) si degenerado.
    Delete: O(log n) si balanceado. O(n) si degenerado.
    Range query: O(log n + k). Excelente.

              15
           /      \
         8         23
        / \       /  \
       5   12   18    27
      /              /  \
     2             ...   31

  Cada nodo tiene: key, value, left child, right child.
  Invariante: left.key < node.key < right.key.
  Inorder traversal: visitar left, luego node, luego right → datos ordenados.

  El problema:
    Insertar [2, 5, 8, 12, 15, 18, 23] EN ORDEN produce:
    2
     \
      5
       \
        8
         \
          12
           \
            15
             \
              18
               \
                23
    → Linked list. Search, insert, delete: O(n).

  Solución: REBALANCEAR después de cada insert/delete.
    AVL: rotaciones para mantener |height(left) - height(right)| ≤ 1.
    Red-Black: colorear nodos + rotaciones para mantener altura ≤ 2 × log₂(n).
```

---

## Tabla de contenidos

- [Sección 7.1 — BST básico: la estructura fundacional](#sección-71--bst-básico-la-estructura-fundacional)
- [Sección 7.2 — El problema del desbalance](#sección-72--el-problema-del-desbalance)
- [Sección 7.3 — AVL Trees: balance estricto con rotaciones](#sección-73--avl-trees-balance-estricto-con-rotaciones)
- [Sección 7.4 — Red-Black Trees: balance relajado para escrituras](#sección-74--red-black-trees-balance-relajado-para-escrituras)
- [Sección 7.5 — Left-Leaning Red-Black Trees: la simplificación de Sedgewick](#sección-75--left-leaning-red-black-trees-la-simplificación-de-sedgewick)
- [Sección 7.6 — Benchmark: BST vs AVL vs Red-Black vs sorted array](#sección-76--benchmark-bst-vs-avl-vs-red-black-vs-sorted-array)
- [Sección 7.7 — Árboles en producción: dónde y por qué](#sección-77--árboles-en-producción-dónde-y-por-qué)

---

## Sección 7.1 — BST Básico: la Estructura Fundacional

### Ejercicio 7.1.1 — Implementar: BST en Python (claridad)

**Tipo: Implementar**

```python
class BSTNode:
    __slots__ = ('key', 'value', 'left', 'right')

    def __init__(self, key, value):
        self.key = key
        self.value = value
        self.left = None
        self.right = None

class BST:
    def __init__(self):
        self.root = None
        self.size = 0

    def put(self, key, value):
        self.root = self._put(self.root, key, value)

    def _put(self, node, key, value):
        if node is None:
            self.size += 1
            return BSTNode(key, value)
        if key < node.key:
            node.left = self._put(node.left, key, value)
        elif key > node.key:
            node.right = self._put(node.right, key, value)
        else:
            node.value = value  # update existing
        return node

    def get(self, key):
        node = self.root
        while node is not None:
            if key < node.key:
                node = node.left
            elif key > node.key:
                node = node.right
            else:
                return node.value
        return None

    def contains(self, key):
        return self.get(key) is not None

    def min_key(self):
        if self.root is None: return None
        node = self.root
        while node.left is not None:
            node = node.left
        return node.key

    def max_key(self):
        if self.root is None: return None
        node = self.root
        while node.right is not None:
            node = node.right
        return node.key

    def inorder(self):
        """Yield keys in sorted order"""
        def _inorder(node):
            if node is not None:
                yield from _inorder(node.left)
                yield node.key
                yield from _inorder(node.right)
        return _inorder(self.root)

    def range_query(self, lo, hi):
        """Yield keys where lo <= key <= hi"""
        def _range(node):
            if node is None: return
            if lo < node.key:
                yield from _range(node.left)
            if lo <= node.key <= hi:
                yield node.key
            if node.key < hi:
                yield from _range(node.right)
        return _range(self.root)

    def height(self):
        def _height(node):
            if node is None: return 0
            return 1 + max(_height(node.left), _height(node.right))
        return _height(self.root)

    def delete(self, key):
        self.root = self._delete(self.root, key)

    def _delete(self, node, key):
        if node is None: return None
        if key < node.key:
            node.left = self._delete(node.left, key)
        elif key > node.key:
            node.right = self._delete(node.right, key)
        else:
            # Found the node to delete
            self.size -= 1
            if node.left is None: return node.right
            if node.right is None: return node.left
            # Two children: replace with in-order successor
            successor = node.right
            while successor.left is not None:
                successor = successor.left
            node.key = successor.key
            node.value = successor.value
            self.size += 1  # compensate: _delete will decrement again
            node.right = self._delete(node.right, successor.key)
        return node


# ═══ Test ═══
import random
bst = BST()
keys = list(range(1000))
random.shuffle(keys)
for k in keys:
    bst.put(k, k * 10)

print(f"Size: {bst.size}, Height: {bst.height()}")
print(f"Min: {bst.min_key()}, Max: {bst.max_key()}")
print(f"get(42) = {bst.get(42)}")
print(f"Range [10, 20]: {list(bst.range_query(10, 20))}")

# Optimal height for 1000 nodes: log₂(1000) ≈ 10
# With random inserts, expected height: ~2 × log₂(1000) ≈ 20
print(f"Optimal height: {int(1000).bit_length()}, Actual: {bst.height()}")
```

**Preguntas:**

1. ¿`_put` es recursivo. ¿Puede causar stack overflow
   para un BST degenerado con 1M nodos?

2. ¿`get` es iterativo pero `_put` es recursivo. ¿Por qué la diferencia?

3. ¿Delete con dos hijos usa el "in-order successor". ¿Se podría
   usar el "in-order predecessor"?

4. ¿`range_query` tiene complejidad O(log n + k) donde k es el número
   de resultados. ¿Por qué?

5. ¿`__slots__` en BSTNode — ¿cuánta memoria ahorra en Python?

---

### Ejercicio 7.1.2 — Implementar: BST en Java con generics

**Tipo: Implementar**

```java
public class BST<K extends Comparable<K>, V> {

    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> left, right;
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    private Node<K, V> root;
    private int size;

    public void put(K key, V value) {
        root = put(root, key, value);
    }

    private Node<K, V> put(Node<K, V> node, K key, V value) {
        if (node == null) { size++; return new Node<>(key, value); }
        int cmp = key.compareTo(node.key);
        if (cmp < 0)      node.left = put(node.left, key, value);
        else if (cmp > 0)  node.right = put(node.right, key, value);
        else                node.value = value;
        return node;
    }

    public V get(K key) {
        Node<K, V> node = root;
        while (node != null) {
            int cmp = key.compareTo(node.key);
            if (cmp < 0)      node = node.left;
            else if (cmp > 0)  node = node.right;
            else                return node.value;
        }
        return null;
    }

    // Floor: largest key ≤ given key
    public K floor(K key) {
        Node<K, V> node = root;
        K result = null;
        while (node != null) {
            int cmp = key.compareTo(node.key);
            if (cmp == 0) return node.key;
            if (cmp < 0) {
                node = node.left;
            } else {
                result = node.key; // candidate
                node = node.right;
            }
        }
        return result;
    }

    // Ceiling: smallest key ≥ given key
    public K ceiling(K key) {
        Node<K, V> node = root;
        K result = null;
        while (node != null) {
            int cmp = key.compareTo(node.key);
            if (cmp == 0) return node.key;
            if (cmp > 0) {
                node = node.right;
            } else {
                result = node.key;
                node = node.left;
            }
        }
        return result;
    }

    public int size() { return size; }

    public int height() { return height(root); }
    private int height(Node<K, V> n) {
        if (n == null) return 0;
        return 1 + Math.max(height(n.left), height(n.right));
    }

    public static void main(String[] args) {
        var bst = new BST<Integer, String>();
        var rng = new java.util.Random(42);
        int n = 1_000_000;

        long start = System.nanoTime();
        for (int i = 0; i < n; i++) bst.put(rng.nextInt(n * 10), "v" + i);
        long elapsed = System.nanoTime() - start;
        System.out.printf("Insert %d random: %d ms, height=%d (optimal≈%d)%n",
            n, elapsed / 1_000_000, bst.height(), (int)(Math.log(n) / Math.log(2)));

        // Floor/Ceiling
        System.out.println("floor(500000) = " + bst.floor(500000));
        System.out.println("ceiling(500000) = " + bst.ceiling(500000));
    }
}
// Output esperado:
// Insert 1000000 random: ~2000-4000 ms, height=~40-45 (optimal≈20)
// floor/ceiling work correctly
```

**Preguntas:**

1. ¿`Comparable<K>` vs `Comparator<K>` — ¿cuándo usar cada uno?

2. ¿Floor y ceiling son operaciones O(log n). ¿Un HashMap puede hacerlo?

3. ¿La altura de ~42 para 1M inserts aleatorios — ¿es ~2× la óptima?

4. ¿`put` asigna un nuevo Node por cada insert.
   ¿Cuánta presión sobre el GC para 1M inserts?

5. ¿TreeMap de Java usa BST? ¿O algo más sofisticado?

---

### Ejercicio 7.1.3 — Analizar: operaciones que solo un BST puede hacer

**Tipo: Analizar**

```
Operaciones que el BST soporta y el HashMap no:

  OPERACIÓN               BST          HashMap    Sorted Array
  ─────────               ───          ───────    ────────────
  get(key)                O(log n)     O(1)       O(log n)
  put(key, value)         O(log n)     O(1) amort O(n)
  delete(key)             O(log n)     O(1) amort O(n)
  min() / max()           O(log n)     O(n)       O(1)
  floor(key)              O(log n)     O(n)       O(log n)
  ceiling(key)            O(log n)     O(n)       O(log n)
  range(lo, hi)           O(log n + k) O(n)       O(log n + k)
  rank(key)               O(log n)*    O(n)       O(log n)
  select(rank)            O(log n)*    O(n)       O(1)
  inorder traversal       O(n)         O(n log n)†O(n)

  * Con augmented BST (almacenar subtree size en cada nodo).
  † HashMap iteration + sort.

  Conclusión:
  - HashMap gana para get/put/delete aislados.
  - BST gana para CUALQUIER operación que requiera ORDEN:
    min, max, floor, ceiling, range, rank, ordered iteration.
  - Sorted array gana para datos estáticos (sin inserts/deletes).

  Regla de decisión:
  ¿Necesitas solo get/put/delete? → HashMap.
  ¿Necesitas operaciones de orden? → BST (TreeMap, BTreeMap).
  ¿Datos estáticos + búsqueda? → Sorted array + binary search.
```

**Preguntas:**

1. ¿`rank(key)` retorna cuántas keys son menores que key.
   ¿Cómo se implementa en un BST augmentado?

2. ¿`select(3)` retorna la 3ra key más pequeña.
   ¿Es posible en O(log n)?

3. ¿Un hash map ordenado (LinkedHashMap) puede hacer range queries?

4. ¿Para un índice de base de datos con range scans frecuentes,
   ¿BST o hash index?

5. ¿Kafka usa TreeMap para offset indexes.
   ¿Qué operaciones de orden necesita?

---

### Ejercicio 7.1.4 — Implementar: range query benchmark — BST vs sorted ArrayList

**Tipo: Implementar**

```java
import java.util.*;

public class RangeQueryBenchmark {

    public static void main(String[] args) {
        int n = 1_000_000;
        var rng = new Random(42);
        int[] data = new int[n];
        for (int i = 0; i < n; i++) data[i] = rng.nextInt(n * 10);

        // Build TreeMap
        var treeMap = new TreeMap<Integer, Integer>();
        for (int d : data) treeMap.put(d, d);

        // Build sorted array
        int[] sorted = treeMap.keySet().stream().mapToInt(Integer::intValue).toArray();

        int queries = 10_000;
        int rangeSize = 100; // expect ~100 results per query

        // Generate random range queries
        int[][] ranges = new int[queries][2];
        for (int i = 0; i < queries; i++) {
            int lo = rng.nextInt(n * 10 - rangeSize * 10);
            ranges[i][0] = lo;
            ranges[i][1] = lo + rangeSize * 10;
        }

        // Warmup
        for (int[] r : ranges) {
            treeMap.subMap(r[0], r[1]).size();
            Arrays.binarySearch(sorted, r[0]);
        }

        // Benchmark: TreeMap.subMap
        long start = System.nanoTime();
        long totalResults = 0;
        for (int[] r : ranges) {
            totalResults += treeMap.subMap(r[0], r[1]).size();
        }
        long treeTime = System.nanoTime() - start;

        // Benchmark: sorted array with binary search
        start = System.nanoTime();
        long totalResults2 = 0;
        for (int[] r : ranges) {
            int lo = Arrays.binarySearch(sorted, r[0]);
            if (lo < 0) lo = -(lo + 1);
            int hi = Arrays.binarySearch(sorted, r[1]);
            if (hi < 0) hi = -(hi + 1);
            totalResults2 += (hi - lo);
        }
        long arrTime = System.nanoTime() - start;

        System.out.printf("Range queries (%d queries, ~%d results each):%n", queries, totalResults / queries);
        System.out.printf("  TreeMap.subMap: %d ms (%.0f ns/query)%n",
            treeTime / 1_000_000, (double) treeTime / queries);
        System.out.printf("  Sorted array:  %d ms (%.0f ns/query)%n",
            arrTime / 1_000_000, (double) arrTime / queries);
        System.out.printf("  Ratio: %.1f×%n", (double) treeTime / arrTime);
    }
}
// Resultado esperado:
// TreeMap.subMap: ~50-100 ms (~5000-10000 ns/query)
// Sorted array:  ~10-30 ms (~1000-3000 ns/query)
// Ratio: ~3-5× (sorted array gana por cache locality)
```

**Preguntas:**

1. ¿El sorted array es 3-5× más rápido para range queries.
   ¿Es la cache locality?

2. ¿`TreeMap.subMap()` retorna una VISTA, no una copia.
   ¿Eso afecta el benchmark?

3. ¿Si los datos cambian frecuentemente (inserts/deletes),
   ¿el sorted array sigue siendo viable?

4. ¿Un B-Tree (Cap.09) tendría mejor cache locality que un Red-Black tree?

5. ¿Para un índice de base de datos que necesita range scans Y updates,
   ¿qué estructura es mejor?

---

### Ejercicio 7.1.5 — Analizar: por qué el BST no es suficiente

**Tipo: Analizar**

```
El BST sin balanceo tiene un defecto fatal:
la altura depende del ORDEN de inserción.

  Inserts aleatorios (1M claves):
    Altura esperada: ~2 × log₂(n) ≈ 40.
    Search: ~40 comparaciones. Aceptable.

  Inserts ordenados (1, 2, 3, ..., 1M):
    Altura: n = 1,000,000.
    Search: ~1,000,000 comparaciones. Desastre.

  Inserts casi-ordenados (sorted con pequeñas perturbaciones):
    Altura: ~n/perturbación. Sigue siendo malo.

  En producción, los datos llegan frecuentemente en orden
  o casi en orden:
  - Timestamps (monótonamente crecientes)
  - Auto-increment IDs
  - Datos pre-sorted de un pipeline
  - Log entries

  Un BST sin balanceo con datos ordenados
  es una linked list que pretende ser un árbol.

  → Necesitamos GARANTÍAS de altura máxima.
  → AVL: h ≤ 1.44 × log₂(n). Estricto.
  → Red-Black: h ≤ 2 × log₂(n). Relajado.
  → Ambos: O(log n) garantizado para CUALQUIER secuencia de inserts.
```

**Preguntas:**

1. ¿Inserts aleatorios dan altura ~2× la óptima.
   ¿Es suficientemente bueno sin balanceo?

2. ¿Si los datos llegan en orden aleatorio,
   ¿el BST sin balanceo es aceptable?

3. ¿Randomizar los inserts (shuffle antes de insertar)
   evita el problema?

4. ¿Un skip list (Cap.10) resuelve el problema de otra forma?

5. ¿El peor caso del BST sin balanceo es teórico
   o ocurre en producción?

---

## Sección 7.2 — El Problema del Desbalance

### Ejercicio 7.2.1 — Implementar: demostrar la degeneración

**Tipo: Implementar**

```java
public class BSTDegeneration {

    static int bstHeight(int[] keys) {
        int[] left = new int[keys.length + 1];
        int[] right = new int[keys.length + 1];
        int[] vals = new int[keys.length + 1];
        java.util.Arrays.fill(left, -1);
        java.util.Arrays.fill(right, -1);
        int root = -1;
        int nodeCount = 0;

        for (int key : keys) {
            int newNode = nodeCount++;
            vals[newNode] = key;
            if (root == -1) { root = newNode; continue; }
            int curr = root;
            while (true) {
                if (key < vals[curr]) {
                    if (left[curr] == -1) { left[curr] = newNode; break; }
                    curr = left[curr];
                } else if (key > vals[curr]) {
                    if (right[curr] == -1) { right[curr] = newNode; break; }
                    curr = right[curr];
                } else break; // duplicate
            }
        }

        // Compute height iteratively with stack
        if (root == -1) return 0;
        int maxH = 0;
        int[] stack = new int[keys.length];
        int[] heights = new int[keys.length];
        int top = 0;
        stack[top] = root; heights[top] = 1; top++;
        while (top > 0) {
            top--;
            int node = stack[top]; int h = heights[top];
            maxH = Math.max(maxH, h);
            if (left[node] != -1) { stack[top] = left[node]; heights[top] = h + 1; top++; }
            if (right[node] != -1) { stack[top] = right[node]; heights[top] = h + 1; top++; }
        }
        return maxH;
    }

    public static void main(String[] args) {
        int n = 100_000;
        var rng = new java.util.Random(42);

        // Case 1: random
        int[] random = new int[n];
        for (int i = 0; i < n; i++) random[i] = rng.nextInt(n * 10);
        int hRandom = bstHeight(random);

        // Case 2: sorted ascending
        int[] sorted = new int[n];
        for (int i = 0; i < n; i++) sorted[i] = i;
        // Don't actually build this BST — would take O(n²)
        // Just report the known height
        int hSorted = n; // degenerate: linked list

        // Case 3: sorted then shuffled slightly (5% perturbation)
        int[] almostSorted = sorted.clone();
        for (int i = 0; i < n * 5 / 100; i++) {
            int a = rng.nextInt(n), b = rng.nextInt(n);
            int tmp = almostSorted[a]; almostSorted[a] = almostSorted[b]; almostSorted[b] = tmp;
        }
        int hAlmost = bstHeight(almostSorted);

        int optimal = (int)(Math.log(n) / Math.log(2));
        System.out.printf("n = %,d (optimal height = %d)%n", n, optimal);
        System.out.printf("Random inserts:       height = %d (%.1f× optimal)%n", hRandom, (double) hRandom / optimal);
        System.out.printf("Sorted inserts:       height = %,d (%.0f× optimal) — DEGENERATE%n", hSorted, (double) hSorted / optimal);
        System.out.printf("Almost-sorted (5%%):   height = %,d (%.0f× optimal)%n", hAlmost, (double) hAlmost / optimal);
    }
}
// Output esperado:
// n = 100,000 (optimal height = 16)
// Random inserts:       height = ~34 (2.1× optimal)
// Sorted inserts:       height = 100,000 (6250× optimal) — DEGENERATE
// Almost-sorted (5%):   height = ~5,000 (312× optimal)
```

**Preguntas:**

1. ¿6,250× la altura óptima para inserts ordenados. ¿Es esto aceptable
   en cualquier escenario?

2. ¿5% de perturbación reduce la altura de 100K a ~5K.
   ¿Sigue siendo 312× peor que óptimo?

3. ¿Construir un BST con n inserts ordenados cuesta O(n²) en total.
   ¿Por qué?

4. ¿Un treap (BST + random priorities) evita la degeneración?

5. ¿Si Java TreeMap fuera un BST sin balanceo,
   ¿qué pasaría con datos de timestamps?

---

## Sección 7.3 — AVL Trees: Balance Estricto con Rotaciones

### Ejercicio 7.3.1 — Leer: qué es un AVL tree y cómo funciona

**Tipo: Leer**

```
AVL tree (Adelson-Velsky y Landis, 1962):
  Un BST donde la diferencia de altura entre subárboles
  izquierdo y derecho de CADA nodo es como máximo 1.

  Balance factor = height(left) - height(right)
  Invariante: balance_factor ∈ {-1, 0, +1} para TODOS los nodos.

  Si un insert o delete viola el invariante (|balance| > 1),
  se rebalancea con ROTACIONES.

  4 casos de desbalance y sus rotaciones:

  CASO 1: Left-Left (LL) → Rotación derecha
    z está desbalanceado (balance = +2)
    y es hijo izquierdo de z
    x es hijo izquierdo de y
         z (+2)                y (0)
        / \                   / \
       y   T4      →        x   z
      / \                  / \ / \
     x   T3              T1 T2 T3 T4
    / \
   T1  T2

  CASO 2: Right-Right (RR) → Rotación izquierda
    Simétrico al LL.
         z (-2)               y (0)
        / \                  / \
      T1   y       →       z   x
          / \              / \ / \
         T2  x            T1 T2 T3 T4
            / \
           T3  T4

  CASO 3: Left-Right (LR) → Rotación izquierda en y, luego derecha en z
         z (+2)               z (+2)              x (0)
        / \                  / \                 / \
       y   T4    →         x   T4    →         y   z
      / \                 / \                  / \ / \
     T1  x              y   T3               T1 T2 T3 T4
        / \            / \
       T2  T3         T1  T2

  CASO 4: Right-Left (RL) → Rotación derecha en y, luego izquierda en z
    Simétrico al LR.

  Garantías:
    Altura máxima: h ≤ 1.44 × log₂(n + 2) - 0.328
    Para n = 1M: h ≤ 1.44 × 20 ≈ 29 (vs óptimo 20).
    Para n = 1M BST degenerado: h = 1,000,000.

  Costo de las rotaciones: O(1) — solo reasignar punteros.
  Máximo rotaciones por insert: 2 (una doble rotación).
  Máximo rotaciones por delete: O(log n) (puede rebalancear todo el path).
```

**Preguntas:**

1. ¿AVL garantiza h ≤ 1.44 × log₂(n). ¿Es un factor constante?

2. ¿Una rotación es O(1)? ¿Cuántos punteros cambian?

3. ¿Insert necesita máximo 2 rotaciones pero delete necesita O(log n)?
   ¿Por qué la diferencia?

4. ¿El balance factor se almacena en cada nodo? ¿O se recalcula?

5. ¿Si nunca haces deletes, ¿el AVL es perfecto?

---

### Ejercicio 7.3.2 — Implementar: AVL tree en Java

**Tipo: Implementar**

```java
public class AVLTree<K extends Comparable<K>, V> {

    private static class Node<K, V> {
        K key; V value;
        Node<K, V> left, right;
        int height;
        Node(K key, V value) { this.key = key; this.value = value; this.height = 1; }
    }

    private Node<K, V> root;
    private int size;

    private int height(Node<K, V> n) { return n == null ? 0 : n.height; }
    private int balance(Node<K, V> n) { return n == null ? 0 : height(n.left) - height(n.right); }

    private void updateHeight(Node<K, V> n) {
        n.height = 1 + Math.max(height(n.left), height(n.right));
    }

    // ═══ Rotations ═══

    private Node<K, V> rotateRight(Node<K, V> z) {
        Node<K, V> y = z.left;
        z.left = y.right;
        y.right = z;
        updateHeight(z);
        updateHeight(y);
        return y;
    }

    private Node<K, V> rotateLeft(Node<K, V> z) {
        Node<K, V> y = z.right;
        z.right = y.left;
        y.left = z;
        updateHeight(z);
        updateHeight(y);
        return y;
    }

    private Node<K, V> rebalance(Node<K, V> node) {
        updateHeight(node);
        int bal = balance(node);

        if (bal > 1) { // Left-heavy
            if (balance(node.left) < 0) // Left-Right case
                node.left = rotateLeft(node.left);
            return rotateRight(node); // Left-Left case
        }

        if (bal < -1) { // Right-heavy
            if (balance(node.right) > 0) // Right-Left case
                node.right = rotateRight(node.right);
            return rotateLeft(node); // Right-Right case
        }

        return node; // balanced
    }

    // ═══ Insert ═══

    public void put(K key, V value) { root = put(root, key, value); }

    private Node<K, V> put(Node<K, V> node, K key, V value) {
        if (node == null) { size++; return new Node<>(key, value); }
        int cmp = key.compareTo(node.key);
        if (cmp < 0)      node.left = put(node.left, key, value);
        else if (cmp > 0)  node.right = put(node.right, key, value);
        else              { node.value = value; return node; }
        return rebalance(node);
    }

    // ═══ Delete ═══

    public void delete(K key) { root = delete(root, key); }

    private Node<K, V> delete(Node<K, V> node, K key) {
        if (node == null) return null;
        int cmp = key.compareTo(node.key);
        if (cmp < 0) node.left = delete(node.left, key);
        else if (cmp > 0) node.right = delete(node.right, key);
        else {
            size--;
            if (node.left == null) return node.right;
            if (node.right == null) return node.left;
            // Find in-order successor
            Node<K, V> succ = node.right;
            while (succ.left != null) succ = succ.left;
            node.key = succ.key; node.value = succ.value;
            size++; // compensate
            node.right = delete(node.right, succ.key);
        }
        return rebalance(node);
    }

    public V get(K key) {
        Node<K, V> n = root;
        while (n != null) {
            int cmp = key.compareTo(n.key);
            if (cmp < 0) n = n.left;
            else if (cmp > 0) n = n.right;
            else return n.value;
        }
        return null;
    }

    public int size() { return size; }
    public int height() { return height(root); }

    public static void main(String[] args) {
        var avl = new AVLTree<Integer, Integer>();
        int n = 1_000_000;

        // Worst case for BST: sorted inserts
        long start = System.nanoTime();
        for (int i = 0; i < n; i++) avl.put(i, i);
        long elapsed = System.nanoTime() - start;

        System.out.printf("AVL sorted inserts (%,d): %d ms%n", n, elapsed / 1_000_000);
        System.out.printf("Height: %d (optimal: %d, max AVL: %d)%n",
            avl.height(), (int)(Math.log(n) / Math.log(2)),
            (int)(1.44 * Math.log(n) / Math.log(2)));

        // Lookup
        start = System.nanoTime();
        long dummy = 0;
        for (int i = 0; i < n; i++) dummy += avl.get(i);
        long lookupTime = System.nanoTime() - start;
        System.out.printf("Lookup %,d: %d ms (%.0f ns/op)%n",
            n, lookupTime / 1_000_000, (double) lookupTime / n);
        if (dummy == Long.MIN_VALUE) System.out.println(dummy);
    }
}
// Output esperado:
// AVL sorted inserts (1,000,000): ~2000-3000 ms
// Height: 20 (optimal: 19, max AVL: 28) — BALANCED even with sorted input!
// Lookup 1,000,000: ~400-600 ms (~400-600 ns/op)
```

**Preguntas:**

1. ¿Sorted inserts en AVL dan altura 20, cerca del óptimo 19.
   ¿El BST sin balanceo daría 1,000,000?

2. ¿`rebalance()` se llama en CADA nivel de la recursión.
   ¿Es costoso?

3. ¿El AVL tree con 1M sorted inserts tarda ~2-3 segundos.
   ¿Es aceptable?

4. ¿Lookup de 400-600 ns es más lento que HashMap (28 ns).
   ¿27× más lento?

5. ¿El AVL tree es mejor que Red-Black para lookups
   porque tiene menor altura?

---

## Sección 7.4 — Red-Black Trees: Balance Relajado para Escrituras

### Ejercicio 7.4.1 — Leer: las 5 propiedades del Red-Black tree

**Tipo: Leer**

```
Un Red-Black tree es un BST con las siguientes propiedades:

  1. Cada nodo es ROJO o NEGRO.
  2. La raíz es NEGRA.
  3. Cada hoja (NIL/null) es NEGRA.
  4. Si un nodo es ROJO, sus dos hijos son NEGROS.
     (No hay dos nodos rojos consecutivos.)
  5. Para cada nodo, todos los paths desde el nodo hasta
     las hojas NIL tienen el MISMO número de nodos negros.
     (La "black-height" es uniforme.)

  Consecuencia de estas propiedades:
    El path más largo es como máximo 2× el path más corto.
    Proof: el path más corto tiene solo nodos negros (bh nodos).
           el path más largo alterna rojo-negro (2×bh nodos).
    → Altura máxima: h ≤ 2 × log₂(n + 1).
    → Para n = 1M: h ≤ 2 × 20 = 40 (vs AVL: h ≤ 29).

  El Red-Black tree es MENOS balanceado que el AVL:
    AVL: h ≤ 1.44 × log₂(n)  → lookups más rápidos.
    RBT: h ≤ 2 × log₂(n)     → lookups más lentos.

  Pero el Red-Black tree necesita MENOS rotaciones:
    AVL insert: hasta 2 rotaciones + update heights por path.
    RBT insert: hasta 2 rotaciones + recolorings (O(log n) pero baratos).
    AVL delete: hasta O(log n) rotaciones.
    RBT delete: hasta 3 rotaciones + recolorings.

  → Red-Black es mejor para workloads con MUCHAS escrituras.
  → AVL es mejor para workloads con MUCHAS lecturas.

  Uso real:
    Java TreeMap:          Red-Black tree.
    C++ std::map:          Red-Black tree (típicamente).
    Linux CFS scheduler:   Red-Black tree (procesos ordenados por vruntime).
    epoll (Linux):         Red-Black tree (file descriptors).
```

**Preguntas:**

1. ¿La "black-height" — ¿es fácil de mantener?

2. ¿Recoloring es más barato que rotación. ¿Por qué?

3. ¿Java eligió Red-Black sobre AVL para TreeMap.
   ¿Fue por el ratio reads/writes?

4. ¿Linux CFS usa un Red-Black tree de procesos.
   ¿Qué operaciones son más frecuentes?

5. ¿La altura máxima de 2× log₂(n) vs 1.44× — ¿cuánto
   importa para n = 1M?

---

### Ejercicio 7.4.2 — Implementar: Red-Black tree insert en Java (fix-up)

**Tipo: Implementar**

```java
public class RedBlackTree<K extends Comparable<K>, V> {

    private static final boolean RED = true;
    private static final boolean BLACK = false;

    private static class Node<K, V> {
        K key; V value;
        Node<K, V> left, right, parent;
        boolean color;
        Node(K key, V value, boolean color, Node<K, V> parent) {
            this.key = key; this.value = value; this.color = color; this.parent = parent;
        }
    }

    private Node<K, V> root;
    private int size;

    private boolean isRed(Node<K, V> n) { return n != null && n.color == RED; }

    // ═══ Rotations ═══

    private void rotateLeft(Node<K, V> x) {
        Node<K, V> y = x.right;
        x.right = y.left;
        if (y.left != null) y.left.parent = x;
        y.parent = x.parent;
        if (x.parent == null) root = y;
        else if (x == x.parent.left) x.parent.left = y;
        else x.parent.right = y;
        y.left = x;
        x.parent = y;
    }

    private void rotateRight(Node<K, V> x) {
        Node<K, V> y = x.left;
        x.left = y.right;
        if (y.right != null) y.right.parent = x;
        y.parent = x.parent;
        if (x.parent == null) root = y;
        else if (x == x.parent.right) x.parent.right = y;
        else x.parent.left = y;
        y.right = x;
        x.parent = y;
    }

    // ═══ Insert ═══

    public void put(K key, V value) {
        Node<K, V> parent = null;
        Node<K, V> curr = root;

        while (curr != null) {
            parent = curr;
            int cmp = key.compareTo(curr.key);
            if (cmp < 0) curr = curr.left;
            else if (cmp > 0) curr = curr.right;
            else { curr.value = value; return; } // update
        }

        Node<K, V> newNode = new Node<>(key, value, RED, parent);
        if (parent == null) root = newNode;
        else if (key.compareTo(parent.key) < 0) parent.left = newNode;
        else parent.right = newNode;
        size++;

        fixAfterInsert(newNode);
    }

    private void fixAfterInsert(Node<K, V> z) {
        while (z != root && isRed(z.parent)) {
            if (z.parent == z.parent.parent.left) {
                Node<K, V> uncle = z.parent.parent.right;
                if (isRed(uncle)) {
                    // Case 1: uncle is red → recolor
                    z.parent.color = BLACK;
                    uncle.color = BLACK;
                    z.parent.parent.color = RED;
                    z = z.parent.parent;
                } else {
                    if (z == z.parent.right) {
                        // Case 2: z is right child → rotate left
                        z = z.parent;
                        rotateLeft(z);
                    }
                    // Case 3: z is left child → rotate right
                    z.parent.color = BLACK;
                    z.parent.parent.color = RED;
                    rotateRight(z.parent.parent);
                }
            } else {
                // Mirror: parent is right child of grandparent
                Node<K, V> uncle = z.parent.parent.left;
                if (isRed(uncle)) {
                    z.parent.color = BLACK;
                    uncle.color = BLACK;
                    z.parent.parent.color = RED;
                    z = z.parent.parent;
                } else {
                    if (z == z.parent.left) {
                        z = z.parent;
                        rotateRight(z);
                    }
                    z.parent.color = BLACK;
                    z.parent.parent.color = RED;
                    rotateLeft(z.parent.parent);
                }
            }
        }
        root.color = BLACK;
    }

    public V get(K key) {
        Node<K, V> n = root;
        while (n != null) {
            int cmp = key.compareTo(n.key);
            if (cmp < 0) n = n.left;
            else if (cmp > 0) n = n.right;
            else return n.value;
        }
        return null;
    }

    public int size() { return size; }

    public int height() { return height(root); }
    private int height(Node<K, V> n) {
        if (n == null) return 0;
        return 1 + Math.max(height(n.left), height(n.right));
    }

    public static void main(String[] args) {
        var rbt = new RedBlackTree<Integer, Integer>();
        int n = 1_000_000;

        // Sorted inserts
        long start = System.nanoTime();
        for (int i = 0; i < n; i++) rbt.put(i, i);
        long elapsed = System.nanoTime() - start;
        System.out.printf("RBT sorted inserts (%,d): %d ms%n", n, elapsed / 1_000_000);
        System.out.printf("Height: %d (max RBT: %d)%n",
            rbt.height(), (int)(2 * Math.log(n) / Math.log(2)));
    }
}
// Output esperado:
// RBT sorted inserts (1,000,000): ~1500-2500 ms
// Height: ~25-30 (max RBT: 40) — well within bounds
```

**Preguntas:**

1. ¿`fixAfterInsert` tiene 3 cases + mirror. ¿Es mucho código?

2. ¿Case 1 (recoloring) puede propagarse hasta la raíz.
   ¿Es O(log n)?

3. ¿Cases 2 y 3 requieren rotaciones. ¿Máximo cuántas por insert?

4. ¿El parent pointer usa 8 bytes extra por nodo. ¿Vale la pena?

5. ¿Nuestra implementación es comparable a `java.util.TreeMap`?

---

### Ejercicio 7.4.3 — Analizar: AVL vs Red-Black — la decisión real

**Tipo: Analizar**

```
Comparación rigurosa para n = 1M:

  Métrica                 AVL              Red-Black
  ──────                  ───              ─────────
  Max height              ~29              ~40
  Avg comparisons/lookup  ~19-20           ~22-25
  Rotations per insert    ≤ 2              ≤ 2 (+ recolorings)
  Rotations per delete    O(log n)         ≤ 3 (+ recolorings)
  Memory per node         +4 bytes (height) +1 bit (color)
  Insert throughput       Base             ~5-10% más rápido
  Lookup throughput       ~5-10% más rápido Base
  Delete throughput       Base             ~10-20% más rápido

  ¿Quién gana?
  - Read-heavy (90% reads): AVL (lookups más rápidos por menor altura).
  - Write-heavy (50%+ writes): Red-Black (menos rotaciones en delete).
  - Mixed (equal reads/writes): Red-Black (la diferencia en lookups es pequeña,
    la ventaja en deletes es significativa).
  - General purpose: Red-Black (por eso Java, C++, Linux lo eligieron).

  ¿Por qué Red-Black domina en la práctica?
  1. Delete de AVL es complicado (hasta log n rotaciones).
  2. Red-Black delete: máximo 3 rotaciones (el resto son recolorings baratos).
  3. La diferencia en lookup (5-10%) rara vez justifica la complejidad de AVL delete.
  4. Red-Black es más simple de implementar correctamente.
     (Excepto LLRB que es aún más simple — Sección 7.5.)

  Excepciones donde AVL gana:
  - Base de datos read-only (lookup-heavy, zero deletes).
  - Diccionarios inmutables (build once, read many).
  - Embedded systems con memoria limitada (4 bytes vs 8 para parent pointer).
```

**Preguntas:**

1. ¿La diferencia de 5-10% en lookups justifica usar AVL?

2. ¿Si nunca haces deletes, ¿AVL es estrictamente mejor?

3. ¿B-Tree (Cap.09) hace que tanto AVL como Red-Black sean
   irrelevantes para almacenamiento en disco?

4. ¿Hay un caso real donde la diferencia entre AVL y Red-Black
   afecte la latencia de producción?

5. ¿Scala usa Red-Black tree internamente para `TreeMap`?

---

## Sección 7.5 — Left-Leaning Red-Black Trees: la Simplificación de Sedgewick

### Ejercicio 7.5.1 — Leer: LLRB — la mitad de código del Red-Black estándar

**Tipo: Leer**

```
Robert Sedgewick (2008) observó que un Red-Black tree
tiene demasiados casos a manejar.
Propuso una restricción adicional:

  LLRB constraint: los enlaces rojos solo van a la IZQUIERDA.
  (Un nodo rojo solo puede ser hijo izquierdo.)

  Esta restricción reduce los casos de insert fix-up de 6 a 3,
  y los de delete de ~10 a ~5.
  El resultado: una implementación de ~50 líneas vs ~150+ del RBT estándar.

  Correspondencia: un LLRB es isomorfo a un 2-3 tree.
    Nodo negro con hijo rojo izquierdo = 3-node en 2-3 tree.
    Nodo negro solo = 2-node en 2-3 tree.

  Tres operaciones de rebalanceo:

  1. ROTATE LEFT: si el hijo derecho es rojo y el izquierdo no.
     (Restaurar el invariante "lean left".)

  2. ROTATE RIGHT: si el hijo izquierdo Y su hijo izquierdo son rojos.
     (Dos rojos consecutivos a la izquierda = temporal 4-node, split.)

  3. FLIP COLORS: si ambos hijos son rojos.
     (Ambos hijos rojos = 4-node, split it up.)

  Insert en LLRB:
    1. Insertar como en BST normal, con color ROJO.
    2. En el camino de vuelta (después de la recursión):
       if (isRed(right) && !isRed(left))    → rotateLeft
       if (isRed(left) && isRed(left.left)) → rotateRight
       if (isRed(left) && isRed(right))     → flipColors
    3. root.color = BLACK

  Eso es todo. 3 líneas de rebalanceo.
```

**Preguntas:**

1. ¿"Solo enlaces rojos a la izquierda" — ¿pierde generalidad?

2. ¿La correspondencia con 2-3 trees — ¿ayuda a entender el LLRB?

3. ¿3 líneas de rebalanceo vs 20+ del RBT estándar.
   ¿Hay un costo en rendimiento?

4. ¿Delete en LLRB es más simple que en RBT estándar?

5. ¿Go usa un LLRB internamente para algo?

---

### Ejercicio 7.5.2 — Implementar: LLRB en Scala con pattern matching

**Tipo: Implementar**

```scala
sealed trait Color
case object Red extends Color
case object Black extends Color

case class LLRBNode[K: Ordering, V](
    key: K, value: V,
    color: Color = Red,
    left: Option[LLRBNode[K, V]] = None,
    right: Option[LLRBNode[K, V]] = None
) {
  def isRed: Boolean = color == Red
}

class LLRB[K: Ordering, V] {
  private var root: Option[LLRBNode[K, V]] = None
  private var _size: Int = 0

  def size: Int = _size

  private def isRed(node: Option[LLRBNode[K, V]]): Boolean =
    node.exists(_.isRed)

  private def rotateLeft(h: LLRBNode[K, V]): LLRBNode[K, V] = {
    val x = h.right.get
    val newH = h.copy(right = x.left, color = Red)
    x.copy(left = Some(newH), color = h.color)
  }

  private def rotateRight(h: LLRBNode[K, V]): LLRBNode[K, V] = {
    val x = h.left.get
    val newH = h.copy(left = x.right, color = Red)
    x.copy(right = Some(newH), color = h.color)
  }

  private def flipColors(h: LLRBNode[K, V]): LLRBNode[K, V] = {
    h.copy(
      color = Red,
      left = h.left.map(_.copy(color = Black)),
      right = h.right.map(_.copy(color = Black))
    )
  }

  private def balance(h: LLRBNode[K, V]): LLRBNode[K, V] = {
    var node = h
    if (isRed(node.right) && !isRed(node.left))    node = rotateLeft(node)
    if (isRed(node.left) && isRed(node.left.flatMap(_.left))) node = rotateRight(node)
    if (isRed(node.left) && isRed(node.right))      node = flipColors(node)
    node
  }

  def put(key: K, value: V): Unit = {
    root = Some(insert(root, key, value).copy(color = Black))
  }

  private def insert(node: Option[LLRBNode[K, V]], key: K, value: V): LLRBNode[K, V] = {
    val ord = implicitly[Ordering[K]]
    node match {
      case None =>
        _size += 1
        LLRBNode(key, value, Red)
      case Some(n) =>
        val cmp = ord.compare(key, n.key)
        val updated =
          if (cmp < 0) n.copy(left = Some(insert(n.left, key, value)))
          else if (cmp > 0) n.copy(right = Some(insert(n.right, key, value)))
          else { n.copy(value = value) } // no size change for update
        balance(updated)
    }
  }

  def get(key: K): Option[V] = {
    val ord = implicitly[Ordering[K]]
    var node = root
    while (node.isDefined) {
      val n = node.get
      val cmp = ord.compare(key, n.key)
      if (cmp < 0) node = n.left
      else if (cmp > 0) node = n.right
      else return Some(n.value)
    }
    None
  }

  def height: Int = {
    def h(n: Option[LLRBNode[K, V]]): Int = n match {
      case None => 0
      case Some(node) => 1 + math.max(h(node.left), h(node.right))
    }
    h(root)
  }
}

object LLRBDemo extends App {
  val tree = new LLRB[Int, Int]

  // Sorted inserts — the worst case for an unbalanced BST
  val n = 100_000
  val start = System.nanoTime()
  for (i <- 0 until n) tree.put(i, i * 10)
  val elapsed = (System.nanoTime() - start) / 1_000_000

  println(s"LLRB sorted inserts ($n): $elapsed ms")
  println(s"Height: ${tree.height} (optimal: ${(math.log(n) / math.log(2)).toInt})")
  println(s"get(42) = ${tree.get(42)}")
  println(s"get(-1) = ${tree.get(-1)}")
}
```

**Preguntas:**

1. ¿La implementación en Scala usa `copy()` (case classes inmutables).
   ¿Eso crea muchos objetos temporales?

2. ¿`balance()` son 3 if-statements. ¿El RBT estándar tiene 6+ cases?

3. ¿Esta implementación es funcional (inmutable por nodo)?
   ¿Podría ser un persistent LLRB?

4. ¿El pattern matching de Scala hace el código más legible
   que los if-else de Java?

5. ¿Go's `container/list` y `container/heap` están en la std library,
   pero no un tree balanceado. ¿Por qué?

---

### Ejercicio 7.5.3 — Implementar: LLRB mutable en Rust

**Tipo: Implementar**

```rust
#[derive(Clone, Copy, PartialEq)]
enum Color { Red, Black }

struct Node<K, V> {
    key: K,
    value: V,
    color: Color,
    left: Option<Box<Node<K, V>>>,
    right: Option<Box<Node<K, V>>>,
}

pub struct LLRB<K: Ord, V> {
    root: Option<Box<Node<K, V>>>,
    size: usize,
}

impl<K: Ord, V> LLRB<K, V> {
    pub fn new() -> Self { LLRB { root: None, size: 0 } }

    fn is_red(node: &Option<Box<Node<K, V>>>) -> bool {
        matches!(node, Some(n) if n.color == Color::Red)
    }

    fn rotate_left(mut h: Box<Node<K, V>>) -> Box<Node<K, V>> {
        let mut x = h.right.take().unwrap();
        h.right = x.left.take();
        x.color = h.color;
        h.color = Color::Red;
        x.left = Some(h);
        x
    }

    fn rotate_right(mut h: Box<Node<K, V>>) -> Box<Node<K, V>> {
        let mut x = h.left.take().unwrap();
        h.left = x.right.take();
        x.color = h.color;
        h.color = Color::Red;
        x.right = Some(h);
        x
    }

    fn flip_colors(h: &mut Box<Node<K, V>>) {
        h.color = match h.color { Color::Red => Color::Black, Color::Black => Color::Red };
        if let Some(ref mut l) = h.left {
            l.color = match l.color { Color::Red => Color::Black, Color::Black => Color::Red };
        }
        if let Some(ref mut r) = h.right {
            r.color = match r.color { Color::Red => Color::Black, Color::Black => Color::Red };
        }
    }

    fn balance(mut h: Box<Node<K, V>>) -> Box<Node<K, V>> {
        if Self::is_red(&h.right) && !Self::is_red(&h.left) {
            h = Self::rotate_left(h);
        }
        if Self::is_red(&h.left) && Self::is_red(&h.left.as_ref().unwrap().left) {
            h = Self::rotate_right(h);
        }
        if Self::is_red(&h.left) && Self::is_red(&h.right) {
            Self::flip_colors(&mut h);
        }
        h
    }

    pub fn put(&mut self, key: K, value: V) {
        self.root = Some(self.insert(self.root.take(), key, value));
        if let Some(ref mut r) = self.root { r.color = Color::Black; }
    }

    fn insert(&mut self, node: Option<Box<Node<K, V>>>, key: K, value: V) -> Box<Node<K, V>> {
        match node {
            None => {
                self.size += 1;
                Box::new(Node { key, value, color: Color::Red, left: None, right: None })
            }
            Some(mut n) => {
                match key.cmp(&n.key) {
                    std::cmp::Ordering::Less => n.left = Some(self.insert(n.left.take(), key, value)),
                    std::cmp::Ordering::Greater => n.right = Some(self.insert(n.right.take(), key, value)),
                    std::cmp::Ordering::Equal => n.value = value,
                }
                Self::balance(n)
            }
        }
    }

    pub fn get(&self, key: &K) -> Option<&V> {
        let mut node = &self.root;
        while let Some(ref n) = node {
            match key.cmp(&n.key) {
                std::cmp::Ordering::Less => node = &n.left,
                std::cmp::Ordering::Greater => node = &n.right,
                std::cmp::Ordering::Equal => return Some(&n.value),
            }
        }
        None
    }

    pub fn size(&self) -> usize { self.size }
}

fn main() {
    let mut tree = LLRB::new();
    let n = 1_000_000;

    let start = std::time::Instant::now();
    for i in 0..n { tree.put(i, i * 2); }
    println!("LLRB sorted inserts ({}): {:?}", n, start.elapsed());
    println!("Size: {}", tree.size());
    println!("get(42) = {:?}", tree.get(&42));
}
```

**Preguntas:**

1. ¿`Option<Box<Node>>` — ¿es el patrón idiomático para árboles en Rust?

2. ¿`take()` mueve el ownership del nodo. ¿Es necesario para la rotación?

3. ¿Sin GC, ¿cuándo se libera la memoria de nodos eliminados?

4. ¿Esta implementación es más rápida que la de Scala?

5. ¿`BTreeMap` de Rust (B-Tree) es siempre mejor que un LLRB
   para uso general?

---

## Sección 7.6 — Benchmark: BST vs AVL vs Red-Black vs Sorted Array

### Ejercicio 7.6.1 — Implementar: benchmark comparativo en Java

**Tipo: Implementar**

```java
import java.util.*;

public class TreeBenchmark {

    public static void main(String[] args) {
        int n = 1_000_000;
        var rng = new Random(42);

        // Prepare data: random and sorted
        int[] randomKeys = new int[n];
        for (int i = 0; i < n; i++) randomKeys[i] = rng.nextInt(n * 10);
        int[] sortedKeys = new int[n];
        for (int i = 0; i < n; i++) sortedKeys[i] = i;

        System.out.println("═══ INSERT BENCHMARK (1M entries) ═══\n");

        for (String label : new String[]{"Random", "Sorted"}) {
            int[] keys = label.equals("Random") ? randomKeys : sortedKeys;
            System.out.println("--- " + label + " inserts ---");

            // TreeMap (Red-Black)
            var tm = new TreeMap<Integer, Integer>();
            long start = System.nanoTime();
            for (int k : keys) tm.put(k, k);
            long tmTime = System.nanoTime() - start;

            // HashMap (for comparison)
            var hm = new HashMap<Integer, Integer>(n * 2);
            start = System.nanoTime();
            for (int k : keys) hm.put(k, k);
            long hmTime = System.nanoTime() - start;

            System.out.printf("  TreeMap (RBT):  %4d ms  (%.0f ns/op)%n",
                tmTime / 1_000_000, (double) tmTime / n);
            System.out.printf("  HashMap:        %4d ms  (%.0f ns/op)%n",
                hmTime / 1_000_000, (double) hmTime / n);
            System.out.printf("  Ratio:          %.1f× (HashMap faster)%n%n",
                (double) tmTime / hmTime);
        }

        // Lookup benchmark
        System.out.println("═══ LOOKUP BENCHMARK (1M lookups) ═══\n");
        var treeMap = new TreeMap<Integer, Integer>();
        var hashMap = new HashMap<Integer, Integer>(n * 2);
        for (int k : randomKeys) { treeMap.put(k, k); hashMap.put(k, k); }

        // Sorted array + binary search
        int[] sortedArr = treeMap.keySet().stream().mapToInt(Integer::intValue).toArray();

        int[] queries = new int[n];
        for (int i = 0; i < n; i++) queries[i] = randomKeys[rng.nextInt(n)];

        // Warmup
        long dummy = 0;
        for (int i = 0; i < 10000; i++) {
            dummy += treeMap.getOrDefault(queries[i % n], 0);
            dummy += hashMap.getOrDefault(queries[i % n], 0);
            dummy += Arrays.binarySearch(sortedArr, queries[i % n]);
        }

        long start = System.nanoTime();
        for (int q : queries) dummy += treeMap.getOrDefault(q, 0);
        long treeTime = System.nanoTime() - start;

        start = System.nanoTime();
        for (int q : queries) dummy += hashMap.getOrDefault(q, 0);
        long hashTime = System.nanoTime() - start;

        start = System.nanoTime();
        for (int q : queries) dummy += Arrays.binarySearch(sortedArr, q);
        long arrTime = System.nanoTime() - start;

        System.out.printf("  TreeMap:        %4d ms  (%.0f ns/op)%n",
            treeTime / 1_000_000, (double) treeTime / n);
        System.out.printf("  HashMap:        %4d ms  (%.0f ns/op)%n",
            hashTime / 1_000_000, (double) hashTime / n);
        System.out.printf("  Sorted array:   %4d ms  (%.0f ns/op)%n",
            arrTime / 1_000_000, (double) arrTime / n);

        if (dummy == Long.MIN_VALUE) System.out.println(dummy);
    }
}
// Resultado esperado:
// ═══ INSERT ═══
// Random inserts:
//   TreeMap (RBT):  ~800 ms  (~800 ns/op)
//   HashMap:        ~200 ms  (~200 ns/op)
//   Ratio: ~4× (HashMap faster)
// Sorted inserts:
//   TreeMap (RBT):  ~600 ms  (~600 ns/op)  ← RBT handles sorted well
//   HashMap:        ~150 ms  (~150 ns/op)
//
// ═══ LOOKUP ═══
//   TreeMap:         ~600 ms  (~600 ns/op)  — pointer chasing, cache misses
//   HashMap:         ~200 ms  (~200 ns/op)  — O(1) amortized
//   Sorted array:    ~300 ms  (~300 ns/op)  — binary search, good locality
```

**Preguntas:**

1. ¿TreeMap es 4× más lento que HashMap para inserts.
   ¿Y 3× más lento para lookups?

2. ¿Sorted array es más rápido que TreeMap para lookups. ¿Cache locality?

3. ¿Sorted inserts son más rápidos que random inserts en TreeMap.
   ¿Por qué?

4. ¿Para un workload de 50% inserts + 50% lookups,
   ¿TreeMap es viable?

5. ¿`BTreeMap` de Rust sería más rápido que `TreeMap` de Java
   para lookups?

---

## Sección 7.7 — Árboles en Producción: Dónde y Por Qué

### Ejercicio 7.7.1 — Analizar: árboles en la infraestructura real

**Tipo: Analizar**

```
Dónde se usan árboles balanceados en producción:

  JAVA TreeMap (Red-Black):
    - Kafka: TimerTaskList ordena operaciones delayed por expiration time.
    - Kafka: OffsetIndex usa un ConcurrentSkipListMap (similar a tree).
    - Cualquier lugar que necesite datos ordenados en memoria.

  LINUX CFS SCHEDULER (Red-Black):
    - Cada proceso tiene un "virtual runtime" (vruntime).
    - Los procesos se almacenan en un Red-Black tree ordenado por vruntime.
    - El scheduler siempre ejecuta el proceso con el MENOR vruntime.
    - min() es O(log n) en el RBT (realmente O(1) porque cachean el leftmost).
    - Insert/delete al cambiar estado de procesos: O(log n).

  LINUX EPOLL (Red-Black):
    - Los file descriptors monitoreados se almacenan en un RBT.
    - Lookup por fd: O(log n).
    - Insert/delete: O(log n).
    - ¿Por qué no un hash map? Porque epoll necesita ordered iteration
      para ciertos paths internos del kernel.

  C++ std::map / std::set (Red-Black):
    - Estándar de C++. Ordered. O(log n) garantizado.
    - Muchos game engines, sistemas embebidos, databases lo usan.

  DATABASES — En memoria:
    - Redis sorted sets (ZSETs) usan skip list + hash table.
      ¿Por qué no Red-Black? Skip list es más simple para rangos.
    - SQLite usa B-Tree (Cap.09), no Red-Black.
    - Los árboles binarios son para MEMORIA.
      Para DISCO, los B-Trees dominan (Cap.09).

  Regla: Red-Black tree = estructura por defecto para
  datos ordenados en memoria. B-Tree = para disco.
```

**Preguntas:**

1. ¿Linux CFS cachea el nodo más izquierdo. ¿Eso convierte min()
   de O(log n) a O(1)?

2. ¿Redis usa skip list en vez de Red-Black tree.
   ¿Es mejor para range queries?

3. ¿Para disco, ¿por qué B-Tree es mejor que Red-Black?

4. ¿epoll usa RBT para file descriptors. ¿Cuántos fds
   maneja un server típico?

5. ¿En 10 años, ¿los árboles binarios seguirán siendo relevantes
   o los B-Trees los reemplazarán completamente?

---

### Ejercicio 7.7.2 — Resumen: las reglas de los árboles

**Tipo: Leer**

```
Reglas nuevas del Cap.07:

  Regla 26: El BST sin balanceo es inútil en producción.
    Datos ordenados → linked list. O(n) garantizado.
    SIEMPRE usa un árbol balanceado: AVL, Red-Black, o B-Tree.

  Regla 27: Red-Black tree es el default para datos ordenados en memoria.
    Menos rotaciones que AVL en deletes.
    Java TreeMap, C++ std::map, Linux CFS, Linux epoll.
    5-10% más lento que AVL en lookups, pero más rápido en writes.

  Regla 28: Los árboles son para ORDEN, no para velocidad de acceso.
    HashMap: 28 ns/lookup. TreeMap: 600 ns/lookup. 20× más lento.
    Usa un árbol SOLO cuando necesitas operaciones de orden:
    min, max, floor, ceiling, range, sorted iteration.

  Regla 29: Sorted array vence al árbol si los datos no cambian.
    Binary search en sorted array: mejor cache locality que tree traversal.
    Si los datos son estáticos (no inserts/deletes), usa sorted array.

  Regla 30: LLRB es Red-Black simplificado — ideal para aprender.
    3 líneas de rebalanceo vs 20+ del RBT estándar.
    Mismo O(log n) garantizado. Mismas aplicaciones.

  La Parte 3 continúa:
    Cap.08: Heaps (arrays como árboles implícitos).
    Cap.09: B-Trees (árboles anchos para disco).
    Cap.10: LSM-Trees (escribir rápido, leer después).
```

**Preguntas:**

1. ¿Las 30 reglas hasta ahora son memorables y accionables?

2. ¿La Regla 28 es la más importante del capítulo?

3. ¿Falta una regla sobre cuándo AVL es mejor que Red-Black?

4. ¿Un data engineer necesita implementar un RBT desde cero?
   ¿O basta con usar TreeMap?

5. ¿El Cap.09 (B-Trees) hará que los árboles binarios
   parezcan obsoletos?
