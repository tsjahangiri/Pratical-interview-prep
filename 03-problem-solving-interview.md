# Data Structures in Java — Part 3: Problem-Solving for Interviews

---

## How to Think in a Live Coding Interview

Most candidates fail not because they don't know data structures — but because they don't have a **structured thinking process**. Interviewers want to see your brain working, not just your output.

### The 5-Step Process (Do This Every Time)

```
1. UNDERSTAND — Repeat the problem in your own words
               Ask clarifying questions BEFORE touching code
               
2. EXAMPLES    — Walk through 2-3 examples by hand
               Include edge cases: empty input, single element, duplicates

3. CHOOSE DS   — Say out loud which structure and WHY
               "I'll use a HashMap here because I need O(1) lookup by key"

4. CODE        — Write clean, readable code
               Name variables well, add short comments at key steps

5. VERIFY      — Trace through your code with an example
               State time and space complexity at the end
```

---

## Pattern Recognition — Read the Clues

Certain words in a problem statement are clues to the data structure:

| Problem says... | Think... |
|---|---|
| "find if exists / contains" | HashSet or HashMap |
| "count frequency / occurrences" | HashMap |
| "unique / distinct" | HashSet |
| "sorted order / range" | TreeMap or TreeSet |
| "top K / largest K / smallest K" | PriorityQueue (heap) |
| "sliding window / subarray" | Deque or two pointers |
| "shortest path / minimum steps" | BFS + Queue |
| "all paths / cycle detection" | DFS + Stack/recursion |
| "valid parentheses / matching" | Stack |
| "undo / history / backtracking" | Stack |
| "schedule / priority / urgency" | PriorityQueue |
| "adjacent / connected / network" | Graph |
| "hierarchical / parent-child" | Tree |

---

## Problem 1 — Two Sum

**Problem:** Given an array of integers and a target sum, return the indices of the two numbers that add up to the target. Assume exactly one solution exists.

```
Input:  nums = [2, 7, 11, 15], target = 9
Output: [0, 1]  (because nums[0] + nums[1] = 2 + 7 = 9)
```

### Thought Process

**Clue:** "Find if a number exists" — that is HashMap territory.

**Brute force:** Check every pair — O(n²). Too slow.

**Better question:** For each number `x`, I need to know if `target - x` exists in the array. I need fast lookup of previously seen values. That is exactly what a HashMap does.

**Key insight:** As I scan left to right, for each `nums[i]`, I check if its **complement** (`target - nums[i]`) has already been stored in my map. If yes — found! If no — store `nums[i]` and its index for future lookups.

```
nums = [2, 7, 11, 15], target = 9
map = {}

i=0: num=2, need=9-2=7, map has 7? No  → store {2:0}
i=1: num=7, need=9-7=2, map has 2? YES → return [map.get(2), 1] = [0, 1]
```

### Java Code

```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>(); // value → index

    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];

        if (seen.containsKey(complement)) {
            return new int[]{seen.get(complement), i};
        }

        seen.put(nums[i], i); // store current number for future lookups
    }

    throw new IllegalArgumentException("No solution found");
}
```

**Complexity:**
- Time: O(n) — single pass through the array
- Space: O(n) — map stores at most n entries

**Common mistakes:**
- Storing all values first then searching — works but misses same-index pairs
- Using array index approach — O(n²), too slow

---

## Problem 2 — Valid Parentheses

**Problem:** Given a string of brackets `()`, `[]`, `{}`, return true if all brackets are properly closed and nested.

```
Input: "({[]})"  → true
Input: "([)]"    → false
Input: "{[]}"    → true
Input: "["       → false
```

### Thought Process

**Clue:** "Matching pairs" and "last opened must be first closed" — that is the definition of LIFO. Stack.

**Key insight:** When I see an opening bracket, I push it. When I see a closing bracket, I check if it matches the **most recently opened** bracket (top of stack). If it doesn't match — invalid. If the stack is empty at the end — valid.

