# Data Structures in Java — Part 1: Core Data Structures

---

## How to Read This Guide

Every data structure is explained with the same structure:
1. **What it is** — plain English definition
2. **How it works inside** — what Java actually does under the hood
3. **Big-O complexity** — time and space costs
4. **When you'd use it** — real-world intuition
5. **Clean Java code** — production-quality examples

---

## 1. Arrays

### What It Is

An array is the most primitive data structure. It stores elements in a **fixed-size, contiguous block of memory**. Every element sits right next to the previous one in RAM.

```
Index:   0     1     2     3     4
       ┌─────┬─────┬─────┬─────┬─────┐
       │ 10  │ 20  │ 30  │ 40  │ 50  │
       └─────┴─────┴─────┴─────┴─────┘
Memory: 1000  1004  1008  1012  1016   (each int = 4 bytes)
```

### How It Works Inside

Because memory is contiguous, Java can jump directly to any element using math:
```
address of element[i] = base_address + (i × element_size)
element[3] = 1000 + (3 × 4) = 1012   → direct jump, no traversal
```

This is why random access is O(1) — it is arithmetic, not searching.

### Big-O Complexity

| Operation | Time | Notes |
|---|---|---|
| Access by index | O(1) | Direct memory jump |
| Search (unsorted) | O(n) | Must scan every element |
| Search (sorted) | O(log n) | Binary search |
| Insert at end | O(1) | If space exists |
| Insert at middle | O(n) | Must shift all elements right |
| Delete at middle | O(n) | Must shift all elements left |
| **Space** | O(n) | n elements |

### Java Code

```java
// Declaration and initialization
int[] numbers = new int[5];          // fixed size, all zeros
int[] primes  = {2, 3, 5, 7, 11};   // inline initialization

// Access — O(1)
int third = primes[2]; // 5

// Iteration
for (int i = 0; i < primes.length; i++) {
    System.out.println(primes[i]);
}

// Enhanced for — when you don't need the index
for (int p : primes) {
    System.out.println(p);
}

// 2D array — matrix
int[][] matrix = new int[3][3];
matrix[0][0] = 1;
matrix[1][1] = 5;
matrix[2][2] = 9;

// Common operations
Arrays.sort(primes);                         // O(n log n)
int idx = Arrays.binarySearch(primes, 5);    // O(log n) — array must be sorted
int[] copy = Arrays.copyOf(primes, 10);      // copy with new size
Arrays.fill(numbers, -1);                    // fill all with -1
System.out.println(Arrays.toString(primes)); // [2, 3, 5, 7, 11]
```

### Real-World Intuition

Use arrays when:
- Size is known and fixed (days of week, RGB values, a fixed config)
- You need maximum raw performance (game loops, image pixel buffers)
- You will mostly read, rarely insert or delete

---

## 2. ArrayList

### What It Is

`ArrayList` is a **resizable array** backed by a plain Java array internally. It gives you dynamic sizing while keeping O(1) random access.

### How It Works Inside

```
Initial capacity = 10

ArrayList after 10 adds:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ A │ B │ C │ D │ E │ F │ G │ H │ I │ J │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

On 11th add — RESIZE triggers:
 1. Create new array of size 10 × 1.5 = 15
 2. Copy all 10 elements to new array   ← O(n) one-time cost
 3. Add new element
 4. Old array is garbage collected

┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ A │ B │ C │ D │ E │ F │ G │ H │ I │ J │ K │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

The resize cost is amortized — it happens rarely enough that the **average cost of add is still O(1)**.

### Big-O Complexity

| Operation | Time | Notes |
|---|---|---|
| get(index) | O(1) | Direct array access |
| add(element) | O(1) amortized | Occasionally O(n) on resize |
| add(index, element) | O(n) | Shifts elements right |
| remove(index) | O(n) | Shifts elements left |
| contains(element) | O(n) | Linear scan |
| size() | O(1) | Stored as a field |
| **Space** | O(n) | May have unused capacity |

### Java Code

```java
// Creation
List<String> cities = new ArrayList<>();
List<String> preset = new ArrayList<>(Arrays.asList("London", "Paris", "Tokyo"));

// Hint capacity upfront to avoid resizing if you know the size
List<String> large = new ArrayList<>(10_000);

