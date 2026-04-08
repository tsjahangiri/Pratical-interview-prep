# Data Structures in Java — Part 2: When to Use What

---

## The Decision Framework

When you face a problem — in an interview or at work — ask these four questions in order:

```
1. Do I need ORDER?
   └── Sorted?     → TreeMap / TreeSet / PriorityQueue
   └── Insertion?  → LinkedHashMap / LinkedHashSet
   └── Don't care? → HashMap / HashSet (fastest)

2. Do I need to find things FAST?
   └── By key     → HashMap (O(1))
   └── By value   → Not a map job — rethink your structure
   └── By rank    → TreeMap (O(log n))

3. Do I need UNIQUE elements?
   └── Yes → Set (HashSet or TreeSet)
   └── No  → List or Multimap (Guava) or Map<K, List<V>>

4. Do I need FAST INSERT/DELETE?
   └── At both ends?       → Deque (ArrayDeque)
   └── By priority?        → PriorityQueue
   └── Anywhere in middle? → LinkedList (but rethink — usually a design smell)
```

---

## Head-to-Head Comparisons

---

### ArrayList vs LinkedList

This is the most common beginner mistake. Most developers reach for `LinkedList` thinking "I need fast inserts" — but in practice, `ArrayList` wins almost every time.

| Operation | ArrayList | LinkedList | Winner |
|---|---|---|---|
| get(index) | O(1) | O(n) | ArrayList |
| add at end | O(1) amortized | O(1) | Tie |
| add at front | O(n) | O(1) | LinkedList |
| add at middle | O(n) | O(n)* | Tie |
| remove at end | O(1) | O(1) | Tie |
| remove at front | O(n) | O(1) | LinkedList |
| Memory per element | ~8 bytes | ~40 bytes | ArrayList |
| Cache performance | Excellent | Poor | ArrayList |

*LinkedList add at middle is O(n) to find the position, then O(1) to insert.

**The hidden cost of LinkedList:** CPU cache lines are 64 bytes. When iterating an `ArrayList`, elements are contiguous — they load together into cache. `LinkedList` nodes are scattered in memory — every node access is a potential cache miss. In benchmarks, iterating a `LinkedList` is **3–10× slower** than `ArrayList` even though both are O(n).

**When to use LinkedList in practice:**
- You need O(1) insert/remove at **both ends** simultaneously → use `ArrayDeque` instead (faster than LinkedList even for this)
- You are implementing a queue or deque → still use `ArrayDeque`
- **Honest answer: almost never in production Java**

```java
// Almost always this:
List<String> list = new ArrayList<>();
Deque<String> deque = new ArrayDeque<>(); // not LinkedList

// Only this when you have a proven profiling need for mid-list insertions
// at the cost of O(n) random access and 5x memory per element
List<String> linked = new LinkedList<>();
```

---

### HashMap vs LinkedHashMap vs TreeMap

| | HashMap | LinkedHashMap | TreeMap |
|---|---|---|---|
| Order | None | Insertion order | Sorted by key |
| get / put | O(1) | O(1) | O(log n) |
| Memory | Low | Medium | Higher (tree nodes) |
| Navigation (floor, ceiling) | No | No | Yes |
| Use case | Fast lookup | Cache (LRU), predictable output | Range queries, ordered data |

```java
// HashMap — fastest, no order
Map<String, Integer> fast = new HashMap<>();

// LinkedHashMap — maintains insertion order, useful for LRU cache
Map<String, Integer> ordered = new LinkedHashMap<>();
ordered.put("C", 3); ordered.put("A", 1); ordered.put("B", 2);
// Iteration: C, A, B — insertion order preserved

// LRU Cache using LinkedHashMap (access-order mode)
Map<String, String> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
        return size() > 100; // evict when over 100 entries
    }
};

// TreeMap — sorted, navigation methods, O(log n)
TreeMap<Integer, String> sorted = new TreeMap<>();
sorted.put(3, "C"); sorted.put(1, "A"); sorted.put(2, "B");
// Iteration: 1:A, 2:B, 3:C — always sorted by key
```

---

### HashSet vs LinkedHashSet vs TreeSet

