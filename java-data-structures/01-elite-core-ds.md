# Elite Java Interview Prep — Part 1: Core Data Structures + Decision Frameworks

---

## How to Use This Guide

Every data structure entry follows this structure:
1. **What + Why** — definition and the reason it exists
2. **Internal mechanics** — what Java actually does
3. **Decision framework** — when to choose it, when to reject it
4. **Complexity** — time AND space, with hidden costs
5. **Code** — production-quality with edge case handling
6. **Common failure modes** — where it breaks

---

## The Master Decision Tree

```
START: What do you need to do most often?
│
├── LOOKUP by key → HashMap O(1) avg
│     └── Need sorted keys? → TreeMap O(log n)
│     └── Need insertion order? → LinkedHashMap O(1)
│
├── UNIQUENESS check → HashSet O(1) avg
│     └── Need sorted? → TreeSet O(log n)
│     └── Need insertion order? → LinkedHashSet O(1)
│
├── ORDERED sequence with fast ends → ArrayDeque O(1)
│     └── Need priority? → PriorityQueue O(log n)
│     └── FIFO only? → ArrayDeque as Queue
│     └── LIFO only? → ArrayDeque as Stack
│
├── INDEXED access (by position) → ArrayList O(1)
│     └── Fixed size, primitives? → int[] array (fastest)
│     └── Frequent mid-insert? → rethink algorithm first
│
├── RANGE queries on sorted data → TreeMap / TreeSet
│
├── HIERARCHICAL data → Tree (custom or use TreeMap)
│
├── RELATIONSHIPS between nodes → Graph (adjacency list)
│
└── MINIMUM or MAXIMUM repeatedly → PriorityQueue / Heap
```

---

## 1. Array

### What + Why

A contiguous block of memory. The most fundamental structure — all others are built on top of it (ArrayList, ArrayDeque, PriorityQueue all use arrays internally).

```
Memory layout:
Address: 1000  1004  1008  1012  1016
         ┌─────┬─────┬─────┬─────┬─────┐
         │  5  │ 12  │  3  │  8  │  1  │
         └─────┴─────┴─────┴─────┴─────┘
Index:     0     1     2     3     4

element[i] address = base + (i × sizeof(element))
element[3] = 1000 + (3 × 4) = 1012 → direct memory jump, no traversal
```

### Decision Framework

| Use array when | Avoid array when |
|---|---|
| Size is fixed and known upfront | Size is dynamic or unknown |
| Working with primitives (int[], long[]) | You need objects with methods |
| Maximum raw performance needed | You need frequent insert/delete at middle |
| Passing to system APIs | You need to grow/shrink frequently |
| Implementing other structures | |

**The key insight:** `int[]` beats `ArrayList<Integer>` by 4–6× in iteration speed due to:
- No object header overhead (16 bytes per Integer vs 4 bytes per int)
- No pointer indirection (direct value vs reference to object)
- Perfect CPU cache utilization (no gaps, no pointer chasing)

### Complexity

| Operation | Time | Hidden cost |
|---|---|---|
| Access by index | O(1) | None — pure arithmetic |
| Search (unsorted) | O(n) | Cache-friendly scan |
| Search (sorted, binary) | O(log n) | Must be sorted first |
| Insert at end | O(1) | Only if pre-allocated space |
| Insert at middle | O(n) | Memory copy via System.arraycopy |
| Delete at middle | O(n) | Memory copy |
| **Space** | O(n) | No overhead per element |

### Code — Production Patterns

```java
// Primitive vs boxed — always prefer primitive for performance
int[]     primitiveArr = new int[1000];        // stack-allocatable, no GC pressure
Integer[] boxedArr     = new Integer[1000];    // 6x memory, GC pressure

// Pre-fill
Arrays.fill(primitiveArr, -1);                 // O(n), uses System.arraycopy internally

// Copy — use Arrays.copyOf not manual loop
int[] original = {1, 2, 3, 4, 5};
int[] copy      = Arrays.copyOf(original, original.length);     // same size
int[] extended  = Arrays.copyOf(original, 10);                  // extends with 0s
int[] partial   = Arrays.copyOfRange(original, 1, 4);           // [2, 3, 4]

// Sort
Arrays.sort(original);                          // O(n log n) dual-pivot quicksort
Arrays.sort(original, 2, 5);                   // sort only indices 2..4

// Binary search — ONLY on sorted array
int idx = Arrays.binarySearch(original, 3);    // O(log n), returns negative if not found
// If not found: -(insertion point) - 1
// idx >= 0 → found at idx
// idx < 0  → not found, would insert at -(idx+1)

// 2D array
int[][] grid = new int[3][4];                  // 3 rows, 4 cols
int rows = grid.length;
int cols = grid[0].length;

// Flatten/iterate
for (int[] row : grid) Arrays.fill(row, 0);    // efficient row-by-row
```