// Core operations
cities.add("Berlin");                  // O(1) amortized
cities.add(0, "Amsterdam");            // O(n) — inserts at front, shifts everything
cities.get(0);                         // O(1)
cities.set(0, "Vienna");               // O(1) — replace at index
cities.remove(0);                      // O(n) — removes by index
cities.remove("Berlin");               // O(n) — removes by value (first match)
cities.contains("Paris");             // O(n)
cities.size();                         // O(1)
cities.isEmpty();                      // O(1)

// Iteration — all equivalent for ArrayList
for (String city : cities) { System.out.println(city); }
cities.forEach(System.out::println);
cities.stream().filter(c -> c.startsWith("A")).forEach(System.out::println);

// Sorting
Collections.sort(cities);                              // alphabetical
cities.sort(Comparator.comparingInt(String::length)); // by length

// Sublist — a view, not a copy
List<String> sub = cities.subList(1, 3); // index 1 inclusive to 3 exclusive

// Convert to array
String[] arr = cities.toArray(new String[0]);
```

---

## 3. LinkedList

### What It Is

A `LinkedList` stores elements in **nodes** scattered anywhere in memory. Each node holds its value and a pointer (reference) to the next node. There is no contiguous memory — nodes connect like a chain.

### Singly Linked List — Internal Structure

```
head
 │
 ▼
┌──────┬──────┐    ┌──────┬──────┐    ┌──────┬──────┐
│  10  │  ────┼──→ │  20  │  ────┼──→ │  30  │ null │
└──────┴──────┘    └──────┴──────┘    └──────┴──────┘
  Node              Node               Node (tail)
 (data)(next)      (data)(next)       (data)(next=null)
```

### Doubly Linked List — Java's Implementation

Java's `LinkedList` is a **doubly linked list** — each node has both a `next` and `prev` pointer.

```
null ← [10] ↔ [20] ↔ [30] ↔ [40] → null
        ↑                      ↑
       head                  tail
```

This allows O(1) insertion/removal at **both ends** and efficient backward traversal.

### How It Works Inside — Node Class

```java
// Simplified version of what Java does internally
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

When you call `addFirst("A")`:
1. Create a new Node with value "A"
2. Set its `next` to the current head
3. Set current head's `prev` to the new node
4. Update `head` reference to the new node
5. If list was empty, also update `tail`

### Big-O Complexity

| Operation | Time | Notes |
|---|---|---|
| addFirst / addLast | O(1) | Update head/tail pointer |
| removeFirst / removeLast | O(1) | Update head/tail pointer |
| get(index) | O(n) | Must traverse from head or tail |
| add(index, element) | O(n) | Traverse to position, then O(1) insert |
| contains(element) | O(n) | Linear scan |
| **Space** | O(n) | Extra memory per node for two pointers |

### Java Code

```java
// Java's LinkedList implements both List and Deque
LinkedList<String> list = new LinkedList<>();

// Adding — O(1) at both ends
list.addFirst("B");   // [B]
list.addLast("C");    // [B, C]
list.addFirst("A");   // [A, B, C]

// Removing — O(1) at both ends
String first = list.removeFirst(); // "A"  →  [B, C]
String last  = list.removeLast();  // "C"  →  [B]

// Peeking without removing
String head = list.peekFirst(); // "B" — does not remove
String tail = list.peekLast();  // "B" — does not remove

// Random access — O(n) — avoid this pattern
String middle = list.get(5); // must walk 5 nodes from head

// Use as a Stack (LIFO)
list.push("X"); // addFirst
list.pop();     // removeFirst

// Use as a Queue (FIFO)
list.offer("Y"); // addLast
list.poll();     // removeFirst

// Iteration — efficient (uses internal iterator, not index)
for (String s : list) { System.out.println(s); }
```

### Build Your Own — Singly Linked List (Interview Classic)

