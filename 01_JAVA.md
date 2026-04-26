# 01 — Java Core & Avancé

## 1. Collections Framework

### Concepts Clés

```
java.util.Collection
├── List (ArrayList, LinkedList, Vector)
├── Set (HashSet, LinkedHashSet, TreeSet)
└── Queue (PriorityQueue, ArrayDeque)

java.util.Map
├── HashMap, LinkedHashMap, TreeMap
└── ConcurrentHashMap (thread-safe)
```

### Exemples

```java
// ArrayList vs LinkedList
List<String> arrayList = new ArrayList<>();  // accès O(1), insertion O(n)
List<String> linkedList = new LinkedList<>(); // accès O(n), insertion O(1)

// HashMap : complexité O(1) amortie, collision => liste chaînée ou arbre (Java 8+)
Map<String, Integer> scores = new HashMap<>();
scores.put("Alice", 95);
scores.getOrDefault("Bob", 0); // évite NullPointerException

// ConcurrentHashMap pour environnement multi-thread
Map<String, Integer> safeMap = new ConcurrentHashMap<>();

// TreeMap : clés triées (ordre naturel ou Comparator)
Map<String, Integer> sorted = new TreeMap<>(Comparator.reverseOrder());
```

---

## 2. Streams & Lambdas (Java 8+)

### Concepts Clés

Un Stream est un pipeline de traitement **paresseux** (lazy evaluation).
- **Intermédiaires** (lazy) : `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`
- **Terminaux** (déclenchent l'exécution) : `collect`, `forEach`, `count`, `findFirst`, `reduce`

```java
// Exemple complet : filtrer, transformer, collecter
List<String> names = List.of("Alice", "Bob", "Charlie", "Anna");

List<String> result = names.stream()
    .filter(n -> n.startsWith("A"))          // intermédiaire
    .map(String::toUpperCase)                 // intermédiaire (method reference)
    .sorted()                                 // intermédiaire
    .collect(Collectors.toList());            // terminal

// Groupement
Map<Integer, List<String>> byLength = names.stream()
    .collect(Collectors.groupingBy(String::length));

// Reduce
int sum = IntStream.rangeClosed(1, 10)
    .reduce(0, Integer::sum); // 55

// flatMap : aplatir une liste de listes
List<List<Integer>> matrix = List.of(List.of(1,2), List.of(3,4));
List<Integer> flat = matrix.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList()); // [1, 2, 3, 4]
```

### Optional

```java
Optional<String> opt = Optional.ofNullable(getValue());

// BAD : opt.get() sans vérification => NoSuchElementException
// GOOD :
String value = opt
    .filter(s -> s.length() > 2)
    .map(String::toUpperCase)
    .orElse("DEFAULT");
```

---

## 3. Programmation Concurrente

### Thread vs Runnable vs Callable

```java
// Runnable : pas de retour, pas d'exception checked
Runnable r = () -> System.out.println("Thread: " + Thread.currentThread().getName());

// Callable : retour + exception
Callable<Integer> c = () -> {
    Thread.sleep(1000);
    return 42;
};

// ExecutorService : pool de threads géré
ExecutorService executor = Executors.newFixedThreadPool(4);
Future<Integer> future = executor.submit(c);
Integer result = future.get(); // bloquant
executor.shutdown();
```

### CompletableFuture (Java 8+)

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> fetchUser(userId))         // exécution async
    .thenApply(user -> user.getName())            // transformation
    .thenApplyAsync(name -> name.toUpperCase())   // async pool différent
    .exceptionally(ex -> "UNKNOWN");              // gestion d'erreur

// Combiner deux futures
CompletableFuture<String> combined = CompletableFuture
    .allOf(cf1, cf2)
    .thenApply(v -> cf1.join() + " " + cf2.join());
```

### synchronized vs ReentrantLock

```java
// synchronized : simple mais moins flexible
public synchronized void increment() { count++; }

// ReentrantLock : tryLock, timeout, interruptible
private final ReentrantLock lock = new ReentrantLock();
public void increment() {
    lock.lock();
    try { count++; }
    finally { lock.unlock(); } // TOUJOURS dans finally
}
```

---

## 4. JVM, Garbage Collection & Mémoire

### Zones Mémoire JVM

| Zone | Contenu | GC ? |
|------|---------|------|
| **Heap** | Objets alloués dynamiquement | Oui |
| **Stack** | Frames de méthodes, variables locales | Non |
| **Metaspace** (Java 8+) | Métadonnées de classes | Oui (CMS/G1) |
| **Code Cache** | JIT compilé | Non |

### Algorithmes GC

- **Serial GC** : mono-thread, petites applications
- **Parallel GC** : multi-thread, throughput maximal
- **G1 GC** (défaut Java 9+) : faible latence, régions de 1-32MB
- **ZGC / Shenandoah** : latence < 10ms, très grands heaps

```bash
# Options JVM utiles
-Xms512m -Xmx2g         # heap min/max
-XX:+UseG1GC            # forcer G1
-XX:+PrintGCDetails     # logs GC
-XX:MaxGCPauseMillis=200 # cible pause G1
```

---

## 5. Nouveautés Java (11, 17, 21)

```java
// Java 11 : var, String methods
var list = new ArrayList<String>(); // inférence de type (Java 10)
"  hello  ".strip();   // unicode-aware (vs trim)
"".isBlank();          // true si vide ou espaces
"a\nb\nc".lines().collect(Collectors.toList());