### Common Failure Modes

```java
// 1. ArrayIndexOutOfBoundsException — most common
int[] arr = {1, 2, 3};
arr[arr.length];   // WRONG: valid indices are 0..length-1
arr[-1];           // WRONG: no negative indices in Java (unlike Python)

// 2. Integer overflow in midpoint calculation
int mid = (left + right) / 2;         // OVERFLOW if both are near Integer.MAX_VALUE
int mid = left + (right - left) / 2;  // CORRECT — no overflow possible

// 3. Shallow copy confusion
int[][] original = {{1, 2}, {3, 4}};
int[][] shallowCopy = original.clone(); // copies outer array only
shallowCopy[0][0] = 99;               // MODIFIES original[0][0] too!
// Fix: deep copy each row
int[][] deepCopy = Arrays.stream(original).map(int[]::clone).toArray(int[][]::new);

// 4. Modifying array while iterating (for-each hides this)
// for-each on arrays is safe — it's an indexed loop under the hood
// for-each on Collections can throw ConcurrentModificationException
```

---

## 2. ArrayList

### What + Why

A resizable array. The default go-to sequential collection. Backed by an `Object[]` array internally. Provides dynamic sizing while preserving O(1) random access.

### Internal Mechanics — The Resize Cycle

```
Initial: capacity=10, size=0
Object[] elementData = new Object[10];

After 10 adds: [A][B][C][D][E][F][G][H][I][J]  size=10, capacity=10

Add 11th element → RESIZE:
  newCapacity = oldCapacity + (oldCapacity >> 1) = 10 + 5 = 15
  System.arraycopy(old, 0, new, 0, size)   ← O(n) but amortized O(1)
  
  [A][B][C][D][E][F][G][H][I][J][K][_][_][_][_]  size=11, capacity=15
```

**Amortized O(1) insert proof:**
- n elements inserted total
- Copies happen at sizes: 10, 15, 22, 33, 49, ...
- Total copies = 10 + 15 + 22 + ... ≈ 3n
- Cost per element = 3n / n = O(1) amortized

### Decision Framework

```
Use ArrayList when:
  ✓ You primarily READ by index (O(1))
  ✓ You primarily ADD to the end (O(1) amortized)
  ✓ You iterate sequentially (cache-friendly, fast)
  ✓ You don't know the size upfront

Use array (int[]) instead when:
  ✓ Size is fixed
  ✓ Primitives (no boxing cost)
  ✓ Performance is critical

Use ArrayDeque instead when:
  ✓ You need fast add/remove at FRONT
  ✓ You're using it as Stack or Queue

NEVER use LinkedList as a drop-in replacement — see LinkedList section
```

### Complexity

| Operation | Time | Space note |
|---|---|---|
| get(i) | O(1) | |
| set(i, v) | O(1) | |
| add(v) at end | O(1) amortized | Triggers O(n) copy rarely |
| add(i, v) at middle | O(n) | System.arraycopy shifts |
| remove(i) | O(n) | System.arraycopy shifts |
| contains(v) | O(n) | Linear scan |
| size() | O(1) | Stored field |
| **Space** | O(n) | Up to 1.5× wasted capacity |

### Code — Production Patterns

