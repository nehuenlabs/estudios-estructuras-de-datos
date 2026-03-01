# Guía de Ejercicios — Cap.09: B-Trees: la Estructura que Domina el Almacenamiento

> Cada vez que ejecutas un SELECT con WHERE en PostgreSQL,
> el query planner decide si hacer un sequential scan o un index scan.
> Si elige el index, atraviesa un B+ Tree.
>
> Cada vez que MySQL/InnoDB inserta una fila,
> actualiza al menos un B+ Tree (el clustered index).
>
> Cada vez que SQLite abre un archivo .db,
> la primera página es la raíz de un B-Tree.
>
> Los filesystems NTFS, ext4, HFS+ almacenan directorios en B-Trees.
> etcd (usado por Kubernetes) almacena estado en BoltDB,
> que es un copy-on-write B+ Tree.
>
> El B-Tree es, posiblemente, la estructura de datos más importante
> de la informática aplicada. No la más elegante, ni la más simple.
> Pero la que más datos del mundo real almacena y recupera.
>
> ¿Por qué no un Red-Black tree?
> Porque un Red-Black tree con 1 billón de claves tiene
> altura ~30. Cada nivel es un acceso a disco. 30 × 10 ms = 300 ms.
> Un B-Tree con branching factor 1000 tiene altura 3.
> 3 × 10 ms = 30 ms. 10× más rápido.
>
> Menos niveles = menos accesos a disco = más rápido. Fin.

---

## El modelo mental: del árbol binario al árbol ancho

```
El problema fundamental:
  Los discos (HDD y SSD) son LENTOS comparados con RAM.
    RAM: ~100 ns por acceso.
    SSD: ~100,000 ns (100 μs) por acceso.
    HDD: ~10,000,000 ns (10 ms) por acceso.

  Si cada nodo del árbol requiere un acceso a disco,
  la ALTURA del árbol determina la velocidad.

  Red-Black tree (binario, 2 hijos):
    n = 1 billón → altura ≈ 30
    30 accesos a disco × 10 ms (HDD) = 300 ms por query.
    30 accesos a disco × 100 μs (SSD) = 3 ms por query.

  B-Tree (ancho, B hijos):
    B = 1000 hijos por nodo.
    n = 1 billón → altura = log₁₀₀₀(10⁹) = 3.
    3 accesos a disco × 10 ms (HDD) = 30 ms por query.
    3 accesos a disco × 100 μs (SSD) = 300 μs por query.

  10× menos accesos a disco. 10× más rápido.

  ¿Por qué B=1000?
    Un "nodo" del B-Tree es una PÁGINA de disco (típicamente 4 KB o 16 KB).
    Cada página almacena muchas claves y punteros a hijos.
    Si cada clave ocupa 8 bytes y cada puntero 8 bytes:
      4 KB / 16 bytes ≈ 250 claves por nodo.
      16 KB / 16 bytes ≈ 1000 claves por nodo.
    → El branching factor B es determinado por el PAGE SIZE.

  B-Tree:
    Raíz → 1000 hijos → cada hijo tiene 1000 hijos → cada nieto tiene 1000 hijos.
    Nivel 0 (raíz): 1 nodo, 1000 claves.
    Nivel 1: 1,000 nodos, 1,000,000 claves.
    Nivel 2: 1,000,000 nodos, 1,000,000,000 claves.
    → 1 billón de claves en 3 niveles.
    → La raíz y posiblemente el nivel 1 caben en RAM (buffer pool).
    → En la práctica: 1-2 accesos a disco por query.
```

---

## Tabla de contenidos