// Java 14+ : Records (immuables, equals/hashCode/toString auto)
public record Point(int x, int y) {}
Point p = new Point(1, 2);
p.x(); // getter automatique

// Java 17 : Sealed Classes
public sealed class Shape permits Circle, Rectangle {}
public final class Circle extends Shape { double radius; }

// Java 21 : Pattern Matching for switch
Object obj = "Hello";
String result = switch (obj) {
    case Integer i -> "int: " + i;
    case String s when s.length() > 3 -> "long string: " + s;
    case String s -> "short string: " + s;
    default -> "other";
};

// Java 21 : Virtual Threads (Project Loom)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> System.out.println("Virtual thread!"));
}
```

---

## Questions d'Interview — Java

---

### Q1 : Quelle est la différence entre `==` et `.equals()` en Java ?

**Réponse :**

`==` compare les **références** (adresses mémoire). `.equals()` compare le **contenu** (valeur sémantique).

```java
String a = new String("hello");
String b = new String("hello");

a == b         // false : deux objets différents en heap
a.equals(b)    // true  : même contenu

// Cas piège : String pool
String c = "hello";
String d = "hello";
c == d         // true : même référence dans le pool de constantes
```

**Pour un Référent :** Toujours surcharger `equals()` ET `hashCode()` ensemble (contrat général). Si `a.equals(b)` est vrai, alors `a.hashCode() == b.hashCode()` DOIT être vrai.

---

### Q2 : Comment fonctionne `HashMap` en interne ?

**Réponse :**

1. `put(key, value)` : calcule `hash(key)` → détermine l'index dans le tableau (bucket)
2. **Collision** : si deux clés ont le même bucket → chaîne (LinkedList) jusqu'à Java 7, puis **arbre rouge-noir** si > 8 éléments (Java 8+)
3. **Resize** : si `size > capacity * loadFactor (0.75)` → doublement du tableau + rehash

```java
// Clé custom : OBLIGATOIRE de surcharger equals + hashCode
public class Employee {
    private int id;
    private String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee e)) return false;
        return id == e.id && Objects.equals(name, e.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }
}
```

**Piège :** Si `hashCode()` retourne toujours la même valeur → toutes les clés dans le même bucket → `O(n)` au lieu de `O(1)`.

---

### Q3 : Expliquez la différence entre `Checked` et `Unchecked` exceptions.

**Réponse :**

| Type | Héritage | Obligation | Exemples |
|------|----------|------------|---------|
| **Checked** | `Exception` | `try/catch` ou `throws` | `IOException`, `SQLException` |
| **Unchecked** | `RuntimeException` | Aucune | `NullPointerException`, `IllegalArgumentException` |
| **Error** | `Error` | Aucune (irrécupérable) | `OutOfMemoryError`, `StackOverflowError` |

```java
// Checked : le compilateur force la gestion
public String readFile(String path) throws IOException {
    return Files.readString(Path.of(path));
}

// Unchecked : erreur de programmation
public int divide(int a, int b) {
    if (b == 0) throw new IllegalArgumentException("Diviseur ne peut être 0");
    return a / b;
}

// Pattern : wrap checked en unchecked dans les lambdas
.map(path -> {
    try { return Files.readString(path); }
    catch (IOException e) { throw new UncheckedIOException(e); }
})
```

---

### Q4 : Qu'est-ce que le `volatile` keyword et quand l'utiliser ?

**Réponse :**

`volatile` garantit que les lectures/écritures d'une variable sont **visibles par tous les threads** (via la mémoire principale, pas le cache CPU).

Il **ne garantit pas l'atomicité** des opérations composées (`i++` = lecture + incrémentation + écriture).

```java
// Cas d'usage valide : flag d'arrêt
private volatile boolean running = true;

public void stop() { running = false; }

public void run() {
    while (running) {  // lit toujours la valeur fraîche
        doWork();
    }
}

