# Elite Java Interview Prep — Part 4: Java Deep Dives & System Considerations

---

## Section 1: HashMap Internals — Complete Deep Dive

### Object Memory Layout

Before understanding HashMap, understand how Java lays out objects in memory:

```
Java Object Memory Layout (64-bit JVM, compressed OOPs):
┌──────────────────────────────┐
│ Object Header: 16 bytes      │ mark word (8b) + class pointer (4b) + padding (4b)
├──────────────────────────────┤
│ Instance fields               │
│   int:     4 bytes           │
│   long:    8 bytes           │
│   boolean: 1 byte (padded)   │
│   Object reference: 4 bytes  │ (compressed OOP, 8 without)
└──────────────────────────────┘

HashMap.Node<K,V> actual size:
  Header:    16 bytes
  int hash:   4 bytes
  K key:      4 bytes (reference)
  V value:    4 bytes (reference)
  Node next:  4 bytes (reference)
  Padding:    4 bytes
  Total:     ~36 bytes per entry (plus the key and value objects themselves)

String "hello" actual size:
  Header:    16 bytes
  char[] value: 4 bytes (reference) + 16 byte array header + 10 bytes chars = 30 bytes
  int hash:   4 bytes
  Total:     ~54 bytes

So: map.put("hello", 42) costs roughly 36 + 54 + 16 (Integer) = ~106 bytes
    array: words[i] = "hello" costs ~58 bytes (the String itself)
```

### Full put() Lifecycle

```java
// What happens when you call: map.put("Alice", 30)

// Step 1: hashCode()
int rawHash = "Alice".hashCode(); // Java computes: 31*...*31 + chars
// "Alice".hashCode() = 63438891

// Step 2: Secondary hash (spreads bits to reduce low-bucket clustering)
// Java 8 implementation (HashMap.hash()):
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// XOR with upper 16 bits: ensures upper bits participate in bucket selection
// Without this: many strings with same suffix would all land in same buckets

// Step 3: Bucket index
int index = hash & (capacity - 1); // = hash & 15 for capacity=16
// Equivalent to hash % 16 but faster (bitwise AND on power of 2)

// Step 4: Insert into bucket
Node<K,V>[] table = ...; // the backing array
if (table[index] == null) {
    table[index] = new Node<>(hash, key, value, null); // empty bucket
} else {
    Node<K,V> e = table[index];
    // Walk the chain — check hash first (fast), then equals (only if hash matches)
    if (e.hash == hash && (e.key == key || key.equals(e.key))) {
        e.value = value; // key already exists — update value
    } else {
        for (Node<K,V> p = e; ; p = p.next) {
            if (p.next == null) {
                p.next = new Node<>(hash, key, value, null); // append to chain
                if (binCount >= TREEIFY_THRESHOLD - 1) treeifyBin(table, hash); // maybe convert
                break;
            }
            if (p.next.hash == hash && key.equals(p.next.key)) {
                e = p.next; e.value = value; break; // found in chain — update
            }
        }
    }
}

// Step 5: Check resize
if (++size > threshold) resize(); // threshold = capacity * loadFactor
```

### Treeification Deep Dive

```java
// Java 8 converts a bucket's linked list to a Red-Black Tree when:
// Condition 1: bucket chain length >= TREEIFY_THRESHOLD (8)
// Condition 2: table.length >= MIN_TREEIFY_CAPACITY (64)
//   If table < 64: RESIZE instead of treeify (more buckets = fewer collisions)

// Why 8? Statistical argument:
// Assuming good hashCode distribution, P(chain length >= 8) ≈ 0.00000006
// So treeification is rare — it's a safety net, not the normal path

// Why Red-Black Tree? Why not AVL Tree?
// Red-Black Tree: O(log n) lookup, O(1) rotation on insert/delete (max 2 rotations)
// AVL Tree: O(log n) lookup, O(log n) rotations (stricter balancing)
// For HashMap: inserts and lookups both common → Red-Black wins

// After treeification: TreeNode<K,V> has 6 pointers (parent, left, right, prev, next, red flag)
// Memory: TreeNode is ~52 bytes vs Node's ~36 bytes
// So treeification COSTS more memory — it's a performance, not memory, optimization
```

### Resize Deep Dive

