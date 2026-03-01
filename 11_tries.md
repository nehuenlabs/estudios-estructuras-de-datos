# Guía de Ejercicios — Cap.11: Tries y Estructuras para Strings

> Escribes "amaz" en la barra de búsqueda de Google.
> En menos de 50 ms, aparecen:
>   "amazon", "amazon prime", "amazing race", "amazon music"
>
> ¿Cómo buscas entre millones de sugerencias
> por prefijo en menos de 50 ms?
>
> Un HashMap no sirve. HashMap busca por CLAVE EXACTA.
> Para encontrar todas las claves que empiezan con "amaz",
> necesitarías recorrer TODAS las claves: O(n).
>
> Un sorted array con binary search puede encontrar
> el rango de prefijos en O(log n + k),
> pero insertar una sugerencia nueva cuesta O(n).
>
> Un Trie (del inglés "reTRIEval") resuelve ambos:
> - Buscar por prefijo: O(m) donde m = longitud del prefijo.
>   No depende de n (número total de claves).
> - Insert: O(m).
> - Autocomplete: O(m + k) donde k = resultados.
>
> El trie es la estructura detrás del autocomplete,
> los HTTP routers de Go (chi, gin, httprouter),
> las routing tables de Linux (LPC trie),
> y los diccionarios de Lucene/Elasticsearch (FST).
>
> Este capítulo implementa tres variantes:
> 1. Trie básico: la idea pura.
> 2. Radix tree (compressed trie): lo que usan los routers HTTP.
> 3. Suffix array: buscar CUALQUIER substring, no solo prefijos.

---

## El modelo mental: de hash a árbol de prefijos

```
Problema: dado un conjunto de strings, encontrar todos los que
empiezan con un prefijo dado.

  HashMap:
    {"amazon": 1, "amazing": 2, "apple": 3, "app": 4, "azure": 5}
    prefix("am") → recorrer TODAS las keys. O(n). No escala.

  Trie (prefix tree):
    Cada nodo es un carácter. Cada camino root→leaf es una clave.

              (root)
             /  |   \
            a   .    .
           / \
          m   p
         / \    \
        a   a    p
        |   |   / \
        z   z  l   (✓ "app")
        |   |  |
        o   i  e
        |   |  |
        n   n  (✓ "apple")
        |   |
     (✓)   g
   "amazon" |
          (✓)
        "amazing"

    prefix("am"):
      root → 'a' → 'm' → collect all descendants.
      Resultado: "amazon", "amazing". O(m + k).
      No recorres las keys que empiezan con 'p' ni con 'z'.

  El trie ORGANIZA las claves por su estructura.
  Las claves con prefijo común COMPARTEN nodos.
  Buscar por prefijo = navegar por el árbol = O(longitud del prefijo).
```

---

## Tabla de contenidos

