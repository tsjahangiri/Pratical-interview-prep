# Senior-Level Deep Understanding: equals(), hashCode(), HashMap, HashSet, HashTable

---

## Part 1: The Contract — equals() and hashCode()

### The Golden Rule (Java Specification)

These two methods are bound by an **unbreakable contract**:

> **If `a.equals(b)` is true → `a.hashCode() == b.hashCode()` MUST also be true.**
> The reverse does NOT have to hold — two objects can share the same hashCode and still not be equal. That's called a **collision**.

Break this contract and you get **silent, catastrophic bugs** in any hash-based collection.

---

### How hashCode() Works Internally

A hash code is just an `int` — a numeric fingerprint of an object. Collections like `HashMap` use it to find the right **bucket** (array slot) without scanning every element.

```
hashCode() → bucket index = hash % capacity
```

Java's default `Object.hashCode()` returns a value derived from the object's memory address — so two distinct objects with the same field values will get **different** hash codes unless you override it.

```java
public class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
}

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);

System.out.println(p1.equals(p2));   // false ← uses Object.equals (reference)
System.out.println(p1.hashCode() == p2.hashCode()); // false ← different memory
```

Once you override both:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Point)) return false;
    Point p = (Point) o;
    return x == p.x && y == p.y;
}

@Override
public int hashCode() {
    return Objects.hash(x, y);  // 31 * x + y internally
}
```

Now `p1.equals(p2)` → true, and `p1.hashCode() == p2.hashCode()` → true. ✅

---

### The 31 Multiplier — Why?

`Objects.hash()` and `String.hashCode()` use the formula:

```java
result = 31 * result + field.hashCode();
```

**31** is chosen because:
- It's an odd prime (avoids even-number bit shift issues)
- It can be optimized by the JVM: `31 * i == (i << 5) - i`
- It produces good distribution with low collision rates

---

### equals() Rules (Reflexive, Symmetric, Transitive, Consistent, Non-null)

```java
// Reflexive
x.equals(x) == true

// Symmetric
x.equals(y) == y.equals(x)

// Transitive
if x.equals(y) && y.equals(z) → x.equals(z)

// Consistent (no side effects)
x.equals(y) always returns same result if no fields change

// Non-null
x.equals(null) == false always
```

Violating **symmetry** is the most common trap:

```java
// ❌ Broken symmetry
class Animal {
    String name;
    public boolean equals(Object o) {
        if (o instanceof Dog d) return name.equals(d.name); // Animal can equal Dog
        return false;
    }
}

class Dog extends Animal {
    public boolean equals(Object o) {
        if (!(o instanceof Dog)) return false; // Dog cannot equal Animal
        return name.equals(((Dog) o).name);
    }
}
// animal.equals(dog) → true
// dog.equals(animal) → false  ← VIOLATED
```

---

## Part 2: HashMap — Deep Internals

### Structure

```
HashMap<K, V> internally:
┌─────────────────────────────────────────────────────┐
│  Node<K,V>[] table  (array of buckets)              │
│                                                     │
│  [0] → null                                         │
│  [1] → Node("a", 1) → Node("x", 5) → null  (chain) │
│  [2] → null                                         │
│  [3] → Node("b", 2) → null                          │
│  ...                                                │
└─────────────────────────────────────────────────────┘
```

Each `Node` holds: `hash`, `key`, `value`, `next` (linked list pointer).

### Key Parameters

| Parameter | Default | Meaning |
|---|---|---|
| `initialCapacity` | 16 | Starting number of buckets |
| `loadFactor` | 0.75 | Resize when 75% full |
| `threshold` | capacity × loadFactor | Entry count that triggers resize |

### Put Operation — Step by Step

```java
map.put("foo", 42);
```

1. Compute `hash = hash(key.hashCode())` — applies a **secondary hash** to spread bits
2. Compute `index = (n - 1) & hash` — bitwise AND (faster than modulo, works because capacity is always power of 2)
3. If bucket is empty → insert new Node directly
4. If bucket is occupied → walk the linked list checking `hash == existing.hash && key.equals(existing.key)`
   - Match found → **update** the value
   - No match → **append** new Node to end of chain
5. If `++size > threshold` → **resize** (double capacity, rehash all entries)

### Java 8+ Treeification

When a single bucket's chain reaches **8 nodes**, Java converts it from a linked list to a **Red-Black Tree** (O(log n) instead of O(n)). When it shrinks back to 6, it converts back.

```java
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64; // only treeify if table size ≥ 64
```

### Get Operation

```java
map.get("foo");
```

1. Compute hash → find bucket index
2. Walk the chain (or tree) checking equality
3. Return value or null

**Best case: O(1)** — one bucket lookup, no chain
**Worst case: O(log n)** — treeified bucket (Java 8+), or O(n) if somehow all keys hash to same bucket (pre Java 8)

---

## Part 3: HashSet — Deep Internals

`HashSet` is nothing more than a `HashMap` where every value is a **dummy sentinel object**:

```java
// Actual source code in OpenJDK
private transient HashMap<E, Object> map;
private static final Object PRESENT = new Object(); // dummy value

