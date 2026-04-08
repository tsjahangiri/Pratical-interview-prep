# Data Structures in Java — Part 5: Advanced & Senior-Level Insights

---

## 1. HashMap Deep Dive — How It Really Works

This is the most common senior interview topic. You must be able to explain the entire lifecycle.

---

### The Bucket Array

A `HashMap` is internally an array called `table`. Each slot in the array is a **bucket**. The initial size is 16 (capacity).

```
table[] (capacity = 16):
[0]  → null
[1]  → Entry("Bob", 42)
[2]  → null
...
[7]  → Entry("Alice", 30)
...
[12] → Entry("Dave", 25) → Entry("Eve", 31)  ← collision: linked list
...
[15] → null
```

### Hashing — How the Bucket Index Is Computed

```java
// Step 1: get Java hash
int rawHash = key.hashCode();

// Step 2: apply secondary hash (spreads bits better, reduces clustering)
// Java 8 implementation:
int h = rawHash ^ (rawHash >>> 16); // XOR upper 16 bits with lower 16 bits

// Step 3: compute bucket index
// capacity is always a power of 2 (16, 32, 64...)
// Using & instead of % is faster and gives same result for powers of 2
int index = h & (capacity - 1);
// e.g. capacity=16: h & 15 = h & 0b1111 = last 4 bits of h
```

**Why capacity is always a power of 2:** `h % 16` = `h & 15` only when 16 is a power of 2. The bitwise AND is significantly faster.

---

### Collision Resolution — Chaining

When two keys hash to the same bucket (collision), Java uses a **linked list** of entries in that bucket. Each entry stores key, value, hashCode, and next pointer.

```java
// Simplified Entry structure (Java 8)
static class Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next; // pointer to next entry in bucket (null if no collision)
}
```

**On get("Bob") when bucket [1] has a chain:**
```
[1] → Node("Bob", 42) → Node("Alice", 30) → null

1. Compute hash("Bob") → index = 1
2. Traverse chain from head
3. Compare hash first (fast) — then equals (only if hash matches)
4. Return value when found
```

---

### Java 8 Upgrade — Tree Bins

When any bucket accumulates ≥ 8 entries (due to many collisions), Java converts that bucket's linked list to a **Red-Black Tree**. This changes worst-case performance from O(n) to O(log n).

```
Before (8+ collisions in bucket [3]):
[3] → A → B → C → D → E → F → G → H    O(n) lookup

After treeification:
[3] → Red-Black Tree (D at root)         O(log n) lookup
           D
          / \
         B   F
        / \ / \
       A  C E  G
                \
                 H
```

When the bucket shrinks back below 6 entries, it converts back to a linked list.

---

### Resize (Rehashing)

When `size > capacity × loadFactor (0.75 default)`:
- Capacity doubles (16 → 32 → 64...)
- **Every entry is rehashed** into the new table
- This is O(n) and temporarily doubles memory usage
- Happens infrequently — amortized cost is O(1) per put

```
capacity=16, loadFactor=0.75
Resize triggers when size > 16 × 0.75 = 12

capacity=32, resize triggers when size > 32 × 0.75 = 24
capacity=64, resize triggers when size > 64 × 0.75 = 48
```

**Senior tip — pre-size to avoid rehashing:**
```java
// Will insert 10,000 entries — avoid 4 resizes (at 12, 24, 48, 96...)
int expectedSize = 10_000;
Map<String, Object> map = new HashMap<>((int)(expectedSize / 0.75) + 1);
// Sets initial capacity to 13334 → next power of 2 = 16384 → no resize needed
```

---

### What Happens When Two Keys Have the Same hashCode?

This is a collision — handled by chaining. Correctness is maintained. Performance degrades.

```java
// Artificial example: keys that always collide
class BadKey {
    String value;
    @Override public int hashCode() { return 42; } // all instances hash to 42!
    @Override public boolean equals(Object o) {
        return o instanceof BadKey && ((BadKey)o).value.equals(this.value);
    }
}

Map<BadKey, String> map = new HashMap<>();
// All entries go to the same bucket
// get() degrades to O(n) for linked list, O(log n) for tree bucket
```