```java
// Resize doubles capacity and rehashes all entries
// Key optimization: entries don't need full rehash — just check one bit!

// Old capacity = 16: index = hash & 0b1111 (last 4 bits)
// New capacity = 32: index = hash & 0b11111 (last 5 bits)

// The new index is either:
//   Same as old index (if bit 5 of hash is 0)
//   OR old index + 16 (if bit 5 of hash is 1)

// This means chains can be split into two without recomputing hash!
// Java 8 takes advantage of this for O(1) per entry rehash (just bit check)

// Example:
// hash = 0b10101: old index = 0b0101 = 5, new index = 0b10101 & 0b11111 = 21 = 5 + 16
// hash = 0b00101: old index = 0b0101 = 5, new index = 0b00101 & 0b11111 = 5 (same!)
```

---

## Section 2: ArrayList vs LinkedList — CPU and Memory Analysis

### Memory Layout Comparison

```
ArrayList with 5 Integers [1,2,3,4,5]:

Object[]  elementData (the backing array):
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ ref1 │ ref2 │ ref3 │ ref4 │ ref5 │ null │ null │ null │ null │ null │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
   4B     4B    4B    4B    4B    (capacity=10, 5 unused slots)

Each Integer object (scattered in heap):
  Integer(1): 16 bytes header + 4 bytes value = 20 bytes
  Integer(2): 20 bytes
  ... × 5 = 100 bytes in Integer objects

Total: 16 (ArrayList header) + 40 (array header + refs) + 100 (Integers) = 156 bytes

LinkedList with same 5 Integers:
  Node(1): 16 header + 4 ref(item) + 4 ref(next) + 4 ref(prev) + padding = 32 bytes
  Node(2..5): 32 bytes each × 5 = 160 bytes in nodes
  Integer objects: 100 bytes
  LinkedList header: 16 bytes + 8 bytes (head ref + tail ref)
  Total: 284 bytes — 82% more memory

For 1 million elements:
  ArrayList: ~8MB (refs) + ~20MB (Integer objects) ≈ 28MB
  LinkedList: ~40MB (nodes) + ~20MB (Integer objects) ≈ 60MB
```

### CPU Cache Behavior

```
CPU cache line = 64 bytes

ArrayList iteration:
  Element refs are contiguous: [ref1][ref2][ref3]...[ref8] in 32 bytes
  CPU loads 8 refs per cache line → prefetcher predicts next line
  L1 cache hit rate: very high (>95% in typical workloads)

LinkedList iteration:
  Each node has: item, next, prev pointers
  Nodes allocated at different times → scattered in heap
  CPU must follow pointer to next node → potential cache miss each node
  L1 cache miss cost: ~4ns, L2 miss: ~12ns, L3 miss: ~40ns, RAM: ~100ns

Benchmark (1M integers, sequential sum):
  int[]       : ~1ms   (no boxing, perfect cache)
  ArrayList   : ~5ms   (boxed, but contiguous refs, good cache)
  LinkedList  : ~25ms  (boxed + scattered nodes + cache misses)
  Ratio: LinkedList is 5× slower than ArrayList for sequential scan
```

---

## Section 3: Autoboxing — The Hidden Tax

### What Autoboxing Costs

```java
// What you write:
Integer x = 5;        // autobox: Integer.valueOf(5)
int y = x;            // unbox: x.intValue()
map.put("key", 5);    // autobox int → Integer for Map<String, Integer>

// What Java actually does:
Integer x = Integer.valueOf(5);    // check cache [-128, 127], create object if outside
int y = x.intValue();              // method call + potential NPE if x is null!

// Integer cache: values -128 to 127 are CACHED (same object reused)
Integer a = 127; Integer b = 127;
System.out.println(a == b);   // TRUE  — same cached object
Integer c = 128; Integer d = 128;
System.out.println(c == d);   // FALSE — different objects (outside cache range)
// This is why you should ALWAYS use .equals() for Integer comparison, never ==
```

### Autoboxing Performance Benchmark

```java
// Cost comparison for 10M operations:
long sum = 0;

// With boxing (Map<Integer, Integer>)
Map<Integer, Integer> boxedMap = new HashMap<>();
for (int i = 0; i < 10_000_000; i++) {
    boxedMap.put(i, i); // autobox key AND value = 20M object creations
    sum += boxedMap.get(i); // unbox during addition
}
// Cost: ~3-4 seconds, ~480MB GC pressure

// Without boxing (using Trove or custom structure)
int[] directArray = new int[10_000_000];
for (int i = 0; i < 10_000_000; i++) {
    directArray[i] = i;
    sum += directArray[i]; // no boxing, no unboxing
}
// Cost: ~40ms, ~40MB, zero GC pressure

// In production: for int-keyed maps or int-valued maps, consider:
// - Eclipse Collections: IntIntMap, IntObjectMap
// - Koloboke: primitive collections
// - Or just design data access patterns to batch operations
```