```
Input: "({[]})"
Stack operations:
  '(' → push:  ['(']
  '{' → push:  ['(', '{']
  '[' → push:  ['(', '{', '[']
  ']' → top is '[', matches → pop:  ['(', '{']
  '}' → top is '{', matches → pop:  ['(']
  ')' → top is '(', matches → pop:  []
End: stack is empty → true ✓
```

### Java Code

```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> matchMap = Map.of(
        ')', '(',
        ']', '[',
        '}', '{'
    );

    for (char c : s.toCharArray()) {
        if (c == '(' || c == '[' || c == '{') {
            stack.push(c); // opening bracket — push
        } else {
            // closing bracket — must match top of stack
            if (stack.isEmpty() || stack.peek() != matchMap.get(c)) {
                return false;
            }
            stack.pop();
        }
    }

    return stack.isEmpty(); // all brackets matched
}
```

**Complexity:**
- Time: O(n)
- Space: O(n) — worst case all opening brackets

**Edge cases to mention:**
- Empty string → true (no brackets = valid)
- Single character `"("` → false
- `")("` → closing before opening → stack is empty when we need to pop

---

## Problem 3 — Top K Frequent Elements

**Problem:** Given an integer array, return the K most frequent elements.

```
Input: nums = [1,1,1,2,2,3], k = 2
Output: [1, 2]  (1 appears 3 times, 2 appears 2 times)
```

### Thought Process

**Clue:** "Frequency" → HashMap. "Top K" → PriorityQueue (min-heap of size K).

**Step 1:** Count frequencies — HashMap.
**Step 2:** Find top K — if we sort the map by frequency, that is O(n log n). Can we do better?

**Better:** Use a **min-heap of size K**. As we process each number's frequency:
- Add to heap
- If heap grows beyond K, remove the smallest frequency (that is why we use min-heap — we want to evict the least frequent)
- At the end, the heap contains exactly the K most frequent elements

```
freq = {1:3, 2:2, 3:1}, k=2

Process 1 (freq=3): heap = [3]
Process 2 (freq=2): heap = [2, 3]
Process 3 (freq=1): heap = [1, 2, 3], size > k → remove min(1)
                    heap = [2, 3]

Result: numbers with freq 2 and 3 → [2, 1]
```

### Java Code

```java
public int[] topKFrequent(int[] nums, int k) {
    // Step 1: Count frequencies
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : nums) {
        freq.merge(num, 1, Integer::sum);
    }

    // Step 2: Min-heap ordered by frequency — keeps K largest frequencies
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(
        Comparator.comparingInt(freq::get) // compare by frequency, not value
    );

    for (int num : freq.keySet()) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll(); // remove element with lowest frequency
        }
    }

    // Step 3: Extract results
    int[] result = new int[k];
    for (int i = k - 1; i >= 0; i--) {
        result[i] = minHeap.poll();
    }
    return result;
}
```

**Complexity:**
- Time: O(n log k) — n for counting, log k per heap operation for n unique elements
- Space: O(n) — map + heap

**Why not sort?** Sorting the frequency map is O(n log n). Heap approach is O(n log k). When k << n, this is significantly faster.

---

## Problem 4 — Sliding Window Maximum

**Problem:** Given an array and a window size k, return the maximum element in every window of size k as the window slides from left to right.

```
Input:  nums = [1,3,-1,-3,5,3,6,7], k = 3
Output: [3, 3, 5, 5, 6, 7]

Windows:
[1  3  -1] -3  5  3  6  7  → max = 3
 1 [3  -1  -3] 5  3  6  7  → max = 3
 1  3 [-1  -3  5] 3  6  7  → max = 5
 1  3  -1 [-3  5  3] 6  7  → max = 5
 1  3  -1  -3 [5  3  6] 7  → max = 6
 1  3  -1  -3  5 [3  6  7] → max = 7
```