public boolean add(E e) {
    return map.put(e, PRESENT) == null;
}

public boolean contains(Object o) {
    return map.containsKey(o);
}
```

So every rule about `HashMap` keys applies directly to `HashSet` elements. There's no separate logic.

---

## Part 4: Hashtable — The Legacy Dinosaur

`Hashtable` is the ancient (Java 1.0) predecessor of `HashMap`. Key differences:

| Feature | HashMap | Hashtable |
|---|---|---|
| Thread safety | ❌ Not thread-safe | ✅ Synchronized (method-level) |
| Null keys | ✅ 1 allowed | ❌ NullPointerException |
| Null values | ✅ Allowed | ❌ NullPointerException |
| Performance | Faster | Slower (locks entire table) |
| Inheritance | AbstractMap | Dictionary (legacy) |
| Iterator | Fail-fast | Enumerator (not fail-fast) |
| Java version | Java 2 (1.2) | Java 1.0 |

**Never use `Hashtable` in new code.** Use `ConcurrentHashMap` for thread safety instead.

```java
// ❌ Old
Hashtable<String, Integer> ht = new Hashtable<>();

// ✅ Modern thread-safe alternative
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
```

---

## Part 5: Senior-Level Bugs & Traps

### 🐛 Bug 1 — Mutable Key in HashMap (The Deadliest Trap)

```java
List<Integer> key = new ArrayList<>(Arrays.asList(1, 2, 3));
Map<List<Integer>, String> map = new HashMap<>();
map.put(key, "value");

System.out.println(map.get(key)); // "value" ✅

key.add(4); // MUTATE the key after insertion!

System.out.println(map.get(key)); // null ❌ — hashCode changed, wrong bucket!
System.out.println(map.containsKey(key)); // false ❌
// The entry is LOST — it's in the old bucket, but we're looking in the new one
```

**Rule: Never mutate an object that's being used as a HashMap key.**

---

### 🐛 Bug 2 — Override equals() but Forget hashCode()

```java
class Employee {
    String id;
    Employee(String id) { this.id = id; }

    @Override
    public boolean equals(Object o) {
        return o instanceof Employee e && id.equals(e.id);
    }
    // hashCode() NOT overridden → uses Object's (memory-based)
}

Set<Employee> set = new HashSet<>();
set.add(new Employee("E001"));
set.add(new Employee("E001")); // Same ID, different object

System.out.println(set.size()); // 2 ❌ — should be 1!
// Two different hashCodes → two different buckets → duplicates in Set
```

---

### 🐛 Bug 3 — HashMap in Multithreaded Environment

```java
// Two threads calling map.put() simultaneously on a non-thread-safe HashMap
// In Java 7: could cause an infinite loop during resize (circular linked list)
// In Java 8+: data corruption, lost entries, or ConcurrentModificationException
Map<String, Integer> map = new HashMap<>(); // ❌ not thread-safe

// ✅ Fix:
Map<String, Integer> safe = new ConcurrentHashMap<>();
```

---

### 🐛 Bug 4 — ConcurrentModificationException

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1); map.put("b", 2); map.put("c", 3);

for (String key : map.keySet()) {
    if (key.equals("b")) {
        map.remove(key); // ❌ ConcurrentModificationException!
    }
}

// ✅ Fix: use iterator's remove
Iterator<String> it = map.keySet().iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove(); // safe
}

// ✅ Or Java 8+:
map.entrySet().removeIf(e -> e.getKey().equals("b"));
```