### Where Autoboxing Causes Subtle Bugs

```java
// 1. NullPointerException on unbox
Map<String, Integer> scores = new HashMap<>();
int score = scores.get("absent"); // NPE! get returns null, unbox null → NPE
int score = scores.getOrDefault("absent", 0); // safe

// 2. == comparison fails above 127
Long a = 1000L, b = 1000L;
if (a == b) { } // FALSE — different objects
if (a.equals(b)) { } // TRUE — correct

// 3. Method overloading resolution surprise
List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3));
list.remove(1);             // removes by INDEX (int overload) — removes element 2
list.remove(Integer.valueOf(1)); // removes by VALUE (Object overload) — removes element 1

// 4. Autoboxing in conditional — NPE
Integer count = null;
int val = condition ? count : 0; // unboxes count → NPE if condition is true!
int val = count != null ? count : 0; // safe
```

---

## Section 4: Concurrent Collections

### Standard vs Concurrent — What Actually Happens

```java
// HashMap in concurrent code — what goes wrong:
Map<String, Integer> map = new HashMap<>();
// Thread A: put("key1", 1)
// Thread B: put("key2", 2) simultaneously

// Problem 1 (pre-Java 8): During resize, two threads can create a cycle in
// the linked list → infinite loop in get() on any thread
// Problem 2: Lost updates — both threads read same slot, both write, one overwrites other
// Problem 3: Partial visibility — one thread sees half-written state

// The fix — use the right tool for the right situation:
```

### ConcurrentHashMap — Internals

```java
// Java 8 ConcurrentHashMap uses:
// 1. CAS (Compare-And-Swap) for lock-free empty-bucket inserts
// 2. synchronized on individual BIN HEADS for collision chains
// 3. No global lock — multiple threads can write to different buckets simultaneously
// 4. Volatile reads on table entries — no lock needed for reads

// Key insight: reads are NEVER blocked (even during resize)
// Resize is incremental: threads HELP with migration (not just one thread)

ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// These are all ATOMIC — safe without external synchronization:
map.putIfAbsent("key", 0);                           // add only if absent
map.merge("key", 1, Integer::sum);                   // atomic increment
map.computeIfAbsent("key", k -> new ArrayList<>());  // atomic lazy init
map.compute("key", (k, v) -> v == null ? 1 : v + 1); // atomic read-modify-write

// These are NOT atomic — still a race condition:
Integer val = map.get("key");
if (val != null) map.put("key", val + 1); // gap between get and put!
// Use merge() or compute() instead
```

### When to Use Which Concurrent Collection

```
ConcurrentHashMap:
  ✓ Most concurrent key-value scenarios
  ✓ When reads dominate (reads never block)
  ✓ When you need atomic compound operations
  ✗ Not for: when you need iteration to be a consistent snapshot

CopyOnWriteArrayList:
  ✓ Event listener lists (many readers, rare modifications)
  ✓ Configuration/allowlist that's read constantly but updated rarely
  ✗ Not for: frequent writes (each write copies the entire array)

LinkedBlockingQueue:
  ✓ Producer-consumer with blocking semantics
  ✓ Thread pool work queues
  ✓ When producers should pause when queue is full

ConcurrentSkipListMap (sorted, concurrent):
  ✓ When you need TreeMap-like operations concurrently
  ✓ Lock-free, O(log n) all operations
  ✓ Range queries under concurrent access

Collections.synchronizedXxx():
  ✗ Avoid: single lock on entire collection = no concurrency
  ✗ Iteration still requires external synchronized block
  ✗ Much worse performance than ConcurrentHashMap
```

---

## Section 5: Real-World System Considerations

### Memory — When Data Structures Blow Up

```java
// Problem: 10M users, each with a HashMap<String, String> of 10 properties

// Naive approach:
List<Map<String, String>> users = new ArrayList<>();
// Memory per user:
//   HashMap object: 48 bytes
//   Object[] table: 16 bytes header + 16 * 4 bytes refs = 80 bytes (capacity=16)
//   10 entries × (Node: 36 bytes + key String: ~50 bytes + value String: ~50 bytes) = 1360 bytes
//   Total per user: ~1.5KB
// 10M users × 1.5KB = 15GB — likely OOM

// Production approach: use an object with fixed fields
class User {
    String name;     // only what you need
    String email;
    int age;
    // ... 10 specific fields
}
// Memory per user: 16 header + 4*reference fields + primitive fields ≈ 80 bytes
// 10M users × 80 bytes = 800MB — manageable

// Even better for analytics: store column-oriented (arrays per property)
String[] names  = new String[10_000_000];
String[] emails = new String[10_000_000];
int[]    ages   = new int[10_000_000];
// No object overhead, cache-friendly for batch operations
```