| | HashSet | LinkedHashSet | TreeSet |
|---|---|---|---|
| Order | None | Insertion order | Sorted |
| add / contains / remove | O(1) | O(1) | O(log n) |
| Memory | Low | Medium | Higher |
| Use case | Uniqueness check | Dedup preserving order | Sorted unique elements |

```java
// HashSet — fastest uniqueness check
Set<String> seen = new HashSet<>();

// LinkedHashSet — deduplicate while preserving order
Set<String> unique = new LinkedHashSet<>(Arrays.asList("C","A","B","A","C"));
// Result: [C, A, B] — duplicates removed, order preserved

// TreeSet — sorted unique elements with navigation
TreeSet<Integer> sorted = new TreeSet<>(Arrays.asList(5, 1, 3, 2, 4));
// Result: [1, 2, 3, 4, 5]
sorted.floor(3);  // 3
sorted.lower(3);  // 2
```

---

### Stack vs Queue vs Deque

| | Stack (ArrayDeque) | Queue (ArrayDeque) | PriorityQueue | Deque (ArrayDeque) |
|---|---|---|---|---|
| Order | LIFO | FIFO | By priority | Both ends |
| Insert | push (front) | offer (back) | offer | addFirst / addLast |
| Remove | pop (front) | poll (front) | poll (min/max) | removeFirst / removeLast |
| Use case | Undo, parentheses, DFS | BFS, task queues | Scheduling, top-K | Sliding window, palindrome |

```java
// All backed by ArrayDeque — use the right interface for your intent
Deque<Integer> stack = new ArrayDeque<>(); // used as LIFO stack
Queue<Integer> queue = new ArrayDeque<>();  // used as FIFO queue
Deque<Integer> deque = new ArrayDeque<>();  // used as both ends
PriorityQueue<Integer> pq = new PriorityQueue<>(); // heap-based priority
```

---

## Real-World Backend Scenarios

---

### Caching — LinkedHashMap (LRU)

**Problem:** You have 10,000 users hitting a product catalog. You want to cache the 500 most recently accessed products. When the cache is full and a new product is accessed, evict the least recently used one.

```java
public class ProductCache {
    private static final int MAX_SIZE = 500;

    // LinkedHashMap with access-order=true: most recently accessed moves to end
    private final Map<Long, Product> cache = new LinkedHashMap<>(16, 0.75f, true) {
        @Override
        protected boolean removeEldestEntry(Map.Entry<Long, Product> eldest) {
            return size() > MAX_SIZE; // auto-evict oldest when full
        }
    };

    public Product get(long productId) {
        return cache.getOrDefault(productId, null);
    }

    public void put(long productId, Product product) {
        cache.put(productId, product);
    }
}
```

**Why LinkedHashMap:** O(1) get and put, maintains LRU order natively, built-in eviction hook.

---

### Frequency Count / Analytics — HashMap

**Problem:** Count how many times each product was viewed in a session log.

```java
public Map<String, Long> countViews(List<String> eventLog) {
    Map<String, Long> viewCount = new HashMap<>();
    for (String productId : eventLog) {
        viewCount.merge(productId, 1L, Long::sum);
    }
    return viewCount;
}

// Find top 5 most viewed products
public List<String> topK(Map<String, Long> viewCount, int k) {
    PriorityQueue<Map.Entry<String, Long>> minHeap =
        new PriorityQueue<>(Comparator.comparingLong(Map.Entry::getValue));

    for (Map.Entry<String, Long> entry : viewCount.entrySet()) {
        minHeap.offer(entry);
        if (minHeap.size() > k) minHeap.poll(); // evict lowest count
    }

    List<String> result = new ArrayList<>();
    while (!minHeap.isEmpty()) result.add(0, minHeap.poll().getKey());
    return result; // ordered from most to least viewed
}
```

---

### Scheduling / Task Queue — PriorityQueue

**Problem:** A background job system where jobs have priorities. Critical jobs run before normal jobs regardless of submission order.

