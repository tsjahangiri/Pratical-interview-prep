# Elite Java Interview Prep — Part 3: Problems with Full Optimization Ladder

---

## How to Use This Section

Every problem follows the same 4-tier optimization ladder:
1. **Brute Force** — naive, always O(n²) or worse. Say it first, then dismiss it.
2. **Improved** — reduce one dimension. Shows you can optimize.
3. **Optimal** — best asymptotic complexity. What interviews expect.
4. **Production-Ready** — handles edge cases, clean API, documented.

Interviewers don't want you to start with optimal — they want to see the **thought process**.

---

## Problem 1: Two Sum

**Statement:** Given an integer array and a target, return indices of the two numbers that add up to target. Exactly one solution exists.

```
Input:  [2, 7, 11, 15], target = 9
Output: [0, 1]
```

### The Optimization Ladder

**Tier 1: Brute Force — O(n²) time, O(1) space**
```java
// Check every pair
public int[] twoSumBrute(int[] nums, int target) {
    for (int i = 0; i < nums.length; i++)
        for (int j = i + 1; j < nums.length; j++)
            if (nums[i] + nums[j] == target)
                return new int[]{i, j};
    return new int[]{};
}
// Say to interviewer: "Brute force is O(n²) — all pairs. We can do better."
```

**Tier 2: Improved — Sort + Two Pointers — O(n log n) time, O(n) space**
```java
// Sort with index tracking, then two pointers
// Problem: sorting loses original indices — need to track them
// This is more complex than the optimal. Show it to demonstrate awareness.
```

**Tier 3: Optimal — HashMap — O(n) time, O(n) space**
```java
// Key insight: for each nums[i], we need to know if (target - nums[i]) exists
// Scan left-to-right, storing each number. For each new number, check the store.
public int[] twoSumOptimal(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>(); // value → index

    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (seen.containsKey(complement)) {
            return new int[]{seen.get(complement), i};
        }
        seen.put(nums[i], i); // store AFTER checking to avoid using same element twice
    }
    return new int[]{};
}
```

**Tier 4: Production-Ready**
```java
/**
 * Returns indices of two elements that sum to target.
 * @throws IllegalArgumentException if no solution exists
 * Complexity: O(n) time, O(n) space
 */
public int[] twoSum(int[] nums, int target) {
    if (nums == null || nums.length < 2) throw new IllegalArgumentException("Invalid input");

    Map<Integer, Integer> seen = new HashMap<>((int)(nums.length / 0.75f) + 1); // pre-sized

    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        Integer complementIdx = seen.get(complement);
        if (complementIdx != null) {
            return new int[]{complementIdx, i};
        }
        seen.put(nums[i], i);
    }
    throw new IllegalArgumentException("No solution found for target: " + target);
}
```

**Edge Cases to State:**
- `null` or empty input
- Single element array
- Negative numbers (complement can be negative — HashMap handles it fine)
- Overflow: `target = 2 × Integer.MAX_VALUE` — use long or state constraint assumption

---

## Problem 2: Longest Substring Without Repeating Characters

**Statement:** Find the length of the longest substring without repeating characters.

```
"abcabcbb" → 3 ("abc")
"bbbbb"    → 1 ("b")
"pwwkew"   → 3 ("wke")
```

### The Optimization Ladder

**Tier 1: Brute Force — O(n³)**
```java
// Generate all substrings O(n²), check each for uniqueness O(n)
// Total: O(n³), Space: O(min(n, charset))
// Dismiss immediately: "O(n³) — too slow, let's use sliding window"
```

**Tier 2: Improved — Sliding Window with Set — O(n)**
```java
public int lengthOfLongestSubstringSet(String s) {
    Set<Character> window = new HashSet<>();
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        while (window.contains(s.charAt(right))) { // shrink until no duplicate
            window.remove(s.charAt(left++));
        }
        window.add(s.charAt(right));
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
// O(n) but each character can be added/removed twice → 2n operations worst case
```

