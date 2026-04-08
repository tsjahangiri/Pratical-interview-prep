# Elite Java Interview Prep — Part 2: Reusable Problem-Solving Patterns

---

## Why Patterns Matter

FAANG interviewers don't just want a solution — they want to see **pattern recognition**. When you say "this is a sliding window problem" before you write code, you signal senior-level thinking. Every pattern below is a reusable template — memorize the structure, apply it to any problem that fits.

---

## Pattern 1: Sliding Window

### The Core Idea

Maintain a window `[left, right]` over a sequential structure. Expand right to grow the window, shrink left to satisfy constraints. Avoids O(n²) nested loops.

```
Array: [2, 1, 5, 2, 3, 2], target sum = 7

Sliding window:
[2,1,5] sum=8 > 7 → shrink left
  [1,5] sum=6 < 7 → expand right
  [1,5,2] sum=8 > 7 → shrink left
    [5,2] sum=7 = 7 ✓ → window of size 2
```

### Fixed Window Template

Use when: window size is exactly K.

```java
/**
 * Fixed sliding window template
 * Problem: maximum sum of subarray of size K
 */
public int maxSumFixedWindow(int[] nums, int k) {
    if (nums.length < k) return -1;

    // Build first window
    int windowSum = 0;
    for (int i = 0; i < k; i++) windowSum += nums[i];

    int maxSum = windowSum;

    // Slide: add right element, remove left element
    for (int right = k; right < nums.length; right++) {
        windowSum += nums[right];           // expand right
        windowSum -= nums[right - k];       // shrink left (remove element that fell out)
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
// Time: O(n), Space: O(1)
```

### Dynamic Window Template

Use when: window size varies based on a constraint.

```java
/**
 * Dynamic sliding window template
 * Problem: smallest subarray with sum ≥ target
 */
public int minSubarrayLen(int target, int[] nums) {
    int left = 0, sum = 0;
    int minLen = Integer.MAX_VALUE;

    for (int right = 0; right < nums.length; right++) {
        sum += nums[right];                          // EXPAND: always add right element

        while (sum >= target) {                      // SHRINK: while constraint met
            minLen = Math.min(minLen, right - left + 1);
            sum -= nums[left];                       // remove left element
            left++;                                  // shrink window
        }
    }
    return minLen == Integer.MAX_VALUE ? 0 : minLen;
}
// Time: O(n), Space: O(1)
```

### Sliding Window with HashMap Template

Use when: window tracks character/element frequency.

```java
/**
 * Dynamic sliding window with frequency map
 * Problem: longest substring with at most K distinct characters
 */
public int longestSubstringKDistinct(String s, int k) {
    Map<Character, Integer> freq = new HashMap<>();
    int left = 0, maxLen = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        freq.merge(c, 1, Integer::sum);              // EXPAND: add right char

        while (freq.size() > k) {                   // SHRINK: while constraint violated
            char leftChar = s.charAt(left);
            freq.merge(leftChar, -1, Integer::sum);
            if (freq.get(leftChar) == 0) freq.remove(leftChar);
            left++;
        }

        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
// Time: O(n), Space: O(k)
```

### When to Recognize Sliding Window

```
Problem has: "subarray", "substring", "contiguous"
Problem has: "maximum/minimum length", "at most K", "exactly K"
Problem has: sequential input (array or string)
Problem has: O(n) expected time (hints away from O(n²) brute force)
```

---

## Pattern 2: Two Pointers

### The Core Idea

Two pointers scan from different positions simultaneously — reducing O(n²) pair-checking to O(n).

### Opposite Ends Template

Use when: sorted array, finding pairs with a target.

```java
/**
 * Two pointers from both ends
 * Problem: two sum in sorted array
 */
public int[] twoSumSorted(int[] nums, int target) {
    int left = 0, right = nums.length - 1;

    while (left < right) {
        int sum = nums[left] + nums[right];
        if      (sum == target) return new int[]{left, right};
        else if (sum < target)  left++;   // need larger sum → move left right
        else                    right--;  // need smaller sum → move right left
    }
    return new int[]{-1, -1};
}
// Time: O(n), Space: O(1)
// Requires: sorted array
```

### Same Direction Template (Fast/Slow)

Use when: finding cycle, removing duplicates, partitioning.

```java
/**
 * Fast/slow two pointers
 * Problem: remove duplicates from sorted array in-place
 */
public int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;
    int slow = 0;                              // slow: position for next unique element

    for (int fast = 1; fast < nums.length; fast++) {
        if (nums[fast] != nums[slow]) {        // found a new unique element
            slow++;
            nums[slow] = nums[fast];           // write it to next unique position
        }
        // if duplicate: fast advances, slow stays
    }
    return slow + 1; // length of deduplicated array
}
// Time: O(n), Space: O(1)
```

