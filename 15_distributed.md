# Guía de Ejercicios — Cap.15: Consistent Hashing y Estructuras para Sistemas Distribuidos

> Tienes 5 servidores y 1 millón de keys.
> Distribución simple: server = hash(key) % 5.
> Funciona. Hasta que agregas un 6to servidor.
> hash(key) % 6 ≠ hash(key) % 5 para la mayoría de keys.
> → TODAS las keys se redistribuyen. Cache invalidation masiva.
>
> Consistent hashing resuelve esto:
> al agregar un servidor, solo ~1/N de las keys se mueven.
>
> Pero distribuir datos es solo el inicio.
> ¿Cómo verificas que dos réplicas tienen los mismos datos?
> Merkle tree: comparar un solo hash (la raíz) para detectar
> diferencias en billones de registros.
>
> ¿Cómo manejas escrituras concurrentes en réplicas sin coordinación?
> CRDTs: estructuras que convergen automáticamente sin conflictos.
>
> Este capítulo cubre las tres estructuras fundamentales
> de los sistemas distribuidos:
> 1. Consistent hashing → distribución de datos (Kafka, Cassandra, DynamoDB).
> 2. Merkle trees → verificación de integridad (Cassandra repair, Git, blockchain).
> 3. CRDTs → convergencia sin conflictos (Redis CRDB, Riak, collaborative editing).

---

## El modelo mental: distribuir, verificar, converger

```
Tres problemas fundamentales de los sistemas distribuidos:

  1. DISTRIBUCIÓN: ¿En qué nodo vive cada key?
     Naive: hash(key) % N. Agregar/quitar nodo → redistribuir todo.
     Consistent hashing: ring de hashes. Agregar nodo → mover ~1/N keys.

  2. VERIFICACIÓN: ¿Dos réplicas tienen los mismos datos?
     Naive: comparar todos los datos byte a byte. O(n).
     Merkle tree: comparar la raíz. Si difieren, descender.
     Verificar 1 billón de registros: ~30 comparaciones de hashes.

  3. CONVERGENCIA: ¿Cómo resolver conflictos entre réplicas?
     Naive: "last writer wins" → pérdida de datos.
     CRDTs: operaciones que SIEMPRE convergen sin coordinación.
     Ejemplo: G-Counter. Nodo A incrementa su slot. Nodo B incrementa su slot.
     Merge: max por slot. Nunca hay conflicto.
```

---

## Tabla de contenidos