```java
public class SinglyLinkedList<T> {
    private Node<T> head;
    private int size;

    private static class Node<T> {
        T data;
        Node<T> next;
        Node(T data) { this.data = data; }
    }

    public void addFirst(T data) {
        Node<T> newNode = new Node<>(data);
        newNode.next = head;
        head = newNode;
        size++;
    }

    public void addLast(T data) {
        Node<T> newNode = new Node<>(data);
        if (head == null) { head = newNode; size++; return; }
        Node<T> current = head;
        while (current.next != null) current = current.next; // traverse to tail
        current.next = newNode;
        size++;
    }

    public T removeFirst() {
        if (head == null) throw new NoSuchElementException();
        T data = head.data;
        head = head.next;
        size--;
        return data;
    }

    public void reverse() {
        Node<T> prev = null, current = head, next = null;
        while (current != null) {
            next = current.next;   // save next
            current.next = prev;   // reverse pointer
            prev = current;        // move prev forward
            current = next;        // move current forward
        }
        head = prev; // prev is now the new head
    }

    public void print() {
        Node<T> current = head;
        while (current != null) {
            System.out.print(current.data + " → ");
            current = current.next;
        }
        System.out.println("null");
    }
}
```

---

## 4. Stack

### What It Is

A Stack is a **Last-In-First-Out (LIFO)** structure. Think of a stack of plates — you add to the top and remove from the top. The last plate placed is the first one taken.

```
       PUSH →  ┌───┐
               │ D │  ← top (last in, first out)
               ├───┤
               │ C │
               ├───┤
               │ B │
               ├───┤
               │ A │  ← bottom (first in, last out)
               └───┘
```

### How It Works Inside

Java's `Stack` class extends `Vector` (legacy, synchronized, slow). In modern code, **use `Deque` backed by `ArrayDeque`** — it is faster and not synchronized.

### Big-O Complexity

| Operation | Time |
|---|---|
| push (add to top) | O(1) |
| pop (remove from top) | O(1) |
| peek (see top without removing) | O(1) |
| search | O(n) |
| **Space** | O(n) |

### Java Code

```java
// Modern way — use Deque as a Stack (NOT java.util.Stack)
Deque<Integer> stack = new ArrayDeque<>();

stack.push(10);   // addFirst — [10]
stack.push(20);   // addFirst — [20, 10]
stack.push(30);   // addFirst — [30, 20, 10]

int top = stack.peek();  // 30 — look without removing
int val = stack.pop();   // 30 — remove and return top

System.out.println(stack.isEmpty()); // false
System.out.println(stack.size());    // 2

// Real use: check balanced parentheses
public boolean isBalanced(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '[' || c == '{') {
            stack.push(c);
        } else {
            if (stack.isEmpty()) return false;
            char top = stack.pop();
            if (c == ')' && top != '(') return false;
            if (c == ']' && top != '[') return false;
            if (c == '}' && top != '{') return false;
        }
    }
    return stack.isEmpty();
}
```

---

## 5. Queue

### What It Is

A Queue is a **First-In-First-Out (FIFO)** structure. Like a checkout line — the first person in is the first person served.

```
ENQUEUE →  [A] → [B] → [C] → [D]  → DEQUEUE
           ↑                   ↑
          back                front
```

### Simple Queue

```java
// Queue is an interface — use ArrayDeque (not LinkedList) for best performance
Queue<String> queue = new ArrayDeque<>();

queue.offer("Alice");   // add to back — returns false if full (for bounded queues)
queue.offer("Bob");
queue.offer("Charlie");

String front = queue.peek();   // "Alice" — look without removing
String served = queue.poll();  // "Alice" — remove and return front
// queue.remove() does same but throws exception if empty — poll() returns null

System.out.println(queue); // [Bob, Charlie]
```

### PriorityQueue — Always Serves Highest Priority First

A `PriorityQueue` is a **min-heap by default** — the smallest element is always at the front (dequeued first). It does NOT maintain FIFO order — it orders by priority.

```
Min-Heap internal structure (not a sorted list):
            1
          /   \
         3     2
        / \   / \
       5   4 6   7

poll() always returns the minimum: 1
After poll():
            2
          /   \
         3     6
        / \   /
       5   4 7
```

```java
// Default: min-heap (smallest element dequeued first)
PriorityQueue<Integer> minPQ = new PriorityQueue<>();
minPQ.offer(30);
minPQ.offer(10);
minPQ.offer(20);

System.out.println(minPQ.poll()); // 10 — smallest
System.out.println(minPQ.poll()); // 20
System.out.println(minPQ.poll()); // 30

// Max-heap — largest element dequeued first
PriorityQueue<Integer> maxPQ = new PriorityQueue<>(Comparator.reverseOrder());
maxPQ.offer(30);
maxPQ.offer(10);
maxPQ.offer(20);
System.out.println(maxPQ.poll()); // 30 — largest

// Custom priority — by task priority field
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
);
taskQueue.offer(new Task("Low priority task", 3));
taskQueue.offer(new Task("Urgent task", 1));
taskQueue.offer(new Task("Normal task", 2));

Task next = taskQueue.poll(); // Task with priority 1 (Urgent task)
```