### Floyd's Cycle Detection (Tortoise and Hare)

```java
/**
 * Detect cycle in linked list
 * Slow moves 1 step, fast moves 2 steps
 * If there's a cycle, they must meet eventually
 */
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;           // 1 step
        fast = fast.next.next;      // 2 steps
        if (slow == fast) return true; // meeting point = cycle exists
    }
    return false; // fast reached end = no cycle
}
```

### When to Recognize Two Pointers

```
Problem has: sorted array + pair/triplet sum
Problem has: remove duplicates / partition in-place
Problem has: palindrome check
Problem has: linked list cycle
Problem has: "O(1) space" constraint
Problem has: comparing two sequences
```

---

## Pattern 3: Binary Search

### The Core Idea

Halve the search space each step. Not just for finding elements — binary search on the **answer space** solves problems you wouldn't expect.

### Standard Binary Search Template

```java
/**
 * Standard: find target in sorted array
 * Returns index or -1
 */
public int binarySearch(int[] nums, int target) {
    int left = 0, right = nums.length - 1;

    while (left <= right) {                          // ≤ because range includes right
        int mid = left + (right - left) / 2;         // overflow-safe midpoint

        if      (nums[mid] == target) return mid;
        else if (nums[mid] < target)  left  = mid + 1;
        else                          right = mid - 1;
    }
    return -1;
}
```

### Left Boundary Template (First occurrence)

```java
/**
 * Find leftmost position where condition is true
 * E.g.: first index where nums[i] >= target
 */
public int leftBound(int[] nums, int target) {
    int left = 0, right = nums.length; // right is exclusive

    while (left < right) {             // strict < because right is exclusive
        int mid = left + (right - left) / 2;
        if (nums[mid] < target) left  = mid + 1;
        else                    right = mid;  // don't exclude mid yet
    }
    // left == right == first index where nums[i] >= target
    return left; // or check if nums[left] == target
}
```

### Binary Search on Answer Space

```java
/**
 * Binary search on ANSWER, not array index
 * Problem: minimum capacity to ship packages within D days
 * 
 * Key insight: "minimum X such that condition(X) is true"
 * → Binary search on X
 */
public int shipWithinDays(int[] weights, int days) {
    // Search space: [max_weight, sum_of_all_weights]
    int left  = Arrays.stream(weights).max().getAsInt();  // must fit at least 1 package
    int right = Arrays.stream(weights).sum();             // ship everything in 1 day

    while (left < right) {
        int mid = left + (right - left) / 2;
        if (canShip(weights, days, mid)) right = mid;    // mid works → try smaller
        else                             left  = mid + 1; // too small → need bigger
    }
    return left;
}

private boolean canShip(int[] weights, int days, int capacity) {
    int daysNeeded = 1, currentLoad = 0;
    for (int w : weights) {
        if (currentLoad + w > capacity) { daysNeeded++; currentLoad = 0; }
        currentLoad += w;
    }
    return daysNeeded <= days;
}
// Time: O(n log(sum)), Space: O(1)
```

### When to Recognize Binary Search on Answer

```
Problem has: "minimum/maximum X such that..."
Problem has: monotonic condition (if X works, X+1 also works)
Problem has: large answer range but O(n log n) acceptable
Classic examples:
  - Minimum capacity to finish task in K days
  - Koko eating bananas (minimum eating speed)
  - Find smallest divisor given threshold
  - Magnetic force between balls (maximum minimum distance)
```

---

## Pattern 4: BFS Templates

### Standard BFS (Graph/Grid)

```java
/**
 * BFS template — works for graphs, grids, trees
 * Guarantees shortest path in unweighted graph
 */
public int bfs(int[][] grid, int startR, int startC, int endR, int endC) {
    int rows = grid.length, cols = grid[0].length;
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    boolean[][] visited = new boolean[rows][cols];

    Queue<int[]> queue = new ArrayDeque<>();
    queue.offer(new int[]{startR, startC});
    visited[startR][startC] = true;
    int dist = 0;

    while (!queue.isEmpty()) {
        int size = queue.size(); // process one level at a time
        for (int i = 0; i < size; i++) {
            int[] cell = queue.poll();
            if (cell[0] == endR && cell[1] == endC) return dist;

            for (int[] dir : dirs) {
                int nr = cell[0] + dir[0];
                int nc = cell[1] + dir[1];
                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                        && !visited[nr][nc] && grid[nr][nc] != WALL) {
                    visited[nr][nc] = true;
                    queue.offer(new int[]{nr, nc});
                }
            }
        }
        dist++;
    }
    return -1; // not reachable
}
```

### Multi-Source BFS