```java
// Always program to the interface
List<String> list = new ArrayList<>();

// Pre-size when you know the approximate size — avoids multiple resizes
List<String> list = new ArrayList<>(10_000);

// Trim to free wasted capacity after bulk load
list.trimToSize(); // reduces internal array to exact size — useful before long-term storage

// Efficient bulk operations
list.addAll(otherList);                            // batch add
list.removeAll(toRemove);                          // batch remove
list.retainAll(toKeep);                            // keep only intersection
boolean changed = list.removeIf(s -> s.isEmpty()); // Java 8+, O(n), no CME risk

// Sorting — always use list.sort or Collections.sort, not your own
list.sort(Comparator.naturalOrder());
list.sort(Comparator.comparing(String::length).thenComparing(Comparator.naturalOrder()));

// Safe iteration with modification
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (predicate(it.next())) it.remove(); // safe
}

// Convert to/from
String[] arr    = list.toArray(new String[0]); // size hint 0 is idiomatic, JVM optimizes
List<String> l2 = Arrays.asList(arr);          // FIXED SIZE — add/remove throws!
List<String> l3 = new ArrayList<>(Arrays.asList(arr)); // mutable copy
List<String> l4 = List.of("a", "b", "c");     // Java 9+ immutable
```

### Common Failure Modes

```java
// 1. ConcurrentModificationException
for (String s : list) {
    if (s.isEmpty()) list.remove(s); // throws CME
}
// Fix: list.removeIf(String::isEmpty)

// 2. Arrays.asList returns fixed-size list
List<String> fixed = Arrays.asList("a", "b");
fixed.add("c"); // UnsupportedOperationException!
// Fix: new ArrayList<>(Arrays.asList(...))

// 3. subList is a VIEW — modifying it modifies original
List<Integer> sub = list.subList(2, 5);
sub.clear(); // removes elements 2-4 from original list!
// Fix: new ArrayList<>(list.subList(2, 5)) for independent copy

// 4. contains() on a List<Integer> with int literal
List<Integer> nums = new ArrayList<>(Arrays.asList(1, 2, 3));
nums.remove(1);    // removes INDEX 1 (element "2"), not VALUE 1!
nums.remove(Integer.valueOf(1)); // removes VALUE 1
```

---

## 3. LinkedList

### What + Why

A doubly-linked list. Each node holds a value plus two pointers (prev/next). No contiguous memory. Implements both `List` and `Deque`.

```
null ← [prev|10|next] ↔ [prev|20|next] ↔ [prev|30|next] → null
        ↑                                          ↑
       head                                       tail

Node structure (Java internals):
  class Node<E> {
      E item;      // 8 bytes reference
      Node<E> next; // 8 bytes reference
      Node<E> prev; // 8 bytes reference
  }
  // + 16 bytes object header = 40 bytes per node MINIMUM
  // Compare: ArrayList: 8 bytes per reference to element
```

### The Honest Performance Reality

```
Myth: LinkedList has O(1) insert anywhere
Truth: Finding the position is O(n). The actual pointer update is O(1).
       Net cost: O(n) — same as ArrayList for mid-list insert.

Myth: LinkedList is better for queues
Truth: ArrayDeque is 2-3× faster as a queue.
       ArrayDeque: contiguous memory, CPU-cache-friendly circular buffer
       LinkedList: scattered nodes, cache miss per node = ~100ns each

Myth: LinkedList saves memory on inserts (no shifting)
Truth: 40 bytes per node vs ArrayList's 8 bytes per reference.
       For 1M elements: LinkedList = ~40MB, ArrayList = ~8MB + 2MB wasted = ~10MB
```

### Decision Framework

```
Use LinkedList ONLY when:
  ✓ You have an Iterator positioned mid-list and need O(1) insert there
  ✓ You're implementing a structure that requires O(1) splice of two lists
  ✓ (Extremely rare in production code)

Use ArrayDeque instead for:
  ✓ Stack (LIFO)
  ✓ Queue (FIFO)
  ✓ Deque (both ends)
  — All O(1) at both ends, cache-friendly, less memory

Use ArrayList instead for:
  ✓ Everything else
```

### Complexity (With Real Hidden Costs)

| Operation | Time (asymptotic) | Real-world cost |
|---|---|---|
| addFirst / addLast | O(1) | Fast — pointer update only |
| removeFirst / removeLast | O(1) | Fast — pointer update only |
| get(i) | O(n) | SLOW — walks from nearest end |
| add(i, v) | O(n) | Walk + O(1) insert |
| Iteration | O(n) | SLOW — cache miss per node |
| **Space per element** | O(1) nodes | 40 bytes vs 8 bytes in ArrayList |

