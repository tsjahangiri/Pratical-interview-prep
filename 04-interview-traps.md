# Data Structures in Java — Part 4: Interview Traps & Mistakes

---

## Category 1 — Wrong Data Structure Choice

---

### Trap 1: Using List.contains() in a Loop

```java
// WRONG — O(n²) total — looks innocent, kills performance at scale
List<String> blocklist = Arrays.asList("spam", "ad", "promo");

for (String email : emails) {
    if (blocklist.contains(email.getSubject())) { // O(n) per call!
        block(email);
    }
}

// CORRECT — O(n) total
Set<String> blocklist = new HashSet<>(Arrays.asList("spam", "ad", "promo"));

for (String email : emails) {
    if (blocklist.contains(email.getSubject())) { // O(1) per call
        block(email);
    }
}
```

**Rule:** Any time you call `List.contains()` inside a loop, replace the List with a HashSet.

---

### Trap 2: Using LinkedList When You Think You Need Fast Inserts

```java
// WRONG — LinkedList is almost never the right answer
List<String> logs = new LinkedList<>();
for (String line : readFile()) {
    logs.add(line); // O(1) — but so is ArrayList!
}

// Random access later destroys performance
String entry = logs.get(50000); // O(n) traversal — SLOW

// CORRECT — ArrayList is faster for both sequential adds AND access
List<String> logs = new ArrayList<>();
```

**The misconception:** LinkedList's O(1) insert at known position sounds great, but you almost always need to find that position first (O(n)), and ArrayList's contiguous memory makes iteration 3–10× faster in practice.

---

### Trap 3: Sorting When You Only Need Min or Max

```java
// WRONG — O(n log n) just to get the minimum
List<Integer> prices = fetchPrices();
Collections.sort(prices);
int cheapest = prices.get(0); // O(1) after sort — but sort was wasteful

// CORRECT — O(n) to find min
int cheapest = Collections.min(prices);

// For top K — WRONG
Collections.sort(prices, Comparator.reverseOrder());
List<Integer> topK = prices.subList(0, k); // O(n log n)

// CORRECT — O(n log k)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
for (int price : prices) {
    minHeap.offer(price);
    if (minHeap.size() > k) minHeap.poll();
}
```

---

### Trap 4: Using TreeMap When Order Doesn't Matter

```java
// WRONG — paying O(log n) for operations when you don't need sort order
Map<String, User> users = new TreeMap<>(); // O(log n) per operation

// CORRECT — O(1) when order is irrelevant
Map<String, User> users = new HashMap<>();
```

**Rule:** Only use `TreeMap`/`TreeSet` when you need sorted iteration, range queries, or navigation methods (floor, ceiling). Otherwise, use `HashMap`/`HashSet`.

---

## Category 2 — HashMap Hidden Traps

---

### Trap 5: Mutable Objects as HashMap Keys

This is one of the most dangerous bugs — causes silent data loss.

```java
// WRONG — using a mutable object as a map key
public class UserFilter {
    List<String> roles; // mutable list!

    // equals and hashCode based on 'roles'
    @Override public int hashCode() { return roles.hashCode(); }
    @Override public boolean equals(Object o) { ... }
}

Map<UserFilter, String> cache = new HashMap<>();
UserFilter filter = new UserFilter(Arrays.asList("ADMIN"));
cache.put(filter, "admin-view");

filter.roles.add("VIEWER"); // mutate the key AFTER inserting!

// The key's hashCode changed — it's now in the wrong bucket
System.out.println(cache.get(filter)); // NULL — entry is lost forever
System.out.println(cache.size());      // 1 — entry still exists, just unreachable

// CORRECT — use immutable keys
// Make roles an unmodifiable list, or use a record, or use String
```

**Rule:** HashMap keys must be **effectively immutable** after insertion. Strings, Integer, Long — all safe because they are immutable. Custom classes — only if you guarantee immutability.

---

### Trap 6: Missing hashCode() When You Override equals()

```java
// WRONG — only overriding equals()
public class Employee {
    private int id;
    private String name;

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Employee)) return false;
        Employee other = (Employee) o;
        return this.id == other.id;
    }
    // hashCode() NOT overridden — inherits from Object → uses memory address
}

Set<Employee> set = new HashSet<>();
Employee e1 = new Employee(1, "Alice");
Employee e2 = new Employee(1, "Alice");

System.out.println(e1.equals(e2)); // true  — you said they're equal
set.add(e1);
set.add(e2);
System.out.println(set.size());    // 2 — but they're "equal"! HashSet treats them as different!

// CORRECT — always override both
@Override
public int hashCode() {
    return Objects.hash(id); // same fields as equals()
}
```