```java
/**
 * Multi-source BFS: start from multiple sources simultaneously
 * Problem: distance from nearest 0 for every cell in matrix
 */
public int[][] updateMatrix(int[][] mat) {
    int rows = mat.length, cols = mat[0].length;
    int[][] dist = new int[rows][cols];
    Queue<int[]> queue = new ArrayDeque<>();

    // Initialize: all 0s are sources, all 1s are unvisited (MAX_VALUE)
    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            if (mat[r][c] == 0) {
                queue.offer(new int[]{r, c}); // all 0s start simultaneously
            } else {
                dist[r][c] = Integer.MAX_VALUE; // 1s need to be filled
            }
        }
    }

    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    while (!queue.isEmpty()) {
        int[] cell = queue.poll();
        for (int[] dir : dirs) {
            int nr = cell[0] + dir[0], nc = cell[1] + dir[1];
            if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                    && dist[nr][nc] > dist[cell[0]][cell[1]] + 1) {
                dist[nr][nc] = dist[cell[0]][cell[1]] + 1;
                queue.offer(new int[]{nr, nc});
            }
        }
    }
    return dist;
}
```

---

## Pattern 5: DFS Templates

### Recursive DFS

```java
/**
 * Recursive DFS template — tree/graph
 */
public void dfsRecursive(TreeNode node, int depth, List<Integer> result) {
    if (node == null) return;                // BASE CASE: always first

    result.add(node.val);                   // PRE-ORDER: process before recursing

    dfsRecursive(node.left,  depth + 1, result); // recurse left
    dfsRecursive(node.right, depth + 1, result); // recurse right

    // POST-ORDER: process after recursing (for cleanup, size calculation, etc.)
}
```

### Iterative DFS (Stack) — Avoids Stack Overflow

```java
/**
 * Iterative DFS — same result as recursive, no stack overflow risk
 * Use for deep trees/graphs (depth > 10,000)
 */
public List<Integer> dfsIterative(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);

    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        result.add(node.val);

        // Push RIGHT first so LEFT is processed first (LIFO)
        if (node.right != null) stack.push(node.right);
        if (node.left  != null) stack.push(node.left);
    }
    return result; // pre-order
}
```

### DFS with Backtracking Template

```java
/**
 * Backtracking DFS template
 * Problem: all subsets / permutations / combinations
 */
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int start,
                        List<Integer> current, List<List<Integer>> result) {
    result.add(new ArrayList<>(current));           // ADD SNAPSHOT (deep copy!)

    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);                       // CHOOSE
        backtrack(nums, i + 1, current, result);    // EXPLORE
        current.remove(current.size() - 1);         // UN-CHOOSE (backtrack)
    }
}
// Time: O(2^n × n), Space: O(n) recursion depth
```

---

## Pattern 6: Heap Patterns

### Top-K Pattern

```java
/**
 * Top K most frequent elements
 * Min-heap of size K — evicts smallest frequency, retains K largest
 */
public int[] topKFrequent(int[] nums, int k) {
    // Step 1: frequency map
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    // Step 2: min-heap ordered by frequency
    PriorityQueue<Integer> heap = new PriorityQueue<>(
        Comparator.comparingInt(freq::get)
    );
    for (int num : freq.keySet()) {
        heap.offer(num);
        if (heap.size() > k) heap.poll(); // evict least frequent
    }

    // Step 3: extract
    int[] result = new int[k];
    for (int i = k - 1; i >= 0; i--) result[i] = heap.poll();
    return result;
}
// Time: O(n log k), Space: O(n)
```

### K-Way Merge Pattern

```java
/**
 * Merge K sorted arrays using a min-heap
 * Each heap entry: [value, arrayIndex, elementIndex]
 */
public int[] mergeKSorted(int[][] arrays) {
    PriorityQueue<int[]> heap = new PriorityQueue<>(
        Comparator.comparingInt(a -> a[0])
    );

    // Seed heap with first element of each array
    for (int i = 0; i < arrays.length; i++) {
        if (arrays[i].length > 0) {
            heap.offer(new int[]{arrays[i][0], i, 0});
        }
    }

    List<Integer> result = new ArrayList<>();
    while (!heap.isEmpty()) {
        int[] curr = heap.poll();
        result.add(curr[0]);
        int nextIdx = curr[2] + 1;
        if (nextIdx < arrays[curr[1]].length) {
            heap.offer(new int[]{arrays[curr[1]][nextIdx], curr[1], nextIdx});
        }
    }
    return result.stream().mapToInt(i -> i).toArray();
}
// Time: O(N log K) where N = total elements, K = number of arrays
```

---

## Pattern 7: Monotonic Deque

### The Core Idea

Maintain a deque in monotonically increasing or decreasing order. Gives O(1) access to the window's minimum or maximum without rescanning.

```
Goal: maximum in every window of size K

Key insight: if nums[j] >= nums[i] and j > i (j came after i),
             then nums[i] will NEVER be the window maximum again.
             → discard nums[i] when nums[j] arrives

Monotone decreasing deque maintains: each entry ≥ all entries to its right
Front = always the maximum of current window
```