### Code

```java
// If you must use LinkedList — use Deque interface
Deque<String> deque = new LinkedList<>(); // but prefer ArrayDeque
deque.addFirst("A");
deque.addLast("B");
deque.removeFirst();
deque.removeLast();

// The ONE pattern where LinkedList shines: iterator-based mid-insertion
LinkedList<Integer> list = new LinkedList<>(Arrays.asList(1, 2, 4, 5));
ListIterator<Integer> it = list.listIterator(2); // position at index 2
it.add(3); // O(1) insert at iterator position — inserts 3 between 2 and 4
// Result: [1, 2, 3, 4, 5]
// This is genuinely O(1) — no searching needed
```

---

## 4. Stack

### What + Why

LIFO (Last-In-First-Out). The last element pushed is the first popped. Models undo history, recursive call frames, DFS traversal, expression evaluation.

```
PUSH 10 → PUSH 20 → PUSH 30

         ┌────┐
top →    │ 30 │  ← most recently added
         ├────┤
         │ 20 │
         ├────┤
         │ 10 │  ← first added, last out
         └────┘
```

### Decision Framework

```
Use Stack (ArrayDeque) for:
  ✓ Undo/redo functionality
  ✓ Balanced parentheses / bracket matching
  ✓ DFS traversal (iterative)
  ✓ Expression evaluation (infix → postfix)
  ✓ Monotonic stack problems (next greater element, histogram)
  ✓ Backtracking state management
  ✓ Call stack simulation

NEVER use java.util.Stack:
  — Extends Vector (synchronized — slow)
  — Has search(element) which returns 1-based position (confusing)
  — Java's own docs recommend Deque instead
```

### Code

```java
// Correct: Deque as stack
Deque<Integer> stack = new ArrayDeque<>();
stack.push(10);   // addFirst
stack.push(20);
stack.push(30);
int top  = stack.peek();     // 30 — view without removing (null if empty)
int val  = stack.pop();      // 30 — remove and return (throws if empty)
int safe = stack.pollFirst(); // same as pop but returns null if empty
boolean empty = stack.isEmpty();
int size = stack.size();

// Pattern: Monotone decreasing stack (next greater element)
public int[] nextGreater(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>(); // stores indices

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            result[stack.pop()] = nums[i]; // current is the "next greater" for stack top
        }
        stack.push(i);
    }
    return result;
}
```

---

## 5. Queue and PriorityQueue

### What + Why

**Queue:** FIFO — first in, first out. Models any waiting line, BFS traversal, task pipelines.

**PriorityQueue:** Min-heap internally. Always dequeues the **minimum** element first (or maximum with reverse comparator). Models any "serve most important first" scenario.

```
Queue (FIFO):
Enqueue → [D][C][B][A] → Dequeue
           back        front

PriorityQueue (Min-Heap):
             1           ← always dequeues this
           /   \
          3     2
         / \   / \
        5   4 6   7
```

### Decision Framework

```
Use Queue (ArrayDeque) for:
  ✓ BFS graph/tree traversal
  ✓ Level-order processing
  ✓ Producer-consumer (consider BlockingQueue for concurrent)
  ✓ Task processing in order received

Use PriorityQueue for:
  ✓ "Top K" problems (heap of size K)
  ✓ Dijkstra's shortest path
  ✓ Merge K sorted lists
  ✓ Median of data stream
  ✓ Task scheduling by priority
  ✓ Greedy algorithms that need next best choice

Use Deque (ArrayDeque) for:
  ✓ Sliding window maximum/minimum
  ✓ Palindrome checking
  ✓ When you need both Stack and Queue behavior
```

### Complexity

| Operation | Queue (ArrayDeque) | PriorityQueue |
|---|---|---|
| offer / add | O(1) | O(log n) |
| poll / remove | O(1) | O(log n) |
| peek | O(1) | O(1) |
| contains | O(n) | O(n) |
| **Space** | O(n) | O(n) |

### Code