// Pour atomicité : utiliser AtomicInteger
private AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet(); // thread-safe
```

---

### Q5 : Expliquez les types de références Java (Strong, Soft, Weak, Phantom).

**Réponse :**

| Type | GC collecte ? | Usage |
|------|--------------|-------|
| **Strong** | Jamais (si référencé) | Référence normale |
| **Soft** | Seulement si mémoire insuffisante | Caches |
| **Weak** | Dès le prochain GC | `WeakHashMap`, listeners |
| **Phantom** | Après finalisation | Nettoyage de ressources natives |

```java
// Soft : cache image
SoftReference<BufferedImage> imageCache = new SoftReference<>(loadImage());
BufferedImage img = imageCache.get(); // peut être null si GC est passé

// Weak : éviter memory leak avec listeners
WeakReference<MyListener> listenerRef = new WeakReference<>(listener);
// Si listener n'a plus de strong reference → GC peut le collecter

// WeakHashMap : clés collectables par GC
Map<Object, String> cache = new WeakHashMap<>();
```

---

### Q6 : Qu'est-ce que le principe SOLID ? Donnez un exemple Java pour chaque lettre.

**Réponse :**

```java
// S — Single Responsibility : une classe = une raison de changer
// BAD
class UserService {
    public void saveUser(User u) { /* DB */ }
    public void sendWelcomeEmail(User u) { /* Email */ }
}
// GOOD
class UserRepository { public void save(User u) {} }
class EmailService { public void sendWelcome(User u) {} }

// O — Open/Closed : ouvert à l'extension, fermé à la modification
interface Discount { double apply(double price); }
class SeasonalDiscount implements Discount { /* ... */ }
class LoyaltyDiscount implements Discount { /* ... */ }
// Ajouter un discount = nouvelle classe, pas modifier l'existant

// L — Liskov Substitution : sous-classe substituable à sa parente
// BAD : Square extends Rectangle casse le contrat si setWidth modifie height
// GOOD : modéliser avec interface Shape { double area(); }

// I — Interface Segregation : interfaces spécialisées
interface Printable { void print(); }
interface Scannable { void scan(); }
// Plutôt qu'une grosse interface MultiFunctionDevice

// D — Dependency Inversion : dépendre des abstractions
// BAD
class OrderService { private MySQLOrderRepo repo = new MySQLOrderRepo(); }
// GOOD
class OrderService {
    private final OrderRepository repo; // interface
    public OrderService(OrderRepository repo) { this.repo = repo; } // injection
}
```

---

### Q7 : Comment fonctionne le Garbage Collector G1 ?

**Réponse :**

G1 (Garbage First) divise le heap en **régions de taille égale** (1-32 MB). Contrairement à l'ancien modèle Young/Old contiguës, les régions sont assignées dynamiquement.

**Cycle de collecte :**
1. **Young GC** : collecte les régions Eden + Survivor → STW (Stop-The-World) court
2. **Concurrent Marking** : marque les objets vivants pendant que l'app tourne
3. **Mixed GC** : collecte Young + quelques régions Old (celles avec le plus de garbage → "Garbage First")
4. **Full GC** (rare) : si allocation trop rapide → STW long

```bash
# Tuning G1
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200    # cible de pause (best effort)
-XX:G1HeapRegionSize=16m    # taille des régions
-XX:G1NewSizePercent=20     # % min du heap pour Young
-XX:InitiatingHeapOccupancyPercent=45  # déclenchement Concurrent Marking
```

---

### Q8 : Différence entre `parallelStream()` et un `ExecutorService` ?

**Réponse :**

| | `parallelStream()` | `ExecutorService` |
|-|-------------------|-------------------|
| **Pool** | ForkJoinPool.commonPool() (partagé JVM) | Pool dédié configurable |
| **Contrôle** | Minimal | Total (taille, stratégie) |
| **Usage** | Opérations CPU-bound simples | Tâches complexes, I/O, timeout |
| **Risque** | Contention si pool partagé saturé | Risque de memory leak si pas shutdown |

```java
// parallelStream : attention au pool partagé
List<Integer> result = bigList.parallelStream()
    .map(this::heavyComputation)
    .collect(Collectors.toList());

// ExecutorService : contrôle total
ExecutorService pool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);
List<Future<Integer>> futures = bigList.stream()
    .map(item -> pool.submit(() -> heavyComputation(item)))
    .collect(Collectors.toList());
pool.shutdown();
```

---

## Pièges Classiques

- Dire que `String` est mutable → **FAUX**, `String` est immuable (pool de constantes)
- Oublier `hashCode()` quand on surcharge `equals()` → bugs subtils dans les Maps/Sets
- Utiliser `parallelStream()` pour des opérations I/O → contention sur le pool commun
- `ConcurrentModificationException` : modifier une collection pendant l'itération → utiliser `Iterator.remove()` ou `CopyOnWriteArrayList`
- `volatile` ne suffit pas pour `i++` → utiliser `AtomicInteger`