**Interview question:** "What happens if all keys have the same hashCode?"
Answer: All entries go to one bucket. LinkedList until 8 entries, then Red-Black Tree. get/put becomes O(log n). This is a hash collision attack — some systems use randomized hashing to prevent it.

---

## 2. Why LinkedList Is Rarely Used in Production

Three reasons LinkedList loses to ArrayList and ArrayDeque in almost every real scenario:

### CPU Cache Miss Problem

```
ArrayList (array in memory):
[A][B][C][D][E][F][G][H]
 ↑ CPU loads this chunk into cache line (64 bytes)
   All 8 elements available — 0 additional cache misses to iterate

LinkedList (scattered nodes):
[A] at 0x1000 → [B] at 0x3F40 → [C] at 0x0A20 → [D] at 0x7810
     ↑ each arrow = potential cache miss
     For 1M elements: potentially 1M cache misses vs ~15K for ArrayList
```

Benchmark reality: iterating 1M elements — LinkedList is 3–10× slower than ArrayList due to cache behavior, even though both are O(n).

### Memory Overhead Per Node

```
ArrayList element (Integer):
  16 bytes (Integer object) + 4 bytes (array reference) = ~20 bytes

LinkedList node (Integer):
  16 bytes (Integer object)
+ 16 bytes (Node object header)
+ 8 bytes (next pointer)
+ 8 bytes (prev pointer)
= ~48 bytes per element — 2.4× more memory
```

### ArrayDeque Beats LinkedList Even for Its Strengths

```java
// LinkedList's supposed advantage: O(1) add/remove at both ends
// ArrayDeque has the SAME O(1) complexity with BETTER cache performance

// LinkedList:
LinkedList<String> ll = new LinkedList<>();
ll.addFirst("A"); ll.addLast("B"); ll.removeFirst(); ll.removeLast();

// ArrayDeque: same O(1), uses a circular array, cache-friendly
ArrayDeque<String> ad = new ArrayDeque<>();
ad.addFirst("A"); ad.addLast("B"); ad.removeFirst(); ad.removeLast();

// Benchmark: ArrayDeque is 2-3× faster than LinkedList as a queue
```

**When LinkedList IS appropriate (rare):**
- You have an `Iterator` positioned mid-list and need O(1) insert at that position
- You are implementing a data structure that requires splicing lists together
- Memory churn from ArrayList resizing is a measured problem in your specific case

---

## 3. When NOT to Use Certain Data Structures

---

### When NOT to Use HashMap

```java
// 1. When you need sorted output
// HashMap.keySet() order is undefined and can change on resize
// Use TreeMap if sorted keys matter

// 2. When keys are mutated after insertion
// Use only immutable keys (String, Integer, Long, records)

// 3. In concurrent code without external synchronization
Map<String, Integer> map = new HashMap<>();
// Two threads putting simultaneously → possible infinite loop (Java 6), data corruption (Java 7)
// Use ConcurrentHashMap instead

// 4. When you have very few entries (< 5)
// HashMap overhead (array allocation, hashing) outweighs its benefits
// For tiny maps, a simple array of pairs or EnumMap may be better

// 5. When keys are int/long and you need a range → use array or TreeMap
int[] frequency = new int[26]; // frequency['a'-'a'] — 100× faster than HashMap<Character,Integer>
```

### When NOT to Use ArrayList

```java
// 1. Frequent insertion at the front or middle
// Shifting O(n) elements on each insert becomes O(n²) total
// Use LinkedList (rare), or rethink your algorithm

// 2. As a stack or queue — use ArrayDeque instead
// ArrayDeque.push/pop/offer/poll are cleaner and slightly faster

// 3. When you know the exact size upfront and it never changes — use array
// int[] vs Integer[] — no boxing, no resizing overhead, pure speed
```