**Tier 3: Optimal — Sliding Window with HashMap (Jump Left Pointer)**
```java
// Optimization: HashMap stores LAST SEEN INDEX so left pointer jumps directly
// instead of advancing character by character
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int left = 0, maxLen = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        // If c was seen and its last position is within current window
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1; // jump past the previous occurrence
        }
        lastSeen.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
// O(n) — each character processed exactly once
```

**Tier 4: Production-Ready (with ASCII optimization)**
```java
/**
 * Longest substring without repeating characters.
 * Uses int[128] for ASCII strings — faster than HashMap for common case.
 * Falls back to HashMap for Unicode.
 * Complexity: O(n) time, O(1) space for ASCII (fixed-size array)
 */
public int lengthOfLongestSubstring(String s) {
    if (s == null || s.isEmpty()) return 0;
    if (s.length() == 1) return 1;

    // Check if all chars are ASCII — use array for O(1) with better constants
    boolean isAscii = s.chars().allMatch(c -> c < 128);

    if (isAscii) {
        int[] lastSeen = new int[128];
        Arrays.fill(lastSeen, -1);
        int left = 0, maxLen = 0;
        for (int right = 0; right < s.length(); right++) {
            int c = s.charAt(right);
            if (lastSeen[c] >= left) left = lastSeen[c] + 1;
            lastSeen[c] = right;
            maxLen = Math.max(maxLen, right - left + 1);
        }
        return maxLen;
    }

    // Unicode fallback
    Map<Character, Integer> lastSeen = new HashMap<>();
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left)
            left = lastSeen.get(c) + 1;
        lastSeen.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

**Edge Cases:**
- Empty string → 0
- All same characters (`"aaaa"`) → 1
- All unique (`"abcd"`) → length of string
- Special characters, Unicode
- Length-1 string → 1

---

## Problem 3: Top K Frequent Elements

**Statement:** Given an integer array, return the K most frequent elements.

```
[1,1,1,2,2,3], k=2 → [1,2]
```

### The Optimization Ladder

**Tier 1: Brute Force — O(n log n)**
```java
// Count frequencies, sort by frequency, take top K
// O(n) to count + O(n log n) to sort = O(n log n)
Map<Integer, Integer> freq = new HashMap<>();
for (int n : nums) freq.merge(n, 1, Integer::sum);
List<Map.Entry<String, Integer>> list = new ArrayList<>(freq.entrySet());
list.sort((a, b) -> b.getValue() - a.getValue()); // sort descending by count
// Take first K
```

**Tier 2: Improved — Heap — O(n log k)**
```java
// Min-heap of size K — O(n log k) instead of O(n log n)
// When k << n, this is significantly better
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    PriorityQueue<Integer> heap = new PriorityQueue<>(
        Comparator.comparingInt(freq::get)
    );
    for (int num : freq.keySet()) {
        heap.offer(num);
        if (heap.size() > k) heap.poll();
    }
    return heap.stream().mapToInt(i -> i).toArray();
}
// O(n log k) time, O(n) space
```

**Tier 3: Optimal — Bucket Sort — O(n)**
```java
// Maximum frequency is n (all same element)
// Use array of size n+1 where index = frequency
public int[] topKFrequentOptimal(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    // Bucket: bucket[i] contains all numbers with frequency i
    List<Integer>[] buckets = new List[nums.length + 1];
    for (int num : freq.keySet()) {
        int f = freq.get(num);
        if (buckets[f] == null) buckets[f] = new ArrayList<>();
        buckets[f].add(num);
    }

    // Collect top K from highest frequency downward
    int[] result = new int[k];
    int idx = 0;
    for (int f = buckets.length - 1; f >= 0 && idx < k; f--) {
        if (buckets[f] != null) {
            for (int num : buckets[f]) {
                result[idx++] = num;
                if (idx == k) break;
            }
        }
    }
    return result;
}
// O(n) time, O(n) space
```

**Tier 4: Production-Ready**
```java
/**
 * Returns k most frequent elements.
 * Uses heap for O(n log k) — better than O(n log n) sort for small k.
 * Uses bucket sort for O(n) when k is close to n.
 * @param k must be 1 ≤ k ≤ number of unique elements
 */