### Scalability — When O(n) Isn't Good Enough

```java
// Scenario: autocomplete for 10M words, user types prefix
// O(n) scan through 10M strings → ~10-50ms → too slow for real-time

// Better: Trie (prefix tree)
//   Insert: O(L) where L = word length
//   Search prefix: O(L)
//   List all with prefix: O(L + results)

// Better still: external search engine (Elasticsearch, Solr)
//   For true production at scale — don't implement in-memory

// Scenario: real-time ranking/leaderboard, 1M users
// O(n log n) re-sort per score change → too slow
// TreeMap<Integer, Set<String>>: O(log n) per update, O(k) to get top K
// Redis ZSET: O(log n) add/update, O(k) range query — production choice
```

### Thread Safety — Common Patterns

```java
// Pattern 1: Immutable + Replace (most scalable)
// Cache config: one thread writes (infrequently), many threads read
private volatile Map<String, String> config = Collections.emptyMap(); // volatile reference

public void updateConfig(Map<String, String> newConfig) {
    this.config = Map.copyOf(newConfig); // atomic reference swap
    // Readers always see a complete, consistent map (no partial updates)
}
public String getConfig(String key) {
    return config.get(key); // no locking needed — reads an immutable map
}

// Pattern 2: ConcurrentHashMap with atomic operations (for mutable shared state)
private final ConcurrentHashMap<String, AtomicLong> counters = new ConcurrentHashMap<>();

public void increment(String metric) {
    counters.computeIfAbsent(metric, k -> new AtomicLong(0)).incrementAndGet();
}

// Pattern 3: ThreadLocal (per-thread isolation — no sharing needed)
private static final ThreadLocal<SimpleDateFormat> formatter =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
// SimpleDateFormat is not thread-safe — each thread gets its own copy
// Remember: call formatter.remove() in thread pool scenarios!
```

### Latency — Choosing for Real-Time Systems

```
Operation           │ Typical latency  │ Notes
────────────────────┼──────────────────┼─────────────────────────────────
HashMap get/put     │ ~50-100ns        │ + GC pause risk (ms range)
TreeMap get/put     │ ~200-500ns       │ more predictable, no GC pressure difference
PriorityQueue poll  │ ~100-300ns       │ depends on heap size
ArrayList.get(i)    │ ~10-30ns         │ near memory access speed
LinkedList.get(i)   │ ~500ns-5μs       │ cache miss per node
ConcurrentHashMap   │ ~100-200ns       │ read: ~50ns, write: ~200ns (CAS)
synchronized block  │ ~200ns-10μs      │ ~200ns uncontended, μs contended
```

---

## Section 6: Common Failure Modes Reference

### Where Solutions Break in Production

```
Data Structure          │ Failure Mode                              │ Fix
────────────────────────┼───────────────────────────────────────────┼──────────────────────────
HashMap                 │ Keys mutated after insertion              │ Use immutable keys
                        │ equals() without hashCode()              │ Always override both
                        │ Concurrent modification                  │ ConcurrentHashMap
                        │ High collision (bad hashCode)            │ Fix hashCode distribution
                        │ Full traversal is slow (sparse map)      │ Set initial capacity
                        │                                           │
ArrayList               │ grow() during tight loop → GC pause      │ Pre-size with capacity
                        │ ConcurrentModification                   │ removeIf() or iterator
                        │ Arrays.asList() is fixed-size            │ new ArrayList<>(list)
                        │                                           │
PriorityQueue           │ Contains() is O(n) — users expect O(1)  │ Maintain parallel Set
                        │ Null elements throw NPE                  │ Guard null inserts
                        │ Modified objects break heap order        │ Remove, modify, re-add
                        │                                           │
Stack/Recursion         │ StackOverflowError on deep input         │ Convert to iterative
                        │ java.util.Stack is synchronized          │ Use ArrayDeque
                        │                                           │
String operations       │ += in loop is O(n²)                      │ StringBuilder
                        │ split() compiles regex every call        │ Cache Pattern.compile()
                        │ substring() shares backing array (Java 6)│ Fixed in Java 7+
                        │                                           │
int arithmetic          │ Overflow in (left + right) / 2           │ left + (right - left) / 2
                        │ int * int overflow before cast           │ (long)a * b
                        │ Arrays.asList int[] doesn't work         │ use Integer[] or stream
```