### When NOT to Use Recursion (for tree/graph traversal)

```java
// For very deep trees/graphs, recursive DFS can overflow the call stack
// Default JVM stack size: ~512KB-1MB → ~10,000-20,000 frames

// WRONG for deep trees (> 10,000 depth)
public void dfs(TreeNode node) {
    if (node == null) return;
    dfs(node.left);  // may cause StackOverflowError for deep trees
    dfs(node.right);
}

// CORRECT — iterative DFS with explicit stack
public void dfsIterative(TreeNode root) {
    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);
    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        // process node
        if (node.right != null) stack.push(node.right);
        if (node.left  != null) stack.push(node.left);
    }
}
```

---

## 4. Concurrency Considerations

Standard Java collections are **not thread-safe**. Using them from multiple threads without synchronization causes data corruption, lost updates, and infinite loops.

---

### The Concurrent Alternatives

| Single-threaded | Multi-threaded replacement | Notes |
|---|---|---|
| `HashMap` | `ConcurrentHashMap` | Segment-level locking, reads never block |
| `HashSet` | `ConcurrentHashMap.newKeySet()` | No ConcurrentHashSet in JDK |
| `ArrayList` | `CopyOnWriteArrayList` | Write = full copy; best for rare writes |
| `ArrayDeque` | `LinkedBlockingDeque` | Producer-consumer, blocking operations |
| `PriorityQueue` | `PriorityBlockingQueue` | Thread-safe, blocking poll |
| `TreeMap` | `ConcurrentSkipListMap` | Lock-free, sorted, O(log n) |
| `TreeSet` | `ConcurrentSkipListSet` | Lock-free, sorted |

### ConcurrentHashMap — The Most Important

```java
// Not just thread-safe — it also provides atomic compound operations
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Non-atomic (WRONG even with ConcurrentHashMap):
Integer old = map.get("key");
map.put("key", old == null ? 1 : old + 1); // race between get and put!

// Atomic (CORRECT):
map.merge("key", 1, Integer::sum);              // atomic increment
map.putIfAbsent("key", 0);                      // atomic add-if-missing
map.computeIfAbsent("key", k -> new ArrayList<>()); // atomic compute
map.compute("key", (k, v) -> v == null ? 1 : v + 1); // atomic read-modify-write
```

### Collections.synchronized* — Usually the Wrong Choice

```java
// Synchronized wrapper — correct but coarse: ENTIRE map locked per operation
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());

// TRAP — even with synchronized wrapper, iteration requires external sync:
synchronized (syncMap) { // must hold lock for entire iteration
    for (Map.Entry<String, Integer> e : syncMap.entrySet()) {
        // ... if you forget this synchronized block, ConcurrentModificationException
    }
}

// ConcurrentHashMap avoids all this — use it instead
```

---

## 5. Memory Model — How Java Stores Objects

Understanding object memory layout helps explain why primitive arrays outperform object collections.

```
int[] array of 10 elements:
┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│0 │1 │2 │3 │4 │5 │6 │7 │8 │9 │  → 40 bytes total (10 × 4 bytes)
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘   Contiguous, CPU cache loves this

Integer[] array of 10 elements (boxed):
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ref │ref │ref │ref │ref │ref │ref │ref │ref │ref │  → 40 bytes for refs
└─┬──┴─┬──┴────┴────┴────┴────┴────┴────┴────┴────┘
  ↓    ↓
Integer Integer  (each at a random heap location)
object  object   16 bytes header + 4 bytes value = 20 bytes EACH
                 10 objects = 200 bytes + 40 bytes refs = 240 bytes total
                 6× more memory, scattered across heap = cache misses
```

This is why `ArrayList<Integer>` uses 6× more memory than `int[]` and iterates slower.

---

## 6. Advanced Patterns Senior Engineers Use

---

### Pattern 1: Using Map to Count Groups