public int[] topKFrequent(int[] nums, int k) {
    Objects.requireNonNull(nums, "nums must not be null");
    if (nums.length == 0 || k <= 0) throw new IllegalArgumentException("Invalid input");

    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    if (k >= freq.size()) return freq.keySet().stream().mapToInt(i -> i).toArray();

    // Use bucket sort (O(n)) when dealing with large datasets
    List<Integer>[] buckets = new List[nums.length + 1];
    freq.forEach((num, count) -> {
        if (buckets[count] == null) buckets[count] = new ArrayList<>();
        buckets[count].add(num);
    });

    int[] result = new int[k];
    int idx = 0;
    for (int f = buckets.length - 1; f >= 0 && idx < k; f--) {
        if (buckets[f] != null) {
            for (int num : buckets[f]) {
                if (idx == k) break;
                result[idx++] = num;
            }
        }
    }
    return result;
}
```

---

## Problem 4: LRU Cache

**Statement:** Design a data structure that follows the Least Recently Used cache eviction policy. Implement `get(key)` and `put(key, value)`, both in O(1).

### The Optimization Ladder

**Tier 1: Brute Force — List + Linear Search — O(n)**
```java
// Keep a list ordered by recency. On access, move to front.
// O(n) to find element in list → too slow.
```

**Tier 2: Improved — LinkedHashMap (Built-in LRU)**
```java
// Java provides LinkedHashMap with access-order mode — built-in LRU
public class LRUCache {
    private final Map<Integer, Integer> cache;

    public LRUCache(int capacity) {
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;
            }
        };
    }

    public int get(int key) { return cache.getOrDefault(key, -1); }
    public void put(int key, int value) { cache.put(key, value); }
}
// O(1) amortized — but uses Java internals. Good to show, then explain limitations.
// Limitation: not thread-safe, uses synchronized wrapper needed for concurrent access
```

**Tier 3: Optimal — HashMap + Custom Doubly Linked List**
```java
// True O(1) for all operations using:
// HashMap: O(1) lookup by key → get node reference
// Doubly Linked List: O(1) move-to-front and evict-from-back (with node reference)
public class LRUCache {
    private static class Node {
        int key, val;
        Node prev, next;
        Node(int k, int v) { key = k; val = v; }
    }

    private final int capacity;
    private final Map<Integer, Node> map = new HashMap<>();
    private final Node head = new Node(0, 0); // dummy head (most recent)
    private final Node tail = new Node(0, 0); // dummy tail (least recent)

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        Node node = map.get(key);
        if (node == null) return -1;
        moveToFront(node);   // mark as most recently used
        return node.val;
    }

    public void put(int key, int value) {
        Node node = map.get(key);
        if (node != null) {
            node.val = value;
            moveToFront(node);
        } else {
            node = new Node(key, value);
            map.put(key, node);
            addToFront(node);
            if (map.size() > capacity) {
                Node evicted = removeTail();
                map.remove(evicted.key);
            }
        }
    }

    private void addToFront(Node node) {
        node.next = head.next; node.prev = head;
        head.next.prev = node; head.next = node;
    }

    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToFront(Node node) { removeNode(node); addToFront(node); }

    private Node removeTail() {
        Node node = tail.prev;
        removeNode(node);
        return node;
    }
}
// All operations: O(1) — true O(1), not amortized
```

**Tier 4: Production-Ready (Thread-Safe)**
```java
/**
 * Thread-safe LRU Cache using read-write lock.
 * Reads (get) allow concurrent access; writes (put) are exclusive.
 */