- [Sección 11.1 — Trie básico: la idea y la implementación](#sección-111--trie-básico-la-idea-y-la-implementación)
- [Sección 11.2 — Implementar un trie con autocomplete](#sección-112--implementar-un-trie-con-autocomplete)
- [Sección 11.3 — Compressed trie (radix tree)](#sección-113--compressed-trie-radix-tree)
- [Sección 11.4 — Radix tree como HTTP router](#sección-114--radix-tree-como-http-router)
- [Sección 11.5 — Suffix arrays: buscar cualquier substring](#sección-115--suffix-arrays-buscar-cualquier-substring)
- [Sección 11.6 — Benchmark: trie vs hash map vs sorted array](#sección-116--benchmark-trie-vs-hash-map-vs-sorted-array)
- [Sección 11.7 — Tries en producción: de routers a Elasticsearch](#sección-117--tries-en-producción-de-routers-a-elasticsearch)

---

## Sección 11.1 — Trie Básico: la Idea y la Implementación

### Ejercicio 11.1.1 — Leer: anatomía de un trie

**Tipo: Leer**

```
Un trie (también llamado prefix tree o digital tree):

  Estructura:
    Cada nodo tiene:
    - Un array (o map) de hijos, indexado por carácter.
    - Un flag "is_end" que indica si este nodo termina una clave.
    - (Opcionalmente) un valor asociado a la clave.

  Para un alfabeto de 26 letras minúsculas:
    Cada nodo tiene un array de 26 punteros a hijos.
    children[0] = hijo para 'a', children[1] = hijo para 'b', ...

  Operaciones:

  INSERT("cat"):
    root → children['c'] (crear si no existe)
         → children['a'] (crear si no existe)
         → children['t'] (crear si no existe, marcar is_end=true)

  SEARCH("cat"):
    root → children['c'] (existe? sí)
         → children['a'] (existe? sí)
         → children['t'] (existe? sí, is_end? sí) → FOUND

  SEARCH("ca"):
    root → children['c'] → children['a'] (is_end? no) → NOT FOUND
    "ca" no es una clave, aunque es un prefijo de "cat".

  STARTS_WITH("ca"):
    root → children['c'] → children['a'] (existe? sí) → TRUE
    El nodo para "ca" existe → hay al menos una clave con ese prefijo.

  Complejidad:
    Insert:      O(m) donde m = longitud de la clave.
    Search:      O(m).
    Starts_with: O(m).
    Autocomplete (prefix + collect): O(m + k) donde k = resultados.

    ¡No depende de n (número total de claves)!
    1 millón de claves o 1 billón: la búsqueda es O(m).

  Memoria:
    Cada nodo: 26 punteros × 8 bytes = 208 bytes (para ASCII lowercase).
    Para Unicode: necesitarías un HashMap por nodo en vez de array.
    Un trie con 1M strings de 10 caracteres promedio:
    ~10M nodos × 208 bytes = ~2 GB. Mucha memoria.
    → La motivación del compressed trie (radix tree).
```

**Preguntas:**

1. ¿O(m) independiente de n — ¿es mejor que HashMap O(1) amortizado?

2. ¿208 bytes por nodo para solo 26 letras. ¿Es mucha memoria?

3. ¿Para Unicode con ~150K caracteres, ¿un array por nodo es imposible?

4. ¿Un trie almacena prefijos compartidos UNA sola vez.
   ¿Eso ahorra memoria comparado con un HashMap?

5. ¿`is_end` distingue "prefijo" de "clave completa".
   ¿Es necesario siempre?

---

### Ejercicio 11.1.2 — Implementar: trie básico en Python

**Tipo: Implementar**

```python
class TrieNode:
    __slots__ = ('children', 'is_end', 'value')

    def __init__(self):
        self.children = {}  # char → TrieNode
        self.is_end = False
        self.value = None

class Trie:
    def __init__(self):
        self.root = TrieNode()
        self.size = 0

    def insert(self, key: str, value=None):
        node = self.root
        for ch in key:
            if ch not in node.children:
                node.children[ch] = TrieNode()
            node = node.children[ch]
        if not node.is_end:
            self.size += 1
        node.is_end = True
        node.value = value

    def search(self, key: str):
        node = self._find_node(key)
        if node and node.is_end:
            return node.value
        return None

    def contains(self, key: str) -> bool:
        node = self._find_node(key)
        return node is not None and node.is_end

    def starts_with(self, prefix: str) -> bool:
        return self._find_node(prefix) is not None

    def autocomplete(self, prefix: str, limit: int = 10) -> list:
        """Return up to `limit` keys that start with `prefix`"""
        node = self._find_node(prefix)
        if node is None:
            return []
        results = []
        self._collect(node, list(prefix), results, limit)
        return results

    def _find_node(self, prefix: str):
        node = self.root
        for ch in prefix:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node

    def _collect(self, node, path, results, limit):
        if len(results) >= limit:
            return
        if node.is_end:
            results.append(''.join(path))
        for ch in sorted(node.children):
            path.append(ch)
            self._collect(node.children[ch], path, results, limit)
            path.pop()

    def __len__(self):
        return self.size


# ═══ Test ═══
trie = Trie()
words = ["apple", "app", "application", "apply", "apt",
         "amazon", "amazing", "amaze", "azure", "bat", "bath"]
for w in words:
    trie.insert(w, len(w))

print(f"Size: {len(trie)}")
print(f"search('app') = {trie.search('app')}")
print(f"search('ap') = {trie.search('ap')}")
print(f"contains('apple') = {trie.contains('apple')}")
print(f"starts_with('app') = {trie.starts_with('app')}")
print(f"starts_with('xyz') = {trie.starts_with('xyz')}")
print(f"autocomplete('app') = {trie.autocomplete('app')}")
print(f"autocomplete('am') = {trie.autocomplete('am')}")
print(f"autocomplete('a') = {trie.autocomplete('a', limit=5)}")

# Benchmark
import time, random, string

n = 100_000
keys = [''.join(random.choices(string.ascii_lowercase, k=random.randint(5, 15))) for _ in range(n)]

trie = Trie()
start = time.perf_counter_ns()
for k in keys:
    trie.insert(k)
insert_time = time.perf_counter_ns() - start

start = time.perf_counter_ns()
found = sum(1 for k in keys[:10000] if trie.contains(k))
search_time = time.perf_counter_ns() - start

start = time.perf_counter_ns()
for _ in range(10000):
    prefix = random.choice(keys)[:3]
    trie.autocomplete(prefix, limit=5)
auto_time = time.perf_counter_ns() - start

print(f"\nBenchmark ({n:,} keys):")
print(f"  Insert: {insert_time // 1_000_000} ms ({insert_time // n} ns/op)")
print(f"  Search: {search_time // 1_000_000} ms ({search_time // 10000} ns/op)")
print(f"  Autocomplete: {auto_time // 1_000_000} ms ({auto_time // 10000} ns/op)")
```

**Preguntas:**

1. ¿`children` como dict vs array de 26 — ¿cuál usa menos memoria?

2. ¿`autocomplete` con `limit=10` deja de buscar temprano. ¿Es O(m+k)?

3. ¿`sorted(node.children)` en `_collect` — ¿eso da resultados
   en orden lexicográfico gratis?

4. ¿Para 100K strings aleatorias, ¿el trie usa más memoria
   que un dict de Python?

5. ¿El trie soporta delete? ¿Es difícil de implementar?

---

### Ejercicio 11.1.3 — Implementar: trie en Java con array de 26

**Tipo: Implementar**

```java
public class Trie {

    private static final int ALPHABET = 26;

    private static class Node {
        Node[] children = new Node[ALPHABET];
        boolean isEnd;
        int value;
    }

    private final Node root = new Node();
    private int size;

    public void insert(String key, int value) {
        Node node = root;
        for (int i = 0; i < key.length(); i++) {
            int idx = key.charAt(i) - 'a';
            if (node.children[idx] == null)
                node.children[idx] = new Node();
            node = node.children[idx];
        }
        if (!node.isEnd) size++;
        node.isEnd = true;
        node.value = value;
    }

    public boolean contains(String key) {
        Node node = findNode(key);
        return node != null && node.isEnd;
    }

    public boolean startsWith(String prefix) {
        return findNode(prefix) != null;
    }

    public java.util.List<String> autocomplete(String prefix, int limit) {
        Node node = findNode(prefix);
        if (node == null) return java.util.Collections.emptyList();
        var results = new java.util.ArrayList<String>();
        collect(node, new StringBuilder(prefix), results, limit);
        return results;
    }

    private Node findNode(String prefix) {
        Node node = root;
        for (int i = 0; i < prefix.length(); i++) {
            int idx = prefix.charAt(i) - 'a';
            if (node.children[idx] == null) return null;
            node = node.children[idx];
        }
        return node;
    }

    private void collect(Node node, StringBuilder sb, java.util.List<String> results, int limit) {
        if (results.size() >= limit) return;
        if (node.isEnd) results.add(sb.toString());
        for (int c = 0; c < ALPHABET; c++) {
            if (node.children[c] != null) {
                sb.append((char)('a' + c));
                collect(node.children[c], sb, results, limit);
                sb.deleteCharAt(sb.length() - 1);
            }
        }
    }

    public int size() { return size; }

    // Memory estimate
    public long nodeCount() { return countNodes(root); }
    private long countNodes(Node n) {
        if (n == null) return 0;
        long count = 1;
        for (Node child : n.children)
            count += countNodes(child);
        return count;
    }

    public static void main(String[] args) {
        var trie = new Trie();
        int n = 1_000_000;
        var rng = new java.util.Random(42);

        // Generate random words
        var words = new String[n];
        for (int i = 0; i < n; i++) {
            int len = 5 + rng.nextInt(11); // 5-15 chars
            var sb = new StringBuilder(len);
            for (int j = 0; j < len; j++)
                sb.append((char)('a' + rng.nextInt(26)));
            words[i] = sb.toString();
        }

        // Insert
        long start = System.nanoTime();
        for (String w : words) trie.insert(w, 1);
        long insertTime = System.nanoTime() - start;

        // Search
        start = System.nanoTime();
        int found = 0;
        for (int i = 0; i < 100_000; i++) {
            if (trie.contains(words[rng.nextInt(n)])) found++;
        }
        long searchTime = System.nanoTime() - start;

        long nodes = trie.nodeCount();
        long memEst = nodes * (26 * 8 + 16); // 26 pointers + object header

        System.out.printf("Trie: %,d keys, %,d nodes%n", trie.size(), nodes);
        System.out.printf("Estimated memory: %,d MB%n", memEst / (1024 * 1024));
        System.out.printf("Insert %,d: %d ms (%.0f ns/op)%n",
            n, insertTime / 1_000_000, (double) insertTime / n);
        System.out.printf("Search 100K: %d ms (%.0f ns/op), found=%d%n",
            searchTime / 1_000_000, (double) searchTime / 100_000, found);
    }
}
// Resultado esperado:
// Trie: ~999,000 keys, ~5-8M nodes
// Estimated memory: ~1000-1600 MB (¡mucha memoria!)
// Insert: ~1500-3000 ms
// Search: ~100-200 ms
// → El trie es rápido para search pero usa MUCHA memoria.
```

**Preguntas:**

1. ¿~1.5 GB para 1M strings aleatorias. ¿Un HashMap usaría cuánto?

2. ¿Cada nodo tiene 26 punteros incluso si solo 1-2 están llenos.
   ¿Cuánto desperdicio?

3. ¿El trie escala a 100M strings? ¿O la memoria es prohibitiva?

4. ¿Un HashMap con `startsWith` manual sería más práctico
   para 1M strings?

5. ¿La solución al problema de memoria es el compressed trie?

---

## Sección 11.2 — Implementar un Trie con Autocomplete

### Ejercicio 11.2.1 — Implementar: autocomplete con ranking en Go

**Tipo: Implementar**

```go
package main

import (
    "fmt"
    "sort"
    "strings"
)

type TrieNode struct {
    children map[byte]*TrieNode
    isEnd    bool
    weight   int // popularity/frequency
}

type Trie struct {
    root *TrieNode
    size int
}

func NewTrie() *Trie {
    return &Trie{root: &TrieNode{children: make(map[byte]*TrieNode)}}
}

func (t *Trie) Insert(key string, weight int) {
    node := t.root
    for i := 0; i < len(key); i++ {
        ch := key[i]
        if _, ok := node.children[ch]; !ok {
            node.children[ch] = &TrieNode{children: make(map[byte]*TrieNode)}
        }
        node = node.children[ch]
    }
    if !node.isEnd {
        t.size++
    }
    node.isEnd = true
    node.weight = weight
}

type Suggestion struct {
    Word   string
    Weight int
}

func (t *Trie) Autocomplete(prefix string, limit int) []Suggestion {
    node := t.root
    for i := 0; i < len(prefix); i++ {
        child, ok := node.children[prefix[i]]
        if !ok {
            return nil
        }
        node = child
    }

    var results []Suggestion
    var buf []byte
    buf = append(buf, prefix...)
    t.collect(node, buf, &results)

    // Sort by weight descending
    sort.Slice(results, func(i, j int) bool {
        return results[i].Weight > results[j].Weight
    })

    if len(results) > limit {
        results = results[:limit]
    }
    return results
}

func (t *Trie) collect(node *TrieNode, buf []byte, results *[]Suggestion) {
    if node.isEnd {
        *results = append(*results, Suggestion{string(buf), node.weight})
    }
    for ch := byte(0); ch < 128; ch++ {
        if child, ok := node.children[ch]; ok {
            t.collect(child, append(buf, ch), results)
        }
    }
}

func main() {
    trie := NewTrie()

    // Simulate search suggestions with popularity weights
    suggestions := map[string]int{
        "amazon":       10000, "amazon prime":  8000,
        "amazing":       5000, "amazing race":  3000,
        "amaze":         1000, "amber":          500,
        "apple":         9000, "apple music":   7000,
        "application":   6000, "apply":          4000,
        "app store":     8500, "app":            2000,
        "azure":         7500, "azure devops":  4500,
    }
    for word, weight := range suggestions {
        trie.Insert(word, weight)
    }

    prefixes := []string{"am", "app", "az", "a", "amazon p"}
    for _, p := range prefixes {
        results := trie.Autocomplete(p, 5)
        strs := make([]string, len(results))
        for i, r := range results {
            strs[i] = fmt.Sprintf("%s(%d)", r.Word, r.Weight)
        }
        fmt.Printf("'%s' → [%s]\n", p, strings.Join(strs, ", "))
    }
}
// Output:
// 'am' → [amazon(10000), amazon prime(8000), amazing(5000), amazing race(3000), amaze(1000)]
// 'app' → [app store(8500), application(6000), apply(4000), app(2000)]
// 'az' → [azure(7500), azure devops(4500)]
// 'a'  → [amazon(10000), apple(9000), app store(8500), ...]
// 'amazon p' → [amazon prime(8000)]
```

**Preguntas:**

1. ¿El ranking por weight — ¿Google hace algo similar?

2. ¿Collect ALL y luego sort es O(k log k). ¿Se puede hacer mejor
   con un heap de tamaño `limit`?

3. ¿`map[byte]*TrieNode` en Go — ¿cuánta memoria overhead vs array?

4. ¿Para sugerencias de búsqueda real con 100M queries,
   ¿el trie cabe en RAM?

5. ¿Google usa un trie para autocomplete? ¿O algo más sofisticado?

---

## Sección 11.3 — Compressed Trie (Radix Tree)

### Ejercicio 11.3.1 — Leer: del trie al radix tree

**Tipo: Leer**

```
El problema del trie básico: demasiados nodos.
Cada carácter de cada string = un nodo.

  Trie para {"amazon", "amazing", "amaze"}:
    root → a → m → a → z → o → n (✓ "amazon")
                        |
                        i → n → g (✓ "amazing")
                        |
                        e (✓ "amaze")

    11 nodos para 3 strings.

  Radix tree (compressed trie): colapsar cadenas sin bifurcación.
    root → "ama" → "z" → "on" (✓ "amazon")
                    |
                    "ing" (✓ "amazing")
                    |
                    "e" (✓ "amaze")

    5 nodos para 3 strings. Menos de la mitad.

  Regla de compresión:
    Si un nodo tiene UN SOLO hijo y no es is_end,
    colapsar con el hijo. Combinar los caracteres.

  La clave de cada nodo ya no es un solo carácter
  sino un string (el "edge label" o "prefix chunk").

  Complejidad:
    Igual que el trie: O(m) para search, insert, prefix search.
    Pero el número de nodos es O(n) en vez de O(n × m_avg).
    → MUCHA menos memoria.

  Memoria:
    Trie:       O(n × m × ALPHABET) donde m = longitud promedio.
    Radix tree: O(n × ALPHABET) nodos, cada uno con un string.
    → Para strings largos (URLs, paths), la reducción es dramática.

  Uso real:
    - Linux kernel routing tables: LPC trie (Level-Path Compressed trie).
    - HTTP routers en Go: chi, httprouter, gin → radix tree de rutas.
    - IP routing: longest prefix match con radix tree.
```

**Preguntas:**

1. ¿De 11 nodos a 5 — ¿con strings más largos la compresión
   es aún mayor?

2. ¿El edge label es un substring. ¿Cómo se almacena eficientemente?

3. ¿Insert en un radix tree puede necesitar SPLIT de un edge.
   ¿Es como el split de un B-Tree?

4. ¿Patricia tree es otro nombre para radix tree?

5. ¿Para URLs como "/api/v1/users/123", ¿cuántos nodos tiene
   un radix tree vs un trie?

---

### Ejercicio 11.3.2 — Implementar: radix tree en Java

**Tipo: Implementar**

```java
import java.util.*;

public class RadixTree<V> {

    static class Node<V> {
        String prefix;          // edge label
        Map<Character, Node<V>> children = new HashMap<>();
        V value;
        boolean isEnd;

        Node(String prefix) { this.prefix = prefix; }
    }

    private final Node<V> root = new Node<>("");
    private int size;

    public void put(String key, V value) {
        if (key.isEmpty()) { root.isEnd = true; root.value = value; size++; return; }
        put(root, key, value);
    }

    private void put(Node<V> parent, String remaining, V value) {
        char first = remaining.charAt(0);
        Node<V> child = parent.children.get(first);

        if (child == null) {
            // No matching child: create new leaf
            var newNode = new Node<V>(remaining);
            newNode.isEnd = true;
            newNode.value = value;
            parent.children.put(first, newNode);
            size++;
            return;
        }

        // Find common prefix length
        String edgeLabel = child.prefix;
        int common = commonPrefixLength(remaining, edgeLabel);

        if (common == edgeLabel.length() && common == remaining.length()) {
            // Exact match
            if (!child.isEnd) size++;
            child.isEnd = true;
            child.value = value;
        } else if (common == edgeLabel.length()) {
            // Edge label is a prefix of remaining: continue down
            put(child, remaining.substring(common), value);
        } else {
            // Split required: edge label partially matches
            var splitNode = new Node<V>(edgeLabel.substring(0, common));
            parent.children.put(first, splitNode);

            child.prefix = edgeLabel.substring(common);
            splitNode.children.put(child.prefix.charAt(0), child);

            if (common == remaining.length()) {
                splitNode.isEnd = true;
                splitNode.value = value;
                size++;
            } else {
                var newLeaf = new Node<V>(remaining.substring(common));
                newLeaf.isEnd = true;
                newLeaf.value = value;
                splitNode.children.put(newLeaf.prefix.charAt(0), newLeaf);
                size++;
            }
        }
    }

    public V get(String key) {
        Node<V> node = findNode(key);
        return (node != null && node.isEnd) ? node.value : null;
    }

    public boolean containsPrefix(String prefix) {
        return findPrefixNode(prefix) != null;
    }

    private Node<V> findNode(String key) {
        Node<V> node = root;
        String remaining = key;
        while (!remaining.isEmpty()) {
            Node<V> child = node.children.get(remaining.charAt(0));
            if (child == null) return null;
            if (!remaining.startsWith(child.prefix)) return null;
            remaining = remaining.substring(child.prefix.length());
            node = child;
        }
        return node;
    }

    private Node<V> findPrefixNode(String prefix) {
        Node<V> node = root;
        String remaining = prefix;
        while (!remaining.isEmpty()) {
            Node<V> child = node.children.get(remaining.charAt(0));
            if (child == null) return null;
            if (child.prefix.length() >= remaining.length()) {
                return child.prefix.startsWith(remaining) ? child : null;
            }
            if (!remaining.startsWith(child.prefix)) return null;
            remaining = remaining.substring(child.prefix.length());
            node = child;
        }
        return node;
    }

    private int commonPrefixLength(String a, String b) {
        int len = Math.min(a.length(), b.length());
        for (int i = 0; i < len; i++) {
            if (a.charAt(i) != b.charAt(i)) return i;
        }
        return len;
    }

    public int size() { return size; }

    public int nodeCount() { return countNodes(root); }
    private int countNodes(Node<V> n) {
        int count = 1;
        for (Node<V> child : n.children.values())
            count += countNodes(child);
        return count;
    }

    public static void main(String[] args) {
        var rt = new RadixTree<Integer>();
        String[] words = {"amazon", "amazing", "amaze", "apple", "application",
                          "apply", "app", "azure", "azure devops"};
        for (int i = 0; i < words.length; i++) rt.put(words[i], i);

        System.out.printf("Radix tree: %d keys, %d nodes%n", rt.size(), rt.nodeCount());
        for (String w : words)
            System.out.printf("  get('%s') = %s%n", w, rt.get(w));
        System.out.printf("  get('xyz') = %s%n", rt.get("xyz"));
        System.out.printf("  containsPrefix('app') = %s%n", rt.containsPrefix("app"));
        System.out.printf("  containsPrefix('xyz') = %s%n", rt.containsPrefix("xyz"));

        // Compare node count: trie vs radix tree
        var trie = new Trie();
        for (String w : words) trie.insert(w, 1);
        System.out.printf("%nTrie nodes:       %d%n", trie.nodeCount());
        System.out.printf("Radix tree nodes: %d (%.0f%% reduction)%n",
            rt.nodeCount(),
            (1 - (double) rt.nodeCount() / trie.nodeCount()) * 100);
    }
}
// Resultado esperado:
// Radix tree: 9 keys, ~12-15 nodes
// Trie nodes: ~40-50
// Radix tree nodes: ~12-15 (60-70% reduction)
```

**Preguntas:**

1. ¿60-70% menos nodos que un trie. ¿Con URLs más largas?

2. ¿El split es la operación más compleja. ¿Cuándo ocurre?

3. ¿`substring()` crea nuevos strings en Java.
   ¿Es un costo significativo?

4. ¿Un radix tree en Rust podría evitar las copias con slices?

5. ¿Este radix tree es lo que usa un HTTP router?

---

## Sección 11.4 — Radix Tree como HTTP Router

### Ejercicio 11.4.1 — Implementar: HTTP router simplificado en Go

**Tipo: Implementar**

```go
package main

import "fmt"

type route struct {
    method  string
    handler string // simplified: just a name
}

type routeNode struct {
    prefix   string
    children map[byte]*routeNode
    route    *route    // non-nil if this node is a complete route
    paramName string   // for ":param" segments
    isParam   bool
}

type Router struct {
    root *routeNode
}

func NewRouter() *Router {
    return &Router{root: &routeNode{children: make(map[byte]*routeNode)}}
}

func (r *Router) Handle(method, path, handler string) {
    node := r.root
    remaining := path

    for len(remaining) > 0 {
        // Check for parameter segment
        if remaining[0] == ':' {
            end := len(remaining)
            for i := 1; i < len(remaining); i++ {
                if remaining[i] == '/' { end = i; break }
            }
            paramName := remaining[1:end]

            child, ok := node.children[':']
            if !ok {
                child = &routeNode{
                    prefix:    ":",
                    children:  make(map[byte]*routeNode),
                    paramName: paramName,
                    isParam:   true,
                }
                node.children[':'] = child
            }
            node = child
            remaining = remaining[end:]
            continue
        }

        // Find matching child
        ch := remaining[0]
        child, ok := node.children[ch]
        if !ok {
            // Create new node for remaining path
            newNode := &routeNode{prefix: remaining, children: make(map[byte]*routeNode)}
            node.children[ch] = newNode
            node = newNode
            remaining = ""
        } else {
            // Find common prefix
            common := 0
            for common < len(child.prefix) && common < len(remaining) &&
                child.prefix[common] == remaining[common] {
                common++
            }

            if common == len(child.prefix) {
                node = child
                remaining = remaining[common:]
            } else {
                // Split
                splitNode := &routeNode{
                    prefix:   child.prefix[:common],
                    children: make(map[byte]*routeNode),
                }
                node.children[ch] = splitNode
                child.prefix = child.prefix[common:]
                splitNode.children[child.prefix[0]] = child
                node = splitNode
                remaining = remaining[common:]
                if len(remaining) > 0 {
                    newNode := &routeNode{prefix: remaining, children: make(map[byte]*routeNode)}
                    splitNode.children[remaining[0]] = newNode
                    node = newNode
                    remaining = ""
                }
            }
        }
    }
    node.route = &route{method: method, handler: handler}
}

type matchResult struct {
    handler string
    params  map[string]string
}

func (r *Router) Match(method, path string) *matchResult {
    params := make(map[string]string)
    node := r.root
    remaining := path

    for len(remaining) > 0 {
        // Try parameter match
        if paramChild, ok := node.children[':']; ok {
            end := len(remaining)
            for i := 0; i < len(remaining); i++ {
                if remaining[i] == '/' { end = i; break }
            }
            params[paramChild.paramName] = remaining[:end]
            node = paramChild
            remaining = remaining[end:]
            continue
        }

        ch := remaining[0]
        child, ok := node.children[ch]
        if !ok { return nil }
        if len(remaining) < len(child.prefix) { return nil }
        if remaining[:len(child.prefix)] != child.prefix { return nil }
        node = child
        remaining = remaining[len(child.prefix):]
    }

    if node.route == nil || node.route.method != method { return nil }
    return &matchResult{handler: node.route.handler, params: params}
}

func main() {
    r := NewRouter()
    r.Handle("GET", "/api/users", "listUsers")
    r.Handle("GET", "/api/users/:id", "getUser")
    r.Handle("POST", "/api/users", "createUser")
    r.Handle("GET", "/api/users/:id/posts", "getUserPosts")
    r.Handle("GET", "/api/posts", "listPosts")
    r.Handle("GET", "/api/posts/:id", "getPost")
    r.Handle("GET", "/health", "healthCheck")

    tests := []struct{ method, path string }{
        {"GET", "/api/users"},
        {"GET", "/api/users/42"},
        {"GET", "/api/users/42/posts"},
        {"POST", "/api/users"},
        {"GET", "/api/posts"},
        {"GET", "/api/posts/99"},
        {"GET", "/health"},
        {"GET", "/unknown"},
    }

    for _, t := range tests {
        result := r.Match(t.method, t.path)
        if result != nil {
            fmt.Printf("%-4s %-25s → %s %v\n", t.method, t.path, result.handler, result.params)
        } else {
            fmt.Printf("%-4s %-25s → 404\n", t.method, t.path)
        }
    }
}
// Output:
// GET  /api/users                → listUsers map[]
// GET  /api/users/42             → getUser map[id:42]
// GET  /api/users/42/posts       → getUserPosts map[id:42]
// POST /api/users                → createUser map[]
// GET  /api/posts                → listPosts map[]
// GET  /api/posts/99             → getPost map[id:99]
// GET  /health                   → healthCheck map[]
// GET  /unknown                  → 404
```

**Preguntas:**

1. ¿El radix tree comprime `/api/` como prefijo compartido.
   ¿Cuántos nodos para 7 rutas?

2. ¿Parámetros como `:id` se resuelven en O(m). ¿Un regex router
   sería más lento?

3. ¿Go chi/httprouter usan exactamente este enfoque?

4. ¿Para 10,000 rutas en un API, ¿el router sigue siendo O(m)?

5. ¿Un hash map de rutas completas sería más simple.
   ¿Por qué no se usa?

---

## Sección 11.5 — Suffix Arrays: Buscar Cualquier Substring

### Ejercicio 11.5.1 — Leer: el problema de la búsqueda de substrings

**Tipo: Leer**

```
Un trie busca por PREFIJO. Pero a veces necesitas buscar
cualquier SUBSTRING.

  "¿El documento contiene la palabra 'structure'?"
  → No es un prefijo. Es un substring en alguna posición.

  Solución naive: buscar "structure" en cada posición del texto.
  Texto de n caracteres, patrón de m caracteres: O(n × m).
  Para un documento de 1M caracteres: 1M × 9 = 9M comparaciones.
  Para 1000 documentos: 9 billón de comparaciones. Inaceptable.

  SUFFIX ARRAY:
  Construir un array de TODOS los suffixes del texto,
  ordenados lexicográficamente. Luego binary search.

  Texto: "banana"
  Suffixes:
    0: "banana"
    1: "anana"
    2: "nana"
    3: "ana"
    4: "na"
    5: "a"

  Suffix array (sorted by suffix):
    [5, 3, 1, 0, 4, 2]
    → "a", "ana", "anana", "banana", "na", "nana"

  Buscar "ana":
    Binary search en el suffix array.
    Comparar "ana" con suffixes en las posiciones del array.
    → Encontrar posiciones 3 y 1 (suffixes "ana..." y "anana...").
    → "ana" aparece en posiciones 1 y 3 del texto original.

  Complejidad:
    Construir suffix array: O(n log n) con algoritmos eficientes.
    Buscar un patrón: O(m × log n) con binary search.
    Espacio: O(n) — un int por carácter.

  Comparación:
    Naive search:     O(n × m) por query. Sin preproceso.
    Suffix array:     O(m × log n) por query. Preproceso O(n log n).
    Inverted index:   O(1) per term. Preproceso O(n). Solo palabras completas.

  Uso real:
    Bioinformática: buscar secuencias en genomas (billones de bases).
    Full-text search: índices para búsqueda de substrings.
    Data compression: BWT (Burrows-Wheeler Transform) usa suffix arrays.
```

**Preguntas:**

1. ¿O(m × log n) para buscar un patrón. ¿Es mejor que O(n)?

2. ¿El suffix array para un texto de 1 GB ocupa cuánta memoria?

3. ¿El inverted index del Cap.12 es mejor para búsqueda
   de palabras completas. ¿Cuándo necesitas suffix array?

4. ¿Bioinformática busca secuencias de ADN. ¿Suffix array
   es la estructura estándar?

5. ¿LCP (Longest Common Prefix) array complementa al suffix array.
   ¿Para qué?

---

### Ejercicio 11.5.2 — Implementar: suffix array y búsqueda en Python

**Tipo: Implementar**

```python
class SuffixArray:
    def __init__(self, text: str):
        self.text = text
        self.n = len(text)
        # Build suffix array: sort indices by their suffixes
        self.sa = sorted(range(self.n), key=lambda i: text[i:])

    def search(self, pattern: str) -> list[int]:
        """Find all occurrences of pattern in text"""
        m = len(pattern)
        # Binary search for leftmost occurrence
        lo, hi = 0, self.n - 1
        left = self.n
        while lo <= hi:
            mid = (lo + hi) // 2
            suffix = self.text[self.sa[mid]:self.sa[mid] + m]
            if suffix < pattern:
                lo = mid + 1
            else:
                hi = mid - 1
                if suffix == pattern:
                    left = mid

        # Binary search for rightmost occurrence
        lo, hi = left, self.n - 1
        right = -1
        while lo <= hi:
            mid = (lo + hi) // 2
            suffix = self.text[self.sa[mid]:self.sa[mid] + m]
            if suffix > pattern:
                hi = mid - 1
            else:
                lo = mid + 1
                if suffix == pattern:
                    right = mid

        if left > right:
            return []
        return [self.sa[i] for i in range(left, right + 1)]

    def count(self, pattern: str) -> int:
        return len(self.search(pattern))


# ═══ Test ═══
text = "banana"
sa = SuffixArray(text)
print(f"Text: '{text}'")
print(f"Suffix array: {sa.sa}")
print(f"Sorted suffixes:")
for i, idx in enumerate(sa.sa):
    print(f"  sa[{i}] = {idx}: '{text[idx:]}'")

print(f"\nsearch('ana') = {sa.search('ana')}")   # [1, 3]
print(f"search('na') = {sa.search('na')}")       # [2, 4]
print(f"search('ban') = {sa.search('ban')}")     # [0]
print(f"search('xyz') = {sa.search('xyz')}")     # []

# Larger test
import random, string, time

text = ''.join(random.choices(string.ascii_lowercase, k=1_000_000))
start = time.perf_counter_ns()
sa = SuffixArray(text)
build_time = time.perf_counter_ns() - start

# Search for random patterns
patterns = [text[random.randint(0, len(text)-10):random.randint(0, len(text)-10)+5] for _ in range(1000)]
start = time.perf_counter_ns()
total_found = sum(sa.count(p) for p in patterns)
search_time = time.perf_counter_ns() - start

print(f"\nSuffix array for {len(text):,} chars:")
print(f"  Build: {build_time // 1_000_000} ms")
print(f"  Search 1000 patterns: {search_time // 1_000_000} ms ({search_time // 1000} ns/op)")
print(f"  Total matches: {total_found}")
```

**Preguntas:**

1. ¿`sorted(range(n), key=lambda i: text[i:])` es O(n² log n). ¿Hay algo mejor?

2. ¿Binary search con comparación de strings es O(m × log n). ¿Correcto?

3. ¿El suffix array para 1M caracteres cuesta ~4 MB (1M ints)?

4. ¿Para búsqueda full-text en un documento grande,
   ¿suffix array o inverted index?

5. ¿El suffix array se puede construir en O(n) con el algoritmo SA-IS?

---

## Sección 11.6 — Benchmark: Trie vs Hash Map vs Sorted Array

### Ejercicio 11.6.1 — Implementar: comparación de prefix search

**Tipo: Implementar**

```java
import java.util.*;

public class PrefixBenchmark {

    public static void main(String[] args) {
        int n = 500_000;
        var rng = new Random(42);
        var words = new String[n];
        for (int i = 0; i < n; i++) {
            int len = 5 + rng.nextInt(11);
            var sb = new StringBuilder(len);
            for (int j = 0; j < len; j++) sb.append((char)('a' + rng.nextInt(26)));
            words[i] = sb.toString();
        }

        // Build structures
        var trie = new Trie();
        var hashSet = new HashSet<String>(n * 2);
        var sorted = new String[n];
        for (int i = 0; i < n; i++) {
            trie.insert(words[i], 1);
            hashSet.add(words[i]);
            sorted[i] = words[i];
        }
        Arrays.sort(sorted);

        // Generate prefix queries
        int queries = 50_000;
        var prefixes = new String[queries];
        for (int i = 0; i < queries; i++) {
            prefixes[i] = words[rng.nextInt(n)].substring(0, 3);
        }

        System.out.println("═══ PREFIX SEARCH (first 10 matches) ═══\n");

        // Trie autocomplete
        long start = System.nanoTime();
        int totalTrie = 0;
        for (String p : prefixes) totalTrie += trie.autocomplete(p, 10).size();
        long trieTime = System.nanoTime() - start;

        // HashSet: must scan all keys (no prefix support)
        start = System.nanoTime();
        int totalHash = 0;
        for (String p : prefixes) {
            int count = 0;
            for (String s : hashSet) {
                if (s.startsWith(p) && ++count >= 10) break;
            }
            totalHash += count;
        }
        long hashTime = System.nanoTime() - start;

        // Sorted array: binary search for range
        start = System.nanoTime();
        int totalSorted = 0;
        for (String p : prefixes) {
            int idx = Arrays.binarySearch(sorted, p);
            if (idx < 0) idx = -(idx + 1);
            int count = 0;
            while (idx < sorted.length && sorted[idx].startsWith(p) && count < 10) {
                count++; idx++;
            }
            totalSorted += count;
        }
        long sortedTime = System.nanoTime() - start;

        System.out.printf("Trie:         %6d ms  (%5.0f ns/op)  results=%d%n",
            trieTime / 1_000_000, (double) trieTime / queries, totalTrie);
        System.out.printf("HashSet scan: %6d ms  (%5.0f ns/op)  results=%d%n",
            hashTime / 1_000_000, (double) hashTime / queries, totalHash);
        System.out.printf("Sorted array: %6d ms  (%5.0f ns/op)  results=%d%n",
            sortedTime / 1_000_000, (double) sortedTime / queries, totalSorted);
    }
}
// Resultado esperado:
// Trie:           ~200-400 ms  (~4000-8000 ns/op)
// HashSet scan:   ~15000-30000 ms (scanea TODOS los keys)
// Sorted array:   ~100-300 ms  (~2000-6000 ns/op)
//
// Trie y sorted array: 50-100× más rápidos que HashSet para prefix search.
// Sorted array ligeramente más rápido por cache locality.
// Pero trie soporta dynamic inserts; sorted array no.
```

**Preguntas:**

1. ¿HashSet es 50-100× más lento para prefix search. ¿Esperado?

2. ¿Sorted array es ligeramente más rápido que trie. ¿Cache locality?

3. ¿Para datos estáticos, ¿sorted array es suficiente?

4. ¿Para datos dinámicos (inserts frecuentes), ¿trie o TreeMap?

5. ¿Lucene/Elasticsearch usa sorted arrays o tries para terms?

---

## Sección 11.7 — Tries en Producción: de Routers a Elasticsearch

### Ejercicio 11.7.1 — Analizar: el ecosistema de tries

**Tipo: Analizar**

```
Dónde se usan tries y variantes en producción:

  HTTP ROUTERS (Go):
    chi, httprouter, gin: radix tree de rutas.
    Cada path como "/api/users/:id" → nodo del radix tree.
    Match: O(longitud del path). Sin regex.
    → Miles de rutas en microsegundos.

  LINUX KERNEL — LPC TRIE:
    Routing table: longest prefix match para IP addresses.
    Dada una IP destino, encontrar la ruta con el prefijo más largo.
    LPC trie: Level-Path Compressed trie.
    → Millones de rutas en nanosegundos.

  APACHE LUCENE / ELASTICSEARCH — FST:
    Finite State Transducer: evolución del trie.
    Almacena el diccionario de terms de un índice invertido.
    Cada term del documento → FST node.
    Soporta prefix search, fuzzy search, regex.
    Compresión: FST comparte tanto prefijos como SUFFIJOS.
    → Un FST de 1 billón de terms cabe en ~2 GB.

  DNS RESOLUTION:
    Nombres de dominio son jerárquicos: "www.example.com"
    → Trie/radix tree de segmentos de dominio.

  AUTOCOMPLETE / TYPEAHEAD:
    Google Search, IDE code completion, shell tab completion.
    Trie con ranking (peso por frecuencia/popularidad).
    → Sugerencias en <50 ms.

  IP ROUTING (BGP):
    Las routing tables de Internet usan longest prefix match.
    ~900K prefixes IPv4 actuales.
    Estructura: binary trie o multibit trie.
    → Lookup en hardware a velocidad de línea.
```

**Preguntas:**

1. ¿FST (Finite State Transducer) comparte suffijos también.
   ¿Cuánta memoria ahorra vs radix tree?

2. ¿Lucene FST para 1 billón de terms en 2 GB.
   ¿Un HashMap usaría cuánto?

3. ¿Linux LPC trie para routing — ¿por qué no un hash map?

4. ¿Los IDE usan tries para code completion. ¿IntelliJ, VSCode?

5. ¿Para 900K prefixes BGP, ¿un binary trie es suficiente?

---

### Ejercicio 11.7.2 — Resumen: las reglas de las estructuras para strings

**Tipo: Leer**

```
Reglas nuevas del Cap.11:

  Regla 46: Trie = O(m) búsqueda independiente de n.
    Buscar un prefijo de longitud m en un trie con n claves: O(m).
    HashMap busca por key exacta en O(1). No soporta prefijos.
    TreeMap busca en O(log n). Soporta range, pero no prefix natural.
    → Trie cuando necesitas prefix search. HashMap cuando necesitas exact match.

  Regla 47: Radix tree = trie comprimido para el mundo real.
    El trie básico usa demasiada memoria (26 punteros por nodo).
    El radix tree colapsa cadenas sin bifurcación: 60-70% menos nodos.
    Es lo que usan los HTTP routers (Go chi, gin, httprouter)
    y las routing tables de Linux.

  Regla 48: Suffix array para búsqueda de substrings.
    Un trie busca por PREFIJO. ¿Y si necesitas buscar por SUBSTRING?
    Suffix array: construir, luego binary search en O(m × log n).
    Para textos grandes, bioinformática, y full-text search.

  Regla 49: FST (Finite State Transducer) = trie del futuro.
    Lucene/Elasticsearch no usa un trie simple.
    Usa un FST que comparte prefijos Y suffijos.
    Compresión extrema: billones de terms en pocos GB.
    → Si trabajas con Elasticsearch, estás usando un FST.

  La Parte 4 continúa:
    Cap.12: Skip Lists — la alternativa probabilística a los árboles.
    → Redis ZSET, LevelDB/RocksDB MemTable, ConcurrentSkipListMap.
```

**Preguntas:**

1. ¿Las 49 reglas cubren 11 capítulos. ¿Es un ritmo razonable?

2. ¿La Regla 46 (O(m) independiente de n) es contra-intuitiva
   para programadores acostumbrados a HashMap?

3. ¿Un data engineer interactúa directamente con tries
   o solo a través de frameworks (routers, Elasticsearch)?

4. ¿El Cap.12 (Skip Lists) cierra la Parte 4?

5. ¿El Inverted Index del temario original se movió o se integró
   en los FSTs de este capítulo?