**PriorityQueue complexity:**

| Operation | Time |
|---|---|
| offer (insert) | O(log n) |
| poll (remove min/max) | O(log n) |
| peek (view min/max) | O(1) |
| contains | O(n) |
| **Space** | O(n) |

### Deque — Double-Ended Queue

A `Deque` (pronounced "deck") lets you add and remove from **both ends**. It is the most versatile queue structure — it can act as a Stack, Queue, or both.

```
addFirst ← [A] ↔ [B] ↔ [C] ↔ [D] → addLast
removeLast ←                    → removeFirst
```

```java
Deque<String> deque = new ArrayDeque<>();

// Add to both ends
deque.addFirst("B");
deque.addFirst("A");  // [A, B]
deque.addLast("C");   // [A, B, C]
deque.addLast("D");   // [A, B, C, D]

// Remove from both ends
deque.removeFirst();  // "A" → [B, C, D]
deque.removeLast();   // "D" → [B, C]

// Peek both ends
deque.peekFirst();    // "B"
deque.peekLast();     // "C"

// As a sliding window (interview pattern — explained in Part 3)
// Maintains a monotone decreasing deque for max in window
```

---

## 6. HashMap

### What It Is

A `HashMap` stores **key-value pairs** and gives you O(1) average time to get, put, or check any key. It is the most used data structure in backend Java development.

### How It Works Inside

```
HashMap internals — array of "buckets"

Key: "Alice"
  ↓  hashCode("Alice") = 65304 → 65304 % 16 = 8
  ↓
Bucket array:
[0]  → null
[1]  → null
...
[8]  → LinkedList/Tree Node → ("Alice", 30)
...
[12] → ("Bob", 25) → ("Carol", 28)   ← COLLISION: both hashed to 12
[15] → null
```

**Step by step — what happens on `put("Alice", 30)`:**
1. Call `"Alice".hashCode()` → get an int
2. Apply secondary hash to reduce clustering
3. `index = hash & (capacity - 1)` → bucket index
4. If bucket is empty → place entry directly
5. If bucket has entries → check if key already exists (equals) → update if yes, append if no
6. If bucket has ≥ 8 entries → convert LinkedList to Red-Black Tree (Java 8+) → O(log n) per bucket

**Resizing:**
When `size > capacity × loadFactor (default 0.75)`:
1. Double the capacity
2. Rehash every entry into new buckets
3. This is O(n) — expensive, happens rarely

```
After resize (capacity 16 → 32):
Entries are redistributed — the collision at [12] may now separate
```

### Big-O Complexity

| Operation | Average | Worst Case |
|---|---|---|
| get(key) | O(1) | O(n) — all keys same bucket |
| put(key, value) | O(1) | O(n) — all keys same bucket |
| remove(key) | O(1) | O(n) |
| containsKey | O(1) | O(n) |
| **Space** | O(n) | |

**Java 8+ worst case is O(log n)** because buckets convert to trees after 8 entries.

### Java Code

```java
Map<String, Integer> scores = new HashMap<>();

// Put — O(1) average
scores.put("Alice", 95);
scores.put("Bob", 87);
scores.put("Charlie", 92);

// Get — O(1) average
int aliceScore = scores.get("Alice");       // 95
int unknown    = scores.getOrDefault("Dave", 0); // 0 — no NullPointerException

// Check
scores.containsKey("Bob");    // true
scores.containsValue(87);     // true (but O(n) — scans all values)

// Remove
scores.remove("Charlie");

// Update patterns
scores.put("Alice", scores.get("Alice") + 5); // verbose
scores.merge("Alice", 5, Integer::sum);        // cleaner — add 5 to existing or set 5

// Compute patterns
scores.compute("Bob", (k, v) -> v == null ? 1 : v + 1);  // increment or init
scores.computeIfAbsent("Dave", k -> loadFromDB(k));       // only if key missing
scores.computeIfPresent("Bob", (k, v) -> v * 2);          // only if key exists

// Iterate — order is NOT guaranteed
for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + " → " + entry.getValue());
}
scores.forEach((name, score) -> System.out.println(name + ": " + score));

// Word frequency counter — classic interview pattern
String text = "the cat sat on the mat the cat";
Map<String, Integer> freq = new HashMap<>();
for (String word : text.split(" ")) {
    freq.merge(word, 1, Integer::sum);
}
// {the=3, cat=2, sat=1, on=1, mat=1}
```