```java
// Group users by country — Map<country, List<User>>
Map<String, List<User>> byCountry = users.stream()
    .collect(Collectors.groupingBy(User::getCountry));

// Count by status
Map<Status, Long> statusCount = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus, Collectors.counting()));

// Frequency map (manual, more control)
Map<String, Integer> freq = new HashMap<>();
for (String item : items) freq.merge(item, 1, Integer::sum);
```

---

### Pattern 2: Two-Heap Technique (Median of Stream)

```java
// Running median using two heaps
class MedianFinder {
    // lower: max-heap of the smaller half → top = median candidate
    private PriorityQueue<Integer> lower = new PriorityQueue<>(Comparator.reverseOrder());
    // upper: min-heap of the larger half  → top = median candidate
    private PriorityQueue<Integer> upper = new PriorityQueue<>();

    public void addNum(int num) {
        lower.offer(num);
        // Always push lower's max to upper to maintain partition
        upper.offer(lower.poll());
        // Keep lower.size() >= upper.size()
        if (upper.size() > lower.size()) lower.offer(upper.poll());
    }

    public double findMedian() {
        if (lower.size() > upper.size()) return lower.peek();
        return (lower.peek() + upper.peek()) / 2.0;
    }
}
```

---

### Pattern 3: Monotone Stack (Next Greater Element)

```java
// For each element, find the next element that is greater
// Classic: stock span, largest rectangle in histogram
public int[] nextGreaterElement(int[] nums) {
    int[] result = new int[nums.length];
    Arrays.fill(result, -1); // default: no greater element
    Deque<Integer> stack = new ArrayDeque<>(); // stores indices

    for (int i = 0; i < nums.length; i++) {
        // Pop all elements smaller than current — current is their "next greater"
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            result[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return result;
}
// nums = [4, 1, 2, 3]
// result = [-1, 2, 3, -1]
```

---

### Pattern 4: Graph Topological Sort (Dependency Resolution)

```java
// Order tasks respecting dependencies: task B depends on task A → A comes before B
// Used in: build systems, course prerequisites, job scheduling
public List<Integer> topologicalSort(int n, int[][] dependencies) {
    int[] inDegree = new int[n];
    Map<Integer, List<Integer>> graph = new HashMap<>();

    for (int[] dep : dependencies) {
        int prereq = dep[1], task = dep[0];
        graph.computeIfAbsent(prereq, k -> new ArrayList<>()).add(task);
        inDegree[task]++;
    }

    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < n; i++) {
        if (inDegree[i] == 0) queue.offer(i); // start with tasks that have no prerequisites
    }

    List<Integer> order = new ArrayList<>();
    while (!queue.isEmpty()) {
        int task = queue.poll();
        order.add(task);
        for (int dependent : graph.getOrDefault(task, List.of())) {
            inDegree[dependent]--;
            if (inDegree[dependent] == 0) queue.offer(dependent);
        }
    }

    return order.size() == n ? order : List.of(); // empty if cycle detected
}
```

---

## Senior Interview Answer Framework

When asked "How does HashMap work?" — answer in layers:

```
Layer 1 (30 seconds):
"HashMap stores key-value pairs in an array of buckets.
 The bucket index is computed by hashing the key. Average O(1) for get/put."

Layer 2 (60 seconds):
"On collision, entries are chained in a linked list within the bucket.
 Java 8 converts to a Red-Black Tree when a bucket exceeds 8 entries,
 ensuring O(log n) worst case instead of O(n)."

Layer 3 (90 seconds):
"HashMap resizes when size exceeds capacity × 0.75 (load factor).
 All entries are rehashed into a doubled capacity array.
 Keys must be immutable and correctly implement both equals() and hashCode()
 — the contract is: if a.equals(b), then a.hashCode() == b.hashCode()."

Layer 4 (if asked about concurrency):
"HashMap is not thread-safe. Concurrent puts can cause data corruption.
 Use ConcurrentHashMap instead — it uses segment-level locking,
 allows concurrent reads, and provides atomic compound operations
 like computeIfAbsent and merge."
```

This layered answer shows both breadth and depth — exactly what senior interviews test.
