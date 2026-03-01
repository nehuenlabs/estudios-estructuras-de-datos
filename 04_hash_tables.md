# Guía de Ejercicios — Cap.04: Hash Tables: de la Función Hash a la Implementación

> Si pudieras elegir una sola estructura de datos para el resto de tu carrera,
> sería el hash map.
>
> Más del 50% de los accesos a datos en producción
> pasan por alguna variante de hash table.
> Cada vez que usas un diccionario en Python, un HashMap en Java,
> un map en Go, o un HashMap en Rust, estás usando un hash table.
>
> Spark particiona datos entre nodos usando hash partitioning.
> Kafka asigna mensajes a particiones con un hash de la key.
> Las bases de datos usan hash joins para combinar tablas.
> Los caches (Redis, Memcached) son hash tables distribuidos.
>
> Pero la mayoría de los programadores no saben cómo funciona
> un hash table por dentro. Saben que es O(1) y que usa una
> "función hash". Nada más.
>
> Este capítulo desmonta el hash table pieza por pieza:
> la función hash, las colisiones, el load factor, el rehashing.
> Y después lo reconstruye desde cero en 4 lenguajes.
>
> Al final, sabrás por qué Java HashMap usa chaining con TreeBins,
> por qué Rust HashMap usa Robin Hood hashing con SwissTable,
> y por qué esa diferencia importa para el rendimiento.

---

## El modelo mental: de clave a posición en un array

```
Un hash table es un array con un truco:
en vez de acceder por posición (arr[5]),
accedemos por clave (map["user_123"]).

El truco es la FUNCIÓN HASH:
  clave → hash(clave) → índice en el array

  hash("user_123") = 7429301847
  índice = 7429301847 % array.length = 47
  → El valor está en array[47].

  Costo: calcular hash + acceso al array = O(1).

Pero hay un problema:
  hash("user_123") % 100 = 47
  hash("user_456") % 100 = 47   ← COLISIÓN

  Dos claves distintas pueden dar el mismo índice.
  Esto es inevitable (birthday paradox).
  La pregunta no es SI habrá colisiones,
  sino CÓMO las resolvemos.

Dos familias de soluciones:

  SEPARATE CHAINING:
  Cada posición del array tiene una lista de (clave, valor).
  Si hay colisión, se agrega a la lista.

  array[47] → [(user_123, data1), (user_456, data2)]

  Lookup: ir a array[47], recorrer la lista hasta encontrar la clave.
  Peor caso: todas las claves colisionan → O(n).
  Caso promedio (load factor ≤ 0.75): O(1).

  OPEN ADDRESSING:
  Si la posición está ocupada, buscar la siguiente libre.

  array[47] = (user_123, data1)  ← ocupado
  array[48] = (user_456, data2)  ← siguiente libre

  Lookup: ir a array[47], si no es mi clave, ir a array[48], etc.
  Ventaja: sin linked lists → mejor cache locality.
  Desventaja: clustering (grupos de posiciones consecutivas ocupadas).

Decisiones de la industria:
  Java HashMap:       separate chaining (+ TreeBins para chains largas)
  Python dict:        open addressing (variante de linear probing)
  Go map:             separate chaining (con buckets de 8 slots)
  Rust HashMap:       open addressing (SwissTable / hashbrown)
  C++ unordered_map:  separate chaining (estándar requiere iteradores estables)
```

---

## Tabla de contenidos