---

## 7. HashSet

### What It Is

A `HashSet` stores **unique elements** with O(1) average for add, remove, and contains. Internally it is just a `HashMap` where each element is a key and the value is a dummy constant.

```java
// Internally: HashSet wraps HashMap
// set.add("Apple") does: map.put("Apple", PRESENT)
// where PRESENT = new Object() — just a placeholder
```

### Java Code

```java
Set<String> fruits = new HashSet<>();

fruits.add("Apple");
fruits.add("Banana");
fruits.add("Apple");   // duplicate — silently ignored
fruits.add("Cherry");

System.out.println(fruits.size());          // 3 — not 4
System.out.println(fruits.contains("Banana")); // true — O(1)
fruits.remove("Banana");                   // O(1)

// Set operations
Set<Integer> a = new HashSet<>(Arrays.asList(1, 2, 3, 4));
Set<Integer> b = new HashSet<>(Arrays.asList(3, 4, 5, 6));

Set<Integer> union        = new HashSet<>(a); union.addAll(b);        // {1,2,3,4,5,6}
Set<Integer> intersection = new HashSet<>(a); intersection.retainAll(b); // {3,4}
Set<Integer> difference   = new HashSet<>(a); difference.removeAll(b);   // {1,2}

// Classic interview use: find duplicates in an array
public boolean hasDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int n : nums) {
        if (!seen.add(n)) return true; // add returns false if already present
    }
    return false;
}
```

---

## 8. TreeMap

### What It Is

A `TreeMap` stores key-value pairs sorted by key at all times. Internally it is a **Red-Black Tree** (self-balancing binary search tree). Unlike `HashMap`, it guarantees **sorted order** but costs O(log n) for every operation.

```
Keys inserted: 5, 3, 8, 1, 4, 7, 9

Red-Black Tree (balanced):
           5
         /   \
        3     8
       / \   / \
      1   4 7   9

Iteration order: 1, 3, 4, 5, 7, 8, 9  ← always sorted
```

### Java Code

```java
TreeMap<String, Integer> map = new TreeMap<>();
map.put("Banana", 3);
map.put("Apple", 5);
map.put("Cherry", 1);
map.put("Date", 8);

// Sorted iteration — always alphabetical by key
map.forEach((k, v) -> System.out.println(k + ": " + v));
// Apple:5, Banana:3, Cherry:1, Date:8

// Navigation — the killer feature of TreeMap
map.firstKey();                    // "Apple"  — smallest key
map.lastKey();                     // "Date"   — largest key
map.floorKey("Blueberry");         // "Banana" — largest key ≤ "Blueberry"
map.ceilingKey("Blueberry");       // "Cherry" — smallest key ≥ "Blueberry"
map.lowerKey("Cherry");            // "Banana" — strictly less than
map.higherKey("Cherry");           // "Date"   — strictly greater than

// Range views — very powerful
SortedMap<String, Integer> sub = map.subMap("Banana", "Date"); // Banana ≤ k < Date
SortedMap<String, Integer> head = map.headMap("Cherry");       // k < Cherry
SortedMap<String, Integer> tail = map.tailMap("Cherry");       // k ≥ Cherry

// Real use: price tier lookup — "what tier does price X fall into?"
TreeMap<Integer, String> priceTiers = new TreeMap<>();
priceTiers.put(0,    "Free");
priceTiers.put(100,  "Basic");
priceTiers.put(500,  "Pro");
priceTiers.put(2000, "Enterprise");

int price = 350;
String tier = priceTiers.floorEntry(price).getValue(); // "Basic" (100 ≤ 350 < 500)
```

### Big-O Complexity