```java
public enum Priority { CRITICAL, HIGH, NORMAL, LOW }

public class Job implements Comparable<Job> {
    String name;
    Priority priority;
    Instant submittedAt;

    @Override
    public int compareTo(Job other) {
        int priorityCompare = this.priority.compareTo(other.priority);
        if (priorityCompare != 0) return priorityCompare;
        return this.submittedAt.compareTo(other.submittedAt); // FIFO within same priority
    }
}

public class JobScheduler {
    private final PriorityQueue<Job> queue = new PriorityQueue<>();

    public void submit(Job job) { queue.offer(job); }

    public Job next() { return queue.poll(); } // always returns highest priority job
}
```

---

### Time-Range Queries — TreeMap

**Problem:** Given a log of server requests with timestamps, efficiently find all requests in a time window.

```java
public class RequestLog {
    // TreeMap<timestamp, request> — sorted by time
    private final TreeMap<Long, List<Request>> log = new TreeMap<>();

    public void add(Request request) {
        log.computeIfAbsent(request.getTimestamp(), k -> new ArrayList<>())
           .add(request);
    }

    // Find all requests between two timestamps — O(log n + k) where k = results
    public Collection<List<Request>> getInRange(long fromMs, long toMs) {
        return log.subMap(fromMs, true, toMs, true).values();
    }

    // Find requests from last 5 minutes
    public Collection<List<Request>> getRecent() {
        long now = System.currentTimeMillis();
        return getInRange(now - 300_000, now);
    }
}
```

---

### Deduplication with Order — LinkedHashSet

**Problem:** A search engine wants to show autocomplete suggestions without duplicates, in the order they were collected.

```java
public List<String> getAutocompleteSuggestions(String prefix, List<String> sources) {
    Set<String> seen = new LinkedHashSet<>(); // dedup + insertion order

    // Priority sources first — their suggestions appear first
    for (String suggestion : getPrioritySuggestions(prefix)) {
        seen.add(suggestion.toLowerCase());
    }
    // Then regular sources
    for (String suggestion : getRegularSuggestions(prefix)) {
        seen.add(suggestion.toLowerCase()); // ignored if already in set
    }

    return new ArrayList<>(seen);
}
```

---

### Graph — Shortest Path (BFS)

**Problem:** In a social network, find the minimum number of connections between two people ("degrees of separation").

```java
public int degreesOfSeparation(Map<String, List<String>> network,
                                String person1, String person2) {
    if (person1.equals(person2)) return 0;

    Set<String> visited = new HashSet<>();
    Queue<String> queue = new ArrayDeque<>();
    queue.offer(person1);
    visited.add(person1);
    int degrees = 0;

    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        degrees++;
        for (int i = 0; i < levelSize; i++) {
            String current = queue.poll();
            for (String connection : network.getOrDefault(current, List.of())) {
                if (connection.equals(person2)) return degrees;
                if (!visited.contains(connection)) {
                    visited.add(connection);
                    queue.offer(connection);
                }
            }
        }
    }
    return -1; // not connected
}
```

---

## Quick Decision Table

| Scenario | Best Structure | Why |
|---|---|---|
| Fast key lookup | HashMap | O(1) average |
| Unique elements, fast check | HashSet | O(1) average |
| Sorted key lookup + range query | TreeMap | O(log n), navigation methods |
| Sorted unique elements | TreeSet | O(log n), floor/ceiling |
| Preserve insertion order + dedup | LinkedHashSet | O(1) + ordered |
| LRU cache | LinkedHashMap | Access-order mode + eviction hook |
| Job scheduling by priority | PriorityQueue | Always serves min/max first |
| Undo / DFS / parentheses | Stack (ArrayDeque) | LIFO |
| BFS / task processing | Queue (ArrayDeque) | FIFO |
| Sliding window max/min | Deque (ArrayDeque) | O(1) both ends |
| Top K elements | PriorityQueue size K | O(n log k) overall |
| Frequency counting | HashMap | O(1) per update |
| Graph shortest path | BFS + Queue | Level-by-level |
| Graph cycle detection | DFS + visited Set | Deep traversal |
| Count distinct | HashSet | O(1) add + size() |
| Leaderboard (sorted scores) | TreeMap/TreeSet | O(log n), ordered |