---

### 🐛 Bug 5 — equals() Using instanceof vs getClass()

```java
// instanceof approach — allows subclasses to be equal to parent
public boolean equals(Object o) {
    if (!(o instanceof Point)) return false;
    // ...
}

// getClass() approach — strict, subclass != parent
public boolean equals(Object o) {
    if (o == null || getClass() != o.getClass()) return false;
    // ...
}
```

**Rule: Use `getClass()` for final classes. Use `instanceof` carefully with inheritance.**

---

### 🐛 Bug 6 — null Key Behavior

```java
HashMap<String, Integer> map = new HashMap<>();
map.put(null, 100);
System.out.println(map.get(null)); // 100 ✅ — HashMap supports one null key

Hashtable<String, Integer> ht = new Hashtable<>();
ht.put(null, 100); // ❌ NullPointerException!

ConcurrentHashMap<String, Integer> chm = new ConcurrentHashMap<>();
chm.put(null, 100); // ❌ NullPointerException!
```

---

### 🐛 Bug 7 — Poor hashCode() Causing O(n) Performance

```java
// Terrible hashCode — everything maps to bucket 0
@Override
public int hashCode() {
    return 1; // constant hash!
}
// HashMap becomes a linked list → O(n) for every get/put
```

---

### 🐛 Bug 8 — Integer Key Auto-unboxing NullPointerException

```java
Map<String, Integer> scores = new HashMap<>();
scores.put("Alice", 100);

int score = scores.get("Bob"); // ❌ NullPointerException!
// scores.get("Bob") returns null (Integer)
// auto-unboxing null → NPE

// ✅ Fix:
int score = scores.getOrDefault("Bob", 0);
```

---

## Part 6: Interview Questions — Senior Level

**Q1: What happens if two keys have the same hashCode?**
They land in the same bucket. HashMap walks the chain using `equals()` to find the right entry. Before Java 8 it was a linked list (O(n) worst case); after Java 8 it becomes a Red-Black Tree at 8 nodes (O(log n)).

**Q2: Why is HashMap's capacity always a power of 2?**
So the bucket index can be computed with a fast bitwise AND: `index = (n-1) & hash` instead of the slower modulo `hash % n`. This works perfectly only when `n` is a power of 2.

**Q3: What's the difference between `==`, `equals()`, and `hashCode()`?**
- `==` compares **references** (memory addresses) for objects, values for primitives
- `equals()` compares **logical equality** (overrideable)
- `hashCode()` is a numeric fingerprint for **fast bucketing** in hash structures

**Q4: Can two unequal objects have the same hashCode?**
Yes — that's a **collision**, and it's perfectly legal. What's illegal is two **equal** objects having **different** hashCodes.

**Q5: What is the load factor and why does 0.75 make sense?**
It's the threshold ratio at which the map resizes. At 0.75, there's a mathematical balance between time (collision risk) and space (wasted empty buckets).

**Q6: How does HashMap handle resizing?**
It doubles the capacity and **rehashes** all existing entries into the new table. This is O(n) work but amortized O(1) per insertion.

**Q7: Why should you pre-size a HashMap if you know the number of entries?**

```java
// ✅ Pre-sized: capacity = expectedSize / loadFactor + 1
Map<String, Integer> map = new HashMap<>(1334); // no resize for 1000 entries
```

**Q8: What is `HashMap.compute()` and when do you use it?**

```java
map.compute("score", (k, v) -> v == null ? 1 : v + 1);
map.merge("score", 1, Integer::sum);
map.computeIfAbsent("players", k -> new ArrayList<>()).add("Alice");
```

---

## Summary Mental Model

```
hashCode()  →  which BUCKET to look in   (fast, approximate)
equals()    →  which ENTRY inside bucket  (exact match)

HashMap     →  key-value, 1 null key, not thread-safe, O(1) avg
HashSet     →  just HashMap<E, PRESENT> underneath
Hashtable   →  legacy, synchronized, no nulls — never use it
ConcurrentHashMap → modern thread-safe, no nulls, fine-grained locks
```

The core of all hash-based data structures in Java comes down to one insight: **hashCode narrows the search space, equals confirms the match**. Get that contract wrong, and every hash collection breaks silently.