| Operation | Time |
|---|---|
| put / get / remove | O(log n) |
| firstKey / lastKey | O(log n) |
| floorKey / ceilingKey | O(log n) |
| Iteration (in order) | O(n) |
| **Space** | O(n) |

---

## 9. TreeSet

### What It Is

`TreeSet` is to `HashSet` what `TreeMap` is to `HashMap` — a sorted set with O(log n) operations. Internally it is a `TreeMap` with dummy values. Elements are always sorted.

```java
TreeSet<Integer> set = new TreeSet<>();
set.add(5); set.add(3); set.add(8); set.add(1); set.add(4);

System.out.println(set);          // [1, 3, 4, 5, 8] — always sorted
set.first();                      // 1
set.last();                       // 8
set.floor(4);                     // 4 — largest ≤ 4
set.ceiling(4);                   // 4 — smallest ≥ 4
set.lower(4);                     // 3 — strictly less than 4
set.higher(4);                    // 5 — strictly greater than 4
set.headSet(5);                   // [1, 3, 4] — elements < 5
set.tailSet(4);                   // [4, 5, 8] — elements ≥ 4
set.subSet(3, 6);                 // [3, 4, 5] — 3 ≤ x < 6
```

---

## 10. Binary Tree

### What It Is

A Binary Tree is a hierarchical structure where each node has at most **two children** — a left child and a right child. There is no ordering rule — it is just a tree shape.

```
         1         ← root
        / \
       2   3
      / \   \
     4   5   6
```

**Key terms:**
- **Root** — the top node (no parent)
- **Leaf** — a node with no children
- **Height** — longest path from root to any leaf
- **Depth** — distance of a node from root

### Traversal Patterns (Critical for Interviews)

```java
public class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}

public class BinaryTree {

    // In-Order: Left → Root → Right  (gives sorted order for BST)
    public void inOrder(TreeNode node) {
        if (node == null) return;
        inOrder(node.left);
        System.out.print(node.val + " ");
        inOrder(node.right);
    }

    // Pre-Order: Root → Left → Right  (used to copy/serialize a tree)
    public void preOrder(TreeNode node) {
        if (node == null) return;
        System.out.print(node.val + " ");
        preOrder(node.left);
        preOrder(node.right);
    }

    // Post-Order: Left → Right → Root  (used to delete a tree, calculate sizes)
    public void postOrder(TreeNode node) {
        if (node == null) return;
        postOrder(node.left);
        postOrder(node.right);
        System.out.print(node.val + " ");
    }

    // Level-Order (BFS): level by level  (used to find shortest path, min depth)
    public void levelOrder(TreeNode root) {
        if (root == null) return;
        Queue<TreeNode> queue = new ArrayDeque<>();
        queue.offer(root);

        while (!queue.isEmpty()) {
            int levelSize = queue.size();
            for (int i = 0; i < levelSize; i++) {
                TreeNode node = queue.poll();
                System.out.print(node.val + " ");
                if (node.left  != null) queue.offer(node.left);
                if (node.right != null) queue.offer(node.right);
            }
            System.out.println(); // new line per level
        }
    }

    // Tree Height
    public int height(TreeNode node) {
        if (node == null) return 0;
        return 1 + Math.max(height(node.left), height(node.right));
    }
}
```

**Traversal output for this tree:**
```
         4
        / \
       2   6
      / \ / \
     1  3 5  7

In-Order:   1 2 3 4 5 6 7   (sorted!)
Pre-Order:  4 2 1 3 6 5 7
Post-Order: 1 3 2 5 7 6 4
Level-Order:4
            2 6
            1 3 5 7
```

---

## 11. Binary Search Tree (BST)

### What It Is

A BST is a Binary Tree with one extra rule:
- **Left subtree** of any node contains only values **less than** that node
- **Right subtree** of any node contains only values **greater than** that node

This rule enables O(log n) search on average.

```
Insert: 5, 3, 7, 1, 4, 6, 8

         5
        / \
       3   7
      / \ / \
     1  4 6  8

Search 4:
  5 → is 4 < 5? Yes → go left
  3 → is 4 < 3? No → go right
  4 → found!  (3 comparisons, not 7)
```

### Java Code