public class ConcurrentLRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null);
    private final Node<K, V> tail = new Node<>(null, null);
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    // ... same structure as above but with lock.readLock() / lock.writeLock()
    // For high-read workloads: readLock allows parallel reads
    // For write: writeLock ensures exclusive access

    public V get(K key) {
        lock.writeLock().lock(); // write because moveToFront modifies structure
        try {
            Node<K, V> node = map.get(key);
            if (node == null) return null;
            moveToFront(node);
            return node.val;
        } finally { lock.writeLock().unlock(); }
    }
}
```

---

## Problem 5: Word Ladder (BFS Shortest Path)

**Statement:** Transform `beginWord` to `endWord` by changing one letter at a time. Each intermediate word must be in `wordList`. Return the minimum number of transformations.

```
beginWord="hit", endWord="cog", wordList=["hot","dot","dog","lot","log","cog"]
Output: 5 (hit→hot→dot→dog→cog)
```

### The Optimization Ladder

**Tier 1: Brute Force — DFS (Wrong approach)**
```
DFS finds A path, not necessarily the SHORTEST path.
"Shortest" = BFS. State this explicitly to the interviewer.
```

**Tier 2: BFS — O(M² × N)**
```java
public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> dict = new HashSet<>(wordList);
    if (!dict.contains(endWord)) return 0;

    Queue<String> queue = new ArrayDeque<>();
    Set<String> visited = new HashSet<>();
    queue.offer(beginWord); visited.add(beginWord);
    int steps = 1;

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            String word = queue.poll();
            char[] chars = word.toCharArray();
            for (int j = 0; j < chars.length; j++) {
                char original = chars[j];
                for (char c = 'a'; c <= 'z'; c++) {
                    chars[j] = c;
                    String next = new String(chars);
                    if (next.equals(endWord)) return steps + 1;
                    if (dict.contains(next) && !visited.contains(next)) {
                        visited.add(next); queue.offer(next);
                    }
                }
                chars[j] = original;
            }
        }
        steps++;
    }
    return 0;
}
// Time: O(M² × N) where M=word length, N=dict size
```

**Tier 3: Optimal — Bidirectional BFS**
```java
// Key insight: BFS from BOTH ends simultaneously — when they meet, path found
// Reduces search space from O(b^d) to O(b^(d/2)) where b=branching, d=depth
public int ladderLengthBidir(String beginWord, String endWord, List<String> wordList) {
    Set<String> dict = new HashSet<>(wordList);
    if (!dict.contains(endWord)) return 0;

    Set<String> beginSet = new HashSet<>(), endSet = new HashSet<>();
    Set<String> visited = new HashSet<>();
    beginSet.add(beginWord); endSet.add(endWord);
    int steps = 1;

    while (!beginSet.isEmpty() && !endSet.isEmpty()) {
        // Always expand the SMALLER frontier — key optimization
        if (beginSet.size() > endSet.size()) {
            Set<String> temp = beginSet; beginSet = endSet; endSet = temp;
        }

        Set<String> nextLevel = new HashSet<>();
        for (String word : beginSet) {
            char[] chars = word.toCharArray();
            for (int j = 0; j < chars.length; j++) {
                char orig = chars[j];
                for (char c = 'a'; c <= 'z'; c++) {
                    chars[j] = c;
                    String next = new String(chars);
                    if (endSet.contains(next)) return steps + 1; // frontiers met!
                    if (dict.contains(next) && visited.add(next)) nextLevel.add(next);
                }
                chars[j] = orig;
            }
        }
        beginSet = nextLevel;
        steps++;
    }
    return 0;
}
// Same O complexity but dramatically faster in practice
```

---

## Problem 6: Subsets (Backtracking)

**Statement:** Return all possible subsets (the power set) of a distinct integer array.

```
Input: [1,2,3]
Output: [[], [1], [2], [1,2], [3], [1,3], [2,3], [1,2,3]]
```

### The Optimization Ladder

**Tier 1: Brute Force — Bit Manipulation**
```java
// For n elements, there are 2^n subsets
// Each subset corresponds to a bitmask from 0 to 2^n - 1
public List<List<Integer>> subsetsBitMask(int[] nums) {
    int n = nums.length;
    List<List<Integer>> result = new ArrayList<>();
    for (int mask = 0; mask < (1 << n); mask++) {
        List<Integer> subset = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) != 0) subset.add(nums[i]);
        }
        result.add(subset);
    }
    return result;
}
// Elegant but interviewer usually wants backtracking for its generalizability
```

**Tier 2: Iterative Build**
```java
// Start with [[]], for each number add it to all existing subsets
public List<List<Integer>> subsetsIterative(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    result.add(new ArrayList<>());
    for (int num : nums) {
        int size = result.size();
        for (int i = 0; i < size; i++) {
            List<Integer> newSubset = new ArrayList<>(result.get(i));
            newSubset.add(num);
            result.add(newSubset);
        }
    }
    return result;
}
```

**Tier 3: Optimal — Backtracking**
```java
// Most generalizable — same template handles combinations, permutations, etc.
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int start, List<Integer> current,
                        List<List<Integer>> result) {
    result.add(new ArrayList<>(current));  // snapshot at every node (not just leaves)

    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);                       // choose
        backtrack(nums, i + 1, current, result);    // explore
        current.remove(current.size() - 1);         // un-choose
    }
}
// T: O(n × 2^n), S: O(n) recursion depth + O(n × 2^n) output
```

**Tier 4: Production-Ready (with duplicates — Subsets II)**
```java
/**
 * All unique subsets (handles duplicate input elements).
 * Requires sorting to group duplicates — enables skip-duplicate pruning.
 */