```java
// Queue
Queue<String> queue = new ArrayDeque<>();
queue.offer("first");     // add to back — returns false if full (for bounded)
queue.offer("second");
String front = queue.peek();   // "first" — view front, null if empty
String served = queue.poll();  // "first" — remove front, null if empty
// queue.element() and queue.remove() throw instead of returning null — avoid

// PriorityQueue — min-heap by default
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5); minHeap.offer(1); minHeap.offer(3);
minHeap.peek();  // 1 — O(1)
minHeap.poll();  // 1 — O(log n)

// Max-heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// Custom comparator — by task priority
PriorityQueue<Task> pq = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
              .thenComparingLong(Task::getSubmittedAt) // tie-break by time
);

// Top-K pattern (min-heap of size K keeps K largest)
public int[] topKElements(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int n : nums) {
        minHeap.offer(n);
        if (minHeap.size() > k) minHeap.poll(); // evict smallest
    }
    return minHeap.stream().mapToInt(i -> i).toArray();
}
```

### Common Failure Modes

```java
// 1. PriorityQueue.contains() is O(n) — not O(log n)
// Elements are in heap order, not sorted — linear scan needed
pq.contains(x); // O(n) — often surprises people

// 2. PriorityQueue iteration is NOT sorted order
for (Integer i : pq) { } // iterates in heap array order, NOT priority order
// To iterate in order: poll() repeatedly, or convert to sorted list

// 3. Null elements throw NullPointerException
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(null); // NullPointerException — PQ cannot contain null

// 4. Modifying objects after insertion breaks heap order
PriorityQueue<Task> pq = new PriorityQueue<>(Comparator.comparingInt(Task::getPriority));
Task t = new Task(5);
pq.offer(t);
t.setPriority(1); // heap order now broken — t is not at the correct position
// Fix: remove, modify, re-insert
```

---

## 6. HashMap

### What + Why

Key-value store with O(1) average access. The most used collection in backend Java. Powers caching, frequency counting, grouping, deduplication with data.

### Internal Mechanics (Full Depth)

```
Step 1: hashCode()
  key.hashCode() → raw int (e.g., "Alice".hashCode() = 63438891)

Step 2: Secondary hash (spread bits)
  h = rawHash ^ (rawHash >>> 16)
  Mixes upper and lower 16 bits → reduces clustering in low bits

Step 3: Bucket index
  capacity must be power of 2 (16, 32, 64, ...)
  index = h & (capacity - 1)
  = h % capacity  (but bitwise AND is faster for powers of 2)

Step 4: Collision handling
  If bucket[index] is empty → place entry
  If bucket[index] has entries:
    - Compare hash first (cheap int comparison)
    - If hash matches: compare with equals() (potentially expensive)
    - If equals() true: update value
    - If equals() false: append to chain (linked list or tree)

Step 5: Tree conversion (Java 8+)
  When bucket chain length ≥ 8 AND total capacity ≥ 64:
    Convert linked list → Red-Black Tree
    get/put: O(n) → O(log n) per bucket

Step 6: Resize
  When size > capacity × loadFactor (default 0.75):
    newCapacity = oldCapacity × 2
    Rehash ALL entries into new array
    Cost: O(n) amortized to O(1) per insertion
```

```
Before resize (capacity=16, size=12, load=0.75):
[0]  null
[1]  Entry(K1,V1)
[5]  Entry(K2,V2) → Entry(K3,V3)   ← collision chain
...

After resize (capacity=32):
Every entry rehashed: index = hash & 31 (was & 15)
Entries that collided before may now separate into different buckets
```

### Decision Framework

```
Use HashMap when:
  ✓ O(1) lookup by key is needed
  ✓ Key ordering doesn't matter
  ✓ Keys are immutable (String, Integer, Long, enums, records)
  ✓ Single-threaded or externally synchronized

Use TreeMap instead when:
  ✓ Keys must be sorted
  ✓ Need range queries (floorKey, ceilingKey, subMap)
  ✓ O(log n) is acceptable

Use LinkedHashMap instead when:
  ✓ Insertion order matters (predictable output, testing)
  ✓ Implementing LRU cache (access-order mode)

Use ConcurrentHashMap instead when:
  ✓ Multiple threads read/write without external sync
  ✓ Need atomic compound operations (merge, computeIfAbsent)

NEVER use HashMap when:
  — Keys are mutable (silent data loss on mutation)
  — equals() is overridden but hashCode() is not (duplicates in set)
  — Thread safety needed without synchronization
```