```java
public class BST {
    private TreeNode root;

    // Insert — O(log n) average, O(n) worst (unbalanced)
    public void insert(int val) {
        root = insertRec(root, val);
    }

    private TreeNode insertRec(TreeNode node, int val) {
        if (node == null) return new TreeNode(val); // base case: empty spot found
        if (val < node.val)      node.left  = insertRec(node.left,  val);
        else if (val > node.val) node.right = insertRec(node.right, val);
        // if val == node.val: duplicate, ignore (or handle per requirements)
        return node;
    }

    // Search — O(log n) average
    public boolean contains(int val) {
        TreeNode current = root;
        while (current != null) {
            if      (val < current.val) current = current.left;
            else if (val > current.val) current = current.right;
            else    return true;
        }
        return false;
    }

    // Find minimum value (leftmost node)
    public int findMin() {
        if (root == null) throw new NoSuchElementException();
        TreeNode current = root;
        while (current.left != null) current = current.left;
        return current.val;
    }

    // In-order gives sorted output — unique BST property
    public List<Integer> getSorted() {
        List<Integer> result = new ArrayList<>();
        inOrder(root, result);
        return result;
    }
    private void inOrder(TreeNode node, List<Integer> result) {
        if (node == null) return;
        inOrder(node.left, result);
        result.add(node.val);
        inOrder(node.right, result);
    }
}
```

**BST complexity:**

| Operation | Average | Worst (unbalanced) |
|---|---|---|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |

Worst case is a sorted input: `insert(1,2,3,4,5)` creates a right-skewed list, not a tree.

**In interviews: Java's `TreeMap` and `TreeSet` use a Red-Black Tree** (self-balancing BST) that guarantees O(log n) always — you don't need to implement balancing yourself.

---

## 12. Heap (Min/Max Heap)

### What It Is

A Heap is a **complete binary tree** with one ordering rule:
- **Min-Heap**: parent is always ≤ both children (root = minimum element)
- **Max-Heap**: parent is always ≥ both children (root = maximum element)

The key insight: **you only care about the root** — it is always the min or max. You do not care about the order of anything else.

### How It Works Inside — Stored as an Array

The beautiful trick: a complete binary tree maps perfectly to an array.

```
Min-Heap tree:
           1
         /   \
        3     2
       / \   / \
      7   5 4   6

Array:  [1, 3, 2, 7, 5, 4, 6]
Index:   0  1  2  3  4  5  6

For node at index i:
  left child  = 2i + 1
  right child = 2i + 2
  parent      = (i - 1) / 2
```

**Insert (offer) — "bubble up":**
```
Add 0 to heap:
[1, 3, 2, 7, 5, 4, 6, 0]   ← 0 added at end

0 at index 7, parent at (7-1)/2 = 3 → value 7
0 < 7 → swap:  [1, 3, 2, 0, 5, 4, 6, 7]

0 now at index 3, parent at (3-1)/2 = 1 → value 3
0 < 3 → swap:  [1, 0, 2, 3, 5, 4, 6, 7]

0 now at index 1, parent at (1-1)/2 = 0 → value 1
0 < 1 → swap:  [0, 1, 2, 3, 5, 4, 6, 7]

0 at index 0 — done (root)
```

**Remove (poll) — "bubble down":**
```
Remove root (0):
Move last element (7) to root:  [7, 1, 2, 3, 5, 4, 6]

7 > min(1, 2) = 1 → swap with left child:  [1, 7, 2, 3, 5, 4, 6]
7 > min(3, 5) = 3 → swap with left child:  [1, 3, 2, 7, 5, 4, 6]
7 has no children → done
```

### Java Code

```java
// Java's PriorityQueue IS a min-heap
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5); minHeap.offer(1); minHeap.offer(3);
System.out.println(minHeap.peek());  // 1 — minimum, O(1)
System.out.println(minHeap.poll());  // 1 — remove minimum, O(log n)

// Max-heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(5); maxHeap.offer(1); maxHeap.offer(3);
System.out.println(maxHeap.poll());  // 5 — maximum

// Classic interview problem: find K largest elements
public List<Integer> findKLargest(int[] nums, int k) {
    // Min-heap of size k — keeps the k largest seen so far
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll(); // remove smallest — keeps k largest in heap
        }
    }
    return new ArrayList<>(minHeap); // contains k largest elements
}

// Find median of a data stream — classic heap interview problem
class MedianFinder {
    private PriorityQueue<Integer> lower = new PriorityQueue<>(Comparator.reverseOrder()); // max-heap of smaller half
    private PriorityQueue<Integer> upper = new PriorityQueue<>();                          // min-heap of larger half

    public void addNum(int num) {
        lower.offer(num);
        upper.offer(lower.poll());         // balance: push largest of lower to upper
        if (lower.size() < upper.size()) { // keep lower size >= upper size
            lower.offer(upper.poll());
        }
    }

    public double findMedian() {
        if (lower.size() > upper.size()) return lower.peek();
        return (lower.peek() + upper.peek()) / 2.0;
    }
}
```

