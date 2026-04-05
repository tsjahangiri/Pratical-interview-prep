# ConcurrentHashMap vs Collections.synchronizedMap() — Deep Dive

---

## The Core Difference in One Picture

```
Collections.synchronizedMap()          ConcurrentHashMap
──────────────────────────────         ──────────────────────────────
┌─────────────────────────┐            ┌──────┬──────┬──────┬──────┐
│  🔒 ONE LOCK            │            │  🔒  │  🔒  │  🔒  │  🔒  │
│  (entire map locked)    │            │  B0  │  B1  │  B2  │  B3  │
│                         │            │      │      │      │      │
│  Thread A writing  ───► │ ◄── waits  │  T-A │  T-B │  T-C │  T-D │
│  Thread B reading       │  Thread B  │ write│ write│ read │ read │
│  Thread C reading       │  Thread C  │      │      │      │      │
│  Thread D writing       │  Thread D  │  All working simultaneously!
└─────────────────────────┘            └──────┴──────┴──────┴──────┘
  Throughput: ████░░░░░░░░              Throughput: ████████████████
```

---

## Part 1: Collections.synchronizedMap() — How It Actually Works

It's a simple **wrapper** — every single method acquires the **same mutex lock** before doing anything:

```java
// Simplified actual OpenJDK source
public class Collections {
    static class SynchronizedMap<K, V> implements Map<K, V> {
        
        private final Map<K, V> m;     // wrapped map
        final Object mutex;            // THE one lock

        SynchronizedMap(Map<K, V> m) {
            this.m = m;
            this.mutex = this; // lock is the wrapper itself
        }

        public V get(Object key) {
            synchronized (mutex) { return m.get(key); }
        }

        public V put(K key, V value) {
            synchronized (mutex) { return m.put(key, value); }
        }

        public void forEach(BiConsumer<? super K, ? super V> action) {
            synchronized (mutex) { m.forEach(action); }
        }

        // EVERY method → synchronized(mutex) → one lock rules them all
    }
}
```

**This means:**
- Thread A calls `get()` → acquires lock
- Thread B calls `put()` → **blocks and waits**
- Thread C calls `size()` → **blocks and waits**
- Thread D calls `containsKey()` → **blocks and waits**

All threads queue up behind one single lock, even if they're touching completely different keys.

---

## Part 2: ConcurrentHashMap — How It Actually Works

### Java 7: Segment-Based Locking

```
┌─────────────────────────────────────────────┐
│  Segment[0]  Segment[1]  ...  Segment[15]   │
│  🔒 Lock 0   🔒 Lock 1         🔒 Lock 15   │
│  [bucket]    [bucket]          [bucket]      │
│  [bucket]    [bucket]          [bucket]      │
└─────────────────────────────────────────────┘
16 segments by default → 16 threads can write simultaneously
```

### Java 8+: CAS + Bucket-Level Locking (Much Better)

```java
// Simplified internal logic of Java 8 ConcurrentHashMap.put()
final V putVal(K key, V value) {
    int hash = spread(key.hashCode());
    
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f = tabAt(tab, i = (n - 1) & hash);
        
        if (f == null) {
            // ✅ Bucket is empty → use CAS (no lock needed!)
            if (casTabAt(tab, i, null, new Node<>(hash, key, value)))
                break;
        } else {
            // 🔒 Bucket has entries → lock ONLY this bucket's head node
            synchronized (f) {
                // insert or update within this bucket only
            }
        }
    }
}
```

Key insight: **reads are almost always lock-free** using `volatile` fields, and writes only lock the specific bucket being written to. Two threads writing to different buckets never block each other.

---

## Part 3: Side-by-Side Comparison

```java
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
Map<String, Integer> concMap = new ConcurrentHashMap<>();
```

| Feature | synchronizedMap | ConcurrentHashMap |
|---|---|---|
| Locking strategy | One lock for whole map | Per-bucket lock + CAS |
| Read performance | Locks on read | Lock-free reads (volatile) |
| Write performance | All writes serialized | Concurrent writes to diff buckets |
| Null keys | ✅ Allowed (from HashMap) | ❌ NullPointerException |
| Null values | ✅ Allowed | ❌ NullPointerException |
| Iteration | Must manually lock | Weakly consistent, no lock needed |
| Atomic compound ops | ❌ Not atomic | ✅ putIfAbsent, compute, merge |
| ConcurrentModificationException | ✅ Throws it | ❌ Never throws it |
| Introduced | Java 1.2 | Java 1.5 |
| Scalability | Poor under contention | Excellent |

---