### Complexity

| Operation | Average | Worst | After Java 8 treeify |
|---|---|---|---|
| get | O(1) | O(n) | O(log n) per bucket |
| put | O(1) | O(n) | O(log n) per bucket |
| remove | O(1) | O(n) | O(log n) per bucket |
| containsKey | O(1) | O(n) | O(log n) per bucket |
| Iteration | O(capacity + n) | — | — |
| **Space** | O(n) | — | 32 bytes per entry |

**Hidden cost:** Iteration is O(capacity + n) — it scans empty buckets too. A HashMap with capacity=10000 but size=10 is slow to iterate.

### Code — Production Patterns

```java
// Construction with pre-sized capacity — avoids resize for large maps
int expectedEntries = 1000;
Map<String, Integer> map = new HashMap<>((int)(expectedEntries / 0.75f) + 1);

// The critical methods (use these, not get-then-put)
map.getOrDefault(key, 0);                          // safe get
map.putIfAbsent(key, defaultValue);                // add only if missing
map.merge(key, 1, Integer::sum);                   // increment counter
map.computeIfAbsent(key, k -> new ArrayList<>());  // lazy initialize
map.compute(key, (k, v) -> v == null ? 1 : v + 1); // full control

// Iteration — entrySet is most efficient
for (Map.Entry<String, Integer> e : map.entrySet()) {
    System.out.println(e.getKey() + " → " + e.getValue());
}
// Java 8+
map.forEach((k, v) -> process(k, v));

// Group-by pattern
Map<String, List<User>> byDept = new HashMap<>();
for (User u : users) {
    byDept.computeIfAbsent(u.getDept(), k -> new ArrayList<>()).add(u);
}
// Or with streams:
Map<String, List<User>> byDept = users.stream()
    .collect(Collectors.groupingBy(User::getDept));

// Frequency count
Map<Character, Integer> freq = new HashMap<>();
for (char c : s.toCharArray()) freq.merge(c, 1, Integer::sum);
```

### Common Failure Modes

```java
// 1. Mutable key — silent data loss
Set<List<Integer>> set = new HashSet<>();
List<Integer> key = new ArrayList<>(Arrays.asList(1, 2, 3));
set.add(key);
key.add(4);                    // mutate after adding
set.contains(key);             // FALSE — hashCode changed, key in wrong bucket

// 2. equals() without hashCode()
class Point { int x, y;
    @Override public boolean equals(Object o) { ... } // no hashCode!
}
Set<Point> set = new HashSet<>();
set.add(new Point(1,1));
set.contains(new Point(1,1)); // FALSE — uses Object.hashCode() = identity

// 3. HashMap in concurrent code
Map<String, Integer> map = new HashMap<>();
// Thread A: map.put("key", 1);
// Thread B: map.put("key", 2);
// Result: undefined behavior, possible infinite loop (pre-Java 8), data corruption

// 4. Iterating while modifying
for (String key : map.keySet()) {
    if (shouldRemove(key)) map.remove(key); // ConcurrentModificationException
}
// Fix:
map.entrySet().removeIf(e -> shouldRemove(e.getKey()));
```

---

## 7. TreeMap

### What + Why

A Red-Black Tree (self-balancing BST) storing entries sorted by key. The same structure guarantees O(log n) for ALL operations — no amortized, no worst-case degradation.

```
Keys: 5, 3, 8, 1, 4, 7, 9

Red-Black Tree (balanced — height ≤ 2 log n):
           5 (black)
         /   \
        3(red) 8(red)
       / \   / \
      1   4 7   9
  (blk)(blk)(blk)(blk)

Property: no red node has a red parent → height bounded
Result: search always O(log n), never degrades
```

### Decision Framework

```
Use TreeMap when:
  ✓ Keys must be in sorted order
  ✓ Need floor/ceiling/range queries (price tiers, time ranges)
  ✓ Need first/last key
  ✓ Need subMap for range iteration
  ✓ O(log n) is acceptable

Use HashMap when:
  ✓ Order doesn't matter and O(1) is needed

Key unique capability: Navigation
  treeMap.floorKey(x)    — largest key ≤ x
  treeMap.ceilingKey(x)  — smallest key ≥ x
  treeMap.subMap(a, b)   — all keys in [a, b)
  treeMap.headMap(x)     — all keys < x
  treeMap.tailMap(x)     — all keys ≥ x
```