---

## 13. Graph

### What It Is

A Graph is a set of **nodes (vertices)** connected by **edges**. Unlike trees, graphs can have cycles, disconnected components, and edges that go in specific directions.

```
Undirected Graph:          Directed Graph (Digraph):
  A --- B                    A → B
  |     |                    ↑   ↓
  C --- D                    C ← D
```

**Types:**
- **Undirected** — edges have no direction (road between two cities)
- **Directed** — edges have direction (one-way streets, dependencies)
- **Weighted** — edges have a cost (distance, time, price)
- **Unweighted** — all edges are equal

### Representation — Adjacency List (Most Common)

```
Graph: A→B, A→C, B→D, C→D

Adjacency List:
A → [B, C]
B → [D]
C → [D]
D → []
```

```java
// Graph using adjacency list
public class Graph {
    private Map<Integer, List<Integer>> adjList = new HashMap<>();

    public void addVertex(int v) {
        adjList.putIfAbsent(v, new ArrayList<>());
    }

    public void addEdge(int from, int to) {
        adjList.computeIfAbsent(from, k -> new ArrayList<>()).add(to);
        adjList.computeIfAbsent(to,   k -> new ArrayList<>()).add(from); // remove for directed
    }

    // BFS — shortest path in unweighted graph, level-order traversal
    public List<Integer> bfs(int start) {
        List<Integer> result = new ArrayList<>();
        Set<Integer> visited = new HashSet<>();
        Queue<Integer> queue = new ArrayDeque<>();

        queue.offer(start);
        visited.add(start);

        while (!queue.isEmpty()) {
            int node = queue.poll();
            result.add(node);
            for (int neighbor : adjList.getOrDefault(node, Collections.emptyList())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.offer(neighbor);
                }
            }
        }
        return result;
    }

    // DFS — explore as deep as possible before backtracking
    public List<Integer> dfs(int start) {
        List<Integer> result = new ArrayList<>();
        Set<Integer> visited = new HashSet<>();
        dfsHelper(start, visited, result);
        return result;
    }

    private void dfsHelper(int node, Set<Integer> visited, List<Integer> result) {
        visited.add(node);
        result.add(node);
        for (int neighbor : adjList.getOrDefault(node, Collections.emptyList())) {
            if (!visited.contains(neighbor)) {
                dfsHelper(neighbor, visited, result);
            }
        }
    }
}
```

### BFS vs DFS at a Glance

| | BFS | DFS |
|---|---|---|
| Structure | Queue | Stack (or recursion) |
| Use case | Shortest path, levels | Cycle detection, path existence |
| Memory | O(width) — can be large | O(depth) — can be large |
| Finds | Shortest path first | Any path first |

---

## Complexity Cheat Sheet

| Structure | Access | Search | Insert | Delete | Space |
|---|---|---|---|---|---|
| Array | O(1) | O(n) | O(n) | O(n) | O(n) |
| ArrayList | O(1) | O(n) | O(1)* | O(n) | O(n) |
| LinkedList | O(n) | O(n) | O(1)† | O(1)† | O(n) |
| Stack / Queue | O(n) | O(n) | O(1) | O(1) | O(n) |
| HashMap | O(1) | O(1) | O(1) | O(1) | O(n) |
| TreeMap | O(log n) | O(log n) | O(log n) | O(log n) | O(n) |
| Min/Max Heap | O(1) peek | O(n) | O(log n) | O(log n) | O(n) |
| BST (balanced) | O(log n) | O(log n) | O(log n) | O(log n) | O(n) |

*ArrayList insert: O(1) amortized at end, O(n) at middle
†LinkedList insert/delete: O(1) if you already have the node reference, O(n) to find it first