### Thought Process

**Clue:** "Window slides" + "maximum per window" → Monotone Deque.

**Brute force:** For each window, scan all k elements to find max — O(n×k). Too slow.

**Insight:** We want a data structure that always gives us the maximum of the current window in O(1). The key is that if a new element coming in is **larger than elements already in the window**, those old elements will **never be the max** for any future window — we can discard them.

This is a **monotone decreasing deque** — we maintain a deque where elements are always in decreasing order. The front is always the current window max.

```
nums = [1,3,-1,-3,5,3,6,7], k=3

i=0: num=1. Deque=[0] (stores indices)
i=1: num=3. 3>nums[0]=1 → remove 0. Deque=[1]
i=2: num=-1. -1<3 → add. Deque=[1,2]. Window complete → max=nums[1]=3

i=3: num=-3. -3<-1 → add. Deque=[1,2,3].
     Check front: index 1, is it out of window? (3-1=2 < k=3, no). Max=nums[1]=3

i=4: num=5. 5>nums[3]=-3→remove. 5>nums[2]=-1→remove. 5>nums[1]=3→remove. Deque=[4].
     Max=nums[4]=5

i=5: num=3. 3<5 → add. Deque=[4,5]. Max=nums[4]=5

i=6: num=6. 6>nums[5]=3→remove. 6>nums[4]=5→remove. Deque=[6]. Max=nums[6]=6

i=7: num=7. 7>nums[6]=6→remove. Deque=[7]. Max=nums[7]=7
```

### Java Code

```java
public int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] result = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>(); // stores INDICES, not values

    for (int i = 0; i < n; i++) {
        // Remove elements outside the current window
        while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }

        // Remove elements smaller than current — they can never be the max
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }

        deque.offerLast(i);

        // Start recording results once first window is complete
        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()]; // front is always the max
        }
    }

    return result;
}
```

**Complexity:**
- Time: O(n) — each element enters and exits the deque at most once
- Space: O(k) — deque holds at most k indices

---

## Problem 5 — Number of Islands

**Problem:** Given a 2D grid of '1' (land) and '0' (water), count the number of islands. An island is surrounded by water and is formed by connecting adjacent lands horizontally or vertically.

```
Input:
11110
11010
11000
00000
Output: 1 (one large island)

Input:
11000
11000
00100
00011
Output: 3 (three separate islands)
```

### Thought Process

**Clue:** "Connected", "adjacent" → Graph traversal.

**Insight:** Each cell is a node. Two cells are connected if they are adjacent and both are '1'. Each island is a connected component of '1' cells.