- [Sección 15.1 — Consistent hashing: el hash ring](#sección-151--consistent-hashing-el-hash-ring)
- [Sección 15.2 — Virtual nodes y balance](#sección-152--virtual-nodes-y-balance)
- [Sección 15.3 — Implementar consistent hashing](#sección-153--implementar-consistent-hashing)
- [Sección 15.4 — Merkle trees: verificación de integridad](#sección-154--merkle-trees-verificación-de-integridad)
- [Sección 15.5 — Implementar un Merkle tree](#sección-155--implementar-un-merkle-tree)
- [Sección 15.6 — CRDTs: convergencia sin conflictos](#sección-156--crdts-convergencia-sin-conflictos)
- [Sección 15.7 — Estructuras distribuidas en producción](#sección-157--estructuras-distribuidas-en-producción)

---

## Sección 15.1 — Consistent Hashing: el Hash Ring

### Ejercicio 15.1.1 — Leer: el problema de hash(key) % N

**Tipo: Leer**

```
Distribución naive: server = hash(key) % N

  N = 3 servidores. 6 keys:
    hash("user_1") % 3 = 0 → Server 0
    hash("user_2") % 3 = 1 → Server 1
    hash("user_3") % 3 = 2 → Server 2
    hash("user_4") % 3 = 0 → Server 0
    hash("user_5") % 3 = 1 → Server 1
    hash("user_6") % 3 = 2 → Server 2

  Agregar Server 3 (N = 4):
    hash("user_1") % 4 = 1 → Server 1  ← MOVIDO (era 0)
    hash("user_2") % 4 = 2 → Server 2  ← MOVIDO (era 1)
    hash("user_3") % 4 = 3 → Server 3  ← MOVIDO (era 2)
    hash("user_4") % 4 = 0 → Server 0  (se quedó)
    hash("user_5") % 4 = 1 → Server 1  (se quedó)
    hash("user_6") % 4 = 2 → Server 2  ← MOVIDO (era 2→2 ok, depende del hash)

  → ~75% de las keys se mueven al agregar 1 servidor.
  → En producción: cache miss masivo, tormenta de recargas.

  CONSISTENT HASHING (Karger et al., 1997):
    Imaginar un anillo (ring) de 0 a 2^32 - 1.
    Cada servidor se ubica en un punto del anillo: hash(server_id).
    Cada key se ubica en un punto: hash(key).
    La key va al PRIMER servidor en sentido horario.

    Ring:
         0
         │  Server A (hash=100)
         │
         │      key_1 (hash=150) → Server B
         │
         │  Server B (hash=200)
         │
         │      key_2 (hash=500) → Server C
         │
         │  Server C (hash=700)
         │
      2^32

    Agregar Server D (hash=300):
      Solo las keys entre Server B (200) y Server D (300)
      se mueven de Server C a Server D.
      → ~1/N de las keys se mueven. No ~75%.
```

**Preguntas:**

1. ¿~1/N de las keys se mueven vs ~75% con modulo. ¿Para N=100?

2. ¿El anillo de 0 a 2^32 — ¿por qué 2^32?

3. ¿"Primer servidor en sentido horario" — ¿qué pasa si un servidor cae?

4. ¿Kafka usa consistent hashing para particiones?

5. ¿DynamoDB fue diseñado sobre consistent hashing (Dynamo paper, 2007)?

---

## Sección 15.2 — Virtual Nodes y Balance

### Ejercicio 15.2.1 — Leer: por qué virtual nodes son necesarios

**Tipo: Leer**

```
Problema: con pocos servidores, la distribución es desigual.

  3 servidores en el ring:
    Server A: hash = 100
    Server B: hash = 200
    Server C: hash = 700

    Server A cubre el rango [700, 100] → ~400/2^32 del ring (~pequeño)
    Server B cubre el rango [100, 200] → ~100/2^32 del ring (muy pequeño)
    Server C cubre el rango [200, 700] → ~500/2^32 del ring (GRANDE)
    → Server C recibe ~50% del tráfico. Desbalance.

  VIRTUAL NODES: cada servidor tiene múltiples puntos en el ring.

    Server A: hash("A-0")=100, hash("A-1")=400, hash("A-2")=800
    Server B: hash("B-0")=200, hash("B-1")=500, hash("B-2")=900
    Server C: hash("C-0")=300, hash("C-1")=600, hash("C-2")=950

    9 puntos en el ring → distribución mucho más uniforme.
    Cada servidor cubre ~33% del ring (±variación estadística).

  Cuántos virtual nodes:
    Cassandra: 256 vnodes por nodo (default).
    DynamoDB: 150-300 vnodes.
    Regla: más vnodes → mejor balance, más metadata.
    → 100-300 es el rango típico.

  Agregar un servidor con vnodes:
    El nuevo servidor coloca 256 puntos en el ring.
    Cada punto "toma" una pequeña fracción de un vecino.
    → La carga se redistribuye uniformemente entre TODOS los nodos existentes.
    → No solo entre los 2 vecinos inmediatos.
```

**Preguntas:**

1. ¿256 vnodes — ¿cuánto metadata extra es?

2. ¿Sin vnodes, ¿el desbalance puede ser 10:1 con 3 servidores?

3. ¿Cassandra cambió a token ranges. ¿Es diferente de vnodes?

4. ¿Los vnodes complican el rebalanceo. ¿Por qué Cassandra
   redujo vnodes en versiones recientes?

5. ¿Para un cluster de 1000 nodos, ¿vnodes siguen siendo necesarios?

---

## Sección 15.3 — Implementar Consistent Hashing

### Ejercicio 15.3.1 — Implementar: consistent hashing en Python

**Tipo: Implementar**

```python
import hashlib
import bisect

class ConsistentHash:
    def __init__(self, vnodes=150):
        self.vnodes = vnodes
        self.ring = {}       # hash_value → node_id
        self.sorted_keys = [] # sorted hash values
        self.nodes = set()

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % (2**32)

    def add_node(self, node_id: str):
        self.nodes.add(node_id)
        for i in range(self.vnodes):
            h = self._hash(f"{node_id}:{i}")
            self.ring[h] = node_id
            bisect.insort(self.sorted_keys, h)

    def remove_node(self, node_id: str):
        self.nodes.discard(node_id)
        self.sorted_keys = [k for k in self.sorted_keys if self.ring.get(k) != node_id]
        self.ring = {k: v for k, v in self.ring.items() if v != node_id}

    def get_node(self, key: str) -> str:
        if not self.sorted_keys:
            return None
        h = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, h)
        if idx == len(self.sorted_keys):
            idx = 0  # wrap around the ring
        return self.ring[self.sorted_keys[idx]]


# ═══ Test ═══
ch = ConsistentHash(vnodes=150)
for node in ["server_A", "server_B", "server_C"]:
    ch.add_node(node)

# Distribute 100K keys
from collections import Counter
dist = Counter()
for i in range(100_000):
    node = ch.get_node(f"key_{i}")
    dist[node] += 1

print("Distribution (3 nodes, 150 vnodes each):")
for node, count in sorted(dist.items()):
    print(f"  {node}: {count:,} keys ({count/1000:.1f}%)")

# Add a node and measure redistribution
old_mapping = {f"key_{i}": ch.get_node(f"key_{i}") for i in range(100_000)}
ch.add_node("server_D")
moved = sum(1 for i in range(100_000) if ch.get_node(f"key_{i}") != old_mapping[f"key_{i}"])
print(f"\nAfter adding server_D: {moved:,} keys moved ({moved/1000:.1f}%)")
print(f"Expected: ~{100_000//4:,} (~25%)")

dist2 = Counter()
for i in range(100_000):
    dist2[ch.get_node(f"key_{i}")] += 1
print("\nNew distribution (4 nodes):")
for node, count in sorted(dist2.items()):
    print(f"  {node}: {count:,} keys ({count/1000:.1f}%)")
```

**Preguntas:**

1. ¿~25% de keys se mueven al agregar 1 de 4 servidores. ¿Esperado?

2. ¿MD5 como hash function — ¿es bueno para consistent hashing?

3. ¿`bisect_right` para encontrar el primer servidor clockwise.
   ¿Es O(log n)?

4. ¿150 vnodes por servidor — ¿cómo varía la uniformidad
   con 10 vs 500 vnodes?

5. ¿Para un sistema de caché (Memcached), ¿consistent hashing
   evita cache storms?

---

### Ejercicio 15.3.2 — Implementar: consistent hashing en Go

**Tipo: Implementar**

```go
package main

import (
    "crypto/md5"
    "encoding/binary"
    "fmt"
    "sort"
    "strconv"
)

type ConsistentHash struct {
    ring     map[uint32]string
    keys     []uint32 // sorted
    vnodes   int
}

func NewConsistentHash(vnodes int) *ConsistentHash {
    return &ConsistentHash{ring: make(map[uint32]string), vnodes: vnodes}
}

func (ch *ConsistentHash) hash(key string) uint32 {
    h := md5.Sum([]byte(key))
    return binary.BigEndian.Uint32(h[:4])
}

func (ch *ConsistentHash) AddNode(node string) {
    for i := 0; i < ch.vnodes; i++ {
        h := ch.hash(node + ":" + strconv.Itoa(i))
        ch.ring[h] = node
        ch.keys = append(ch.keys, h)
    }
    sort.Slice(ch.keys, func(i, j int) bool { return ch.keys[i] < ch.keys[j] })
}

func (ch *ConsistentHash) GetNode(key string) string {
    if len(ch.keys) == 0 { return "" }
    h := ch.hash(key)
    idx := sort.Search(len(ch.keys), func(i int) bool { return ch.keys[i] >= h })
    if idx == len(ch.keys) { idx = 0 }
    return ch.ring[ch.keys[idx]]
}

func main() {
    ch := NewConsistentHash(200)
    nodes := []string{"node-1", "node-2", "node-3", "node-4", "node-5"}
    for _, n := range nodes {
        ch.AddNode(n)
    }

    dist := make(map[string]int)
    n := 1_000_000
    for i := 0; i < n; i++ {
        node := ch.GetNode("key-" + strconv.Itoa(i))
        dist[node]++
    }

    fmt.Printf("Distribution (%d nodes, %d vnodes, %d keys):\n", len(nodes), ch.vnodes, n)
    for _, node := range nodes {
        count := dist[node]
        fmt.Printf("  %s: %d keys (%.1f%%)\n", node, count, float64(count)/float64(n)*100)
    }
}
// Resultado esperado (5 nodes, 200 vnodes):
// Cada nodo: ~200,000 keys (~20%) ± 1-3%
```

**Preguntas:**

1. ¿Con 200 vnodes y 5 nodos, ¿la desviación es <3%?

2. ¿`sort.Search` para binary search en el ring. ¿Es O(log(N×vnodes))?

3. ¿Para Kafka partitions, ¿consistent hashing es diferente?

4. ¿Amazon DynamoDB usa consistent hashing con vnodes. ¿Cuántos?

5. ¿Consistent hashing con weighted nodes — ¿más vnodes para servidores más potentes?

---

## Sección 15.4 — Merkle Trees: Verificación de Integridad

### Ejercicio 15.4.1 — Leer: el poder del hash tree

**Tipo: Leer**

```
Problema: verificar que dos réplicas de 1 TB tienen los mismos datos.

  Naive: transferir los 1 TB completos y comparar. O(n) datos transferidos.
  Merkle tree: comparar UN hash (la raíz). Si difiere, descender.

  Estructura:
    Cada hoja es el hash de un bloque de datos.
    Cada nodo interno es el hash de la concatenación de sus hijos.
    La raíz es el hash de TODO el dataset.

    Datos: [block_0, block_1, block_2, block_3]

                  hash(H01 + H23) ← raíz
                 /                \
           hash(H0 + H1)    hash(H2 + H3)
           /          \      /          \
       hash(B0)  hash(B1)  hash(B2)  hash(B3)

  Verificar integridad:
    Réplica A raíz = 0x1234. Réplica B raíz = 0x5678.
    → Difieren. ¿Dónde?
    Comparar hijo izquierdo: A = 0xABCD, B = 0xABCD. ✓ Iguales.
    Comparar hijo derecho: A = 0xEF01, B = 0x2345. ✗ Difieren.
    Descender al hijo derecho...
    → Encontrar el bloque diferente en O(log n) comparaciones.
    → Transferir SOLO el bloque diferente.

  Para 1 TB en bloques de 4 KB:
    ~250 millones de hojas.
    Altura del árbol: ~28 niveles.
    Encontrar una diferencia: 28 comparaciones de hashes (32 bytes cada uno).
    → 28 × 32 = 896 bytes transferidos para encontrar el bloque corrupto.
    → vs 1 TB para comparación completa.

  Uso real:
    Cassandra: anti-entropy repair. Compara Merkle trees entre réplicas.
    Git: cada commit es un Merkle tree de archivos.
    IPFS: content-addressable storage con Merkle DAGs.
    Blockchain: cada bloque contiene un Merkle root de transacciones.
    Amazon S3: verificación de integridad de objetos.
```

**Preguntas:**

1. ¿28 comparaciones para 1 TB — ¿es realmente así de eficiente?

2. ¿Git almacena cada archivo como un blob con su hash SHA-1.
   ¿Eso es un Merkle tree?

3. ¿Cassandra anti-entropy repair — ¿construye Merkle trees
   de todas las réplicas periódicamente?

4. ¿Si un bloque cambia, ¿todo el path hasta la raíz cambia?

5. ¿El hash function (SHA-256, MD5) importa para el Merkle tree?

---

### Ejercicio 15.5.1 — Implementar: Merkle tree en Java

**Tipo: Implementar**

```java
import java.security.MessageDigest;
import java.util.*;

public class MerkleTree {

    static class Node {
        byte[] hash;
        Node left, right;
        Node(byte[] hash) { this.hash = hash; }
        Node(byte[] hash, Node left, Node right) {
            this.hash = hash; this.left = left; this.right = right;
        }
    }

    private final Node root;
    private final List<byte[]> leaves;

    public MerkleTree(List<byte[]> dataBlocks) {
        this.leaves = new ArrayList<>();
        List<Node> nodes = new ArrayList<>();
        for (byte[] block : dataBlocks) {
            byte[] h = sha256(block);
            leaves.add(h);
            nodes.add(new Node(h));
        }
        // Pad to power of 2
        while (Integer.bitCount(nodes.size()) != 1) {
            nodes.add(new Node(new byte[32]));
        }
        this.root = buildTree(nodes);
    }

    private Node buildTree(List<Node> nodes) {
        if (nodes.size() == 1) return nodes.get(0);
        List<Node> parents = new ArrayList<>();
        for (int i = 0; i < nodes.size(); i += 2) {
            byte[] combined = sha256(concat(nodes.get(i).hash, nodes.get(i+1).hash));
            parents.add(new Node(combined, nodes.get(i), nodes.get(i+1)));
        }
        return buildTree(parents);
    }

    public byte[] rootHash() { return root.hash; }

    // Find differing blocks between two Merkle trees
    public static List<Integer> findDifferences(MerkleTree a, MerkleTree b) {
        List<Integer> diffs = new ArrayList<>();
        findDiffs(a.root, b.root, 0, a.leaves.size(), diffs);
        return diffs;
    }

    private static void findDiffs(Node a, Node b, int start, int count, List<Integer> diffs) {
        if (Arrays.equals(a.hash, b.hash)) return; // subtrees match
        if (count == 1) { diffs.add(start); return; } // leaf difference
        int half = count / 2;
        if (a.left != null && b.left != null)
            findDiffs(a.left, b.left, start, half, diffs);
        if (a.right != null && b.right != null)
            findDiffs(a.right, b.right, start + half, half, diffs);
    }

    private static byte[] sha256(byte[] data) {
        try {
            return MessageDigest.getInstance("SHA-256").digest(data);
        } catch (Exception e) { throw new RuntimeException(e); }
    }

    private static byte[] concat(byte[] a, byte[] b) {
        byte[] result = new byte[a.length + b.length];
        System.arraycopy(a, 0, result, 0, a.length);
        System.arraycopy(b, 0, result, a.length, b.length);
        return result;
    }

    private static String hex(byte[] h) {
        var sb = new StringBuilder();
        for (int i = 0; i < Math.min(4, h.length); i++) sb.append(String.format("%02x", h[i]));
        return sb.toString() + "...";
    }

    public static void main(String[] args) {
        int blocks = 1024;
        var rng = new Random(42);

        // Create two datasets (identical)
        List<byte[]> data1 = new ArrayList<>(), data2 = new ArrayList<>();
        for (int i = 0; i < blocks; i++) {
            byte[] block = new byte[4096];
            rng.nextBytes(block);
            data1.add(block);
            data2.add(block.clone());
        }

        var tree1 = new MerkleTree(data1);
        var tree2 = new MerkleTree(data2);
        System.out.printf("Root hash match: %s%n", Arrays.equals(tree1.rootHash(), tree2.rootHash()));

        // Corrupt 3 blocks in tree2
        data2.get(100)[0] ^= 0xFF;
        data2.get(500)[0] ^= 0xFF;
        data2.get(900)[0] ^= 0xFF;
        tree2 = new MerkleTree(data2);

        System.out.printf("Root hash after corruption: %s%n",
            Arrays.equals(tree1.rootHash(), tree2.rootHash()));

        var diffs = MerkleTree.findDifferences(tree1, tree2);
        System.out.printf("Differences found: %s%n", diffs);
        System.out.printf("Comparisons needed: O(log %d) = ~%d%n",
            blocks, (int)(Math.log(blocks) / Math.log(2)) * diffs.size());
    }
}
// Resultado esperado:
// Root hash match: true
// Root hash after corruption: false
// Differences found: [100, 500, 900]
// Comparisons needed: O(log 1024) = ~30
```

**Preguntas:**

1. ¿3 bloques corruptos en 1024 se encuentran con ~30 comparaciones?

2. ¿SHA-256 por bloque — ¿es costoso para blocks de 4 KB?

3. ¿Cassandra construye el Merkle tree de toda la partition.
   ¿Es online o offline?

4. ¿Git usa SHA-1 (ahora SHA-256). ¿El Merkle tree es un DAG?

5. ¿Para verificar un download de 10 GB, ¿cuántos hashes necesitas?

---

## Sección 15.6 — CRDTs: Convergencia sin Conflictos

### Ejercicio 15.6.1 — Leer: qué es un CRDT

**Tipo: Leer**

```
CRDT = Conflict-free Replicated Data Type.
Una estructura que puede replicarse en múltiples nodos
y converger automáticamente sin coordinación.

  Problema:
    Nodo A: counter = 5 → increment → counter = 6.
    Nodo B: counter = 5 → increment → counter = 6.
    ¿Merge? counter = 6 (last writer wins). ¡Perdimos un increment!

  G-Counter (Grow-only Counter):
    Cada nodo mantiene SU PROPIO contador.
    Nodo A: {A: 3, B: 0, C: 0}
    Nodo B: {A: 0, B: 5, C: 0}
    Nodo C: {A: 0, B: 0, C: 2}

    Merge: max por slot.
    merge({A:3,B:0,C:0}, {A:0,B:5,C:0}) = {A:3,B:5,C:0}
    Valor total: sum(todos los slots) = 3+5+0 = 8.

    Increment en Nodo A: A incrementa solo su slot.
    {A:4, B:0, C:0}.
    Merge con cualquier otro nodo: siempre converge.
    NUNCA hay conflicto.

  PN-Counter (Positive-Negative Counter):
    Dos G-Counters: uno para incrementos (P), otro para decrementos (N).
    Valor = sum(P) - sum(N).
    Increment: P[self]++.
    Decrement: N[self]++.
    Merge: merge P y merge N por separado.

  OR-Set (Observed-Remove Set):
    Cada add(element) genera un unique tag.
    remove(element) elimina todos los tags OBSERVADOS.
    Si Nodo A agrega "x" y Nodo B remueve "x" concurrentemente:
    El add de A tiene un tag que B no observó → "x" sobrevive.
    → "Add wins" semántica.
```

**Preguntas:**

1. ¿G-Counter solo crece. ¿Es útil en la práctica?

2. ¿PN-Counter resuelve decrements. ¿Puede llegar a negativo?

3. ¿OR-Set es complejo. ¿Hay CRDTs para maps?

4. ¿Redis CRDB usa CRDTs. ¿Cuáles específicamente?

5. ¿Google Docs usa CRDTs para edición colaborativa?

---

### Ejercicio 15.6.2 — Implementar: G-Counter y PN-Counter en Scala

**Tipo: Implementar**

```scala
case class GCounter(counts: Map[String, Long] = Map.empty) {
  def increment(nodeId: String): GCounter =
    copy(counts = counts + (nodeId -> (counts.getOrElse(nodeId, 0L) + 1)))

  def value: Long = counts.values.sum

  def merge(other: GCounter): GCounter = {
    val allKeys = counts.keySet ++ other.counts.keySet
    val merged = allKeys.map { k =>
      k -> math.max(counts.getOrElse(k, 0L), other.counts.getOrElse(k, 0L))
    }.toMap
    GCounter(merged)
  }
}

case class PNCounter(
  positive: GCounter = GCounter(),
  negative: GCounter = GCounter()
) {
  def increment(nodeId: String): PNCounter =
    copy(positive = positive.increment(nodeId))

  def decrement(nodeId: String): PNCounter =
    copy(negative = negative.increment(nodeId))

  def value: Long = positive.value - negative.value

  def merge(other: PNCounter): PNCounter =
    PNCounter(positive.merge(other.positive), negative.merge(other.negative))
}

object CRDTDemo extends App {
  // Simulate 3 nodes
  var nodeA = GCounter()
  var nodeB = GCounter()
  var nodeC = GCounter()

  // Each node increments locally
  nodeA = nodeA.increment("A").increment("A").increment("A") // A=3
  nodeB = nodeB.increment("B").increment("B")                 // B=2
  nodeC = nodeC.increment("C")                                 // C=1

  // Merge all
  val merged = nodeA.merge(nodeB).merge(nodeC)
  println(s"G-Counter: A=${nodeA.value}, B=${nodeB.value}, C=${nodeC.value}")
  println(s"Merged value: ${merged.value}") // 3+2+1 = 6
  println(s"Merged counts: ${merged.counts}") // {A:3, B:2, C:1}

  // PN-Counter
  var pnA = PNCounter()
  var pnB = PNCounter()

  pnA = pnA.increment("A").increment("A").increment("A") // +3
  pnB = pnB.increment("B").decrement("B")                 // +1 - 1 = 0

  val pnMerged = pnA.merge(pnB)
  println(s"\nPN-Counter: A=${pnA.value}, B=${pnB.value}")
  println(s"Merged: ${pnMerged.value}") // 3 + 0 = 3

  // Concurrent increments — never lost!
  var c1 = GCounter().increment("X").increment("X") // X=2
  var c2 = GCounter().increment("X")                  // X=1 (different timeline)
  // Both increment "X" but from different starting points
  val wrong = c1.merge(c2) // max(2, 1) = 2 — NOT 3!
  println(s"\nConcurrent: c1=${c1.value}, c2=${c2.value}, merged=${wrong.value}")
  println("Note: concurrent updates to SAME node requires coordination")
}
// Output:
// G-Counter: A=3, B=2, C=1
// Merged value: 6
// PN-Counter: A=3, B=0
// Merged: 3
```

**Preguntas:**

1. ¿Merge = max por slot. ¿Nunca se pierde un increment?

2. ¿Dos nodos incrementando el MISMO slot concurrentemente
   — ¿es un problema?

3. ¿Scala immutable case classes son naturales para CRDTs?

4. ¿El PN-Counter usa DOS G-Counters. ¿Es memoria 2×?

5. ¿Para Cassandra counters, ¿usan algo similar?

---

## Sección 15.7 — Estructuras Distribuidas en Producción

### Ejercicio 15.7.1 — Resumen: las reglas de sistemas distribuidos

**Tipo: Leer**

```
Reglas nuevas del Cap.15:

  Regla 63: Consistent hashing mueve ~1/N keys al agregar un nodo.
    hash(key) % N mueve ~75%+ al cambiar N.
    Consistent hashing con virtual nodes: ~1/N se mueven.
    Kafka, Cassandra, DynamoDB, Memcached: todos lo usan.
    150-256 vnodes por nodo para distribución uniforme.

  Regla 64: Merkle tree verifica integridad en O(log n) comparaciones.
    Comparar la raíz de dos réplicas: 1 hash (32 bytes).
    Si difieren: descender. Encontrar bloques corruptos en O(log n).
    Para 1 TB en bloques de 4 KB: ~28 comparaciones.
    Cassandra anti-entropy repair, Git, IPFS, blockchain.

  Regla 65: CRDTs convergen sin coordinación.
    G-Counter: incrementos por nodo, merge = max por slot.
    PN-Counter: incrementos + decrementos sin conflictos.
    OR-Set: add/remove con semántica "add wins".
    Redis CRDB, Riak, collaborative editing.
    Costo: más metadata (un slot por nodo, tags por operación).

  Regla 66: Distribuir, verificar, converger = las 3 operaciones de datos distribuidos.
    Consistent hashing → distribuir datos entre nodos.
    Merkle trees → verificar que las réplicas son consistentes.
    CRDTs → resolver conflictos cuando las réplicas divergen.
    Cassandra usa las tres.

  La Parte 5 (Concurrencia y Distribuidos) está completa:
    Cap.13: Queues → coordinación entre threads y procesos.
    Cap.14: Thread-safe → patrones de concurrencia.
    Cap.15: Distribuidos → consistent hashing, Merkle trees, CRDTs.

  Parte 6: Estructuras para Datos a Escala.
    Cap.16: Columnar layouts → Parquet, Arrow, OLAP.
    Cap.17: Streaming → Kafka log, Flink state, event sourcing.
    Cap.18: Proyecto final → key-value store completo.
```

**Preguntas:**

1. ¿Cassandra usa consistent hashing + Merkle trees + CRDTs.
   ¿Todo junto?

2. ¿Las 66 reglas cubren 15 capítulos. ¿Son demasiadas para memorizar?

3. ¿Un data engineer interactúa directamente con estas estructuras
   o son internas de Cassandra/Kafka?

4. ¿La Parte 6 cierra con el proyecto final. ¿Qué integra?

5. ¿El Cap.18 key-value store usará consistent hashing?