### Code

```java
TreeMap<Integer, String> priceTiers = new TreeMap<>();
priceTiers.put(0,    "Free");
priceTiers.put(100,  "Starter");
priceTiers.put(500,  "Pro");
priceTiers.put(2000, "Enterprise");

// Price tier lookup — "what tier does price X fall into?"
int price = 350;
Map.Entry<Integer, String> tier = priceTiers.floorEntry(price); // {100, "Starter"}
String tierName = tier.getValue(); // "Starter"

// Time-range query
TreeMap<Long, List<Event>> eventLog = new TreeMap<>();
long from = System.currentTimeMillis() - 3600_000L; // last hour
long to   = System.currentTimeMillis();
eventLog.subMap(from, true, to, true).forEach((ts, events) -> process(events));

// Top K scores — navigate from end
TreeMap<Integer, String> leaderboard = new TreeMap<>();
leaderboard.put(99, "Alice");
leaderboard.put(87, "Bob");
leaderboard.put(95, "Charlie");

// Iterate top 2 in descending order
leaderboard.descendingMap().entrySet().stream()
    .limit(2)
    .forEach(e -> System.out.println(e.getValue() + ": " + e.getKey()));
// Charlie: 95, Alice: 99  → Alice: 99, Charlie: 95
```

---

## 8. Heap (PriorityQueue)

### What + Why

A complete binary tree stored as an array satisfying the heap property: parent ≤ children (min-heap). The root is always the minimum. O(1) to find min, O(log n) to remove it.

```
Min-Heap stored as array:
           1
         /   \         Array: [1, 3, 2, 7, 5, 4, 6]
        3     2                0  1  2  3  4  5  6
       / \   / \
      7   5 4   6       For index i:
                          parent      = (i-1) / 2
                          left child  = 2*i + 1
                          right child = 2*i + 2
```

### The Three Core Heap Patterns (Know All Three)

**Pattern 1: Find K largest elements (use MIN-heap of size K)**
```java
// Counterintuitive: to find K LARGEST, use a MIN-heap
// The heap maintains the K largest seen so far
// The minimum of those K is at the top — evict it when a larger element comes in

PriorityQueue<Integer> minHeap = new PriorityQueue<>();
for (int num : nums) {
    minHeap.offer(num);
    if (minHeap.size() > k) minHeap.poll(); // remove smallest of K
}
// heap contains K largest — top is Kth largest
```

**Pattern 2: Find K smallest elements (use MAX-heap of size K)**
```java
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
for (int num : nums) {
    maxHeap.offer(num);
    if (maxHeap.size() > k) maxHeap.poll(); // remove largest of K
}
// heap contains K smallest
```

**Pattern 3: Merge K sorted sources (use MIN-heap)**
```java
// E.g., merge K sorted arrays or K sorted streams
PriorityQueue<int[]> minHeap = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
// a[0] = value, a[1] = which array, a[2] = index in that array

for (int i = 0; i < arrays.length; i++) {
    if (arrays[i].length > 0) minHeap.offer(new int[]{arrays[i][0], i, 0});
}
List<Integer> result = new ArrayList<>();
while (!minHeap.isEmpty()) {
    int[] curr = minHeap.poll();
    result.add(curr[0]);
    int nextIdx = curr[2] + 1;
    if (nextIdx < arrays[curr[1]].length) {
        minHeap.offer(new int[]{arrays[curr[1]][nextIdx], curr[1], nextIdx});
    }
}
```

---

## 9. Graph

### What + Why

Nodes (vertices) connected by edges. Models relationships: social networks, roads, dependencies, state machines.

### Representation Decision

```
Adjacency List vs Adjacency Matrix:

Adjacency List (Map<Node, List<Node>>):
  Space: O(V + E)
  Check edge: O(degree of node)
  List neighbors: O(degree)
  Best for: sparse graphs (most real-world graphs)

Adjacency Matrix (int[V][V]):
  Space: O(V²)
  Check edge: O(1) — matrix[u][v]
  List neighbors: O(V)
  Best for: dense graphs, or when frequent "does edge X→Y exist" queries needed
```