## Part 4: The Iteration Trap — The Biggest Bug Source

### synchronizedMap() — You MUST lock manually during iteration

```java
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
syncMap.put("a", 1);
syncMap.put("b", 2);

// ❌ WRONG — not thread-safe even though map is "synchronized"
for (Map.Entry<String, Integer> entry : syncMap.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
    // Another thread can modify map here → ConcurrentModificationException!
}

// ✅ CORRECT — manually synchronize on the map during iteration
synchronized (syncMap) {
    for (Map.Entry<String, Integer> entry : syncMap.entrySet()) {
        System.out.println(entry.getKey() + "=" + entry.getValue());
    }
}
```

### ConcurrentHashMap — Iteration is always safe, no locking needed

```java
Map<String, Integer> concMap = new ConcurrentHashMap<>();
concMap.put("a", 1);
concMap.put("b", 2);

// ✅ Always safe — uses weakly consistent iterator
for (Map.Entry<String, Integer> entry : concMap.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
    concMap.put("c", 3); // perfectly fine during iteration!
}
```

---

## Part 5: Compound Operations — The Subtle Race Condition

### The Check-Then-Act Race Condition with synchronizedMap

```java
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());

// ❌ BROKEN — classic race condition even with synchronizedMap
public void increment(String key) {
    Integer val = syncMap.get(key);     // Thread A reads 5
                                        // Thread B reads 5  ← both see 5!
    if (val == null) val = 0;
    syncMap.put(key, val + 1);         // Thread A writes 6
                                        // Thread B writes 6 ← LOST UPDATE!
    // Expected: 7, Actual: 6
}

// ✅ Fix with synchronizedMap — manually synchronize the whole block
public void increment(String key) {
    synchronized (syncMap) {
        Integer val = syncMap.getOrDefault(key, 0);
        syncMap.put(key, val + 1);
    }
}
```

### ConcurrentHashMap — Atomic Operations Built In

```java
Map<String, Integer> concMap = new ConcurrentHashMap<>();

// ✅ Atomic — no race condition possible
public void increment(String key) {
    concMap.merge(key, 1, Integer::sum);
    // merge is a single atomic operation:
    // if key absent → put 1
    // if key present → apply (oldVal, 1) → Integer::sum
}

// ✅ Other atomic operations:
concMap.putIfAbsent("key", 42);
concMap.computeIfAbsent("key", k -> expensiveOp());
concMap.compute("key", (k, v) -> v == null ? 1 : v + 1);
concMap.replace("key", oldVal, newVal);
```

---

## Part 6: Real-World Performance Demo

```java
import java.util.*;
import java.util.concurrent.*;

public class MapPerformanceTest {

    static final int THREADS = 16;
    static final int OPS_PER_THREAD = 100_000;

    public static void main(String[] args) throws InterruptedException {
        benchmark("synchronizedMap", Collections.synchronizedMap(new HashMap<>()));
        benchmark("ConcurrentHashMap", new ConcurrentHashMap<>());
    }

    static void benchmark(String name, Map<String, Integer> map)
            throws InterruptedException {

        ExecutorService pool = Executors.newFixedThreadPool(THREADS);
        CountDownLatch latch = new CountDownLatch(THREADS);
        long start = System.currentTimeMillis();

        for (int t = 0; t < THREADS; t++) {
            final int threadId = t;
            pool.submit(() -> {
                for (int i = 0; i < OPS_PER_THREAD; i++) {
                    String key = "key" + (i % 1000);
                    if (i % 3 == 0) {
                        map.put(key, threadId);   // 33% writes
                    } else {
                        map.get(key);             // 67% reads
                    }
                }
                latch.countDown();
            });
        }

        latch.await();
        long elapsed = System.currentTimeMillis() - start;
        System.out.printf("%-25s → %d ms%n", name, elapsed);
        pool.shutdown();
    }
}

// Typical results on 16-core machine:
// synchronizedMap          → 2400 ms
// ConcurrentHashMap        → 210 ms   ← ~11x faster under contention
```

---

## Part 7: Real-World Use Cases

### Use Case 1 — Web Request Counter (ConcurrentHashMap wins)

```java
public class RequestCounter {

    // ❌ synchronizedMap — bottleneck, all threads queue up
    private final Map<String, Integer> syncCounter =
        Collections.synchronizedMap(new HashMap<>());

    // ✅ ConcurrentHashMap — threads on different endpoints don't block each other
    private final ConcurrentHashMap<String, LongAdder> counter =
        new ConcurrentHashMap<>();

    public void record(String endpoint) {
        counter.computeIfAbsent(endpoint, k -> new LongAdder()).increment();
    }

    public long getCount(String endpoint) {
        LongAdder adder = counter.get(endpoint);
        return adder == null ? 0 : adder.sum();
    }
}
```