**The contract:** If `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must also be true. Violating this breaks all hash-based collections (`HashMap`, `HashSet`, `Hashtable`).

---

### Trap 7: HashMap Iteration While Modifying

```java
// WRONG — ConcurrentModificationException
Map<String, Integer> scores = new HashMap<>();
scores.put("Alice", 50); scores.put("Bob", 30); scores.put("Charlie", 80);

for (String key : scores.keySet()) {
    if (scores.get(key) < 50) {
        scores.remove(key); // modifying while iterating → ConcurrentModificationException
    }
}

// CORRECT — use entrySet iterator with remove()
Iterator<Map.Entry<String, Integer>> it = scores.entrySet().iterator();
while (it.hasNext()) {
    if (it.next().getValue() < 50) {
        it.remove(); // safe removal through iterator
    }
}

// ALSO CORRECT — Java 8+ removeIf
scores.entrySet().removeIf(entry -> entry.getValue() < 50);

// ALSO CORRECT — collect keys to remove, then remove after
Set<String> toRemove = scores.entrySet().stream()
    .filter(e -> e.getValue() < 50)
    .map(Map.Entry::getKey)
    .collect(Collectors.toSet());
toRemove.forEach(scores::remove);
```

---

### Trap 8: getOrDefault vs get Null Check

```java
Map<String, Integer> map = new HashMap<>();
map.put("key", null); // null VALUE is valid in HashMap

// WRONG — doesn't distinguish "key not present" from "key maps to null"
int val = map.get("key") != null ? map.get("key") : 0; // returns 0, but key IS present with null value

// CORRECT — use containsKey when null values are possible
int val = map.containsKey("key") ? map.get("key") : 0;

// SAFEST for non-null default
int val = map.getOrDefault("key", 0); // returns 0 only if key absent
                                       // returns null if key maps to null — careful!
```

---

## Category 3 — Stack and Queue Mistakes

---

### Trap 9: Using java.util.Stack

```java
// WRONG — Stack extends Vector, which is synchronized (slow) and has weird methods
Stack<Integer> stack = new Stack<>();
stack.push(1); stack.push(2);
stack.search(1); // 1-based index — confusing and not part of any interface

// CORRECT — Deque via ArrayDeque
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2);
stack.peek(); stack.pop();
```

Java's own documentation says: **"A more complete and consistent set of LIFO stack operations is provided by the Deque interface and its implementations, which should be used in preference to this class."**

---

### Trap 10: poll() vs remove() — Null vs Exception

```java
Queue<String> queue = new ArrayDeque<>();

// poll() returns null if empty — safe
String item = queue.poll(); // null — no exception

// remove() throws NoSuchElementException if empty — dangerous without guard
String item = queue.remove(); // throws! Don't use without isEmpty() check

// Same for peek vs element
queue.peek();   // null if empty
queue.element();// throws if empty

// Same for offer vs add
queue.offer("x"); // returns false if bounded queue is full
queue.add("x");   // throws if bounded queue is full

// Rule: always use the "safe" versions: offer, poll, peek
```

---

## Category 4 — Complexity Traps

---

### Trap 11: String Concatenation in a Loop

```java
// WRONG — O(n²) because each += creates a new String object
String result = "";
for (String word : words) {
    result += word + " "; // new String object on every iteration
}

// CORRECT — O(n) with StringBuilder
StringBuilder sb = new StringBuilder();
for (String word : words) {
    sb.append(word).append(" ");
}
String result = sb.toString();
```

---

### Trap 12: Autoboxing in Performance-Critical Loops

```java
// WRONG — autoboxing Integer → int and back on every iteration
List<Integer> numbers = new ArrayList<>();
long sum = 0;
for (Integer n : numbers) {
    sum += n; // unboxes Integer to int on every iteration — slow at scale
}

// BETTER — use int[] if you control the data structure
int[] numbers = new int[size];
long sum = 0;
for (int n : numbers) { sum += n; } // no boxing/unboxing

// OR — IntStream
long sum = IntStream.of(numbers).asLongStream().sum();

// For collections — use streams which handle this better
long sum = numbers.stream().mapToLong(Integer::longValue).sum();
```

---

### Trap 13: PriorityQueue.contains() Is O(n)

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();

// TRAP — looks like it should be O(1), is actually O(n)
boolean has = pq.contains(42); // linear scan — not O(log n)

// PriorityQueue is a heap — only the root position is guaranteed
// There is NO fast "does this element exist" — it must scan the array

// If you need fast contains + priority ordering:
// Use a Set alongside the PriorityQueue (common pattern)
Set<Integer> inQueue = new HashSet<>();
PriorityQueue<Integer> pq = new PriorityQueue<>();

pq.offer(42); inQueue.add(42);
boolean has = inQueue.contains(42); // O(1)
```