```java
/**
 * Monotone decreasing deque — sliding window maximum
 */
public int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] result = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>(); // stores INDICES

    for (int i = 0; i < n; i++) {
        // Remove indices outside the window
        while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }

        // Maintain decreasing order: remove all smaller elements from back
        // They can never be the maximum while nums[i] is in the window
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }

        deque.offerLast(i);

        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()]; // front = max of window
        }
    }
    return result;
}
// Time: O(n) — each element enters/exits deque exactly once
// Space: O(k)
```

---

## Pattern 8: Prefix Sum

### The Core Idea

Precompute cumulative sums so any subarray sum can be answered in O(1).

```
nums =     [1,  2,  3,  4,  5]
prefix =   [0,  1,  3,  6, 10, 15]  (prefix[i] = sum of nums[0..i-1])

Sum of nums[2..4] = prefix[5] - prefix[2] = 15 - 3 = 12
Formula: sum(i..j) = prefix[j+1] - prefix[i]
```

```java
/**
 * Prefix sum template
 * Problem: number of subarrays with sum = K
 */
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1);  // empty prefix (sum 0 occurs once before array starts)

    int sum = 0, count = 0;
    for (int num : nums) {
        sum += num;
        // If (sum - k) was seen before, there's a subarray ending here with sum = k
        count += prefixCount.getOrDefault(sum - k, 0);
        prefixCount.merge(sum, 1, Integer::sum);
    }
    return count;
}
// Time: O(n), Space: O(n)
// Key insight: sum(i..j) = prefix[j] - prefix[i-1] = k
//              → prefix[i-1] = prefix[j] - k
//              → count how many times (current_sum - k) appeared before
```

### 2D Prefix Sum

```java
/**
 * 2D prefix sum for rectangle range queries
 */
class NumMatrix {
    private int[][] prefix;

    public NumMatrix(int[][] matrix) {
        int r = matrix.length, c = matrix[0].length;
        prefix = new int[r + 1][c + 1];
        for (int i = 1; i <= r; i++)
            for (int j = 1; j <= c; j++)
                prefix[i][j] = matrix[i-1][j-1]
                    + prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1];
    }

    // Sum of rectangle (r1,c1) to (r2,c2) inclusive — O(1)
    public int sumRegion(int r1, int c1, int r2, int c2) {
        return prefix[r2+1][c2+1]
             - prefix[r1][c2+1]
             - prefix[r2+1][c1]
             + prefix[r1][c1];
    }
}
```

---

## Pattern Recognition Quick-Reference

```
Pattern           │ Trigger words in problem                │ Time complexity
──────────────────┼─────────────────────────────────────────┼────────────────
Sliding Window    │ contiguous subarray/substring, at most K │ O(n)
Two Pointers      │ sorted array, pairs, in-place, O(1) space│ O(n)
Binary Search     │ sorted input, minimize/maximize X        │ O(log n) or O(n log n)
BFS               │ shortest path, min steps, levels         │ O(V + E)
DFS/Backtracking  │ all paths, combinations, cycle detection  │ O(2^n) or O(n!)
Heap (Top K)      │ K largest/smallest/frequent              │ O(n log k)
Monotone Deque    │ sliding window max/min                   │ O(n)
Prefix Sum        │ subarray sum, range queries              │ O(n) preprocess, O(1) query
HashMap           │ frequency, lookup, two-sum style         │ O(n)
Union-Find        │ connected components, dynamic connectivity│ O(α(n)) ≈ O(1)
```

---

## Backtracking Decision Framework

```
Is this a backtracking problem?
  ✓ "Find ALL solutions" (not just one)
  ✓ "Generate all combinations/permutations/subsets"
  ✓ "Determine if a valid arrangement exists"
  ✓ Constraint satisfaction (N-Queens, Sudoku)

Backtracking vs DFS:
  DFS: traverse existing structure (graph/tree that's already there)
  Backtracking: BUILD the structure as you go, undo if invalid

Pruning (critical for performance):
  Add constraint checks BEFORE recursing, not after
  Sort input to enable early termination (e.g., skip duplicates)
  Track visited set to avoid revisiting

// Template with pruning:
private void backtrack(int start, List<Integer> current, List<List<Integer>> result, int[] nums) {
    if (meetsGoal(current)) {
        result.add(new ArrayList<>(current));
        return; // or continue depending on problem
    }
    for (int i = start; i < nums.length; i++) {
        if (i > start && nums[i] == nums[i-1]) continue; // PRUNE: skip duplicates
        if (!isValid(current, nums[i])) continue;        // PRUNE: constraint check
        current.add(nums[i]);
        backtrack(i + 1, current, result, nums);
        current.remove(current.size() - 1);
    }
}
```