- [Sección 4.1 — Funciones hash: de datos a números](#sección-41--funciones-hash-de-datos-a-números)
- [Sección 4.2 — Colisiones: el birthday paradox y por qué son inevitables](#sección-42--colisiones-el-birthday-paradox-y-por-qué-son-inevitables)
- [Sección 4.3 — Separate chaining: la solución clásica](#sección-43--separate-chaining-la-solución-clásica)
- [Sección 4.4 — Implementar un hash map con chaining desde cero](#sección-44--implementar-un-hash-map-con-chaining-desde-cero)
- [Sección 4.5 — Open addressing: linear probing y sus problemas](#sección-45--open-addressing-linear-probing-y-sus-problemas)
- [Sección 4.6 — Robin Hood hashing: ecualizando la miseria](#sección-46--robin-hood-hashing-ecualizando-la-miseria)
- [Sección 4.7 — Load factor, rehashing, y el benchmark final](#sección-47--load-factor-rehashing-y-el-benchmark-final)

---

## Sección 4.1 — Funciones Hash: de Datos a Números

### Ejercicio 4.1.1 — Leer: qué es una función hash y qué propiedades necesita

**Tipo: Leer**

```
Una función hash mapea datos de tamaño arbitrario
a un número de tamaño fijo.

  hash("hello")        = 0x2CF24DBA   (32 bits)
  hash("hello world")  = 0xB94D27B9   (32 bits)
  hash("")             = 0xE3B0C442   (32 bits)

Propiedades de una BUENA función hash para hash tables:

  1. DETERMINISMO
     La misma entrada siempre produce la misma salida.
     hash("hello") siempre retorna el mismo número.
     (Parece obvio, pero hashCode() de Java para Object
     retorna la dirección de memoria, que cambia entre ejecuciones.)

  2. DISTRIBUCIÓN UNIFORME
     Los hashes deben repartirse uniformemente sobre el rango de salida.
     Si hash("a")=0, hash("b")=1, hash("c")=2, ..., hash("z")=25
     eso es distribución perfecta.
     Si hash("a")=0, hash("b")=0, hash("c")=0, ... (todo en 0)
     eso es distribución pésima → todas las claves colisionan.

  3. AVALANCHE EFFECT
     Un cambio mínimo en la entrada produce un cambio masivo en la salida.
     hash("hello")  = 0x2CF24DBA
     hash("hellp")  = 0xE9C07F8B  ← totalmente diferente
     Si hash("hello") y hash("hellp") fueran similares,
     claves parecidas caerían en los mismos buckets → clustering.

  4. VELOCIDAD
     Para hash tables, necesitamos calcular hashes millones de veces/segundo.
     Un hash criptográfico (SHA-256) es seguro pero lento (~200 ns).
     Un hash no-criptográfico (xxHash) es rápido (~5 ns) pero no seguro.
     Para hash tables, la velocidad importa más que la seguridad.
     EXCEPTO cuando hay riesgo de HashDoS (ataques de colisión deliberada).

  NO necesita:
  - Irreversibilidad (eso es para criptografía)
  - Resistencia a colisiones criptográficas
  - Tamaño de salida grande (32-64 bits es suficiente)
```

**Preguntas:**

1. ¿Una función hash que retorna `input.length()` cumple
   con determinismo? ¿Con distribución uniforme?

2. ¿`Object.hashCode()` de Java retorna la dirección de memoria?
   ¿Eso cambia entre ejecuciones?

3. ¿El avalanche effect perfecto significa que cambiar 1 bit del input
   cambia ~50% de los bits del output?

4. ¿SHA-256 se puede usar como hash function para un hash map?
   ¿Sería buena idea?

5. ¿HashDoS es un ataque real? ¿Contra qué sistemas?

---

### Ejercicio 4.1.2 — Analizar: las funciones hash de cada lenguaje

**Tipo: Analizar**

```
Cada lenguaje elige una función hash distinta.
La elección refleja prioridades distintas.

  JAVA — hashCode()
    Cada objeto implementa hashCode() que retorna un int (32 bits).
    String.hashCode(): s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
    Multiplicar por 31 (primo): buena distribución, el JIT lo optimiza
    a (val << 5) - val (shift + resta, sin multiplicación real).
    HashMap mezcla adicionalmente: hash ^= (hash >>> 16)
    para distribuir los bits altos hacia los bajos.
    
    Pros: rápido, bien entendido, estable entre versiones.
    Contras: determinista → vulnerable a HashDoS.
             String hashCode es débil para claves con prefijos comunes.

  PYTHON — SipHash-2-4
    Desde Python 3.3, usa SipHash (keyed hash, RFC 7539).
    La key se genera aleatoriamente al inicio del proceso.
    → hash("hello") retorna un valor DIFERENTE cada vez que ejecutas Python.
    → Inmune a HashDoS (el atacante no conoce la key).
    
    Pros: seguro contra HashDoS.
    Contras: más lento que hashes no-keyed (~10 ns vs ~5 ns).

  GO — runtime hash
    Go usa una hash function específica por tipo (aeshash, memhash).
    En CPUs con AES-NI, usa instrucciones AES del hardware.
    La semilla es aleatoria por mapa.
    → Inmune a HashDoS.
    
    Pros: muy rápido (usa instrucciones de hardware), seguro.
    Contras: no exportada (no la puedes usar directamente).

  RUST — SipHash-1-3 (default) → ahash / SwissTable
    El trait Hash define cómo se hashea un tipo.
    El default Hasher es SipHash-1-3 (seguro contra HashDoS).
    Pero la crate hashbrown (usada por std desde 1.36) internamente
    usa ahash: un hash rápido basado en AES-NI si está disponible.
    → Seguridad por defecto + rendimiento con ahash.

    Pros: seguro Y rápido.
    Contras: la combinación SipHash + ahash es confusa al principio.

  FUNCIONES HASH NO-CRIPTOGRÁFICAS POPULARES:
    xxHash (Yann Collet):  ~5 GB/s. Usado en LZ4, Kafka (CRC32C).
    Murmur3:               ~3 GB/s. Usado en Spark, Cassandra, Guava.
    CityHash (Google):     ~5 GB/s. Optimizado para strings cortas.
    FNV-1a:                ~1 GB/s. Simple. Bueno para hash tables pequeñas.
    wyhash:                ~10 GB/s. Extremadamente rápido. Usado en Go (parcialmente).
```

**Preguntas:**

1. ¿Por qué Java no usa SipHash como Python y Rust?
   ¿Cuándo cambiaría?

2. ¿Spark usa Murmur3 para hash partitioning. ¿Es la mejor opción?

3. ¿`hash("hello")` en Python retorna valores distintos en cada ejecución?
   ¿Eso rompe algo?

4. ¿AES-NI en Go — ¿es una instrucción de hardware?
   ¿Qué pasa en CPUs sin AES-NI?

5. ¿Un hash de 32 bits es suficiente para un hash table de 1 billón de entries?

---

### Ejercicio 4.1.3 — Implementar: hash functions desde cero

**Tipo: Implementar**

```python
# Implementar 3 hash functions y comparar distribución

def hash_naive(key: str) -> int:
    """Suma de bytes. Distribución pésima."""
    return sum(key.encode('utf-8'))

def hash_djb2(key: str) -> int:
    """DJB2 de Daniel J. Bernstein. Clásica. Razonable."""
    h = 5381
    for c in key.encode('utf-8'):
        h = ((h << 5) + h + c) & 0xFFFFFFFF  # h * 33 + c
    return h

def hash_fnv1a(key: str) -> int:
    """FNV-1a. Buena distribución. Usada en muchas hash tables."""
    FNV_OFFSET = 0x811C9DC5
    FNV_PRIME = 0x01000193
    h = FNV_OFFSET
    for c in key.encode('utf-8'):
        h ^= c
        h = (h * FNV_PRIME) & 0xFFFFFFFF
    return h

# Test: distribución sobre 16 buckets
def test_distribution(hash_func, keys, num_buckets=16):
    buckets = [0] * num_buckets
    for key in keys:
        idx = hash_func(key) % num_buckets
        buckets[idx] += 1
    return buckets

# Generar claves
keys_sequential = [f"user_{i}" for i in range(10000)]
keys_similar = [f"item_{i:05d}" for i in range(10000)]

for name, func in [("naive", hash_naive), ("djb2", hash_djb2), ("fnv1a", hash_fnv1a)]:
    dist = test_distribution(func, keys_sequential)
    variance = sum((x - 625) ** 2 for x in dist) / 16  # ideal = 10000/16 = 625
    print(f"{name:8s} sequential: variance={variance:10.0f}  min={min(dist)}  max={max(dist)}")

    dist = test_distribution(func, keys_similar)
    variance = sum((x - 625) ** 2 for x in dist) / 16
    print(f"{name:8s} similar:    variance={variance:10.0f}  min={min(dist)}  max={max(dist)}")
    print()

# Resultado esperado:
# naive    sequential: variance=  ~500000  min=0    max=~3000  ← TERRIBLE
# naive    similar:    variance=  ~500000  min=0    max=~3000
# djb2     sequential: variance=     ~500  min=~580  max=~670  ← OK
# djb2     similar:    variance=     ~800  min=~550  max=~700
# fnv1a    sequential: variance=     ~300  min=~590  max=~660  ← MEJOR
# fnv1a    similar:    variance=     ~400  min=~580  max=~670
```

```java
// Java: implementar y comparar con String.hashCode()

public class HashFunctions {

    static int hashNaive(String key) {
        int h = 0;
        for (int i = 0; i < key.length(); i++) h += key.charAt(i);
        return h & 0x7FFFFFFF; // positive
    }

    static int hashDJB2(String key) {
        int h = 5381;
        for (int i = 0; i < key.length(); i++) {
            h = ((h << 5) + h) + key.charAt(i);
        }
        return h & 0x7FFFFFFF;
    }

    static int hashFNV1a(String key) {
        int h = 0x811C9DC5;
        for (int i = 0; i < key.length(); i++) {
            h ^= key.charAt(i);
            h *= 0x01000193;
        }
        return h & 0x7FFFFFFF;
    }

    // HashMap de Java mezcla el hash para mejorar distribución
    static int hashMapSpread(int h) {
        return h ^ (h >>> 16);
    }

    public static void main(String[] args) {
        String[] keys = new String[100_000];
        for (int i = 0; i < keys.length; i++) keys[i] = "user_" + i;

        int buckets = 1024;
        int[][] counts = new int[4][buckets];
        String[] names = {"naive", "djb2", "fnv1a", "String.hashCode"};

        for (String key : keys) {
            counts[0][hashNaive(key) % buckets]++;
            counts[1][hashDJB2(key) % buckets]++;
            counts[2][hashFNV1a(key) % buckets]++;
            counts[3][(key.hashCode() & 0x7FFFFFFF) % buckets]++;
        }

        double ideal = 100_000.0 / buckets;
        for (int f = 0; f < 4; f++) {
            int min = Integer.MAX_VALUE, max = 0;
            double variance = 0;
            for (int c : counts[f]) {
                min = Math.min(min, c);
                max = Math.max(max, c);
                variance += (c - ideal) * (c - ideal);
            }
            variance /= buckets;
            System.out.printf("%-18s variance=%8.0f  min=%3d  max=%3d  (ideal=%.0f)%n",
                names[f], variance, min, max, ideal);
        }
    }
}
```

**Preguntas:**

1. ¿Por qué el hash naive tiene varianza tan alta?

2. ¿DJB2 usa `h * 33 + c`. ¿Por qué 33? ¿Qué tiene de especial?

3. ¿FNV-1a hace XOR primero y luego multiplica. ¿Importa el orden?

4. ¿`hash ^ (hash >>> 16)` de HashMap mezcla bits altos y bajos.
   ¿Por qué es necesario?

5. ¿Cuántos nanosegundos tarda cada función para un string de 20 caracteres?

---

### Ejercicio 4.1.4 — Implementar: medir velocidad de hash functions

**Tipo: Implementar**

```rust
// Rust: benchmark de hash functions
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;
use std::time::Instant;

// FNV-1a manual
fn fnv1a(data: &[u8]) -> u64 {
    let mut h: u64 = 0xcbf29ce484222325;
    for &b in data {
        h ^= b as u64;
        h = h.wrapping_mul(0x100000001b3);
    }
    h
}

// Usar el default hasher de Rust (SipHash)
fn siphash(data: &[u8]) -> u64 {
    let mut hasher = DefaultHasher::new();
    data.hash(&mut hasher);
    hasher.finish()
}

// Hash trivial (suma)
fn hash_sum(data: &[u8]) -> u64 {
    data.iter().map(|&b| b as u64).sum()
}

fn benchmark_hash(name: &str, data: &[Vec<u8>], hash_fn: fn(&[u8]) -> u64) {
    // Warmup
    for d in data.iter().take(1000) { std::hint::black_box(hash_fn(d)); }

    let start = Instant::now();
    let mut sink = 0u64;
    for d in data {
        sink = sink.wrapping_add(hash_fn(d));
    }
    let elapsed = start.elapsed();
    let ns_per_hash = elapsed.as_nanos() as f64 / data.len() as f64;
    println!("  {:<20} {:>6.1} ns/hash", name, ns_per_hash);
    std::hint::black_box(sink);
}

fn main() {
    let n = 1_000_000;

    // Strings de longitud 8
    let short: Vec<Vec<u8>> = (0..n).map(|i| format!("key_{:04}", i).into_bytes()).collect();
    // Strings de longitud 64
    let medium: Vec<Vec<u8>> = (0..n).map(|i| format!("{:064}", i).into_bytes()).collect();
    // Strings de longitud 256
    let long: Vec<Vec<u8>> = (0..n).map(|i| format!("{:0256}", i).into_bytes()).collect();

    for (label, data) in [("8 bytes", &short), ("64 bytes", &medium), ("256 bytes", &long)] {
        println!("Hash benchmark ({label}, {n} keys):");
        benchmark_hash("sum (naive)", data, hash_sum);
        benchmark_hash("FNV-1a", data, fnv1a);
        benchmark_hash("SipHash (default)", data, siphash);
        println!();
    }
}
// Resultado esperado (varía por hardware):
// Hash benchmark (8 bytes):
//   sum (naive)           1.5 ns/hash
//   FNV-1a                4.0 ns/hash
//   SipHash (default)     8.0 ns/hash
//
// Hash benchmark (64 bytes):
//   sum (naive)           8.0 ns/hash
//   FNV-1a               18.0 ns/hash
//   SipHash (default)    20.0 ns/hash
//
// Hash benchmark (256 bytes):
//   sum (naive)          30.0 ns/hash
//   FNV-1a               65.0 ns/hash
//   SipHash (default)    55.0 ns/hash
```

**Preguntas:**

1. ¿SipHash es más rápido que FNV-1a para strings largos?
   ¿Por qué?

2. ¿El hash naive (suma) es más rápido. ¿Cuándo importa la velocidad
   más que la distribución?

3. ¿xxHash sería más rápido que SipHash aquí?

4. ¿El costo del hash (5-20 ns) es significativo comparado
   con el costo del lookup (28 ns en un HashMap)?

5. ¿Para claves numéricas (i64), ¿necesitas una función hash
   o basta con usar el valor directamente?

---

### Ejercicio 4.1.5 — Analizar: el identity hash y cuándo funciona

**Tipo: Analizar**

```
Para claves numéricas, puedes usar el VALOR como hash directamente.

  hash(42) = 42
  hash(1000000) = 1000000
  index = hash % table_size

  ¿Funciona?
  Depende de la distribución de tus claves.

  CASO 1: claves secuenciales (1, 2, 3, ..., 1000)
    hash(i) % 1024 = i (para i < 1024)
    Distribución perfecta. El identity hash funciona.

  CASO 2: claves con patrón (0, 1024, 2048, 3072, ...)
    hash(i) % 1024 = 0 para todos.
    Todas las claves colisionan. Desastre.

  CASO 3: claves aleatorias
    Distribución buena en general.
    Pero si table_size es potencia de 2 y las claves tienen
    patrones en los bits bajos, puedes tener problemas.

  Solución: mezclar los bits (bit mixing / scrambling)
    hash(key) = key ^ (key >>> 16) — como hace Java HashMap
    O más agresivo: Fibonacci hashing
    hash(key) = (key * 0x9E3779B97F4A7C15) >>> shift
    La constante es floor(2^64 / φ) donde φ es el golden ratio.
    → Esto convierte patrones regulares en distribución uniforme.

  Fibonacci hashing es usado en:
  - Python dict (parcialmente)
  - Algunos allocators de memoria
  - Hash tables experimentales

  Regla: para claves numéricas, no uses el valor directamente.
  Siempre mezcla los bits, al menos con XOR-shift.
```

**Preguntas:**

1. ¿Java Integer.hashCode() retorna el valor int sin modificar?
   ¿Eso es un identity hash?

2. ¿Fibonacci hashing con el golden ratio — ¿por qué funciona?

3. ¿Si table_size es primo (no potencia de 2),
   ¿el identity hash funciona mejor?

4. ¿Rust hashea `i64` con SipHash por defecto.
   ¿Es innecesariamente costoso?

5. ¿Para hash partitioning en Spark, ¿qué pasa si la clave
   tiene patrones regulares?

---

## Sección 4.2 — Colisiones: el Birthday Paradox y Por Qué Son Inevitables

### Ejercicio 4.2.1 — Leer: el birthday paradox aplicado a hash tables

**Tipo: Leer**

```
El birthday paradox dice:
  En un grupo de 23 personas, hay un 50% de probabilidad
  de que dos compartan cumpleaños (365 días posibles).
  Con 70 personas, la probabilidad sube a 99.9%.

Aplicado a hash tables:
  Con un hash de 32 bits (4 mil millones de valores posibles),
  ¿cuántas claves necesitas para tener una colisión con 50% de probabilidad?

  Respuesta: ~77,000 claves.

  Fórmula: n ≈ √(2 × m × p) donde m = rango del hash, p = probabilidad.
  n ≈ √(2 × 2³² × 0.5) ≈ √(4.3 × 10⁹) ≈ 65,536

  Para un hash table con m=1024 buckets:
  n ≈ √(2 × 1024 × 0.5) ≈ 32 claves para 50% de colisión.
  Con 100 claves en 1024 buckets: ~99% de probabilidad de colisión.

  Conclusión: las colisiones NO son un caso raro.
  Son lo NORMAL en cualquier hash table con más de √m elementos.

  Implicaciones para diseño:
  1. SIEMPRE necesitas una estrategia de resolución de colisiones.
  2. El load factor (n/m) controla cuántas colisiones hay.
  3. Un hash perfecto (sin colisiones) solo es posible si
     conoces TODAS las claves de antemano (perfect hashing).
```

**Preguntas:**

1. ¿Con 50,000 claves en un hash table de 1M buckets,
   ¿cuántas colisiones esperas?

2. ¿Un hash de 64 bits tiene colisiones más tarde que uno de 32 bits?
   ¿Cuánto más tarde?

3. ¿Perfect hashing (gperf, CMPH) es viable para claves conocidas
   a priori?

4. ¿El birthday paradox aplica a Bloom filters?

5. ¿Un load factor de 0.75 (Java default) — ¿cuál es la longitud
   promedio de cada chain?

---

### Ejercicio 4.2.2 — Implementar: simular colisiones empíricamente

**Tipo: Implementar**

```java
// Simular colisiones para diferentes tamaños de tabla y número de claves

public class CollisionSimulator {

    static int countCollisions(int numKeys, int tableSize, long seed) {
        var rng = new java.util.Random(seed);
        boolean[] occupied = new boolean[tableSize];
        int collisions = 0;

        for (int i = 0; i < numKeys; i++) {
            int hash = rng.nextInt() & 0x7FFFFFFF;
            int index = hash % tableSize;
            if (occupied[index]) {
                collisions++;
            } else {
                occupied[index] = true;
            }
        }
        return collisions;
    }

    public static void main(String[] args) {
        int[] tableSizes = {1024, 4096, 16384, 65536};
        double[] loadFactors = {0.25, 0.50, 0.75, 0.90, 1.00};

        System.out.printf("%-12s", "Table Size");
        for (double lf : loadFactors) System.out.printf("  LF=%.2f", lf);
        System.out.println();
        System.out.println("─".repeat(70));

        for (int tableSize : tableSizes) {
            System.out.printf("%-12d", tableSize);
            for (double lf : loadFactors) {
                int numKeys = (int)(tableSize * lf);
                // Promedio sobre 100 simulaciones
                long totalCollisions = 0;
                for (int trial = 0; trial < 100; trial++) {
                    totalCollisions += countCollisions(numKeys, tableSize, trial);
                }
                double avgCollisions = totalCollisions / 100.0;
                double collisionRate = avgCollisions / numKeys * 100;
                System.out.printf("  %5.1f%%  ", collisionRate);
            }
            System.out.println();
        }
    }
}
// Resultado esperado:
// Table Size  LF=0.25  LF=0.50  LF=0.75  LF=0.90  LF=1.00
// ──────────────────────────────────────────────────────────────
// 1024        11.7%    21.3%    26.4%    28.4%    36.8%
// 4096        11.8%    21.4%    26.5%    28.5%    36.8%
// 16384       11.8%    21.3%    26.4%    28.5%    36.8%
// 65536       11.8%    21.3%    26.5%    28.5%    36.8%
//
// Observar: el collision rate depende SOLO del load factor,
// no del tamaño absoluto de la tabla.
// Con LF=0.75, ~26% de los inserts causan colisión.
```

**Preguntas:**

1. ¿26% de collision rate con LF=0.75 — ¿es aceptable?

2. ¿El collision rate no depende del tamaño de la tabla?
   ¿Solo del load factor?

3. ¿Un hash perfecto tendría 0% de collision rate?

4. ¿Si la hash function tiene mala distribución,
   ¿el collision rate sube?

5. ¿En un HashMap de Java con 1M entries y LF=0.75,
   ¿cuántas chains tienen longitud > 1?

---

### Ejercicio 4.2.3 — Analizar: expected chain length con chaining

**Tipo: Analizar**

```
Con separate chaining, la longitud esperada de cada chain
sigue una distribución de Poisson con λ = load factor (α).

  Load factor α = n/m  (n claves, m buckets)

  Probabilidad de que un bucket tenga exactamente k elementos:
  P(k) = (α^k × e^(-α)) / k!

  Para α = 0.75:
    P(0) = e^(-0.75) = 0.472  → 47% de buckets vacíos
    P(1) = 0.75 × e^(-0.75) = 0.354  → 35% con 1 elemento
    P(2) = 0.75² × e^(-0.75) / 2 = 0.133  → 13% con 2 elementos
    P(3) = 0.75³ × e^(-0.75) / 6 = 0.033  → 3.3% con 3 elementos
    P(4) = 0.006  → 0.6% con 4 elementos
    P(≥5) < 0.1%

  Impacto en rendimiento:
    Lookup exitoso promedio: 1 + α/2 comparaciones = 1.375
    Lookup fallido promedio: e^(-α) + α = ~1.22 (buckets vacíos + chain scan)

  Para α = 0.75, un lookup típico hace 1-2 comparaciones.
  Eso es efectivamente O(1), con constante ~1.4.

  Pero cada comparación puede ser un cache miss
  si la chain está en una linked list (nodos dispersos en heap).
  1.4 comparaciones × 100 ns/cache miss = 140 ns worst case.

  Por eso Java 8 convierte chains largas (≥8) en Red-Black trees:
  una chain de 8 en linked list: 8 cache misses.
  un Red-Black tree de 8: log₂(8) = 3 comparaciones máximo.
  (Y con peor cache locality, pero para chains >8 el tree gana.)

  Probabilidad de necesitar un tree (chain ≥ 8) con α=0.75:
  P(≥8) < 0.000006 → 6 en un millón de buckets.
  → Los TreeBins de Java casi nunca se activan.
  → Existen como protección contra HashDoS.
```

**Preguntas:**

1. ¿47% de buckets vacíos con LF=0.75 — ¿es desperdicio de memoria?

2. ¿Los TreeBins de Java 8 casi nunca se activan.
   ¿Entonces por qué existen?

3. ¿Si subieras el LF a 0.95, ¿cuál sería la chain length promedio?

4. ¿La distribución de Poisson asume hash function perfecta.
   ¿Qué pasa si la distribución es mala?

5. ¿Go map usa buckets de 8 slots en vez de linked lists.
   ¿Eso mejora la locality?

---

## Sección 4.3 — Separate Chaining: la Solución Clásica

### Ejercicio 4.3.1 — Leer: cómo funciona el chaining paso a paso

**Tipo: Leer**

```
Separate chaining: cada bucket es una lista de (key, value).

  PUT(key, value):
    1. index = hash(key) % table.length
    2. Buscar key en table[index] (recorrer la chain)
    3. Si key existe → actualizar value
    4. Si key no existe → agregar (key, value) al inicio de la chain
    5. Si load_factor > threshold → rehash (duplicar tabla)

  GET(key):
    1. index = hash(key) % table.length
    2. Buscar key en table[index] (recorrer la chain)
    3. Si key existe → return value
    4. Si key no existe → return null

  REMOVE(key):
    1. index = hash(key) % table.length
    2. Buscar key en table[index]
    3. Si key existe → desenlazar nodo de la chain, return value
    4. Si key no existe → return null

  Visualización:

  table: [  0  ] → null
         [  1  ] → (key_a, val_a) → (key_b, val_b) → null
         [  2  ] → null
         [  3  ] → (key_c, val_c) → null
         [  4  ] → (key_d, val_d) → (key_e, val_e) → (key_f, val_f) → null
         [  5  ] → null
         [  6  ] → (key_g, val_g) → null
         [  7  ] → null

  size = 7,  capacity = 8,  load factor = 7/8 = 0.875
  chains: [0, 2, 0, 1, 3, 0, 1, 0]
  max chain: 3 (bucket 4)

  REHASH cuando load factor > 0.75:
    1. Crear nueva tabla de tamaño 2× el actual
    2. Para cada (key, value) en la tabla vieja:
       nuevo_index = hash(key) % nueva_tabla.length
       Agregar a nueva_tabla[nuevo_index]
    3. Reemplazar tabla vieja por nueva tabla

  Costo del rehash: O(n) — recorrer todos los elementos.
  Amortizado sobre n inserts: O(1) por insert (mismo análisis Cap.02).
```

**Preguntas:**

1. ¿Agregar al inicio de la chain (prepend) vs al final (append):
   ¿cuál es mejor? ¿Por qué?

2. ¿El rehash recalcula todos los hashes o puede reutilizarlos?

3. ¿Si duplicas la capacidad (potencia de 2), cada elemento
   va al bucket original o al bucket + vieja_capacidad.
   ¿Por qué?

4. ¿El rehash es O(n). ¿Puede causar un spike de latencia
   en producción?

5. ¿Java HashMap almacena el hash con cada entry para evitar
   recalcularlo durante rehash?

---

### Ejercicio 4.3.2 — Implementar: entry node para separate chaining

**Tipo: Implementar**

```java
// El nodo de la linked list que forma cada chain

public class Entry<K, V> {
    final K key;
    V value;
    final int hash;  // almacenar hash calculado (evitar recalcular)
    Entry<K, V> next;

    Entry(K key, V value, int hash, Entry<K, V> next) {
        this.key = key;
        this.value = value;
        this.hash = hash;
        this.next = next;
    }

    @Override
    public String toString() {
        return key + "=" + value;
    }
}

// Tamaño en memoria (Java 64-bit, compressed oops):
//   Object header:  12 bytes
//   key ref:         4 bytes (compressed oop)
//   value ref:       4 bytes
//   hash:            4 bytes
//   next ref:        4 bytes
//   Padding:         4 bytes (alinear a 8)
//   Total:          32 bytes PER ENTRY
//
//   + el objeto K (ej: Integer = 16 bytes)
//   + el objeto V (ej: Integer = 16 bytes)
//   Total real: ~64 bytes per (Integer, Integer) entry
//
// Comparar con un array de pares:
//   int key: 4 bytes + int value: 4 bytes = 8 bytes per entry
//   Ratio: 64 / 8 = 8× más memoria con chaining.
```

**Preguntas:**

1. ¿32 bytes por Entry + objetos key/value. ¿Es demasiado overhead?

2. ¿Almacenar `hash` en el Entry (4 bytes extra)
   vale la pena para evitar recalcular?

3. ¿`final K key` — ¿por qué la key es final?

4. ¿Si K es `String`, ¿cuántos bytes ocupa una Entry en total?

5. ¿En Rust, ¿una Entry equivalente tiene el mismo overhead?

---

## Sección 4.4 — Implementar un Hash Map con Chaining Desde Cero

### Ejercicio 4.4.1 — Implementar: hash map completo en Java

**Tipo: Implementar**

```java
// HashMap con separate chaining — implementación completa

public class MiHashMap<K, V> {

    private Entry<K, V>[] table;
    private int size;
    private int capacity;
    private static final int DEFAULT_CAPACITY = 16;
    private static final float LOAD_FACTOR = 0.75f;

    @SuppressWarnings("unchecked")
    public MiHashMap() {
        this.capacity = DEFAULT_CAPACITY;
        this.table = new Entry[capacity];
        this.size = 0;
    }

    @SuppressWarnings("unchecked")
    public MiHashMap(int initialCapacity) {
        this.capacity = nextPowerOf2(Math.max(initialCapacity, 4));
        this.table = new Entry[capacity];
        this.size = 0;
    }

    // ═══ Hash function ═══

    private int hash(K key) {
        if (key == null) return 0;
        int h = key.hashCode();
        return h ^ (h >>> 16); // spread high bits into low bits
    }

    private int indexFor(int hash) {
        return hash & (capacity - 1); // equivalent to hash % capacity for power-of-2
    }

    // ═══ Core operations ═══

    public V put(K key, V value) {
        int h = hash(key);
        int index = indexFor(h);

        // Buscar si la key ya existe
        for (Entry<K, V> e = table[index]; e != null; e = e.next) {
            if (e.hash == h && (e.key == key || (key != null && key.equals(e.key)))) {
                V oldValue = e.value;
                e.value = value;
                return oldValue; // updated existing
            }
        }

        // Key no existe: agregar al inicio de la chain
        table[index] = new Entry<>(key, value, h, table[index]);
        size++;

        if (size > capacity * LOAD_FACTOR) {
            resize();
        }

        return null; // new entry
    }

    public V get(K key) {
        int h = hash(key);
        int index = indexFor(h);

        for (Entry<K, V> e = table[index]; e != null; e = e.next) {
            if (e.hash == h && (e.key == key || (key != null && key.equals(e.key)))) {
                return e.value;
            }
        }
        return null;
    }

    public V remove(K key) {
        int h = hash(key);
        int index = indexFor(h);

        Entry<K, V> prev = null;
        Entry<K, V> curr = table[index];

        while (curr != null) {
            if (curr.hash == h && (curr.key == key || (key != null && key.equals(curr.key)))) {
                if (prev == null) {
                    table[index] = curr.next;
                } else {
                    prev.next = curr.next;
                }
                size--;
                return curr.value;
            }
            prev = curr;
            curr = curr.next;
        }
        return null;
    }

    public boolean containsKey(K key) {
        return get(key) != null;
    }

    public int size() { return size; }
    public boolean isEmpty() { return size == 0; }

    // ═══ Resize ═══

    @SuppressWarnings("unchecked")
    private void resize() {
        int newCapacity = capacity * 2;
        Entry<K, V>[] newTable = new Entry[newCapacity];

        // Rehash: redistribuir todos los entries
        for (int i = 0; i < capacity; i++) {
            Entry<K, V> e = table[i];
            while (e != null) {
                Entry<K, V> next = e.next;
                int newIndex = e.hash & (newCapacity - 1);
                e.next = newTable[newIndex];
                newTable[newIndex] = e;
                e = next;
            }
        }

        table = newTable;
        capacity = newCapacity;
    }

    // ═══ Utility ═══

    private static int nextPowerOf2(int n) {
        int p = 1;
        while (p < n) p <<= 1;
        return p;
    }

    // Estadísticas de distribución
    public String stats() {
        int maxChain = 0, empty = 0;
        int[] chainLengths = new int[capacity];
        for (int i = 0; i < capacity; i++) {
            int len = 0;
            for (Entry<K, V> e = table[i]; e != null; e = e.next) len++;
            chainLengths[i] = len;
            maxChain = Math.max(maxChain, len);
            if (len == 0) empty++;
        }
        return String.format("size=%d, capacity=%d, LF=%.2f, empty=%d (%.0f%%), maxChain=%d",
            size, capacity, (double) size / capacity,
            empty, (double) empty / capacity * 100, maxChain);
    }

    public static void main(String[] args) {
        var map = new MiHashMap<String, Integer>();

        // Insert
        for (int i = 0; i < 100_000; i++) {
            map.put("key_" + i, i);
        }
        System.out.println(map.stats());

        // Get
        System.out.println("get(key_42) = " + map.get("key_42"));
        System.out.println("get(key_99999) = " + map.get("key_99999"));
        System.out.println("get(nonexistent) = " + map.get("nonexistent"));

        // Remove
        map.remove("key_42");
        System.out.println("After remove: get(key_42) = " + map.get("key_42"));
        System.out.println("size = " + map.size());

        // Benchmark vs java.util.HashMap
        int n = 1_000_000;
        var ours = new MiHashMap<Integer, Integer>(n * 2);
        var jdk = new java.util.HashMap<Integer, Integer>(n * 2);

        long start = System.nanoTime();
        for (int i = 0; i < n; i++) ours.put(i, i);
        long oursInsert = System.nanoTime() - start;

        start = System.nanoTime();
        for (int i = 0; i < n; i++) jdk.put(i, i);
        long jdkInsert = System.nanoTime() - start;

        // Lookup benchmark
        long dummy = 0;
        for (int w = 0; w < 5; w++) {
            for (int i = 0; i < n; i++) dummy += ours.get(i);
            for (int i = 0; i < n; i++) dummy += jdk.get(i);
        }

        start = System.nanoTime();
        for (int i = 0; i < n; i++) dummy += ours.get(i);
        long oursGet = System.nanoTime() - start;

        start = System.nanoTime();
        for (int i = 0; i < n; i++) dummy += jdk.get(i);
        long jdkGet = System.nanoTime() - start;

        System.out.printf("%nBenchmark (%d entries):%n", n);
        System.out.printf("Insert:  ours=%d ms, JDK=%d ms%n", oursInsert / 1_000_000, jdkInsert / 1_000_000);
        System.out.printf("Get:     ours=%d ms, JDK=%d ms%n", oursGet / 1_000_000, jdkGet / 1_000_000);
        if (dummy == Long.MIN_VALUE) System.out.println(dummy);
    }
}
```

**Preguntas:**

1. ¿Nuestra implementación es competitiva con `java.util.HashMap`?
   ¿Qué nos falta?

2. ¿`e.hash == h` antes de `key.equals()` — ¿por qué comparar
   el hash primero?

3. ¿El resize mueve entries sin crear nuevos objetos Entry.
   ¿Eso es correcto?

4. ¿La tabla empieza en 16. ¿Para 100K entries, cuántos resizes ocurren?

5. ¿`null` como key está soportado. ¿Es una buena práctica?

---

### Ejercicio 4.4.2 — Implementar: hash map en Python

**Tipo: Implementar**

```python
# Hash map con separate chaining en Python
class HashMap:
    def __init__(self, initial_capacity=16, load_factor=0.75):
        self._capacity = initial_capacity
        self._load_factor = load_factor
        self._size = 0
        self._table = [None] * self._capacity  # array of linked lists

    def _hash(self, key):
        h = hash(key)  # Python's built-in hash (SipHash)
        return h ^ (h >> 16)

    def _index(self, h):
        return h & (self._capacity - 1)

    def put(self, key, value):
        h = self._hash(key)
        idx = self._index(h)

        # Search chain
        node = self._table[idx]
        while node is not None:
            if node[0] == key:
                old = node[2]
                node[2] = value  # update
                return old
            node = node[3]

        # Prepend new node: [key, hash, value, next]
        self._table[idx] = [key, h, value, self._table[idx]]
        self._size += 1

        if self._size > self._capacity * self._load_factor:
            self._resize()
        return None

    def get(self, key, default=None):
        h = self._hash(key)
        idx = self._index(h)
        node = self._table[idx]
        while node is not None:
            if node[0] == key:
                return node[2]
            node = node[3]
        return default

    def remove(self, key):
        h = self._hash(key)
        idx = self._index(h)
        prev = None
        node = self._table[idx]
        while node is not None:
            if node[0] == key:
                if prev is None:
                    self._table[idx] = node[3]
                else:
                    prev[3] = node[3]
                self._size -= 1
                return node[2]
            prev = node
            node = node[3]
        return None

    def _resize(self):
        old = self._table
        self._capacity *= 2
        self._table = [None] * self._capacity
        self._size = 0
        for chain in old:
            node = chain
            while node is not None:
                self.put(node[0], node[2])
                node = node[3]

    def __len__(self):
        return self._size

    def __contains__(self, key):
        return self.get(key) is not None

    def __setitem__(self, key, value):
        self.put(key, value)

    def __getitem__(self, key):
        result = self.get(key)
        if result is None and key not in self:
            raise KeyError(key)
        return result

# Test
m = HashMap()
for i in range(100_000):
    m[f"key_{i}"] = i

print(f"size = {len(m)}")
print(f"m['key_42'] = {m['key_42']}")
print(f"'key_99999' in m = {'key_99999' in m}")

# Benchmark vs built-in dict
import time

n = 500_000
ours = HashMap(n * 2)
builtin = {}

start = time.perf_counter_ns()
for i in range(n):
    ours.put(f"k{i}", i)
ours_time = time.perf_counter_ns() - start

start = time.perf_counter_ns()
for i in range(n):
    builtin[f"k{i}"] = i
builtin_time = time.perf_counter_ns() - start

print(f"\nInsert {n}: ours={ours_time // 1_000_000} ms, dict={builtin_time // 1_000_000} ms")
print(f"Ratio: {ours_time / builtin_time:.1f}×")
# Expected: ours is 3-5× slower than built-in dict
# (built-in dict is implemented in C with open addressing)
```

**Preguntas:**

1. ¿Nuestra implementación en Python es 3-5× más lenta que `dict`.
   ¿Por qué?

2. ¿`dict` de Python usa open addressing, no chaining. ¿Eso lo hace más rápido?

3. ¿Usar listas `[key, hash, value, next]` como nodos
   es una buena idea en Python?

4. ¿El `_resize` llama a `put` para cada elemento.
   ¿Eso puede causar resizes recursivos?

5. ¿Los operadores `__setitem__` y `__getitem__` permiten
   la sintaxis `m["key"] = value`. ¿Es idiomático?

---

### Ejercicio 4.4.3 — Implementar: hash map genérico en Go

**Tipo: Implementar**

```go
// Hash map con separate chaining en Go (generics)
package main

import (
    "fmt"
    "hash/fnv"
    "time"
)

type entry[K comparable, V any] struct {
    key   K
    value V
    hash  uint64
    next  *entry[K, V]
}

type HashMap[K comparable, V any] struct {
    table    []*entry[K, V]
    size     int
    capacity int
    hasher   func(K) uint64
}

func NewHashMap[K comparable, V any](initialCap int, hasher func(K) uint64) *HashMap[K, V] {
    cap := nextPow2(initialCap)
    return &HashMap[K, V]{
        table:    make([]*entry[K, V], cap),
        capacity: cap,
        hasher:   hasher,
    }
}

func (m *HashMap[K, V]) Put(key K, value V) (V, bool) {
    h := m.hasher(key)
    idx := int(h) & (m.capacity - 1)

    for e := m.table[idx]; e != nil; e = e.next {
        if e.hash == h && e.key == key {
            old := e.value
            e.value = value
            return old, true // updated
        }
    }

    m.table[idx] = &entry[K, V]{key, value, h, m.table[idx]}
    m.size++

    if float64(m.size) > float64(m.capacity)*0.75 {
        m.resize()
    }

    var zero V
    return zero, false // new
}

func (m *HashMap[K, V]) Get(key K) (V, bool) {
    h := m.hasher(key)
    idx := int(h) & (m.capacity - 1)

    for e := m.table[idx]; e != nil; e = e.next {
        if e.hash == h && e.key == key {
            return e.value, true
        }
    }
    var zero V
    return zero, false
}

func (m *HashMap[K, V]) Remove(key K) (V, bool) {
    h := m.hasher(key)
    idx := int(h) & (m.capacity - 1)

    var prev *entry[K, V]
    for e := m.table[idx]; e != nil; e = e.next {
        if e.hash == h && e.key == key {
            if prev == nil {
                m.table[idx] = e.next
            } else {
                prev.next = e.next
            }
            m.size--
            return e.value, true
        }
        prev = e
    }
    var zero V
    return zero, false
}

func (m *HashMap[K, V]) Size() int { return m.size }

func (m *HashMap[K, V]) resize() {
    newCap := m.capacity * 2
    newTable := make([]*entry[K, V], newCap)

    for i := 0; i < m.capacity; i++ {
        for e := m.table[i]; e != nil; {
            next := e.next
            newIdx := int(e.hash) & (newCap - 1)
            e.next = newTable[newIdx]
            newTable[newIdx] = e
            e = next
        }
    }
    m.table = newTable
    m.capacity = newCap
}

func nextPow2(n int) int {
    p := 1
    for p < n { p <<= 1 }
    return p
}

// String hasher using FNV-1a
func stringHasher(s string) uint64 {
    h := fnv.New64a()
    h.Write([]byte(s))
    return h.Sum64()
}

func main() {
    m := NewHashMap[string, int](16, stringHasher)

    n := 1_000_000
    start := time.Now()
    for i := 0; i < n; i++ {
        m.Put(fmt.Sprintf("key_%d", i), i)
    }
    insertTime := time.Since(start)

    v, ok := m.Get("key_42")
    fmt.Printf("Get(key_42) = %d, found=%v\n", v, ok)
    fmt.Printf("Size: %d\n", m.Size())
    fmt.Printf("Insert %d: %v (%.0f ns/op)\n", n, insertTime,
        float64(insertTime.Nanoseconds())/float64(n))

    // Benchmark vs built-in map
    builtin := make(map[string]int, n)
    start = time.Now()
    for i := 0; i < n; i++ {
        builtin[fmt.Sprintf("key_%d", i)] = i
    }
    builtinTime := time.Since(start)
    fmt.Printf("Built-in map insert: %v (%.0f ns/op)\n", builtinTime,
        float64(builtinTime.Nanoseconds())/float64(n))
}
```

**Preguntas:**

1. ¿Go generics (`[K comparable, V any]`) permiten hash maps
   sin `interface{}`. ¿Cuánto mejora el rendimiento?

2. ¿`hash/fnv` es la mejor opción para el hasher?

3. ¿El built-in `map` de Go es más rápido. ¿Cuánto y por qué?

4. ¿Nuestra implementación tiene el mismo layout de memoria
   que el built-in map?

5. ¿El built-in map de Go usa buckets de 8 slots.
   ¿Eso es mejor que linked list chaining?

---

### Ejercicio 4.4.4 — Analizar: Java HashMap internals — lo que no cuentan los libros

**Tipo: Analizar**

```
Java HashMap tiene optimizaciones que nuestra implementación no tiene:

  1. TREEIFICATION (Java 8+):
     Si una chain crece a ≥ 8 nodos Y la tabla tiene ≥ 64 buckets,
     la chain se convierte en un Red-Black tree.
     Esto convierte el peor caso de O(n) a O(log n).
     → Protección contra HashDoS.
     Si la chain se reduce a ≤ 6 nodos, vuelve a linked list.
     (Hysteresis: 8 para convertir, 6 para revertir, evitando thrashing.)

  2. HIGH BITS MIXING:
     hash ^= (hash >>> 16)
     Distribuye los bits altos hacia los bajos.
     Necesario porque indexFor usa mask (& capacity-1)
     que solo mira los bits BAJOS del hash.
     Sin mixing, claves con mismos bits bajos colisionan.

  3. POWER-OF-2 CAPACITY:
     Siempre potencia de 2 → usa & en vez de % para indexFor.
     & es ~3× más rápido que % en la mayoría de CPUs.

  4. LAZY INITIALIZATION:
     El backing array no se crea hasta el primer put().
     new HashMap<>() no asigna memoria para la tabla.
     → Ahorra memoria si el HashMap nunca se usa.

  5. ITERATION ORDER:
     No hay orden garantizado. Cambia después de un resize.
     LinkedHashMap mantiene insertion order (doubly-linked list).

  6. FAIL-FAST ITERATOR:
     modCount se incrementa en cada modificación.
     El iterator compara modCount al inicio y en cada next().
     Si cambió → ConcurrentModificationException.
     No es thread-safe — solo detecta bugs.

  7. TABLESIDE RESIZE OPTIMIZATION:
     En un resize de capacity a 2×capacity:
     cada entry va al index original O al index + vieja_capacity.
     Ejemplo: capacity=16, entry con hash=0b10110100.
     Old index: hash & 0b1111 = 0b0100 = 4
     New index: hash & 0b11111 = 0b10100 = 20 (4 + 16) O 0b00100 = 4
     Solo necesita mirar UN BIT extra (el bit de la vieja capacidad).
     → El resize es más eficiente que rehash completo.
```

**Preguntas:**

1. ¿La treeification protege contra HashDoS. ¿Cómo funciona HashDoS?

2. ¿El bit trick del resize — ¿es más rápido que recalcular
   hash % newCapacity?

3. ¿Lazy initialization — ¿cuántos HashMap vacíos existen
   en un programa Java típico?

4. ¿`ConcurrentModificationException` no es para concurrencia.
   ¿Entonces para qué es?

5. ¿Nuestra MiHashMap implementa alguna de estas 7 optimizaciones?

---

### Ejercicio 4.4.5 — Analizar: por qué equals y hashCode deben ser consistentes

**Tipo: Analizar**

```
El contrato de Java HashMap:

  Si a.equals(b) → a.hashCode() == b.hashCode()
  (El inverso NO es obligatorio: mismo hash no implica equals.)

  ¿Qué pasa si se viola?

  Caso 1: equals sin hashCode
    class Persona {
        String nombre;
        @Override public boolean equals(Object o) {
            return o instanceof Persona p && nombre.equals(p.nombre);
        }
        // NO override hashCode!
    }

    var map = new HashMap<Persona, String>();
    var p1 = new Persona("Ana");
    map.put(p1, "data");
    var p2 = new Persona("Ana");
    map.get(p2); // → null ← ¡¡Debería retornar "data"!!

    ¿Por qué? p1.hashCode() ≠ p2.hashCode() (heredan de Object: identity hash).
    p1 y p2 caen en buckets DISTINTOS.
    get(p2) busca en el bucket de p2, donde p1 no está.

  Caso 2: hashCode mutable
    class Punto {
        int x, y;
        @Override public int hashCode() { return x * 31 + y; }
        @Override public boolean equals(Object o) {
            return o instanceof Punto p && x == p.x && y == p.y;
        }
    }

    var map = new HashMap<Punto, String>();
    var p = new Punto(1, 2);
    map.put(p, "data");     // hash = 1*31 + 2 = 33, bucket = 33 % 16 = 1
    p.x = 10;               // mutamos la key DESPUÉS de insertarla
    map.get(p); // → null   // hash = 10*31 + 2 = 312, bucket = 312 % 16 = 8
                             // busca en bucket 8, pero el entry está en bucket 1

  Lección: las keys de un hash map deben ser INMUTABLES.
  En Java: String, Integer, Long son inmutables → seguros.
  En Rust: HashMap<K, V> requiere K: Eq + Hash. Ownership previene mutación.
  En Go: map keys deben ser comparable. Slices no pueden ser keys.
  En Python: solo hashable objects (immutables) pueden ser dict keys.
```

**Preguntas:**

1. ¿Scala case classes generan equals y hashCode automáticamente?
   ¿Son consistentes?

2. ¿En Rust, ¿puedes mutar una key de un HashMap mientras está insertada?

3. ¿Go permite structs como map keys. ¿Qué pasa si el struct
   tiene un campo slice?

4. ¿Python permite tuples como dict keys pero no lists. ¿Por qué?

5. ¿El record de Java 16+ genera equals/hashCode.
   ¿Resuelve este problema?

---

## Sección 4.5 — Open Addressing: Linear Probing y sus Problemas

### Ejercicio 4.5.1 — Leer: open addressing vs chaining

**Tipo: Leer**

```
Open addressing: todos los entries están en el MISMO array.
Si la posición calculada está ocupada, buscar otra posición libre.

  PUT(key, value):
    index = hash(key) % capacity
    while table[index] is occupied:
        if table[index].key == key: update value, return
        index = next_probe(index)    ← la "probing strategy"
    table[index] = (key, value)

  GET(key):
    index = hash(key) % capacity
    while table[index] is not empty:
        if table[index].key == key: return value
        index = next_probe(index)
    return null  (slot vacío → key no existe)

  PROBING STRATEGIES:

  1. LINEAR PROBING:
     next(index) = (index + 1) % capacity
     Simple. Excelente cache locality (slots contiguos en memoria).
     Problema: PRIMARY CLUSTERING — grupos de posiciones consecutivas
     ocupadas crecen y se fusionan. Los clusters ralentizan las búsquedas.

  2. QUADRATIC PROBING:
     next(index, i) = (index + i²) % capacity  (i = 1, 2, 3, ...)
     Reduce clustering primario.
     Problema: SECONDARY CLUSTERING — claves con mismo hash inicial
     siguen la misma secuencia de probes.
     Problema: no garantiza visitar todos los slots
     (necesita capacity primo o potencia de 2 con cuidado).

  3. DOUBLE HASHING:
     next(index, i) = (index + i × hash2(key)) % capacity
     Cada clave tiene su propia secuencia de probes.
     Casi elimina clustering.
     Problema: peor cache locality (saltos aleatorios).
     Problema: más costoso (dos funciones hash).

  4. ROBIN HOOD HASHING (Sección 4.6):
     Linear probing + redistribución durante insert.
     Si un entry "rico" (pocos probes) está en el camino de uno "pobre"
     (muchos probes), se intercambian.
     → Ecualiza la distancia de probe entre entries.
     → Reduce la VARIANZA del tiempo de lookup.

  Tradeoff fundamental:
    Chaining:        memoria extra (nodos), mala locality, simple.
    Open addressing: sin memoria extra, buena locality, complejo deletes.
```

**Preguntas:**

1. ¿Open addressing tiene mejor cache locality que chaining?
   ¿Siempre?

2. ¿DELETE en open addressing es complicado. ¿Por qué?

3. ¿Python dict usa open addressing. ¿Cómo maneja los deletes?

4. ¿Con load factor 0.9, ¿cuántos probes promedio necesita
   linear probing?

5. ¿SwissTable de Rust/Google — ¿es linear probing,
   quadratic probing, o algo distinto?

---

### Ejercicio 4.5.2 — Implementar: linear probing hash map en Rust

**Tipo: Implementar**

```rust
// Hash map con linear probing
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

#[derive(Clone)]
enum Slot<K, V> {
    Empty,
    Occupied(K, V, u64), // key, value, hash
    Deleted,             // tombstone for delete support
}

pub struct LinearProbingMap<K, V> {
    table: Vec<Slot<K, V>>,
    size: usize,
    capacity: usize,
    probe_count: usize, // total probes for stats
}

impl<K: Hash + Eq + Clone, V: Clone> LinearProbingMap<K, V> {
    pub fn new(capacity: usize) -> Self {
        let cap = capacity.next_power_of_two().max(16);
        LinearProbingMap {
            table: vec![Slot::Empty; cap],
            size: 0,
            capacity: cap,
            probe_count: 0,
        }
    }

    fn hash_key(key: &K) -> u64 {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        let h = hasher.finish();
        h ^ (h >> 16)
    }

    pub fn put(&mut self, key: K, value: V) -> Option<V> {
        if (self.size + 1) as f64 > self.capacity as f64 * 0.7 {
            self.resize();
        }

        let h = Self::hash_key(&key);
        let mut index = (h as usize) & (self.capacity - 1);
        let mut first_deleted: Option<usize> = None;

        loop {
            match &self.table[index] {
                Slot::Empty => {
                    let insert_idx = first_deleted.unwrap_or(index);
                    self.table[insert_idx] = Slot::Occupied(key, value, h);
                    self.size += 1;
                    return None;
                }
                Slot::Deleted => {
                    if first_deleted.is_none() { first_deleted = Some(index); }
                }
                Slot::Occupied(k, _, _) if *k == key => {
                    let old = std::mem::replace(
                        &mut self.table[index],
                        Slot::Occupied(key, value, h)
                    );
                    if let Slot::Occupied(_, v, _) = old { return Some(v); }
                    unreachable!();
                }
                Slot::Occupied(_, _, _) => {}
            }
            self.probe_count += 1;
            index = (index + 1) & (self.capacity - 1);
        }
    }

    pub fn get(&self, key: &K) -> Option<&V> {
        let h = Self::hash_key(key);
        let mut index = (h as usize) & (self.capacity - 1);

        loop {
            match &self.table[index] {
                Slot::Empty => return None,
                Slot::Occupied(k, v, _) if *k == *key => return Some(v),
                _ => { index = (index + 1) & (self.capacity - 1); }
            }
        }
    }

    pub fn remove(&mut self, key: &K) -> Option<V> {
        let h = Self::hash_key(key);
        let mut index = (h as usize) & (self.capacity - 1);

        loop {
            match &self.table[index] {
                Slot::Empty => return None,
                Slot::Occupied(k, _, _) if *k == *key => {
                    let old = std::mem::replace(&mut self.table[index], Slot::Deleted);
                    self.size -= 1;
                    if let Slot::Occupied(_, v, _) = old { return Some(v); }
                    unreachable!();
                }
                _ => { index = (index + 1) & (self.capacity - 1); }
            }
        }
    }

    fn resize(&mut self) {
        let new_cap = self.capacity * 2;
        let old_table = std::mem::replace(
            &mut self.table,
            vec![Slot::Empty; new_cap]
        );
        self.capacity = new_cap;
        self.size = 0;
        self.probe_count = 0;

        for slot in old_table {
            if let Slot::Occupied(k, v, _) = slot {
                self.put(k, v);
            }
        }
    }

    pub fn size(&self) -> usize { self.size }
    pub fn avg_probes(&self) -> f64 {
        if self.size == 0 { 0.0 } else { self.probe_count as f64 / self.size as f64 }
    }
}

fn main() {
    let n = 1_000_000;
    let mut map = LinearProbingMap::new(n * 2);

    let start = std::time::Instant::now();
    for i in 0..n {
        map.put(i as i64, i as i64 * 2);
    }
    let insert_time = start.elapsed();

    println!("Inserted {} entries in {:?}", n, insert_time);
    println!("Avg probes per insert: {:.2}", map.avg_probes());
    println!("get(42) = {:?}", map.get(&42));
    println!("get(-1) = {:?}", map.get(&-1));
}
```

**Preguntas:**

1. ¿`Slot::Deleted` (tombstone) — ¿por qué no podemos simplemente
   poner `Slot::Empty` al borrar?

2. ¿Los tombstones se acumulan. ¿Cómo se limpian?

3. ¿`avg_probes` debería ser cercano a 1.0 con LF=0.7.
   ¿Es así?

4. ¿Esta implementación es comparable en rendimiento
   a `std::collections::HashMap`?

5. ¿Linear probing con load factor 0.7 vs chaining con LF 0.75:
   ¿cuál tiene menos cache misses?

---

### Ejercicio 4.5.3 — Analizar: clustering — el talón de Aquiles del linear probing

**Tipo: Analizar**

```
Primary clustering con linear probing:

  Estado inicial (table size = 16, _ = vacío):
  [_ _ _ _ X X X X _ _ _ _ _ _ _ _]
                ^^^^^^^
                cluster de 4

  Si hash(nueva_key) cae en posición 4, 5, 6, o 7:
  el nuevo entry se agrega al FINAL del cluster (posición 8).
  El cluster crece a 5.

  Peor aún: si hash(otra_key) cae en posición 8
  (que antes estaba libre), ahora está ocupada.
  Otro entry que debería ir a la posición 8
  se desplaza a la 9. El cluster crece a 7.

  Los clusters se AUTOALIMENTAN:
  - Un cluster más grande es un target más grande.
  - Más hashes caen en el cluster.
  - El cluster crece más rápido.
  - Eventualmente, clusters vecinos se FUSIONAN.

  Longitud esperada del probe para linear probing:

    Lookup exitoso:    ½(1 + 1/(1-α))
    Lookup fallido:    ½(1 + 1/(1-α)²)

    α = 0.50:  exitoso = 1.5,   fallido = 2.5
    α = 0.70:  exitoso = 2.2,   fallido = 6.1
    α = 0.80:  exitoso = 3.0,   fallido = 13.0
    α = 0.90:  exitoso = 5.5,   fallido = 50.5
    α = 0.95:  exitoso = 10.5,  fallido = 200.5

  A LF=0.90, un lookup FALLIDO necesita ~50 probes.
  Cada probe es un acceso al array (posiblemente un cache miss).
  50 probes × ~5 ns (array accesses, mayormente cache hits) = 250 ns.

  Por eso open addressing necesita load factors más bajos que chaining.
  Java HashMap (chaining): LF = 0.75.
  Rust HashMap (open addressing): LF = 0.875 (7/8), pero con SwissTable
  que mitiga el clustering con metadata de control.
```

**Preguntas:**

1. ¿50 probes promedio para lookup fallido con LF=0.90 — ¿es aceptable?

2. ¿Quadratic probing reduce el clustering? ¿Cuánto?

3. ¿Si la hash function tiene distribución perfecta,
   ¿el clustering desaparece?

4. ¿SwissTable mantiene LF=0.875 sin clustering severo.
   ¿Cómo?

5. ¿Para un hash map de solo lecturas (read-only después de construir),
   ¿el clustering importa?

---

## Sección 4.6 — Robin Hood Hashing: Ecualizando la Miseria

### Ejercicio 4.6.1 — Leer: la idea de Robin Hood

**Tipo: Leer**

```
Robin Hood hashing es linear probing con una regla extra:
"Roba de los ricos para dar a los pobres."

  Cada entry tiene una DISTANCIA: cuántos probes se desplazó
  desde su posición ideal (hash % capacity).

  REGLA: durante insert, si el entry que estás insertando
  tiene MAYOR distancia que el entry ocupando la posición actual,
  intercámbialos. El entry "pobre" (más desplazado) se queda,
  y el "rico" (menos desplazado) sigue buscando.

  Ejemplo:
  Insertar key_X con hash ideal = 3.
  Posición 3: ocupada por key_A (distancia = 0 — está en su posición ideal)
  key_X tiene distancia 1 (se desplazó 1 posición).
  1 > 0 → INTERCAMBIAR. key_X se queda en 3, key_A sigue buscando.

  Posición 4: vacía. key_A se inserta aquí (distancia = 1).

  Resultado: ambos entries tienen distancia 1.
  Sin Robin Hood: key_X tendría distancia ≥1, key_A tendría distancia 0.
  → Robin Hood ECUALIZA las distancias.

  Impacto:
  - La distancia MÁXIMA es mucho menor que con linear probing estándar.
  - La VARIANZA del tiempo de lookup se reduce significativamente.
  - El lookup exitoso promedio es similar, pero el PEOR CASO mejora.
  - Lookup puede TERMINAR TEMPRANO: si la distancia del entry actual
    es menor que la distancia que llevas buscando, la key no existe.

  Lookup con early termination:
    GET(key):
      index = hash(key) % capacity
      distance = 0
      while table[index] is occupied:
          if table[index].key == key: return value
          if distance > table[index].distance: return null  ← EARLY TERMINATION
          index = (index + 1) % capacity
          distance++
      return null

  Early termination evita recorrer todo el cluster para lookups fallidos.
  → El lookup fallido con Robin Hood es mucho más rápido que linear probing.
```

**Preguntas:**

1. ¿Robin Hood hashing siempre es mejor que linear probing estándar?

2. ¿La early termination es la ventaja principal?

3. ¿Robin Hood complica los deletes? ¿Necesita backward shifting?

4. ¿SwissTable de Rust usa Robin Hood hashing o algo distinto?

5. ¿La varianza reducida importa para sistemas de baja latencia?

---

### Ejercicio 4.6.2 — Implementar: Robin Hood hashing en Rust

**Tipo: Implementar**

```rust
// Robin Hood hashing con early termination y backward shift delete
pub struct RobinHoodMap<K, V> {
    keys: Vec<Option<K>>,
    values: Vec<Option<V>>,
    hashes: Vec<u64>,
    distances: Vec<i32>,  // -1 = empty, ≥0 = distance from ideal
    size: usize,
    capacity: usize,
    max_distance: usize,
}

impl<K: std::hash::Hash + Eq + Clone, V: Clone> RobinHoodMap<K, V> {
    pub fn new(capacity: usize) -> Self {
        let cap = capacity.next_power_of_two().max(16);
        RobinHoodMap {
            keys: vec![None; cap],
            values: vec![None; cap],
            hashes: vec![0; cap],
            distances: vec![-1; cap],
            size: 0,
            capacity: cap,
            max_distance: 0,
        }
    }

    fn hash_key(key: &K) -> u64 {
        use std::hash::Hasher;
        let mut h = std::collections::hash_map::DefaultHasher::new();
        key.hash(&mut h);
        let hash = h.finish();
        hash ^ (hash >> 16)
    }

    pub fn put(&mut self, mut key: K, mut value: V) -> Option<V> {
        if (self.size + 1) as f64 > self.capacity as f64 * 0.8 {
            self.resize();
        }

        let mut h = Self::hash_key(&key);
        let mut index = (h as usize) & (self.capacity - 1);
        let mut dist: i32 = 0;

        loop {
            if self.distances[index] < 0 {
                // Empty slot: insert here
                self.keys[index] = Some(key);
                self.values[index] = Some(value);
                self.hashes[index] = h;
                self.distances[index] = dist;
                self.size += 1;
                self.max_distance = self.max_distance.max(dist as usize);
                return None;
            }

            // Check if same key
            if self.hashes[index] == h {
                if let Some(ref k) = self.keys[index] {
                    if *k == key {
                        let old = self.values[index].take();
                        self.values[index] = Some(value);
                        return old;
                    }
                }
            }

            // Robin Hood: steal from the rich
            if dist > self.distances[index] {
                // Swap: current entry is "richer" (less displaced)
                std::mem::swap(&mut self.keys[index], &mut Some(key).as_mut().map(|_| &mut key).and_then(|_| self.keys[index].as_mut()));
                // Simplified: just do manual swaps
                let tmp_key = self.keys[index].take();
                let tmp_val = self.values[index].take();
                let tmp_hash = self.hashes[index];
                let tmp_dist = self.distances[index];

                self.keys[index] = Some(key);
                self.values[index] = Some(value);
                self.hashes[index] = h;
                self.distances[index] = dist;

                key = tmp_key.unwrap();
                value = tmp_val.unwrap();
                h = tmp_hash;
                dist = tmp_dist;
            }

            index = (index + 1) & (self.capacity - 1);
            dist += 1;
        }
    }

    pub fn get(&self, key: &K) -> Option<&V> {
        let h = Self::hash_key(key);
        let mut index = (h as usize) & (self.capacity - 1);
        let mut dist: i32 = 0;

        loop {
            if self.distances[index] < 0 {
                return None; // empty slot
            }
            if dist > self.distances[index] {
                return None; // EARLY TERMINATION: would have been here if it existed
            }
            if self.hashes[index] == h {
                if let Some(ref k) = self.keys[index] {
                    if *k == *key {
                        return self.values[index].as_ref();
                    }
                }
            }
            index = (index + 1) & (self.capacity - 1);
            dist += 1;
        }
    }

    pub fn remove(&mut self, key: &K) -> Option<V> {
        let h = Self::hash_key(key);
        let mut index = (h as usize) & (self.capacity - 1);
        let mut dist: i32 = 0;

        loop {
            if self.distances[index] < 0 || dist > self.distances[index] {
                return None;
            }
            if self.hashes[index] == h {
                if let Some(ref k) = self.keys[index] {
                    if *k == *key {
                        let removed = self.values[index].take();
                        self.keys[index] = None;
                        self.distances[index] = -1;
                        self.size -= 1;
                        // Backward shift: move entries closer to ideal
                        self.backward_shift(index);
                        return removed;
                    }
                }
            }
            index = (index + 1) & (self.capacity - 1);
            dist += 1;
        }
    }

    fn backward_shift(&mut self, start: usize) {
        let mut index = (start + 1) & (self.capacity - 1);
        loop {
            if self.distances[index] <= 0 {
                break; // empty or at ideal position
            }
            let prev = (index + self.capacity - 1) & (self.capacity - 1);
            self.keys[prev] = self.keys[index].take();
            self.values[prev] = self.values[index].take();
            self.hashes[prev] = self.hashes[index];
            self.distances[prev] = self.distances[index] - 1;
            self.distances[index] = -1;
            index = (index + 1) & (self.capacity - 1);
        }
    }

    fn resize(&mut self) {
        let old_keys = std::mem::replace(&mut self.keys, vec![None; self.capacity * 2]);
        let old_values = std::mem::replace(&mut self.values, vec![None; self.capacity * 2]);
        let old_hashes = std::mem::take(&mut self.hashes);
        let old_distances = std::mem::take(&mut self.distances);

        self.hashes = vec![0; self.capacity * 2];
        self.distances = vec![-1; self.capacity * 2];
        self.capacity *= 2;
        self.size = 0;
        self.max_distance = 0;

        for i in 0..old_keys.len() {
            if old_distances[i] >= 0 {
                if let (Some(k), Some(v)) = (old_keys[i].clone(), old_values[i].clone()) {
                    self.put(k, v);
                }
            }
        }
    }

    pub fn size(&self) -> usize { self.size }
    pub fn max_distance(&self) -> usize { self.max_distance }
}

fn main() {
    let n = 1_000_000usize;
    let mut map = RobinHoodMap::new(n * 2);

    let start = std::time::Instant::now();
    for i in 0..n {
        map.put(i as i64, i as i64 * 2);
    }
    let elapsed = start.elapsed();

    println!("Inserted {} entries in {:?}", n, elapsed);
    println!("Max distance: {}", map.max_distance());
    println!("get(42) = {:?}", map.get(&42));
    println!("get(-1) = {:?}", map.get(&-1));
}
```

**Preguntas:**

1. ¿Backward shift elimina la necesidad de tombstones?

2. ¿`max_distance` debería ser mucho menor que con linear probing estándar?

3. ¿La early termination mejora los lookups fallidos. ¿Cuánto?

4. ¿Los 4 arrays paralelos (keys, values, hashes, distances)
   tienen buena cache locality? ¿O SoA sería mejor?

5. ¿La implementación real de hashbrown/SwissTable de Rust
   es más rápida? ¿Por qué?

---

## Sección 4.7 — Load Factor, Rehashing, y el Benchmark Final

### Ejercicio 4.7.1 — Implementar: medir el impacto del load factor

**Tipo: Implementar**

```java
// Medir el tiempo de lookup para diferentes load factors

public class LoadFactorBenchmark {

    public static void main(String[] args) {
        int n = 1_000_000;
        double[] loadFactors = {0.25, 0.50, 0.60, 0.70, 0.75, 0.80, 0.90, 0.95};
        int lookups = 1_000_000;

        var rng = new java.util.Random(42);
        int[] keys = new int[n];
        for (int i = 0; i < n; i++) keys[i] = rng.nextInt();
        int[] queryKeys = new int[lookups];
        for (int i = 0; i < lookups; i++) queryKeys[i] = keys[rng.nextInt(n)];

        System.out.printf("%-8s  %-10s  %-12s  %-12s%n",
            "LF", "Capacity", "Get (ns/op)", "Miss (ns/op)");
        System.out.println("─".repeat(50));

        for (double lf : loadFactors) {
            int capacity = (int)(n / lf);
            var map = new java.util.HashMap<Integer, Integer>(capacity, (float) lf);
            for (int i = 0; i < n; i++) map.put(keys[i], i);

            // Warmup
            long dummy = 0;
            for (int i = 0; i < 10_000; i++) dummy += map.getOrDefault(queryKeys[i % lookups], 0);

            // Benchmark: successful lookups
            long start = System.nanoTime();
            for (int q : queryKeys) dummy += map.getOrDefault(q, 0);
            long hitTime = System.nanoTime() - start;

            // Benchmark: failed lookups (keys that don't exist)
            int[] missKeys = new int[lookups];
            for (int i = 0; i < lookups; i++) missKeys[i] = -(i + 1);
            start = System.nanoTime();
            for (int q : missKeys) dummy += map.getOrDefault(q, 0);
            long missTime = System.nanoTime() - start;

            System.out.printf("%-8.2f  %-10d  %-12.1f  %-12.1f%n",
                lf, map.size(), (double) hitTime / lookups, (double) missTime / lookups);

            if (dummy == Long.MIN_VALUE) System.out.println(dummy);
        }
    }
}
// Resultado esperado:
// LF        Capacity    Get (ns/op)   Miss (ns/op)
// ──────────────────────────────────────────────────
// 0.25      1000000     22.0          18.0
// 0.50      1000000     24.0          20.0
// 0.60      1000000     25.0          22.0
// 0.70      1000000     27.0          24.0
// 0.75      1000000     28.0          26.0
// 0.80      1000000     30.0          30.0
// 0.90      1000000     35.0          40.0
// 0.95      1000000     42.0          55.0
```

**Preguntas:**

1. ¿El rendimiento degrada gradualmente o hay un "cliff"?

2. ¿LF=0.75 de Java es un buen equilibrio entre memoria y velocidad?

3. ¿Los miss lookups degradan más rápido. ¿Por qué?

4. ¿Estos resultados son para chaining. ¿Open addressing
   degradaría más rápido?

5. ¿Para un cache (read-heavy, write-rare), ¿qué LF elegirías?

---

### Ejercicio 4.7.2 — Implementar: benchmark cruzado — 4 lenguajes

**Tipo: Implementar**

```
Benchmark: insertar y buscar 1M entries en hash maps de 4 lenguajes.
Ejecutar cada uno por separado y comparar los resultados.
```

```java
// Java
var map = new java.util.HashMap<Long, Long>(2_000_000);
long start = System.nanoTime();
for (long i = 0; i < 1_000_000; i++) map.put(i, i * 2);
long insertNs = System.nanoTime() - start;
// Lookup...
```

```go
// Go
m := make(map[int64]int64, 2_000_000)
start := time.Now()
for i := int64(0); i < 1_000_000; i++ { m[i] = i * 2 }
insertTime := time.Since(start)
```

```python
# Python
d = {}
start = time.perf_counter_ns()
for i in range(1_000_000): d[i] = i * 2
insert_ns = time.perf_counter_ns() - start
```

```rust
// Rust
let mut map = HashMap::with_capacity(2_000_000);
let start = Instant::now();
for i in 0i64..1_000_000 { map.insert(i, i * 2); }
let insert_time = start.elapsed();
```

```
Resultado esperado (orientativo):

  Lenguaje    Insert (ns/op)   Get hit (ns/op)   Get miss (ns/op)
  ────────    ──────────────   ───────────────    ────────────────
  Rust             25               18                 12
  Go               45               25                 18
  Java             55               28                 22
  Python          180              120                 80
```

**Preguntas:**

1. ¿Rust es más rápido que Go y Java. ¿Por la hash function,
   el open addressing, o la ausencia de GC?

2. ¿Python es 7× más lento que Rust. ¿Esperado?

3. ¿Java es competitivo con Go. ¿El JIT compensa el boxing?

4. ¿Estos números son para claves numéricas. ¿Para String keys
   el ranking cambiaría?

5. ¿Para un hash map de 100M entries, ¿los ratios se mantienen?

---

### Ejercicio 4.7.3 — Analizar: hash maps en la infraestructura de datos

**Tipo: Analizar**

```
El hash map aparece en CADA capa de la infraestructura de datos:

  SPARK — Hash Partitioning:
    Cuando haces df.repartition(col("user_id")),
    Spark calcula hash(user_id) % numPartitions.
    Usa Murmur3. Cada executor tiene un hash map interno
    para agregar datos durante shuffles.

  KAFKA — Partition Assignment:
    DefaultPartitioner: hash(key) % numPartitions.
    Usa Murmur2 (legacy) o Murmur3.
    Los consumers usan hash maps para rastrear offsets por partición.

  FLINK — Keyed State:
    Cuando usas keyBy(user_id), Flink hashea la key
    para asignar cada key a un key group.
    El state backend (HashMap o RocksDB) indexa el estado por key.

  DATABASES — Hash Join:
    SELECT * FROM orders JOIN users ON orders.user_id = users.id
    El plan de ejecución construye un hash map de la tabla pequeña (users)
    y luego escanea la tabla grande (orders) buscando matches.
    → O(n + m) en vez de O(n × m) con nested loops.

  REDIS / MEMCACHED:
    Son hash maps distribuidos.
    Cada get/set es un lookup en un hash table en memoria.
    Redis usa su propia implementación (dict) con rehash incremental.

  CASSANDRA — Consistent Hashing:
    Los tokens de cada nodo se determinan hasheando el partition key.
    Usa Murmur3 para asignar datos a nodos del cluster.

  El hash map que implementaste en esta sección
  es la misma estructura que mueve datos en Spark, Kafka, Flink,
  bases de datos, y caches distribuidos.
  La diferencia es escala, no principio.
```

**Preguntas:**

1. ¿Spark usa Murmur3 para shuffles. ¿Qué pasa si Murmur3
   distribuye mal para ciertas claves?

2. ¿El hash join de una base de datos es O(n + m).
   ¿Cuánta memoria necesita?

3. ¿Redis rehash incremental — ¿es como el shrinking
   del dynamic array pero aplicado al hash map?

4. ¿Cassandra consistent hashing (Cap.15) es un hash map
   a nivel de cluster?

5. ¿Si un data engineer entiende el hash map desde las tripas,
   ¿entiende el fundamento de todas estas herramientas?

---

### Ejercicio 4.7.4 — Resumen: las reglas del hash map

**Tipo: Leer**

```
Reglas nuevas del Cap.04:

  Regla 12: El hash map es la estructura de acceso más usada.
    Más del 50% de los accesos a datos en producción
    pasan por un hash table.

  Regla 13: La función hash importa tanto como la estructura.
    Una hash function mala (clustering, colisiones) destruye
    el rendimiento del hash map, independientemente de la implementación.
    Siempre mezcla los bits. Nunca uses el identity hash
    sin verificar la distribución.

  Regla 14: Chaining es simple, open addressing es rápido.
    Chaining: fácil de implementar, soporta LF alto, mala locality.
    Open addressing: más rápido por locality, complicado con deletes,
    necesita LF más bajo.

  Regla 15: El load factor controla el tradeoff memoria↔velocidad.
    LF bajo (0.25-0.50): rápido, usa mucha memoria.
    LF medio (0.70-0.80): buen balance.
    LF alto (0.90+): lento (especialmente para misses), ahorra memoria.
    Elegir LF según tu caso: ¿read-heavy? LF bajo.
    ¿Write-heavy con mucha memoria? LF medio.

  Regla 16: Las keys deben ser inmutables.
    Mutar una key después de insertarla corrompe el hash map.
    Usa String, Integer, records, o value objects como keys.

  Resumen del capítulo:
  Hash function → reduce a índice → colisión inevitable →
  resolución (chaining o open addressing) → load factor controla
  cuántas colisiones → rehash cuando LF excede threshold.
  Todo en O(1) amortizado caso promedio. O(n) peor caso.
```

**Preguntas:**

1. ¿Las 16 reglas cubren todo lo necesario para elegir
   y entender estructuras de datos hasta ahora?

2. ¿La Regla 14 tiene excepciones? ¿Cuándo chaining es más rápido?

3. ¿Robin Hood hashing merece su propia regla o es una optimización
   de open addressing?

4. ¿Qué regla habrías querido saber cuando empezaste
   a usar hash maps?

5. ¿El Cap.05 (hash maps avanzados) agregará más reglas
   o refinará las existentes?