### Use Case 2 — Cache with Lazy Loading (ConcurrentHashMap wins)

```java
public class UserCache {
    private final ConcurrentHashMap<Integer, User> cache = new ConcurrentHashMap<>();

    public User getUser(int userId) {
        // ✅ computeIfAbsent is atomic — DB called ONLY ONCE even with 100 threads
        return cache.computeIfAbsent(userId, id -> {
            System.out.println("Loading from DB: " + id);
            return database.findUser(id);
        });
    }
}

// With synchronizedMap you'd need verbose double-checked locking:
public User getUserSync(int userId) {
    // ❌ Double-checked locking — verbose and error-prone
    User user = syncMap.get(userId);
    if (user == null) {
        synchronized (syncMap) {
            user = syncMap.get(userId); // check again inside lock
            if (user == null) {
                user = database.findUser(userId);
                syncMap.put(userId, user);
            }
        }
    }
    return user;
}
```

### Use Case 3 — When synchronizedMap is Acceptable

```java
// ✅ synchronizedMap is fine when:
// 1. Low concurrency (few threads)
// 2. You need null keys/values
// 3. You need a specific Map implementation underneath (TreeMap, LinkedHashMap)

// Sorted thread-safe map
Map<String, Integer> sortedThreadSafe =
    Collections.synchronizedMap(new TreeMap<>());

// Insertion-order preserving thread-safe map
Map<String, Integer> orderedThreadSafe =
    Collections.synchronizedMap(new LinkedHashMap<>());

// ConcurrentHashMap has no ordering guarantees
```

---

## Part 8: Senior-Level Traps

### Trap 1 — Null Values in ConcurrentHashMap

```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();

map.put("key", null);  // ❌ NullPointerException!

// WHY? You can't distinguish between:
// "key is absent" → get() returns null
// "key maps to null" → get() also returns null
// In a concurrent context this ambiguity causes real bugs.

// ✅ Use a sentinel value instead:
private static final String NULL_SENTINEL = "__NULL__";
map.put("key", NULL_SENTINEL);
```

### Trap 2 — Thinking synchronizedMap Makes Everything Atomic

```java
Map<String, List<String>> map = Collections.synchronizedMap(new HashMap<>());

// ❌ BROKEN — three separate synchronized calls, not one atomic operation
if (!map.containsKey("list")) {          // lock acquired, then released
    map.put("list", new ArrayList<>());  // lock acquired, then released
}
map.get("list").add("item");             // lock acquired, then released
// Another thread can sneak in between ANY of these three calls

// ✅ Fix — manually lock the entire sequence
synchronized (map) {
    map.computeIfAbsent("list", k -> new ArrayList<>()).add("item");
}

// ✅ Better — use ConcurrentHashMap with atomic computeIfAbsent
ConcurrentHashMap<String, List<String>> concMap = new ConcurrentHashMap<>();
concMap.computeIfAbsent("list", k -> new CopyOnWriteArrayList<>()).add("item");
```

### Trap 3 — Weakly Consistent Iteration

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1); map.put("b", 2); map.put("c", 3);

for (String key : map.keySet()) {
    map.put("d", 4);  // might or might not appear in this iteration
    map.remove("b");  // might or might not be iterated
    // No exception thrown, but results are not perfectly predictable
}

// ✅ If you need a consistent snapshot:
Map<String, Integer> snapshot = new HashMap<>(map);
for (Map.Entry<String, Integer> e : snapshot.entrySet()) {
    // perfectly consistent snapshot
}
```

---

## Decision Flowchart

```
Do you need thread safety?
        │
       YES
        │
Do you need null keys or values?
        │
   YES──┘                  NO
   │                        │
   ▼                        ▼
synchronizedMap       High concurrency / many threads?
(with TreeMap or           │
 LinkedHashMap if     YES──┘              NO
 ordering needed)     │                   │
                      ▼                   ▼
              ConcurrentHashMap     Either works,
              (best performance,    ConcurrentHashMap
               atomic operations)  preferred
```

---

## The One-Line Summary

`Collections.synchronizedMap()` is a **blunt instrument** — one lock, complete serialization, iteration still unsafe without extra work. `ConcurrentHashMap` is a **precision tool** — lock-free reads, bucket-level writes, built-in atomic operations, and truly safe concurrent iteration. In any modern multithreaded application with meaningful contention, `ConcurrentHashMap` wins on every front except null support and ordering.