### BFS vs DFS Decision

```
Use BFS when:
  ✓ Shortest path in UNWEIGHTED graph
  ✓ Level-order / distance from source
  ✓ "Minimum number of steps" problems
  ✓ Finding nearest neighbor

Use DFS when:
  ✓ Cycle detection
  ✓ Topological sort
  ✓ Connected components
  ✓ Path existence (not shortest)
  ✓ Maze solving / backtracking
  ✓ Tree problems (pre/in/post-order)

Key difference:
  BFS: explores level by level → guarantees shortest path in unweighted graph
  DFS: explores depth first → does NOT guarantee shortest path
```

### Code

```java
// Graph with adjacency list
public class Graph<T> {
    private final Map<T, List<T>> adj = new HashMap<>();

    public void addEdge(T from, T to) {
        adj.computeIfAbsent(from, k -> new ArrayList<>()).add(to);
        adj.computeIfAbsent(to,   k -> new ArrayList<>()).add(from); // undirected
    }

    // BFS — shortest path (unweighted)
    public int shortestPath(T start, T end) {
        if (start.equals(end)) return 0;
        Set<T> visited = new HashSet<>();
        Queue<T> queue = new ArrayDeque<>();
        queue.offer(start); visited.add(start);
        int dist = 0;
        while (!queue.isEmpty()) {
            int size = queue.size();
            dist++;
            for (int i = 0; i < size; i++) {
                T node = queue.poll();
                for (T neighbor : adj.getOrDefault(node, List.of())) {
                    if (neighbor.equals(end)) return dist;
                    if (visited.add(neighbor)) queue.offer(neighbor);
                }
            }
        }
        return -1; // not connected
    }

    // DFS — cycle detection
    public boolean hasCycle() {
        Set<T> visited  = new HashSet<>();
        Set<T> inStack  = new HashSet<>();
        for (T node : adj.keySet()) {
            if (!visited.contains(node)) {
                if (dfsHasCycle(node, visited, inStack)) return true;
            }
        }
        return false;
    }
    private boolean dfsHasCycle(T node, Set<T> visited, Set<T> inStack) {
        visited.add(node); inStack.add(node);
        for (T neighbor : adj.getOrDefault(node, List.of())) {
            if (!visited.contains(neighbor) && dfsHasCycle(neighbor, visited, inStack)) return true;
            if (inStack.contains(neighbor)) return true; // back edge → cycle
        }
        inStack.remove(node);
        return false;
    }
}
```

---

## Master Complexity Reference

```
Structure        │ Access │ Search │ Insert │ Delete │ Space │ Notes
─────────────────┼────────┼────────┼────────┼────────┼───────┼──────────────────────
int[]            │ O(1)   │ O(n)   │ O(n)   │ O(n)   │ O(n)  │ primitives, no GC
ArrayList        │ O(1)   │ O(n)   │ O(1)*  │ O(n)   │ O(n)  │ *amortized, end only
LinkedList       │ O(n)   │ O(n)   │ O(1)†  │ O(1)†  │ O(n)  │ †if you have the node
ArrayDeque       │ O(1)‡  │ O(n)   │ O(1)   │ O(1)   │ O(n)  │ ‡only at ends
HashMap          │ -      │ O(1)   │ O(1)   │ O(1)   │ O(n)  │ avg; O(log n) worst (Java8)
TreeMap          │ -      │ O(lgn) │ O(lgn) │ O(lgn) │ O(n)  │ always sorted
HashSet          │ -      │ O(1)   │ O(1)   │ O(1)   │ O(n)  │ same as HashMap
TreeSet          │ -      │ O(lgn) │ O(lgn) │ O(lgn) │ O(n)  │ sorted unique
PriorityQueue    │ O(1)§  │ O(n)   │ O(lgn) │ O(lgn) │ O(n)  │ §peek only (min/max)
BST (balanced)   │ O(lgn) │ O(lgn) │ O(lgn) │ O(lgn) │ O(n)  │ Red-Black in Java
Graph (adj list) │ -      │ O(V+E) │ O(1)   │ O(E)   │ O(V+E)│ BFS/DFS
```