**Approach:** Scan every cell. When I find an unvisited '1', I found a new island — increment counter. Then use BFS/DFS to mark all cells in this island as visited (so I don't count them again).

**Why BFS here?** BFS works perfectly. DFS also works but can hit stack overflow on very large grids. BFS is safer.

```
Grid:
1 1 0
1 0 0
0 0 1

Start: (0,0)=1, not visited → island++, BFS from (0,0)
  Visit (0,0) → mark as visited, add neighbors (0,1) and (1,0)
  Visit (0,1) → mark, add neighbor (check right (0,2)=0, down (1,1)=0)
  Visit (1,0) → mark, add neighbors (check down (2,0)=0, right (1,1)=0)
  Queue empty — island 1 fully explored

Continue scan: (0,2)=0, (1,0)=visited, (1,1)=0, (1,2)=0, (2,0)=0, (2,1)=0
  (2,2)=1, not visited → island++, BFS from (2,2)
  Queue empty — island 2 fully explored

Total islands = 2
```

### Java Code

```java
public int numIslands(char[][] grid) {
    if (grid == null || grid.length == 0) return 0;

    int rows = grid.length;
    int cols = grid[0].length;
    int islands = 0;
    int[][] directions = {{0,1},{0,-1},{1,0},{-1,0}}; // right, left, down, up

    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            if (grid[r][c] == '1') {
                islands++;
                bfsMarkVisited(grid, r, c, rows, cols, directions);
            }
        }
    }

    return islands;
}

private void bfsMarkVisited(char[][] grid, int startR, int startC,
                             int rows, int cols, int[][] directions) {
    Queue<int[]> queue = new ArrayDeque<>();
    queue.offer(new int[]{startR, startC});
    grid[startR][startC] = '0'; // mark visited by turning to water

    while (!queue.isEmpty()) {
        int[] cell = queue.poll();
        for (int[] dir : directions) {
            int newR = cell[0] + dir[0];
            int newC = cell[1] + dir[1];
            if (newR >= 0 && newR < rows &&
                newC >= 0 && newC < cols &&
                grid[newR][newC] == '1') {
                grid[newR][newC] = '0'; // mark visited
                queue.offer(new int[]{newR, newC});
            }
        }
    }
}
```

**Complexity:**
- Time: O(m × n) — each cell visited at most once
- Space: O(min(m,n)) — BFS queue width is bounded by the shorter dimension

**Follow-up questions interviewers ask:**
- "Can you do it without modifying the grid?" → use a `Set<String>` of visited coordinates
- "What if it's a 3D grid?" → add a third direction dimension

---

## Bonus Problem — Longest Substring Without Repeating Characters

**Problem:** Find the length of the longest substring without repeating characters.

```
Input: "abcabcbb"  → Output: 3  ("abc")
Input: "bbbbb"     → Output: 1  ("b")
Input: "pwwkew"    → Output: 3  ("wke")
```

### Thought Process

**Clue:** "Substring" (contiguous) + "without repeating" → Sliding window + HashSet.

**Two-pointer sliding window:** Maintain a window `[left, right]`. Expand `right`. If `right` introduces a duplicate, shrink from `left` until the duplicate is removed.

```
"abcabcbb"
left=0, right=0: window="a"  set={a}      len=1
left=0, right=1: window="ab" set={a,b}    len=2
left=0, right=2: window="abc"set={a,b,c}  len=3
left=0, right=3: 'a' in set → shrink: remove s[0]='a', left=1
                 window="bca" set={b,c,a}  len=3
...
```

### Java Code

```java
public int lengthOfLongestSubstring(String s) {
    Set<Character> window = new HashSet<>();
    int left = 0, maxLen = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);

        // Shrink from left until we remove the duplicate
        while (window.contains(c)) {
            window.remove(s.charAt(left));
            left++;
        }

        window.add(c);
        maxLen = Math.max(maxLen, right - left + 1);
    }

    return maxLen;
}

// Optimized: use HashMap to jump left directly (no character-by-character shrinking)
public int lengthOfLongestSubstringOptimized(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>(); // char → last index
    int left = 0, maxLen = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1; // jump left past the duplicate
        }
        lastSeen.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }

    return maxLen;
}
```

**Complexity:**
- Time: O(n) — each character processed at most twice (HashSet version)
- Space: O(min(n, alphabet_size)) — window size bounded by character set

---

## Interview Conversation Cheat Sheet

When choosing a data structure, say it out loud:

```
"I need fast lookup by key — I'll use HashMap, O(1) average."

"I need to find the minimum repeatedly — min-heap (PriorityQueue), O(log n) per removal."

"I need LIFO behavior — stack via ArrayDeque."

"I'm doing BFS — I need a Queue, I'll use ArrayDeque."

"I need to track a window of elements and remove from both ends — Deque."

"I need sorted order with range queries — TreeMap."

"I just need to check uniqueness — HashSet, O(1) add and contains."
```

**State complexity at the end — always:**
```
"Time complexity is O(n log k) because for each of the n elements,
 heap operations take O(log k). Space is O(n) for the frequency map
 and O(k) for the heap."
```