---

## Category 5 — Edge Cases Interviewers Test

---

### Edge Cases to Always Mention

```java
// 1. Empty input
if (nums == null || nums.length == 0) return new int[]{};

// 2. Single element
if (nums.length == 1) return nums[0];

// 3. All same elements
// nums = [2, 2, 2, 2] — does your algorithm handle duplicates?

// 4. Integer overflow
// nums = [Integer.MAX_VALUE, 1] — does sum overflow int?
// Use long when summing large ints
long sum = (long) a + b; // explicit cast before addition

// 5. Negative numbers
// nums = [-5, -1, -3] — does your min/max still work?
int min = Integer.MAX_VALUE; // correct — works for negatives
int min = 0;                  // WRONG — 0 is not a valid "minimum" for negatives

// 6. Two-pointer: array with two elements
// left=0, right=1 — does your loop condition work?

// 7. Sorted vs unsorted input
// Does your algorithm assume sorted input?
// State this assumption explicitly

// 8. Null values in collections
Map<String, String> map = new HashMap<>();
map.put("key", null);          // null value — valid
map.put(null, "value");        // null key — valid in HashMap
// TreeMap does NOT allow null keys — throws NullPointerException
```

---

### The Integer.MAX_VALUE Trap

```java
// WRONG — integer overflow when array has no subarray sum constraint
int sum = Integer.MIN_VALUE; // okay for tracking max

// WRONG — adding to MAX_VALUE overflows to negative
int boundary = Integer.MAX_VALUE;
boundary = boundary + 1; // -2147483648 — overflow!

// CORRECT — use Long for potentially large sums
long sum = 0;
for (int n : nums) sum += n;

// CORRECT — check before adding
if (a > Integer.MAX_VALUE - b) { // would overflow
    // handle overflow case
}
```

---

### The Off-By-One in Binary Search

```java
// Classic: find target in sorted array
// WRONG — infinite loop when left and right are adjacent
int left = 0, right = nums.length;       // WRONG: right should be length-1
while (left < right) {                   // WRONG for some variants
    int mid = left + right / 2;          // WRONG: operator precedence! Should be (left+right)/2
    int mid = (left + right) / 2;        // overflow risk if left+right > Integer.MAX_VALUE
    int mid = left + (right - left) / 2; // CORRECT — overflow safe
}

// CORRECT binary search template
int left = 0, right = nums.length - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (nums[mid] == target)      return mid;
    else if (nums[mid] < target)  left  = mid + 1;
    else                          right = mid - 1;
}
return -1;
```

---

### HashMap Initial Capacity Trap

```java
// WRONG — default capacity 16, load factor 0.75
// Resizes when size > 12 — causes unnecessary rehashing for large maps
Map<String, String> config = new HashMap<>(); // will resize at 12 entries

// CORRECT — provide initial capacity for large maps
// Formula: expected_size / load_factor + 1
int expectedEntries = 1000;
Map<String, String> config = new HashMap<>((int)(expectedEntries / 0.75) + 1);
// No resizing will occur for up to 1000 entries

// Guava helper (if available)
Map<String, String> config = Maps.newHashMapWithExpectedSize(1000);
```

---

## Quick Trap Reference

| Trap | Symptom | Fix |
|---|---|---|
| `List.contains()` in loop | O(n²) slowness | Use `HashSet` |
| Mutable HashMap key | Data silently lost | Use immutable keys |
| `equals()` without `hashCode()` | Duplicates in Set | Always override both |
| Modify map while iterating | `ConcurrentModificationException` | Use iterator.remove() or removeIf |
| `java.util.Stack` | Slow, confusing API | Use `ArrayDeque` |
| `remove()` on empty queue | Exception instead of null | Use `poll()` |
| String `+=` in loop | O(n²) string building | Use `StringBuilder` |
| `PriorityQueue.contains()` | Accidentally O(n) | Maintain parallel `HashSet` |
| `(left + right) / 2` | Integer overflow | Use `left + (right - left) / 2` |
| Missing null/empty check | NullPointerException | Always guard at method start |
| `int` sum of large arrays | Silent overflow | Use `long` |
| Default HashMap capacity | Excessive resizing | Pre-size with `expectedSize / 0.75 + 1` |