public List<List<Integer>> subsetsWithDup(int[] nums) {
    Arrays.sort(nums); // sort to group duplicates together
    List<List<Integer>> result = new ArrayList<>();
    backtrackDedup(nums, 0, new ArrayList<>(), result);
    return result;
}

private void backtrackDedup(int[] nums, int start, List<Integer> current,
                              List<List<Integer>> result) {
    result.add(new ArrayList<>(current));

    for (int i = start; i < nums.length; i++) {
        // Skip duplicates at the same level of recursion
        // If i > start: we're not the first element at this level
        // If nums[i] == nums[i-1]: we'd generate the same subset we already generated
        if (i > start && nums[i] == nums[i - 1]) continue;

        current.add(nums[i]);
        backtrackDedup(nums, i + 1, current, result);
        current.remove(current.size() - 1);
    }
}
```

---

## Edge Case Generation Strategy

For every problem, systematically generate edge cases:

```
1. SIZE EXTREMES
   Empty input:     nums = [], s = ""
   Single element:  nums = [x], s = "a"
   Two elements:    nums = [x, y]
   Maximum size:    nums = int[10^5]

2. DUPLICATES
   All same:        [2, 2, 2, 2]
   Some duplicates: [1, 2, 2, 3]
   Consecutive:     [1, 1, 2, 2]

3. NEGATIVES
   All negative:    [-3, -1, -2]
   Mixed:           [-2, 0, 1, 3]
   Negatives + duplicates: [-1, -1, 0, 1]

4. OVERFLOW
   Large positive:  [Integer.MAX_VALUE, 1]
   Large negative:  [Integer.MIN_VALUE, -1]
   Sum overflows:   [Integer.MAX_VALUE, Integer.MAX_VALUE]
   → Use long for sums: long sum = (long)a + b

5. SORTED INPUT
   Already sorted:  [1, 2, 3, 4, 5] → BST becomes linked list
   Reverse sorted:  [5, 4, 3, 2, 1]

6. SPECIAL VALUES
   Zero:            [0, 0, 0]
   Target = 0:      twoSum(nums, 0)
   k > array size:  topK(nums, k) where k >= nums.length

7. NULL HANDLING
   Null array:      nums = null
   Array with nulls: new Integer[]{1, null, 3} (if using Integer[])
   Null string:     s = null
```