- [Sección 9.1 — De árboles binarios a B-Trees: por qué anchos](#sección-91--de-árboles-binarios-a-b-trees-por-qué-anchos)
- [Sección 9.2 — B-Tree: propiedades, búsqueda, inserción](#sección-92--b-tree-propiedades-búsqueda-inserción)
- [Sección 9.3 — Implementar un B-Tree simplificado](#sección-93--implementar-un-b-tree-simplificado)
- [Sección 9.4 — B+ Tree: la variante que domina las bases de datos](#sección-94--b-tree-la-variante-que-domina-las-bases-de-datos)
- [Sección 9.5 — Implementar un B+ Tree con range scan](#sección-95--implementar-un-b-tree-con-range-scan)
- [Sección 9.6 — B-Trees en la práctica: pages, WAL, concurrencia](#sección-96--b-trees-en-la-práctica-pages-wal-concurrencia)
- [Sección 9.7 — B-Trees en producción: de PostgreSQL a BoltDB](#sección-97--b-trees-en-producción-de-postgresql-a-boltdb)

---

## Sección 9.1 — De Árboles Binarios a B-Trees: Por Qué Anchos

### Ejercicio 9.1.1 — Leer: el costo de un acceso a disco

**Tipo: Leer**

```
Para entender el B-Tree, necesitas entender el DISCO.

  ¿Cuánto cuesta leer datos?

  Ubicación       Latencia         Relativo a RAM
  ─────────       ────────         ──────────────
  L1 cache        ~1 ns            1×
  L2 cache        ~5 ns            5×
  L3 cache        ~20 ns           20×
  RAM             ~100 ns          100×
  SSD (random)    ~100,000 ns      100,000×
  HDD (random)    ~10,000,000 ns   10,000,000×

  Un acceso a disco es 1,000× a 100,000× más lento que RAM.

  Pero la UNIDAD de lectura del disco no es un byte.
  Es una PÁGINA (page):
  - HDD: sector mínimo 512 bytes, pero el OS lee 4 KB típicamente.
  - SSD: page interna de 4 KB o 16 KB.
  - OS page cache: 4 KB (Linux default page size).

  Leer 1 byte del disco cuesta LO MISMO que leer 4 KB.
  → Si vas a hacer un acceso a disco, llena la página.
  → Un nodo de B-Tree = 1 página = muchas claves.

  Comparación de árboles:
    Red-Black (binario): cada nodo tiene 1 clave + 2 punteros.
      Si un nodo = 1 página de 4 KB: desperdicias 4000+ bytes.
      Altura para 1 billón: ~30 páginas = 30 I/Os.

    B-Tree (B=500): cada nodo tiene ~500 claves + 501 punteros.
      Cada nodo llena una página de 4 KB eficientemente.
      Altura para 1 billón: ~3 páginas = 3 I/Os.

  → B-Tree AMORTIZA el costo de acceso a disco
    empacando muchas claves en cada lectura.
```

**Preguntas:**

1. ¿Leer 1 byte vs 4 KB del SSD cuesta lo mismo?

2. ¿NVMe SSDs tienen menor latencia que SATA SSDs.
   ¿Eso reduce la ventaja del B-Tree?

3. ¿El buffer pool de una base de datos cachea las páginas más usadas.
   ¿Cuántas páginas de B-Tree típicamente están en RAM?

4. ¿Con datos completamente en RAM (ej: Redis),
   ¿el B-Tree sigue teniendo ventaja sobre Red-Black?

5. ¿Un filesystem lee 4 KB por defecto. ¿Se puede aumentar
   el page size para un B-Tree más ancho?

---

### Ejercicio 9.1.2 — Analizar: altura del B-Tree vs branching factor

**Tipo: Analizar**

```
La altura de un B-Tree con n claves y branching factor B:

  h = ⌈log_B(n)⌉  (aproximadamente)

  Tabla de alturas para n = 1 billón (10⁹):

  B (branching)  Altura   I/Os por query  Nodos nivel 1
  ─────────────  ──────   ──────────────  ─────────────
  2              30       30              2
  10             9        9               10
  100            5        5               100
  500            4        4               500
  1,000          3        3               1,000
  5,000          3        3               5,000
  10,000         3        3               10,000

  Observaciones:
  1. De B=2 a B=100: la altura se reduce dramáticamente (30→5).
  2. De B=100 a B=10,000: la mejora es marginal (5→3).
  3. A partir de B=500, la altura se estabiliza en 3-4 para 10⁹.
  4. La raíz siempre está en RAM. Nivel 1 usualmente también.
     → En la práctica, solo 1-2 I/Os por query.

  ¿Qué B usan las bases de datos?

  Base de datos  Page size   B aprox.     Claves por hoja
  ────────────   ─────────   ────────     ───────────────
  PostgreSQL     8 KB        ~200-500     ~100-200
  MySQL/InnoDB   16 KB       ~500-1000    ~200-500
  SQLite         4 KB        ~100-250     ~50-100
  SQL Server     8 KB        ~200-500     ~100-200
  BoltDB         4 KB        ~100-250     ~50-100

  El page size es configurable en algunas bases de datos.
  InnoDB eligió 16 KB como balance entre fanout y write amplification.
  SQLite eligió 4 KB como balance entre fanout y tamaño del archivo.
```

**Preguntas:**

1. ¿PostgreSQL con page size 8 KB tiene B≈200-500.
   ¿Podría usar 64 KB pages para B más grande?

2. ¿InnoDB usa 16 KB pages. ¿Cuántas filas caben en una hoja
   si la fila tiene 200 bytes?

3. ¿La altura de 3 significa 3 lecturas a disco.
   ¿Si la raíz y nivel 1 están en RAM, cuántas lecturas reales?

4. ¿B más grande siempre es mejor? ¿Hay desventajas?

5. ¿Un B-Tree en SSD con page size de 4 KB vs 64 KB
   — ¿cuál es más rápido?

---

## Sección 9.2 — B-Tree: Propiedades, Búsqueda, Inserción

### Ejercicio 9.2.1 — Leer: propiedades formales del B-Tree

**Tipo: Leer**

```
Un B-Tree de orden M (máximo M hijos por nodo):

  1. Cada nodo tiene como máximo M hijos.
  2. Cada nodo interno (excepto la raíz) tiene al menos ⌈M/2⌉ hijos.
  3. La raíz tiene al menos 2 hijos (si no es hoja).
  4. Un nodo con k hijos tiene k-1 claves.
  5. Las claves en cada nodo están ORDENADAS.
  6. Todas las hojas están al MISMO nivel.

  Ejemplo: B-Tree de orden M=4 (2-3-4 tree)
    Cada nodo tiene entre 2 y 4 hijos (1 a 3 claves).

            [10, 20]                    ← raíz: 2 claves, 3 hijos
           /    |    \
    [3, 5]   [12, 15]   [25, 30, 35]   ← hojas: 2-3 claves cada una

  Búsqueda de key=15:
    Raíz [10, 20]: 10 < 15 < 20 → hijo del medio.
    [12, 15]: 15 encontrado. ✓
    → 2 accesos a nodos. Si cada nodo es una página: 2 I/Os.

  Búsqueda de key=17:
    Raíz [10, 20]: 10 < 17 < 20 → hijo del medio.
    [12, 15]: 15 < 17 → no encontrado. ✗
    → 2 accesos. Key no existe.

  Búsqueda dentro de un nodo:
    Las claves están ordenadas → binary search.
    Un nodo con 500 claves: binary search = ~9 comparaciones.
    Estas comparaciones son en RAM (el nodo ya fue leído del disco).
    → El costo es 1 I/O + 9 comparaciones en RAM ≈ 1 I/O.

  Complejidad de búsqueda:
    h niveles × (1 I/O + O(log B) comparaciones) = h I/Os + O(h × log B) CPU.
    El costo está DOMINADO por los I/Os, no por las comparaciones.
```

**Preguntas:**

1. ¿"Orden M" vs "grado mínimo t" — ¿son dos formas de definir lo mismo?

2. ¿Todas las hojas al mismo nivel. ¿Cómo se garantiza?

3. ¿⌈M/2⌉ hijos mínimo — ¿por qué este límite inferior?

4. ¿Binary search dentro del nodo vs linear search
   — ¿cuándo importa?

5. ¿Un nodo con 500 claves y binary search hace 9 comparaciones.
   ¿En un RBT serían 9 nodos = 9 I/Os?

---

### Ejercicio 9.2.2 — Leer: inserción con split

**Tipo: Leer**

```
Insert en un B-Tree de orden M=4 (max 3 claves por nodo):

  Estado inicial:
            [10, 20]
           /    |    \
    [3, 5]   [12, 15]   [25, 30, 35]

  Insertar 28:
    1. Buscar posición: raíz [10,20] → 28>20 → hijo derecho [25,30,35].
    2. [25,30,35] tiene espacio? Tiene 3 claves (máximo para M=4). NO.
    3. SPLIT: dividir [25,28,30,35] en dos nodos:
       Mediana: 30. Sube al padre.
       Izquierdo: [25, 28]. Derecho: [35].
    4. Padre [10,20] recibe mediana 30 → [10, 20, 30].
       Tiene 3 claves. OK (cabe en M=4).

  Resultado:
            [10, 20, 30]
           /    |    |    \
    [3, 5]  [12,15] [25,28]  [35]

  ¿Y si el padre también está lleno?
    El split SE PROPAGA hacia arriba.
    Si la raíz se divide, se crea una nueva raíz.
    → El árbol crece en ALTURA solo cuando la raíz se divide.
    → Esto garantiza que todas las hojas estén al mismo nivel.

  Insertar 22 (ahora la raíz tiene 3 claves, está llena):
    1. Buscar: [10,20,30] → 20<22<30 → hijo [25,28].
    2. [25,28] tiene espacio → insertar → [22,25,28].

  Insertar 27:
    1. Buscar: [10,20,30] → 20<27<30 → hijo [22,25,28].
    2. [22,25,28] está lleno. SPLIT.
       Mediana: 25. Sube al padre.
       Izquierdo: [22]. Derecho: [27,28].
    3. Padre [10,20,30] recibe 25 → [10,20,25,30]. 4 claves, LLENO para M=4.
       SPLIT del padre.
       Mediana: 20. Se convierte en NUEVA RAÍZ.
       Izquierdo: [10]. Derecho: [25,30].

  Resultado:
                [20]                    ← nueva raíz
              /       \
         [10]           [25, 30]
        /    \         /   |    \
     [3,5] [12,15]  [22] [27,28] [35]

  El árbol creció de altura 2 a altura 3.
  Esto solo pasa cuando la raíz se divide.
```

**Preguntas:**

1. ¿El split siempre sube la MEDIANA. ¿Por qué no la menor o mayor?

2. ¿Split propagado hasta la raíz — ¿cuántos I/Os cuesta?

3. ¿El árbol crece hacia ARRIBA, no hacia abajo.
   ¿Eso es diferente de un BST?

4. ¿Con M=1000, ¿cada cuántos inserts ocurre un split?

5. ¿Insert de claves ordenadas (1,2,3,...,n)
   — ¿causa muchos splits?

---

## Sección 9.3 — Implementar un B-Tree Simplificado

### Ejercicio 9.3.1 — Implementar: B-Tree búsqueda e inserción en Java

**Tipo: Implementar**

```java
import java.util.*;

public class BTree {

    private static final int M = 6; // order: max 6 children, max 5 keys per node
    private static final int MIN_KEYS = M / 2 - 1; // min 2 keys (except root)

    static class Node {
        int n;              // number of keys
        int[] keys;         // keys[0..n-1]
        Node[] children;    // children[0..n] (null for leaf)
        boolean leaf;

        Node(boolean leaf) {
            this.leaf = leaf;
            this.keys = new int[M - 1];
            this.children = new Node[M];
        }
    }

    private Node root;

    public BTree() {
        root = new Node(true);
    }

    // ═══ Search ═══

    public boolean search(int key) {
        return search(root, key);
    }

    private boolean search(Node node, int key) {
        int i = 0;
        while (i < node.n && key > node.keys[i]) i++;
        if (i < node.n && key == node.keys[i]) return true;
        if (node.leaf) return false;
        return search(node.children[i], key);
    }

    // ═══ Insert ═══

    public void insert(int key) {
        Node r = root;
        if (r.n == M - 1) {
            // Root is full: split it
            Node newRoot = new Node(false);
            newRoot.children[0] = r;
            splitChild(newRoot, 0);
            root = newRoot;
            insertNonFull(newRoot, key);
        } else {
            insertNonFull(r, key);
        }
    }

    private void insertNonFull(Node node, int key) {
        int i = node.n - 1;
        if (node.leaf) {
            // Shift keys and insert
            while (i >= 0 && key < node.keys[i]) {
                node.keys[i + 1] = node.keys[i];
                i--;
            }
            node.keys[i + 1] = key;
            node.n++;
        } else {
            while (i >= 0 && key < node.keys[i]) i--;
            i++;
            if (node.children[i].n == M - 1) {
                splitChild(node, i);
                if (key > node.keys[i]) i++;
            }
            insertNonFull(node.children[i], key);
        }
    }

    private void splitChild(Node parent, int i) {
        Node full = parent.children[i];
        Node newNode = new Node(full.leaf);
        int mid = (M - 1) / 2;

        // Copy upper half of keys to new node
        newNode.n = full.n - mid - 1;
        for (int j = 0; j < newNode.n; j++)
            newNode.keys[j] = full.keys[mid + 1 + j];

        // Copy upper half of children (if not leaf)
        if (!full.leaf) {
            for (int j = 0; j <= newNode.n; j++)
                newNode.children[j] = full.children[mid + 1 + j];
        }

        // Median key moves up to parent
        // Shift parent's keys and children
        for (int j = parent.n - 1; j >= i; j--)
            parent.keys[j + 1] = parent.keys[j];
        parent.keys[i] = full.keys[mid];
        for (int j = parent.n; j >= i + 1; j--)
            parent.children[j + 1] = parent.children[j];
        parent.children[i + 1] = newNode;
        parent.n++;
        full.n = mid;
    }

    // ═══ Stats ═══

    public int height() { return height(root); }
    private int height(Node n) {
        if (n.leaf) return 1;
        return 1 + height(n.children[0]);
    }

    public int size() { return size(root); }
    private int size(Node n) {
        int count = n.n;
        if (!n.leaf) {
            for (int i = 0; i <= n.n; i++)
                count += size(n.children[i]);
        }
        return count;
    }

    public static void main(String[] args) {
        var bt = new BTree();
        int n = 1_000_000;
        var rng = new Random(42);

        long start = System.nanoTime();
        for (int i = 0; i < n; i++) bt.insert(rng.nextInt(n * 10));
        long elapsed = System.nanoTime() - start;

        System.out.printf("B-Tree (M=%d): %,d keys%n", M, bt.size());
        System.out.printf("Height: %d (optimal log_%d(%d) ≈ %.1f)%n",
            bt.height(), M, bt.size(), Math.log(bt.size()) / Math.log(M));
        System.out.printf("Insert time: %d ms (%.0f ns/op)%n",
            elapsed / 1_000_000, (double) elapsed / n);

        // Search benchmark
        start = System.nanoTime();
        int found = 0;
        for (int i = 0; i < n; i++) {
            if (bt.search(rng.nextInt(n * 10))) found++;
        }
        long searchTime = System.nanoTime() - start;
        System.out.printf("Search: %d ms (%.0f ns/op), found=%d%n",
            searchTime / 1_000_000, (double) searchTime / n, found);

        // Sorted inserts (worst case for BST, fine for B-Tree)
        var btSorted = new BTree();
        start = System.nanoTime();
        for (int i = 0; i < n; i++) btSorted.insert(i);
        long sortedTime = System.nanoTime() - start;
        System.out.printf("%nSorted inserts: %d ms, height=%d%n",
            sortedTime / 1_000_000, btSorted.height());
    }
}
// Resultado esperado (M=6):
// B-Tree (M=6): ~951,000 keys (some duplicates lost)
// Height: 8 (optimal log_6(951000) ≈ 7.7)
// Insert time: ~600-1000 ms
// Search: ~400-700 ms
// Sorted inserts: height=8 — B-Tree handles sorted input perfectly!
```

**Preguntas:**

1. ¿M=6 da un B-Tree con ~8 niveles para 1M claves.
   ¿Con M=1000 serían ~2 niveles?

2. ¿`splitChild` mueve la mediana al padre. ¿Es la operación más costosa?

3. ¿Sorted inserts producen la misma altura que random inserts.
   ¿Es una ventaja del B-Tree sobre BST?

4. ¿Esta implementación usa arrays in-memory.
   ¿En una base de datos, cada nodo sería una página de disco?

5. ¿Falta delete. ¿Es más complejo que insert?

---

### Ejercicio 9.3.2 — Analizar: delete en un B-Tree

**Tipo: Analizar**

```
Delete en un B-Tree es más complejo que insert.

  Tres casos:

  CASO 1: La clave está en una HOJA y la hoja tiene más que el mínimo de claves.
    → Simplemente eliminar la clave. Nada más.

  CASO 2: La clave está en un nodo INTERNO.
    → Reemplazar con el predecesor (mayor clave del subárbol izquierdo)
      o el sucesor (menor clave del subárbol derecho).
    → Luego eliminar el predecesor/sucesor de la hoja.
    → Similar a delete en BST, pero con el constraint de tamaño mínimo.

  CASO 3: La hoja donde debe eliminarse tiene exactamente el mínimo de claves.
    Antes de eliminar, garantizar que la hoja tenga claves extra.
    Opciones:
    a) REDISTRIBUTION: tomar prestada una clave de un hermano adyacente
       que tenga más del mínimo. Rotar a través del padre.
    b) MERGE: si ningún hermano tiene claves extra,
       fusionar la hoja con un hermano y bajar una clave del padre.
       → Si el padre queda con menos del mínimo: propagar el merge hacia arriba.

  Merge puede propagarse hasta la raíz:
    Si la raíz se queda con 0 claves después de un merge,
    la raíz se elimina y el único hijo se convierte en nueva raíz.
    → El árbol reduce su altura.

  Complejidad: O(h) = O(log_B n) accesos a nodos.
  Número de I/Os: como máximo 2h + 1 (lecturas y escrituras).

  ¿Por qué no implementamos delete?
    1. Es 3× más código que insert.
    2. Muchas bases de datos no hacen delete real del B-Tree:
       usan "tombstones" (marcar como eliminado, limpiar después).
    3. El concepto es el mismo: mantener el invariante de tamaño mínimo.
```

**Preguntas:**

1. ¿"Redistribution" vs "merge" — ¿cuándo se elige cada uno?

2. ¿Tombstones en vez de delete real — ¿es lo que hace InnoDB?

3. ¿Merge puede reducir la altura del árbol. ¿Cuándo pasa?

4. ¿Delete es más costoso que insert por los merges?

5. ¿Un B-Tree que nunca hace deletes (append-only) es más simple?

---

## Sección 9.4 — B+ Tree: la Variante que Domina las Bases de Datos

### Ejercicio 9.4.1 — Leer: B-Tree vs B+ Tree

**Tipo: Leer**

```
B+ Tree: la evolución del B-Tree para bases de datos.

  Diferencias clave respecto al B-Tree:

  1. DATOS SOLO EN LAS HOJAS.
     Los nodos internos solo almacenan CLAVES como guías de navegación.
     Los datos (values, filas, punteros a filas) solo están en las hojas.
     → Los nodos internos son más compactos → mayor branching factor
       → menor altura.

  2. HOJAS ENLAZADAS.
     Las hojas forman una LINKED LIST (doubly-linked).
     → Range scans: encontrar la primera hoja, luego seguir los enlaces.
     → No necesitas volver a subir al árbol para el siguiente rango.

  3. CLAVES DUPLICADAS EN NODOS INTERNOS.
     Los nodos internos contienen COPIAS de las claves de las hojas.
     Una clave puede aparecer tanto en un nodo interno como en una hoja.
     (En un B-Tree estándar, cada clave aparece exactamente una vez.)

  Visualización B+ Tree:

    Nodos internos (solo claves para navegación):
                    [20]
                  /      \
             [5, 10]     [25, 30]

    Hojas (contienen todos los datos, enlazadas):
    [1,3,5] ↔ [7,8,10] ↔ [12,15,20] ↔ [22,23,25] ↔ [28,30,35]
    ←───────────── linked list ──────────────────→

    Range scan (10 to 25):
    1. Buscar 10 → navegar nodos internos → llegar a hoja [7,8,10].
    2. Empezar en key=10.
    3. Seguir los enlaces: [12,15,20] → [22,23,25]. Stop.
    → Sin necesidad de volver a navegar desde la raíz.

  ¿Por qué B+ Tree es mejor para bases de datos?

  1. NODOS INTERNOS MÁS COMPACTOS:
     Sin datos en nodos internos → más claves por nodo → más fanout.
     B-Tree (M=500, datos en todos los nodos).
     B+ Tree (M=1000, datos solo en hojas, nodos internos el doble de compactos).
     → Menor altura para el mismo n.

  2. RANGE SCANS EFICIENTES:
     "SELECT * FROM users WHERE age BETWEEN 20 AND 30"
     Encontrar primera hoja: O(log n). Scan: O(k) siguiendo enlaces.
     En un B-Tree estándar: cada key podría requerir navegar desde la raíz.

  3. PREFETCH FRIENDLY:
     Las hojas son contiguas en disco (idealmente).
     Range scan = sequential I/O → mucho más rápido que random I/O.

  Esto es lo que usan:
    PostgreSQL: B+ Tree indexes.
    MySQL/InnoDB: B+ Tree (clustered y secondary indexes).
    SQLite: B+ Tree para tablas, B-Tree para indexes.
    BoltDB: B+ Tree.
    LMDB: B+ Tree con copy-on-write.
```

**Preguntas:**

1. ¿"Datos solo en hojas" — ¿los nodos internos son significativamente
   más pequeños?

2. ¿Las hojas enlazadas hacen range scans O(k) donde k es el resultado.
   ¿Un B-Tree estándar sería O(k log n)?

3. ¿InnoDB clustered index: las hojas contienen las FILAS completas.
   ¿Eso es diferente de un secondary index?

4. ¿Si todas las hojas están enlazadas, ¿una iteración completa
   recorre SOLO las hojas sin tocar nodos internos?

5. ¿PostgreSQL y MySQL usan B+ Tree. ¿Alguna base de datos
   usa B-Tree estándar?

---

### Ejercicio 9.4.2 — Analizar: clustered vs secondary index

**Tipo: Analizar**

```
En InnoDB (MySQL), hay dos tipos de B+ Tree indexes:

  CLUSTERED INDEX:
    Las hojas contienen la FILA COMPLETA de datos.
    La tabla completa está ALMACENADA en el B+ Tree.
    Ordenado por la primary key.
    Cada tabla tiene exactamente UN clustered index.

    Hoja del clustered index:
    [..., (id=42, name="Alice", age=30, email="alice@..."), ...]

    Buscar por PK: 1-2 I/Os (navegar B+ Tree → hoja con datos).
    Range scan por PK: encontrar primera hoja → sequential scan.
    → Extremadamente eficiente.

  SECONDARY INDEX:
    Las hojas contienen la clave del index + la PRIMARY KEY.
    Para obtener la fila completa: buscar en el secondary index,
    luego buscar la PK en el clustered index.
    → Doble lookup ("bookmark lookup").

    Hoja del secondary index (index en age):
    [..., (age=30, pk=42), (age=30, pk=78), (age=31, pk=15), ...]

    Buscar por age=30:
    1. Navegar secondary index → hoja con (age=30, pk=42). 1-2 I/Os.
    2. Buscar pk=42 en clustered index → fila completa. 1-2 I/Os.
    → 2-4 I/Os total.

  Covering index:
    Si TODAS las columnas del query están en el secondary index,
    no necesitas el segundo lookup al clustered index.
    CREATE INDEX idx_age_name ON users(age, name);
    SELECT name FROM users WHERE age = 30;
    → El secondary index tiene (age, name, pk). Todo lo necesario.
    → Sin doble lookup. Tan rápido como buscar por PK.

  PostgreSQL es diferente:
    No tiene "clustered index" permanente.
    Los datos (heap) están separados de todos los indexes.
    Cada index apunta a un (page, offset) en el heap.
    → Siempre hay un "bookmark lookup" para obtener la fila.
    → CLUSTER command re-ordena el heap, pero no se mantiene.
```

**Preguntas:**

1. ¿InnoDB sin primary key explícita — ¿qué usa como clustered index?

2. ¿"Bookmark lookup" en un secondary index. ¿Cuántos I/Os extra?

3. ¿Covering index elimina el doble lookup. ¿Es siempre la solución?

4. ¿PostgreSQL vs InnoDB para range scans por PK
   — ¿cuál es más rápido?

5. ¿Un index con 10 columnas ocupa mucha más memoria?

---

## Sección 9.5 — Implementar un B+ Tree con Range Scan

### Ejercicio 9.5.1 — Implementar: B+ Tree en Java

**Tipo: Implementar**

```java
import java.util.*;

public class BPlusTree {

    private static final int ORDER = 6; // max children per internal node
    private static final int MAX_KEYS = ORDER - 1;
    private static final int MIN_KEYS = ORDER / 2 - 1;

    abstract static class Node {
        int[] keys;
        int n; // number of keys
        Node(int maxKeys) { keys = new int[maxKeys + 1]; } // +1 for overflow during split
    }

    static class InternalNode extends Node {
        Node[] children;
        InternalNode() {
            super(MAX_KEYS);
            children = new Node[ORDER + 1];
        }
    }

    static class LeafNode extends Node {
        int[] values;       // data associated with each key
        LeafNode next;      // linked list of leaves
        LeafNode prev;
        LeafNode() {
            super(MAX_KEYS);
            values = new int[MAX_KEYS + 1];
        }
    }

    private Node root;
    private int size;

    public BPlusTree() {
        root = new LeafNode();
    }

    // ═══ Search ═══

    public Integer get(int key) {
        LeafNode leaf = findLeaf(key);
        for (int i = 0; i < leaf.n; i++) {
            if (leaf.keys[i] == key) return leaf.values[i];
        }
        return null;
    }

    private LeafNode findLeaf(int key) {
        Node node = root;
        while (node instanceof InternalNode internal) {
            int i = 0;
            while (i < internal.n && key >= internal.keys[i]) i++;
            node = internal.children[i];
        }
        return (LeafNode) node;
    }

    // ═══ Range Scan ═══

    public List<int[]> rangeScan(int lo, int hi) {
        List<int[]> result = new ArrayList<>();
        LeafNode leaf = findLeaf(lo);

        outer:
        while (leaf != null) {
            for (int i = 0; i < leaf.n; i++) {
                if (leaf.keys[i] > hi) break outer;
                if (leaf.keys[i] >= lo) {
                    result.add(new int[]{leaf.keys[i], leaf.values[i]});
                }
            }
            leaf = leaf.next;
        }
        return result;
    }

    // ═══ Insert ═══

    public void put(int key, int value) {
        // Find leaf
        List<InternalNode> path = new ArrayList<>();
        Node node = root;
        while (node instanceof InternalNode internal) {
            path.add(internal);
            int i = 0;
            while (i < internal.n && key >= internal.keys[i]) i++;
            node = internal.children[i];
        }

        LeafNode leaf = (LeafNode) node;

        // Insert into leaf (sorted)
        int pos = 0;
        while (pos < leaf.n && leaf.keys[pos] < key) pos++;
        if (pos < leaf.n && leaf.keys[pos] == key) {
            leaf.values[pos] = value; // update
            return;
        }

        // Shift and insert
        for (int i = leaf.n; i > pos; i--) {
            leaf.keys[i] = leaf.keys[i - 1];
            leaf.values[i] = leaf.values[i - 1];
        }
        leaf.keys[pos] = key;
        leaf.values[pos] = value;
        leaf.n++;
        size++;

        // Split if overflow
        if (leaf.n > MAX_KEYS) {
            splitLeaf(leaf, path);
        }
    }

    private void splitLeaf(LeafNode leaf, List<InternalNode> path) {
        LeafNode newLeaf = new LeafNode();
        int mid = (leaf.n + 1) / 2;

        // Move upper half to new leaf
        newLeaf.n = leaf.n - mid;
        for (int i = 0; i < newLeaf.n; i++) {
            newLeaf.keys[i] = leaf.keys[mid + i];
            newLeaf.values[i] = leaf.values[mid + i];
        }
        leaf.n = mid;

        // Update linked list
        newLeaf.next = leaf.next;
        newLeaf.prev = leaf;
        if (leaf.next != null) leaf.next.prev = newLeaf;
        leaf.next = newLeaf;

        // Push up the first key of new leaf
        int pushKey = newLeaf.keys[0];
        insertIntoParent(leaf, pushKey, newLeaf, path);
    }

    private void insertIntoParent(Node left, int key, Node right, List<InternalNode> path) {
        if (path.isEmpty()) {
            // Create new root
            InternalNode newRoot = new InternalNode();
            newRoot.keys[0] = key;
            newRoot.children[0] = left;
            newRoot.children[1] = right;
            newRoot.n = 1;
            root = newRoot;
            return;
        }

        InternalNode parent = path.remove(path.size() - 1);
        int pos = 0;
        while (pos < parent.n && key > parent.keys[pos]) pos++;

        // Shift and insert
        for (int i = parent.n; i > pos; i--) {
            parent.keys[i] = parent.keys[i - 1];
            parent.children[i + 1] = parent.children[i];
        }
        parent.keys[pos] = key;
        parent.children[pos + 1] = right;
        parent.n++;

        // Split internal if overflow
        if (parent.n > MAX_KEYS) {
            splitInternal(parent, path);
        }
    }

    private void splitInternal(InternalNode node, List<InternalNode> path) {
        InternalNode newNode = new InternalNode();
        int mid = node.n / 2;
        int pushKey = node.keys[mid];

        newNode.n = node.n - mid - 1;
        for (int i = 0; i < newNode.n; i++)
            newNode.keys[i] = node.keys[mid + 1 + i];
        for (int i = 0; i <= newNode.n; i++)
            newNode.children[i] = node.children[mid + 1 + i];
        node.n = mid;

        insertIntoParent(node, pushKey, newNode, path);
    }

    // ═══ Stats ═══

    public int height() {
        int h = 0;
        Node n = root;
        while (n != null) {
            h++;
            n = (n instanceof InternalNode in) ? in.children[0] : null;
        }
        return h;
    }

    public int size() { return size; }

    public static void main(String[] args) {
        var tree = new BPlusTree();
        int n = 1_000_000;
        var rng = new Random(42);

        // Insert
        long start = System.nanoTime();
        for (int i = 0; i < n; i++) tree.put(rng.nextInt(n * 10), i);
        long insertTime = System.nanoTime() - start;
        System.out.printf("B+ Tree (order=%d): %,d entries, height=%d%n",
            ORDER, tree.size(), tree.height());
        System.out.printf("Insert: %d ms (%.0f ns/op)%n",
            insertTime / 1_000_000, (double) insertTime / n);

        // Point query
        rng = new Random(42);
        start = System.nanoTime();
        int found = 0;
        for (int i = 0; i < n; i++) {
            if (tree.get(rng.nextInt(n * 10)) != null) found++;
        }
        long queryTime = System.nanoTime() - start;
        System.out.printf("Point query: %d ms (%.0f ns/op), found=%d%n",
            queryTime / 1_000_000, (double) queryTime / n, found);

        // Range scan
        start = System.nanoTime();
        var results = tree.rangeScan(1_000_000, 1_001_000);
        long rangeTime = System.nanoTime() - start;
        System.out.printf("Range scan [1M, 1M+1000]: %d results in %d μs%n",
            results.size(), rangeTime / 1000);
    }
}
// Resultado esperado:
// B+ Tree (order=6): ~951,000 entries, height=8
// Insert: ~800-1200 ms
// Point query: ~500-800 ms
// Range scan [1M, 1M+1000]: ~100 results in ~20-50 μs
```

**Preguntas:**

1. ¿Las hojas enlazadas hacen range scan eficiente:
   O(log n) para encontrar + O(k) para escanear. ¿Es correcto?

2. ¿Split de hoja vs split de nodo interno — ¿cuál es la diferencia?

3. ¿Esta implementación in-memory. ¿Qué cambiaría para disco?

4. ¿Con ORDER=6, la altura es 8. ¿Con ORDER=500?

5. ¿La implementación no soporta delete. ¿Cómo lo manejan
   las bases de datos?

---

### Ejercicio 9.5.2 — Analizar: height vs branching factor empírico

**Tipo: Analizar**

```
Medir la altura del B+ Tree para distintos branching factors:

  Para n = 10,000,000 entries:

  Order (M)   Max keys/node   Height   Level 0 nodes   Leaf nodes
  ─────────   ─────────────   ──────   ─────────────   ──────────
  4           3               15       1               ~5,000,000
  8           7               8        1               ~1,666,666
  16          15              6        1               ~714,285
  32          31              5        1               ~344,827
  64          63              4        1               ~163,934
  128         127             3        1               ~79,365
  256         255             3        1               ~39,370
  512         511             2        1               ~19,569
  1024        1023            2        1               ~9,775

  Observaciones:
  - De M=4 a M=128: la altura cae dramáticamente (15 → 3).
  - De M=128 a M=1024: la mejora es marginal (3 → 2).
  - Con M=1024, 10M entries caben en 2 niveles.
    La raíz tiene ~10K claves → el nivel 0 apunta directamente a las hojas.
  - En una base de datos real: M=500-1000 (page size 16 KB).
    → Billones de filas en 3-4 niveles = 3-4 I/Os máximo.
```

**Preguntas:**

1. ¿Altura 2 para 10M entries con M=1024. ¿La raíz cabe en RAM?

2. ¿5M hojas con M=4. ¿Cuánta memoria total?

3. ¿Page size de 16 KB da M≈1000. ¿Page size de 64 KB da M≈4000?

4. ¿Para billones de filas en un data warehouse,
   ¿la altura sigue siendo 3-4?

5. ¿Un B+ Tree más ancho tiene más write amplification en inserts?

---

## Sección 9.6 — B-Trees en la Práctica: Pages, WAL, Concurrencia

### Ejercicio 9.6.1 — Leer: page layout en una base de datos real

**Tipo: Leer**

```
En una base de datos, un nodo del B+ Tree es una PAGE en disco.

  PostgreSQL page layout (8 KB):
  ┌────────────────────────────────────────────┐
  │ Page header (24 bytes)                     │
  │   - LSN (log sequence number)              │
  │   - checksum                               │
  │   - flags, free space pointer              │
  ├────────────────────────────────────────────┤
  │ Item pointers (4 bytes each)               │
  │   - offset + length de cada tuple/key      │
  ├────────────────────────────────────────────┤
  │         Free space                         │
  ├────────────────────────────────────────────┤
  │ Tuples / Key-Value pairs                   │
  │   - datos almacenados de abajo hacia arriba│
  └────────────────────────────────────────────┘

  Item pointers crecen hacia abajo, datos crecen hacia arriba.
  → Insertar un nuevo item: agregar item pointer arriba, datos abajo.
  → No necesitas mover datos existentes.

  Write-Ahead Logging (WAL):
    ANTES de modificar una page del B-Tree,
    escribir un log entry en el WAL (sequential write, rápido).
    SI el sistema crashea: reconstruir el B-Tree desde el WAL.
    → Sin WAL: un crash durante un split podría corromper el árbol.

  Write Amplification:
    Un insert lógico puede causar:
    1. Escribir el WAL entry.
    2. Modificar la hoja (dirty page).
    3. Si split: escribir 2 nuevas pages + modificar el padre.
    4. Eventualmente, el background writer escribe las dirty pages a disco.
    → Un insert lógico = 2-5 escrituras físicas.
    → "Write amplification" = escrituras físicas / escrituras lógicas.

  Buffer Pool:
    Las pages no se leen/escriben directamente del disco.
    La base de datos mantiene un BUFFER POOL en RAM.
    Pages frecuentemente accedidas permanecen en el buffer pool.
    → La raíz y niveles superiores del B-Tree casi siempre están en RAM.
    → Solo las hojas requieren I/O en el caso típico.
```

**Preguntas:**

1. ¿El LSN en el page header conecta la page con el WAL.
   ¿Para qué sirve durante recovery?

2. ¿Write amplification de 2-5× — ¿es aceptable?

3. ¿El buffer pool de PostgreSQL por defecto es cuántos MB/GB?

4. ¿"Item pointers crecen hacia abajo, datos hacia arriba"
   — ¿por qué no todo en una dirección?

5. ¿Si el buffer pool es suficientemente grande,
   ¿todo el B-Tree cabe en RAM?

---

### Ejercicio 9.6.2 — Analizar: concurrencia en B-Trees

**Tipo: Analizar**

```
Múltiples queries accediendo al B-Tree simultáneamente.

  Problema: un thread hace un range scan mientras otro hace un insert
  que causa un split. El scan podría ver datos inconsistentes.

  Soluciones:

  1. LOCK-BASED (PostgreSQL, InnoDB):
     Latch (lightweight lock) por page.
     Read latch: múltiples readers.
     Write latch: un solo writer, bloquea readers.

     Protocolo "crabbing" (latch coupling):
     Al navegar de arriba a abajo:
     1. Tomar latch en el nodo hijo.
     2. Soltar latch en el padre (si el hijo es "seguro" — no va a hacer split).
     3. Si el hijo PODRÍA hacer split: mantener latch en el padre.
     → Minimiza el tiempo que los latches están tomados.
     → Readers nunca bloquean a otros readers.

  2. COPY-ON-WRITE (LMDB, BoltDB):
     NUNCA modificar una page existente.
     Cada escritura crea NUEVAS páginas (path copying, como el HAMT del Cap.05).
     Los readers ven una SNAPSHOT consistente (la versión anterior).
     No necesitan latches. Lock-free reads.

     Ventaja: readers nunca se bloquean. Perfecto para read-heavy.
     Desventaja: write amplification alta (copiar todo el path por cada write).

  3. BLINK-TREE (optimización):
     Variante del B+ Tree donde cada nodo interno tiene un "right link"
     al hermano derecho. Si un split ocurre durante una búsqueda,
     el searcher puede seguir el right link sin necesidad de reiniciar.
     → Menos bloqueo durante splits.
     → PostgreSQL usa una variante de Blink-tree.
```

**Preguntas:**

1. ¿Crabbing protocol — ¿es lo que PostgreSQL hace internamente?

2. ¿Copy-on-write como LMDB — ¿es similar a path copying del HAMT?

3. ¿BoltDB (usado por etcd) usa copy-on-write.
   ¿Es por eso que Kubernetes reads son rápidos?

4. ¿Write amplification de copy-on-write — ¿es peor que la de un B-Tree normal?

5. ¿Para un write-heavy workload, ¿copy-on-write B-Tree o latch-based?

---

### Ejercicio 9.6.3 — Analizar: copy-on-write B-Tree (LMDB/BoltDB)

**Tipo: Analizar**

```
Copy-on-Write B-Tree (BoltDB):

  Estructura:
    El B+ Tree vive en un archivo mapeado a memoria (mmap).
    Cada transacción de escritura crea NUEVAS páginas.
    Las transacciones de lectura usan las páginas ANTIGUAS.
    → MVCC (Multi-Version Concurrency Control) sin locks.

  Write:
    1. Copiar el path desde la raíz hasta la hoja modificada.
    2. Modificar la copia de la hoja.
    3. Los nodos internos en el path ahora apuntan a la nueva hoja.
    4. Actualizar la raíz atómicamente (puntero a nueva raíz).
    5. Las páginas antiguas se marcan como libres
       (solo cuando ningún reader las está usando).

  Ejemplo: insertar key=42
    Raíz_v1 → [Internal_v1] → [Leaf_v1 (con datos actuales)]

    1. Copiar Leaf_v1 → Leaf_v2 (con key=42 insertada).
    2. Copiar Internal_v1 → Internal_v2 (apuntando a Leaf_v2).
    3. Crear Raíz_v2 (apuntando a Internal_v2).
    4. Raíz activa = Raíz_v2.

    Readers activos en Raíz_v1 siguen viendo la versión anterior.
    Nuevos readers usan Raíz_v2.

  Ventajas:
    - Reads NUNCA bloquean. Ni por writes ni por otros reads.
    - Crash recovery trivial: si el crash ocurre antes de
      actualizar la raíz, las nuevas páginas simplemente se ignoran.
    - Sin WAL necesario (las antiguas páginas SON el "log").

  Desventajas:
    - Write amplification: copiar todo el path (h páginas) por write.
    - Espacio en disco: versiones antiguas ocupan espacio hasta que
      se liberan (cuando ningún reader las usa).
    - Un solo writer a la vez (BoltDB). Serialización de writes.
```

**Preguntas:**

1. ¿"Sin WAL necesario" — ¿es una simplificación significativa?

2. ¿BoltDB serializa writes (un writer a la vez).
   ¿Es aceptable para etcd?

3. ¿LMDB usa mmap. ¿Eso significa que el OS maneja el caching?

4. ¿Copy-on-write es lo mismo que structural sharing del HAMT?

5. ¿Para un sistema de analytics con 95% reads y 5% writes,
   ¿copy-on-write B-Tree es ideal?

---

## Sección 9.7 — B-Trees en Producción: de PostgreSQL a BoltDB

### Ejercicio 9.7.1 — Analizar: el B+ Tree en PostgreSQL

**Tipo: Analizar**

```
PostgreSQL B-Tree index:

  CREATE INDEX idx_users_email ON users(email);

  Esto crea un B+ Tree donde:
    - Nodos internos: contienen claves (email) para navegación.
    - Hojas: contienen (email, ctid) donde ctid = (page, offset) en el heap.
    - Hojas enlazadas: para range scans.

  Page size: 8 KB (configurable con --with-blocksize al compilar).
  Fill factor: 90% por defecto (10% libre para inserts sin split).
  Typical fanout: ~200-500 (depende del tamaño de la clave).

  Query execution:
    SELECT * FROM users WHERE email = 'alice@example.com';
    Plan: Index Scan using idx_users_email on users
          Index Cond: (email = 'alice@example.com')

    1. Navegar B+ Tree: 2-3 I/Os (root → internal → leaf).
    2. Obtener ctid del heap tuple.
    3. Leer la página del heap: 1 I/O.
    → Total: 3-4 I/Os.

  Range query:
    SELECT * FROM users WHERE email BETWEEN 'a%' AND 'b%';
    Plan: Index Scan (o Bitmap Index Scan)
    1. Encontrar primera hoja: 2-3 I/Os.
    2. Scan hojas enlazadas: sequential reads.
    3. Para cada key: fetch del heap (random I/O).
    → Si muchas rows: el planner puede elegir Sequential Scan.

  EXPLAIN ANALYZE muestra el plan real y los tiempos:
    Index Scan: rows=1, loops=1, time=0.05ms (index) + 0.02ms (heap).
    → ~70 μs total para un point query con index.
```

**Preguntas:**

1. ¿Fill factor 90% — ¿por qué no 100%?

2. ¿El query planner puede elegir Sequential Scan en vez de Index Scan.
   ¿Cuándo?

3. ¿Bitmap Index Scan — ¿qué es y cuándo se usa?

4. ¿70 μs para un point query con index.
   ¿Es más rápido que un hash index?

5. ¿PostgreSQL soporta hash indexes. ¿Cuándo son mejores que B-Tree?

---

### Ejercicio 9.7.2 — Analizar: InnoDB vs PostgreSQL B+ Tree

**Tipo: Analizar**

```
Comparación de implementaciones:

  Aspecto              InnoDB (MySQL)          PostgreSQL
  ───────              ──────────────          ──────────
  Clustered index      Sí (datos en hojas)     No (heap separado)
  Page size            16 KB                   8 KB
  Typical fanout       ~500-1000               ~200-500
  PK lookup            1-2 I/Os (directo)      2-4 I/Os (index + heap)
  Secondary index      2 lookups (idx → PK)    1 lookup (idx → ctid)
  Range scan por PK    Sequential en hojas     Random en heap
  Concurrency          Latch-based + MVCC      Latch-based + MVCC
  Write amplification  Moderate                Moderate + HOT updates

  InnoDB ventajas:
  - PK lookups extremadamente rápidos (datos en las hojas).
  - Range scans por PK son secuenciales (hojas ordenadas).
  - Menor I/O total para workloads PK-heavy.

  PostgreSQL ventajas:
  - Secondary indexes apuntan directamente al heap (ctid) → sin doble lookup.
  - HOT updates: si la nueva versión cabe en la misma page,
    no necesita actualizar los indexes. InnoDB SIEMPRE actualiza.
  - Más flexible para schemas con muchos secondary indexes.

  Para un data engineer:
  - Si tu workload es PK-heavy (lookups/scans por PK): InnoDB.
  - Si tienes muchos secondary indexes y updates: PostgreSQL.
  - Si usas ETL y bulk loading: ambos son comparables.
```

**Preguntas:**

1. ¿InnoDB PK lookup en 1-2 I/Os vs PostgreSQL en 2-4.
   ¿Es siempre 2× más rápido?

2. ¿HOT updates de PostgreSQL — ¿es una ventaja significativa?

3. ¿Para un OLAP workload con columnar scans,
   ¿el B+ Tree es relevante?

4. ¿Parquet no usa B+ Trees. ¿Qué usa?

5. ¿Las bases de datos NewSQL (CockroachDB, TiDB)
   usan B-Trees o LSM-Trees?

---

### Ejercicio 9.7.3 — Resumen: las reglas del B-Tree

**Tipo: Leer**

```
Reglas nuevas del Cap.09:

  Regla 36: El B-Tree existe porque el disco es lento.
    RAM: 100 ns. SSD: 100 μs. HDD: 10 ms.
    Cada nivel del árbol = 1 I/O.
    Reducir la altura de 30 (binario) a 3 (B-Tree) = 10× más rápido.
    El B-Tree amortiza el costo de I/O empacando claves en páginas.

  Regla 37: B+ Tree > B-Tree para bases de datos.
    Datos solo en hojas → nodos internos más compactos → más fanout.
    Hojas enlazadas → range scans eficientes.
    PostgreSQL, MySQL, SQLite, SQL Server: todos usan B+ Tree.

  Regla 38: El branching factor lo determina el page size.
    Page 4 KB → B≈250. Page 16 KB → B≈1000.
    1 billón de claves en 3 niveles con B=1000.
    La raíz y nivel 1 caben en RAM → 1-2 I/Os reales por query.

  Regla 39: B-Tree optimiza reads. LSM-Tree optimiza writes.
    B-Tree: update in-place. O(log_B n) reads, O(log_B n) writes.
    LSM-Tree (Cap.10): append-only. Writes son sequenciales (rápidos).
    Reads requieren buscar en múltiples niveles (más lentos).
    → B-Tree para read-heavy. LSM-Tree para write-heavy.

  Regla 40: Copy-on-write B-Tree = MVCC sin locks.
    Cada write crea nuevas páginas. Readers ven snapshots.
    Sin WAL necesario. Recovery trivial.
    Costo: write amplification. Un writer a la vez.
    Usado por: LMDB, BoltDB (etcd/Kubernetes).

  El B-Tree es la estructura más desplegada del mundo.
  Cada base de datos relacional, cada filesystem moderno,
  cada key-value store basado en disco usa alguna variante de B-Tree.
  Entender el B-Tree es entender cómo funciona el almacenamiento.
```

**Preguntas:**

1. ¿Las 40 reglas cubren fundamentos + hashing + árboles completos?

2. ¿La Regla 36 es la más fundamental del capítulo?

3. ¿La Regla 39 prepara el terreno para el Cap.10 (LSM-Trees)?

4. ¿Para un data engineer que usa Spark + PostgreSQL + Kafka,
   ¿cuáles de estas 40 reglas son las más relevantes?

5. ¿El Cap.10 (LSM-Trees) complementa o compite con el B-Tree?
