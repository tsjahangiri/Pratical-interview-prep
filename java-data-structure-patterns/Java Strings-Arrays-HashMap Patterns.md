# 🎯 CODING INTERVIEW MASTERCLASS — Java
### Arrays · Strings · HashMaps · Interview Ready

---

> **Your Mission:** Recognize the pattern in 60 seconds → Apply the right template → Communicate clearly → Ace the interview.

---

## 📋 TABLE OF CONTENTS

### 🔗 Quick Jump to Section
| Section | Topic |
|---|---|
| [Section 1](#section-1-arrays-master-guide-) | Arrays Master Guide |
| [Section 2](#section-2-strings-master-guide-) | Strings Master Guide |
| [Section 3](#section-3-hashmap--hashset-master-guide-️) | HashMap / HashSet Master Guide |
| [Section 4](#section-4-problem-solving-framework-️-gold) | Problem-Solving Framework (GOLD) |
| [Section 5](#section-5-interview-mode-️) | Interview Mode |
| [Section 6](#section-6-debugging-skills-) | Debugging Skills |
| [Section 7](#section-7-7-day-revision-plan-) | 7-Day Revision Plan |

---

### 📦 Section 1 — Arrays

| Pattern | Description |
|---|---|
| [What is an Array?](#what-is-an-array-beginner-first) | Basics, indexing, guard block |
| [Pattern 1 — Traversal](#pattern-1-traversal) | Visit every element |
| [Pattern 2 — Two Pointers](#pattern-2-two-pointers) | Left & right pointers |
| [Pattern 3 — Sliding Window](#pattern-3-sliding-window) | Fixed & variable window |
| [Pattern 4 — Prefix Sum](#pattern-4-prefix-sum) | Range sum queries |
| [Pattern 5 — Kadane's Algorithm](#pattern-5-kadanes-algorithm) | Maximum subarray |
| [Pattern 6 — Sorting + Greedy](#pattern-6-sorting--greedy) | Merge intervals |
| [Pattern 7 — Binary Search](#pattern-7-binary-search-on-arrays) | Search on sorted arrays |
| [Pattern 8 — In-Place Modification](#pattern-8-in-place-modification) | Read/write pointers |
| [Pattern 9 — Merge Intervals](#pattern-9-merge-intervals) | Insert interval |
| [Pattern 10 — Frequency Counting](#pattern-10-frequency-counting) | Count occurrences |
| **Interview Problems** | |
| [P1 — Two Sum (Unsorted)](#problem-1--two-sum-unsorted) | HashMap lookup |
| [P2 — Best Time to Buy & Sell Stock](#problem-2--best-time-to-buy-and-sell-stock) | Min tracking |
| [P3 — Contains Duplicate](#problem-3--contains-duplicate) | HashSet |
| [P4 — Product Except Self](#problem-4--product-of-array-except-self) | Left/right pass |
| [P5 — Maximum Subarray](#problem-5--maximum-subarray-kadanes) | Kadane's |
| [P6 — 3Sum](#problem-6--3sum) | Sort + Two Pointers |
| [P7 — Trapping Rain Water](#problem-7--trapping-rain-water) | Two Pointers |
| [P8 — Rotate Array](#problem-8--rotate-array) | Reversal trick |
| [P9 — Find Min in Rotated Array](#problem-9--find-minimum-in-rotated-sorted-array) | Binary Search |
| [P10 — Subarray Sum Equals K](#problem-10--subarray-sum-equals-k) | Prefix Sum + HashMap |

---

### 🔤 Section 2 — Strings

| Pattern | Description |
|---|---|
| [What is a String in Java?](#what-is-a-string-in-java-beginner-first) | Immutability, key methods |
| [Pattern 1 — Character Frequency Count](#pattern-1-character-frequency-count) | int[26] vs HashMap |
| [Pattern 2 — Two Pointers on Strings](#pattern-2-two-pointers-on-strings) | Palindrome check |
| [Pattern 3 — Sliding Window on Strings](#pattern-3-sliding-window-on-strings) | Longest/min substring |
| [Pattern 4 — Palindrome Problems](#pattern-4-palindrome-problems) | Expand from center |
| [Pattern 5 — Anagram Problems](#pattern-5-anagram-problems) | Frequency match |
| [Pattern 6 — Substring Problems](#pattern-6-substring-problems) | All substrings |
| [Pattern 7 — StringBuilder Optimization](#pattern-7-stringbuilder-optimization) | O(n) vs O(n²) |
| [Pattern 8 — HashMap on Strings](#pattern-8-hashmap-on-strings) | Group anagrams |
| [Pattern 9 — Pattern Matching](#pattern-9-pattern-matching) | KMP concept |
| [Pattern 10 — Parsing / Tokenization](#pattern-10-parsing--tokenization) | Split, trim, char type |
| **Interview Problems** | |
| [P1 — Valid Anagram](#problem-1--valid-anagram) | Frequency count |
| [P2 — Longest Substring No Repeat](#problem-2--longest-substring-without-repeating) | Sliding window |
| [P3 — Valid Palindrome](#problem-3--valid-palindrome) | Two pointers |
| [P4 — Reverse Words in String](#problem-4--reverse-words-in-a-string) | Split + reverse |
| [P5 — Longest Common Prefix](#problem-5--longest-common-prefix) | Shrink prefix |
| [P6 — String Compression](#problem-6--string-compression) | Count chars |
| [P7 — Roman to Integer](#problem-7--roman-to-integer) | HashMap mapping |
| [P8 — Count and Say](#problem-8--count-and-say) | Simulate |
| [P9 — Decode String](#problem-9--decode-string) | Stack |
| [P10 — Minimum Window Substring](#problem-10--minimum-window-substring) | Sliding window |

---

### 🗺️ Section 3 — HashMap / HashSet

| Pattern | Description |
|---|---|
| [What is a HashMap?](#what-is-a-hashmap-beginner-first) | Basics, key methods |
| [What is a HashSet?](#what-is-a-hashset) | Set operations |
| [How HashMap Works Internally](#how-hashmap-works-internally-interview-gold) | Buckets, collisions |
| [Pattern 1 — Frequency Count](#pattern-1-frequency-count) | getOrDefault, merge |
| [Pattern 2 — Lookup Optimization](#pattern-2-lookup-optimization) | O(n) via O(1) lookup |
| [Pattern 3 — Duplicate Detection](#pattern-3-duplicate-detection) | HashSet.add() trick |
| [Pattern 4 — Pair Sum Problems](#pattern-4-pair-sum-problems) | Complement search |
| [Pattern 5 — Grouping Problems](#pattern-5-grouping-problems) | computeIfAbsent |
| [Pattern 6 — Counting Distinct](#pattern-6-counting-distinct) | Set size |
| [Pattern 7 — Index Mapping](#pattern-7-index-mapping) | Consecutive sequence |
| [Pattern 8 — Prefix Sum + HashMap](#pattern-8-prefix-sum--hashmap) | Subarray sum count |
| [Pattern 9 — Cache Concepts](#pattern-9-cache-concepts-lru-cache) | LRU Cache |
| [Pattern 10 — Set-based Fast Lookup](#pattern-10-set-based-fast-lookup) | TreeSet window |
| **Interview Problems** | |
| [P1 — Two Sum](#problem-1--two-sum) | Classic |
| [P2 — Longest Consecutive Sequence](#problem-2--longest-consecutive-sequence) | HashSet |
| [P3 — Top K Frequent Elements](#problem-3--top-k-frequent-elements) | Heap + Map |
| [P4 — Subarray Sum Equals K](#problem-4--subarray-sum-equals-k) | Prefix + Map |
| [P5 — Intersection of Two Arrays](#problem-5--intersection-of-two-arrays) | Set |
| [P6 — Word Frequency Count](#problem-6--word-frequency-count) | Map |
| [P7 — Check if Array is Subset](#problem-7--check-if-array-is-subset) | Set lookup |
| [P8 — Group Anagrams](#problem-8--group-anagrams) | Map + sort key |
| [P9 — Ransom Note](#problem-9--ransom-note) | Frequency array |
| [P10 — First Missing Positive](#problem-10--first-missing-positive) | In-place hashing |

---

### 📌 Sections 4–7

| Section | Topic |
|---|---|
| [Pattern Recognition Cheat Sheet](#the-60-second-pattern-recognition-checklist) | Keyword → Pattern table |
| [5-Step Interview Approach](#the-5-step-interview-approach) | How to tackle any problem |
| [Complexity Quick Reference](#complexity-quick-reference) | Big-O guide |
| [Interview Problem 1 (Live)](#interview-problem-1-live) | Longest subarray sum = k |
| [Interview Problem 2 (Live)](#interview-problem-2-live) | Longest no-repeat substring |
| [Interview Problem 3 (Live)](#interview-problem-3-live) | Min in rotated sorted array |
| [Bug 1 — Off-By-One](#bug-1-off-by-one-errors) | Loop boundaries |
| [Bug 2 — Null Checks](#bug-2-null-checks) | NPE prevention |
| [Bug 3 — Integer Overflow](#bug-3-integer-overflow) | Safe mid formula |
| [Bug 4 — Duplicate Handling](#bug-4-duplicate-handling-in-two-pointers) | 3Sum edge case |
| [Bug 5 — Forgetting Shrink](#bug-5-forgetting-window-shrink) | Sliding window |
| [Bug 6 — HashMap Update Order](#bug-6-wrong-hashmap-update-order) | Two Sum trap |
| [Bug 7 — String Immutability](#bug-7-string-immutability) | char[] vs String |
| [Bug 8 — HashMap Null Unboxing](#bug-8-hashmapget-returning-null) | getOrDefault |
| [7-Day Study Plan](#section-7-7-day-revision-plan-) | Day by day schedule |
| [Most Repeated Patterns](#most-repeated-patterns-ranked-by-interview-frequency) | Ranked by frequency |
| [1-Hour Pre-Interview Checklist](#what-to-revise-1-hour-before-interview) | Quick revision |
| [How to Communicate](#how-to-communicate-during-a-coding-round) | What to say |
| [Golden Rules](#the-golden-rules-of-interviews) | 8 rules to live by |

---

# SECTION 1: ARRAYS MASTER GUIDE 📦

---

## What is an Array? (Beginner First)

An array is the simplest data structure. Think of it as a **row of boxes**, each holding one value, each box having a number (index) starting from 0.

```
Index:  0    1    2    3    4
      [ 10 ][ 30 ][ 20 ][ 50 ][ 40 ]

arr[0] = 10
arr[2] = 20
arr[4] = 40
```

**In Java:**
```java
int[] arr = {10, 30, 20, 50, 40};
int size = arr.length;   // 5
int first = arr[0];      // 10
int last = arr[arr.length - 1];  // 40
```

**The Guard Block — Always Write First:**
```java
if (arr == null || arr.length == 0) return;
```

---

## Pattern 1: Traversal

### 🧠 Intuition
Visit every element exactly once. Foundation of everything.

### When to Use
- Print elements, sum all, find min/max, count occurrences

### How to Recognize
- "Find all elements that...", "Sum of array", "Maximum value"

### Java Code

```java
// Forward traversal
for (int i = 0; i < arr.length; i++) {
    System.out.print(arr[i] + " ");
}

// Backward traversal
for (int i = arr.length - 1; i >= 0; i--) {
    System.out.print(arr[i] + " ");
}

// Enhanced for loop (when index not needed)
for (int num : arr) {
    System.out.print(num + " ");
}
```

### Dry Run
```
arr = [3, 1, 4, 1, 5]

Find maximum:
  max = arr[0] = 3
  i=1: arr[1]=1 < 3 → no update
  i=2: arr[2]=4 > 3 → max = 4
  i=3: arr[3]=1 < 4 → no update
  i=4: arr[4]=5 > 4 → max = 5

Answer: 5 ✅
```

### Time & Space
- Time: O(n)
- Space: O(1)

### ⚠️ Common Traps
```java
// ❌ Off by one: using <= instead of <
for (int i = 0; i <= arr.length; i++)  // crashes at i=arr.length
for (int i = 0; i < arr.length; i++)   // ✅ correct

// ❌ Modifying array while iterating
for (int num : arr) arr[0] = 5;  // unpredictable behavior
```

---

## Pattern 2: Two Pointers

### 🧠 Intuition
Use two variables (pointers) to scan the array from both ends or at different speeds. Eliminates the need for nested loops.

```
[1, 2, 3, 4, 5]
 ↑           ↑
left        right

Move left → or right ← depending on condition.
```

### When to Use
- Sorted array problems
- Finding pairs with a sum
- Reversing array in-place
- Removing duplicates
- Container with most water

### How to Recognize
Keywords: **"pair"**, **"sorted array"**, **"two elements"**, **"reverse"**, **"palindrome"**

### Brute Force vs Optimized

```java
// BRUTE FORCE: Find pair with target sum → O(n²)
for (int i = 0; i < arr.length; i++)
    for (int j = i+1; j < arr.length; j++)
        if (arr[i] + arr[j] == target) // found

// TWO POINTERS: Same problem on sorted array → O(n)
int left = 0, right = arr.length - 1;
while (left < right) {
    int sum = arr[left] + arr[right];
    if (sum == target) return true;
    else if (sum < target) left++;   // need bigger sum
    else right--;                    // need smaller sum
}
```

### Full Java Code

```java
// Problem: Two Sum in sorted array
public int[] twoSumSorted(int[] arr, int target) {
    int left = 0;
    int right = arr.length - 1;

    while (left < right) {
        int sum = arr[left] + arr[right];

        if (sum == target) {
            return new int[]{left, right};
        } else if (sum < target) {
            left++;    // sum too small → move left pointer right
        } else {
            right--;   // sum too big  → move right pointer left
        }
    }
    return new int[]{-1, -1};  // not found
}

// Problem: Reverse array in-place
public void reverse(int[] arr) {
    int left = 0, right = arr.length - 1;
    while (left < right) {
        int temp = arr[left];
        arr[left] = arr[right];
        arr[right] = temp;
        left++;
        right--;
    }
}
```

### Dry Run — Two Sum

```
arr = [1, 3, 5, 7, 9], target = 10

left=0 (val=1), right=4 (val=9)
  sum = 1+9 = 10 == target → return [0,4] ✅

arr = [1, 3, 5, 7, 9], target = 12

left=0 (1), right=4 (9) → sum=10 < 12 → left++
left=1 (3), right=4 (9) → sum=12 == 12 → return [1,4] ✅

arr = [1, 3, 5, 7, 9], target = 6

left=0 (1), right=4 (9) → sum=10 > 6 → right--
left=0 (1), right=3 (7) → sum=8  > 6 → right--
left=0 (1), right=2 (5) → sum=6 == 6 → return [0,2] ✅
```

### Time & Space
- Time: O(n)
- Space: O(1)

### ⚠️ Common Traps
```java
// ❌ Wrong condition: left <= right allows same element used twice
while (left <= right)   // wrong for pair problems
while (left < right)    // ✅ correct

// ❌ Not sorting first when required
// Two pointers ONLY works on sorted arrays for pair sum
Arrays.sort(arr);  // ← must do this first if array is unsorted
```

---

## Pattern 3: Sliding Window

### 🧠 Intuition
Maintain a "window" (subarray) that slides through the array. Instead of recomputing from scratch, you **add one element from the right and remove one from the left** — like sliding a frame across the array.

```
arr = [1, 3, 2, 6, 4, 1]
Window size = 3

[1  3  2] 6  4  1  → sum=6
 1 [3  2  6] 4  1  → sum=11
 1  3 [2  6  4] 1  → sum=12
 1  3  2 [6  4  1] → sum=11
```

### Two Types of Sliding Window

```
Fixed Window:   window size stays constant (k elements)
Variable Window: window shrinks/grows based on condition
```

### When to Use
- "Maximum/Minimum sum of k elements"
- "Longest subarray with condition"
- "Smallest subarray with sum ≥ target"

### How to Recognize
Keywords: **"subarray"**, **"contiguous"**, **"window"**, **"k elements"**, **"longest"**, **"shortest"**

### Fixed Window Java Code

```java
// Maximum sum of k consecutive elements
public int maxSumFixed(int[] arr, int k) {
    if (arr.length < k) return -1;

    // Build first window
    int windowSum = 0;
    for (int i = 0; i < k; i++) {
        windowSum += arr[i];
    }

    int maxSum = windowSum;

    // Slide the window
    for (int i = k; i < arr.length; i++) {
        windowSum += arr[i];       // add new element on right
        windowSum -= arr[i - k];   // remove old element on left
        maxSum = Math.max(maxSum, windowSum);
    }

    return maxSum;
}
```

### Dry Run — Fixed Window

```
arr = [2, 1, 5, 1, 3, 2], k = 3

Build first window: sum = 2+1+5 = 8, maxSum = 8

i=3: add arr[3]=1, remove arr[0]=2 → sum = 8+1-2 = 7, maxSum=8
i=4: add arr[4]=3, remove arr[1]=1 → sum = 7+3-1 = 9, maxSum=9
i=5: add arr[5]=2, remove arr[2]=5 → sum = 9+2-5 = 6, maxSum=9

Answer: 9 ✅  (window [5,1,3])
```

### Variable Window Java Code

```java
// Longest subarray with sum <= target
public int longestSubarray(int[] arr, int target) {
    int left = 0;
    int windowSum = 0;
    int maxLen = 0;

    for (int right = 0; right < arr.length; right++) {
        windowSum += arr[right];  // expand window

        // Shrink window from left when condition violated
        while (windowSum > target) {
            windowSum -= arr[left];
            left++;
        }

        // Window is valid → update answer
        maxLen = Math.max(maxLen, right - left + 1);
    }

    return maxLen;
}
```

### Dry Run — Variable Window

```
arr = [1, 2, 3, 4, 5], target = 7

right=0: sum=1, left=0 → valid, len=1, maxLen=1
right=1: sum=3, left=0 → valid, len=2, maxLen=2
right=2: sum=6, left=0 → valid, len=3, maxLen=3
right=3: sum=10 > 7 → shrink:
           remove arr[0]=1, left=1, sum=9
           sum=9 > 7 → shrink:
           remove arr[1]=2, left=2, sum=7
           valid! len=2, maxLen=3
right=4: sum=12 > 7 → shrink:
           remove arr[2]=3, left=3, sum=9
           remove arr[3]=4, left=4, sum=5
           valid! len=1, maxLen=3

Answer: 3 ✅  (subarray [1,2,3] or [3,4])
```

### Time & Space
- Time: O(n) — each element added and removed at most once
- Space: O(1)

### ⚠️ Common Traps
```java
// ❌ Forgetting to shrink window
// If you only expand and never shrink → O(n²) at best, wrong at worst

// ❌ Wrong window size formula
int len = right - left;      // ❌ off by one (misses 1 element)
int len = right - left + 1;  // ✅ correct

// ❌ Using if instead of while for shrinking
if (condition) { left++; }    // ❌ only shrinks once
while (condition) { left++; } // ✅ shrinks until valid
```

---

## Pattern 4: Prefix Sum

### 🧠 Intuition
Precompute cumulative sums so any range sum query runs in O(1) instead of O(n).

```
arr     =  [3,  1,  4,  1,  5,  9]
prefix  =  [0,  3,  4,  8,  9, 14, 23]
            ↑
         prefix[0]=0 (empty prefix, makes formula clean)

Sum from index 2 to 4 = prefix[5] - prefix[2] = 14 - 4 = 10
Verify: arr[2]+arr[3]+arr[4] = 4+1+5 = 10 ✅
```

### When to Use
- Multiple range sum queries
- "Subarray sum equals k"
- "Count subarrays with given sum"

### How to Recognize
Keywords: **"range sum"**, **"subarray sum"**, **"cumulative"**, **"queries"**

### Java Code

```java
// Build prefix sum
public int[] buildPrefix(int[] arr) {
    int n = arr.length;
    int[] prefix = new int[n + 1];  // 1-indexed, prefix[0] = 0
    for (int i = 0; i < n; i++) {
        prefix[i + 1] = prefix[i] + arr[i];
    }
    return prefix;
}

// Query: sum from index l to r (inclusive, 0-indexed)
public int rangeSum(int[] prefix, int l, int r) {
    return prefix[r + 1] - prefix[l];
}

// Count subarrays with sum = k
public int subarraySum(int[] arr, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1);  // empty prefix sum = 0, appears once

    int sum = 0, count = 0;
    for (int num : arr) {
        sum += num;
        // If (sum - k) exists as prefix sum → subarray with sum k found
        count += prefixCount.getOrDefault(sum - k, 0);
        prefixCount.put(sum, prefixCount.getOrDefault(sum, 0) + 1);
    }
    return count;
}
```

### Dry Run — Subarray Sum = k

```
arr = [1, 1, 1], k = 2

prefixCount = {0:1}, sum=0, count=0

num=1: sum=1, need sum-k=1-2=-1 → not found
       prefixCount = {0:1, 1:1}

num=1: sum=2, need 2-2=0 → found! count=1
       prefixCount = {0:1, 1:2}

num=1: sum=3, need 3-2=1 → found 2 times! count=3
       prefixCount = {0:1, 1:2, 2:1}  (wait, let's recount)
       Actually count += prefixCount.get(1) = 2 → count = 1+2 = 3? 
       
Hmm let's redo carefully:
After num=1 (first):  sum=1, need -1 → 0 added. count=0. map={0:1,1:1}
After num=1 (second): sum=2, need 0  → 1 added. count=1. map={0:1,1:2}
  Wait, why 1:2? Because sum=1 appears twice? No.
  After processing second element, sum=2. We put sum=2 in map.
  map={0:1,1:1,2:1}
After num=1 (third):  sum=3, need 1  → found 1 time. count=2.
       map={0:1,1:1,2:1,3:1}

Answer: 2 ✅ (subarrays [1,1] starting at 0 and at 1)
```

### Time & Space
- Build: O(n), Query: O(1)
- Subarray sum with HashMap: O(n)

---

## Pattern 5: Kadane's Algorithm

### 🧠 Intuition
Find the **maximum sum contiguous subarray**. The key insight: at each position, decide — should I **extend** the current subarray, or **start fresh** from here?

```
If currentSum + arr[i] > arr[i]  → extend (currentSum is positive, helps us)
If currentSum + arr[i] < arr[i]  → start fresh (currentSum is negative, hurts us)

Simplified: currentSum = max(arr[i], currentSum + arr[i])
```

### When to Use
- Maximum subarray sum
- Any "maximum/minimum subarray" variant

### How to Recognize
Keywords: **"maximum subarray"**, **"contiguous subarray"**, **"largest sum"**

### Java Code

```java
public int maxSubArray(int[] arr) {
    if (arr == null || arr.length == 0) return 0;

    int currentSum = arr[0];   // sum of current subarray
    int maxSum = arr[0];       // best sum seen so far

    for (int i = 1; i < arr.length; i++) {
        // Start fresh or extend?
        currentSum = Math.max(arr[i], currentSum + arr[i]);
        maxSum = Math.max(maxSum, currentSum);
    }

    return maxSum;
}
```

### Dry Run

```
arr = [-2, 1, -3, 4, -1, 2, 1, -5, 4]

i=0: currentSum=-2, maxSum=-2
i=1: max(1, -2+1=-1)=1,  currentSum=1,  maxSum=1
i=2: max(-3, 1-3=-2)=-2, currentSum=-2, maxSum=1
i=3: max(4, -2+4=2)=4,   currentSum=4,  maxSum=4
i=4: max(-1, 4-1=3)=3,   currentSum=3,  maxSum=4
i=5: max(2, 3+2=5)=5,    currentSum=5,  maxSum=5
i=6: max(1, 5+1=6)=6,    currentSum=6,  maxSum=6
i=7: max(-5, 6-5=1)=1,   currentSum=1,  maxSum=6
i=8: max(4, 1+4=5)=5,    currentSum=5,  maxSum=6

Answer: 6 ✅  (subarray [4,-1,2,1])
```

### Time & Space
- Time: O(n)
- Space: O(1)

### ⚠️ Common Traps
```java
// ❌ Initializing maxSum = 0 (wrong when all elements are negative)
int maxSum = 0;        // returns 0 for [-3,-1,-2] → WRONG
int maxSum = arr[0];   // ✅ returns -1 for [-3,-1,-2] → CORRECT

// ❌ Starting loop at i=0 when initialized with arr[0]
for (int i = 0; i < arr.length; i++)   // double-counts arr[0]
for (int i = 1; i < arr.length; i++)   // ✅ start at 1
```

---

## Pattern 6: Sorting + Greedy

### 🧠 Intuition
Sort the array first, then make locally optimal choices at each step. Sorting often reveals structure that makes the problem trivial.

### When to Use
- Meeting rooms / interval problems
- Minimum number of operations
- "Can you achieve X?" type problems

### Java Code — Merge Intervals

```java
public int[][] merge(int[][] intervals) {
    // Sort by start time
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);

    List<int[]> result = new ArrayList<>();
    int[] current = intervals[0];

    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] <= current[1]) {
            // Overlapping → merge
            current[1] = Math.max(current[1], intervals[i][1]);
        } else {
            // Non-overlapping → save current, start new
            result.add(current);
            current = intervals[i];
        }
    }
    result.add(current);
    return result.toArray(new int[0][]);
}
```

### Dry Run

```
intervals = [[1,3],[2,6],[8,10],[15,18]]

After sort: [[1,3],[2,6],[8,10],[15,18]] (already sorted)

current = [1,3]
i=1: [2,6] starts at 2 ≤ 3 (current end) → overlap!
     merge: current = [1, max(3,6)=6]

i=2: [8,10] starts at 8 > 6 → no overlap
     save [1,6], current = [8,10]

i=3: [15,18] starts at 15 > 10 → no overlap
     save [8,10], current = [15,18]

After loop: save [15,18]

Result: [[1,6],[8,10],[15,18]] ✅
```

---

## Pattern 7: Binary Search on Arrays

### 🧠 Intuition
On a SORTED array, find an element or answer in O(log n) by eliminating half the search space each step. (You've already learned this in depth!)

### When to Use
- Sorted array, find target
- Find first/last occurrence
- Find insertion position
- "Minimum possible maximum" / "Maximum possible minimum" (binary search on answer)

### Java Code

```java
public int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

---

## Pattern 8: In-Place Modification

### 🧠 Intuition
Modify the array without using extra space. Usually done by keeping a "write pointer" that only advances when a valid element is found.

### When to Use
- Remove duplicates from sorted array
- Move zeros to end
- Filter elements in-place

### Java Code

```java
// Remove duplicates from sorted array in-place
public int removeDuplicates(int[] arr) {
    if (arr.length == 0) return 0;

    int writePtr = 1;  // next position to write unique element

    for (int readPtr = 1; readPtr < arr.length; readPtr++) {
        if (arr[readPtr] != arr[readPtr - 1]) {  // new unique element
            arr[writePtr] = arr[readPtr];
            writePtr++;
        }
    }
    return writePtr;  // new length
}

// Move zeros to end, maintain relative order
public void moveZeros(int[] arr) {
    int writePtr = 0;

    // First: move all non-zeros forward
    for (int readPtr = 0; readPtr < arr.length; readPtr++) {
        if (arr[readPtr] != 0) {
            arr[writePtr++] = arr[readPtr];
        }
    }
    // Then: fill rest with zeros
    while (writePtr < arr.length) {
        arr[writePtr++] = 0;
    }
}
```

### Dry Run — Move Zeros

```
arr = [0, 1, 0, 3, 12]

writePtr = 0

readPtr=0: arr[0]=0 → skip
readPtr=1: arr[1]=1 ≠ 0 → arr[0]=1, writePtr=1
readPtr=2: arr[2]=0 → skip
readPtr=3: arr[3]=3 ≠ 0 → arr[1]=3, writePtr=2
readPtr=4: arr[4]=12 ≠ 0 → arr[2]=12, writePtr=3

arr = [1, 3, 12, 3, 12]  ← only first 3 matter

Fill zeros from writePtr=3:
arr[3]=0, arr[4]=0

Final: [1, 3, 12, 0, 0] ✅
```

---

## Pattern 9: Merge Intervals

Already covered in Pattern 6. Additional variant:

### Insert Interval

```java
public int[][] insert(int[][] intervals, int[] newInterval) {
    List<int[]> result = new ArrayList<>();
    int i = 0, n = intervals.length;

    // Add all intervals that end before new interval starts
    while (i < n && intervals[i][1] < newInterval[0]) {
        result.add(intervals[i++]);
    }

    // Merge overlapping intervals with newInterval
    while (i < n && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
        newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
        i++;
    }
    result.add(newInterval);

    // Add remaining intervals
    while (i < n) result.add(intervals[i++]);

    return result.toArray(new int[0][]);
}
```

---

## Pattern 10: Frequency Counting

### 🧠 Intuition
Count how many times each element appears. Use an array (if range is small) or HashMap (if range is large/unknown).

### Java Code

```java
// Count frequencies using array (for small range like 0-9)
public int[] frequencyArray(int[] arr, int range) {
    int[] freq = new int[range + 1];
    for (int num : arr) freq[num]++;
    return freq;
}

// Count frequencies using HashMap (for any range)
public Map<Integer, Integer> frequencyMap(int[] arr) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : arr) {
        freq.put(num, freq.getOrDefault(num, 0) + 1);
    }
    return freq;
}

// Find element appearing more than n/2 times (Boyer-Moore)
public int majorityElement(int[] arr) {
    int candidate = arr[0], count = 1;
    for (int i = 1; i < arr.length; i++) {
        if (count == 0) {
            candidate = arr[i];
            count = 1;
        } else if (arr[i] == candidate) {
            count++;
        } else {
            count--;
        }
    }
    return candidate;
}
```

---

## 10 Real Array Interview Problems

---

### Problem 1 — Two Sum (Unsorted)
> "Given array of integers, return indices of two numbers that add up to target."

```java
// HashMap approach: O(n) time, O(n) space
public int[] twoSum(int[] arr, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < arr.length; i++) {
        int complement = target - arr[i];
        if (seen.containsKey(complement)) {
            return new int[]{seen.get(complement), i};
        }
        seen.put(arr[i], i);
    }
    return new int[]{-1, -1};
}
```

**Dry Run:**
```
arr=[2,7,11,15], target=9
i=0: complement=7, not in map. map={2:0}
i=1: complement=2, IN MAP! return [0,1] ✅
```

---

### Problem 2 — Best Time to Buy and Sell Stock
> "Find maximum profit from one buy and one sell."

```java
public int maxProfit(int[] prices) {
    int minPrice = Integer.MAX_VALUE;
    int maxProfit = 0;

    for (int price : prices) {
        minPrice = Math.min(minPrice, price);      // best buy so far
        maxProfit = Math.max(maxProfit, price - minPrice); // best sell
    }
    return maxProfit;
}
```

**Dry Run:**
```
prices = [7,1,5,3,6,4]
min=7, profit=0
price=1: min=1, profit=max(0,1-1)=0
price=5: min=1, profit=max(0,5-1)=4
price=3: min=1, profit=max(4,3-1)=4
price=6: min=1, profit=max(4,6-1)=5
price=4: min=1, profit=max(5,4-1)=5
Answer: 5 ✅
```

---

### Problem 3 — Contains Duplicate
> "Return true if any value appears at least twice."

```java
public boolean containsDuplicate(int[] arr) {
    Set<Integer> seen = new HashSet<>();
    for (int num : arr) {
        if (!seen.add(num)) return true;  // add() returns false if already present
    }
    return false;
}
```

---

### Problem 4 — Product of Array Except Self
> "Return array where output[i] = product of all elements except arr[i]. No division allowed."

```java
public int[] productExceptSelf(int[] arr) {
    int n = arr.length;
    int[] result = new int[n];

    // Left pass: result[i] = product of all elements to LEFT of i
    result[0] = 1;
    for (int i = 1; i < n; i++) {
        result[i] = result[i-1] * arr[i-1];
    }

    // Right pass: multiply by product of all elements to RIGHT of i
    int rightProduct = 1;
    for (int i = n-1; i >= 0; i--) {
        result[i] *= rightProduct;
        rightProduct *= arr[i];
    }

    return result;
}
```

**Dry Run:**
```
arr = [1, 2, 3, 4]

Left pass:
result = [1, 1, 2, 6]
         (1)(1)(1×2)(2×3)

Right pass (rightProduct starts=1):
i=3: result[3]=6×1=6,   rightProduct=1×4=4
i=2: result[2]=2×4=8,   rightProduct=4×3=12
i=1: result[1]=1×12=12, rightProduct=12×2=24
i=0: result[0]=1×24=24, rightProduct=24×1=24

result = [24, 12, 8, 6] ✅
```

---

### Problem 5 — Maximum Subarray (Kadane's)
Already covered in Pattern 5.

---

### Problem 6 — 3Sum
> "Find all triplets that sum to zero. No duplicate triplets."

```java
public List<List<Integer>> threeSum(int[] arr) {
    Arrays.sort(arr);
    List<List<Integer>> result = new ArrayList<>();

    for (int i = 0; i < arr.length - 2; i++) {
        if (i > 0 && arr[i] == arr[i-1]) continue;  // skip duplicates

        int left = i + 1, right = arr.length - 1;
        while (left < right) {
            int sum = arr[i] + arr[left] + arr[right];
            if (sum == 0) {
                result.add(Arrays.asList(arr[i], arr[left], arr[right]));
                while (left < right && arr[left] == arr[left+1]) left++;
                while (left < right && arr[right] == arr[right-1]) right--;
                left++;
                right--;
            } else if (sum < 0) left++;
            else right--;
        }
    }
    return result;
}
```

---

### Problem 7 — Trapping Rain Water
> "Given elevation map, compute how much water it can trap."

```java
// Two pointer approach O(n) time O(1) space
public int trap(int[] height) {
    int left = 0, right = height.length - 1;
    int leftMax = 0, rightMax = 0;
    int water = 0;

    while (left < right) {
        if (height[left] < height[right]) {
            if (height[left] >= leftMax) leftMax = height[left];
            else water += leftMax - height[left];
            left++;
        } else {
            if (height[right] >= rightMax) rightMax = height[right];
            else water += rightMax - height[right];
            right--;
        }
    }
    return water;
}
```

---

### Problem 8 — Rotate Array
> "Rotate array to the right by k steps."

```java
public void rotate(int[] arr, int k) {
    k = k % arr.length;  // handle k > length
    reverse(arr, 0, arr.length - 1);  // reverse whole array
    reverse(arr, 0, k - 1);           // reverse first k
    reverse(arr, k, arr.length - 1);  // reverse rest
}

private void reverse(int[] arr, int l, int r) {
    while (l < r) {
        int temp = arr[l]; arr[l] = arr[r]; arr[r] = temp;
        l++; r--;
    }
}
```

---

### Problem 9 — Find Minimum in Rotated Sorted Array
> "Array was sorted then rotated. Find minimum."

```java
public int findMin(int[] arr) {
    int left = 0, right = arr.length - 1;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] > arr[right]) left = mid + 1;  // min is in right half
        else right = mid;                            // min is in left half (including mid)
    }
    return arr[left];
}
```

---

### Problem 10 — Subarray Sum Equals K
Already covered in Pattern 4 (Prefix Sum with HashMap).

---

---

# SECTION 2: STRINGS MASTER GUIDE 🔤

---

## What is a String in Java? (Beginner First)

A String is a **sequence of characters**. In Java, strings are **immutable** — once created, they cannot be changed. Every "modification" creates a new String object.

```java
String s = "hello";
s.length()      // 5
s.charAt(0)     // 'h'
s.charAt(4)     // 'o'
s.substring(1,3) // "el"  → from index 1 (inclusive) to 3 (exclusive)
s.toCharArray() // ['h','e','l','l','o']
```

**The Most Critical Java String Rule:**
```java
// ❌ NEVER concatenate strings in a loop (creates new object each time → O(n²))
String result = "";
for (char c : arr) result += c;  // creates new String each iteration!

// ✅ ALWAYS use StringBuilder in loops
StringBuilder sb = new StringBuilder();
for (char c : arr) sb.append(c);
String result = sb.toString();
```

---

## Pattern 1: Character Frequency Count

### 🧠 Intuition
Count how many times each character appears. The building block of anagram, palindrome, and uniqueness problems.

### When to Use
- "Is this an anagram?"
- "Find first non-repeating character"
- "Check if two strings have same characters"

### How to Recognize
Keywords: **"characters"**, **"frequency"**, **"count"**, **"anagram"**, **"unique"**

### Two Approaches

```java
// Approach 1: Array (fastest, only for lowercase a-z)
int[] freq = new int[26];
for (char c : s.toCharArray()) {
    freq[c - 'a']++;  // 'a' maps to 0, 'b' to 1, etc.
}

// Approach 2: HashMap (works for any characters including Unicode)
Map<Character, Integer> freq = new HashMap<>();
for (char c : s.toCharArray()) {
    freq.put(c, freq.getOrDefault(c, 0) + 1);
}
```

### Full Example — First Unique Character

```java
public int firstUniqChar(String s) {
    int[] freq = new int[26];

    // Count all characters
    for (char c : s.toCharArray()) {
        freq[c - 'a']++;
    }

    // Find first with count = 1
    for (int i = 0; i < s.length(); i++) {
        if (freq[s.charAt(i) - 'a'] == 1) return i;
    }

    return -1;  // no unique character
}
```

### Dry Run
```
s = "leetcode"

Count:
l:1, e:3, t:1, c:1, o:1, d:1

Second pass:
i=0: 'l' → freq=1 → return 0 ✅

s = "aabb"
Count: a:2, b:2
Second pass: all freq > 1 → return -1 ✅
```

### ⚠️ Common Bugs
```java
// ❌ Wrong index calculation
freq[c]           // uses ASCII value (97 for 'a') → index out of bounds for size-26 array
freq[c - 'a']     // ✅ maps 'a'→0, 'b'→1, ..., 'z'→25

// ❌ Assuming lowercase only without checking
// Always ask: "Are there uppercase letters? Numbers? Spaces?"
// If yes → use HashMap instead of int[26]
```

---

## Pattern 2: Two Pointers on Strings

### 🧠 Intuition
Use left and right pointers moving toward center. Most useful for palindrome checking and reversal problems.

### When to Use
- "Is this string a palindrome?"
- "Reverse a string"
- "Valid palindrome ignoring non-alphanumeric"

### Java Code

```java
// Check palindrome
public boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) return false;
        left++;
        right--;
    }
    return true;
}

// Valid palindrome ignoring non-alphanumeric (interview version)
public boolean isPalindromeII(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        // Skip non-alphanumeric from left
        while (left < right && !Character.isLetterOrDigit(s.charAt(left))) left++;
        // Skip non-alphanumeric from right
        while (left < right && !Character.isLetterOrDigit(s.charAt(right))) right--;

        if (Character.toLowerCase(s.charAt(left)) !=
            Character.toLowerCase(s.charAt(right))) return false;
        left++;
        right--;
    }
    return true;
}
```

### Dry Run
```
s = "A man, a plan, a canal: Panama"

left=0 ('A'), right=29 ('a')
toLowerCase: 'a' == 'a' ✅ → left++, right--

left=1 (' ') → skip
left=2 ('m'), right=28 ('m') → 'm'=='m' ✅

... (continues matching from outside in)

All characters match → return true ✅
```

---

## Pattern 3: Sliding Window on Strings

### 🧠 Intuition
Variable-size window that expands and contracts to find optimal substring. Identical to array sliding window but we track character frequencies.

### When to Use
- "Longest substring without repeating characters"
- "Minimum window substring"
- "Longest substring with at most k distinct characters"

### Java Code — Longest Substring Without Repeating

```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int left = 0, maxLen = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);

        // If character seen and is inside current window → shrink
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1;  // move left past the duplicate
        }

        lastSeen.put(c, right);  // update last seen position
        maxLen = Math.max(maxLen, right - left + 1);
    }

    return maxLen;
}
```

### Dry Run
```
s = "abcabcbb"

right=0 'a': not seen. map={a:0}. len=1, max=1
right=1 'b': not seen. map={a:0,b:1}. len=2, max=2
right=2 'c': not seen. map={a:0,b:1,c:2}. len=3, max=3
right=3 'a': seen at 0, >= left(0) → left=0+1=1
             map={a:3,b:1,c:2}. len=3-1+1=3, max=3
right=4 'b': seen at 1, >= left(1) → left=1+1=2
             map={a:3,b:4,c:2}. len=4-2+1=3, max=3
right=5 'c': seen at 2, >= left(2) → left=2+1=3
             map={a:3,b:4,c:5}. len=5-3+1=3, max=3
right=6 'b': seen at 4, >= left(3) → left=4+1=5
             map={a:3,b:6,c:5}. len=6-5+1=2, max=3
right=7 'b': seen at 6, >= left(5) → left=6+1=7
             len=7-7+1=1, max=3

Answer: 3 ✅ ("abc")
```

### Java Code — Minimum Window Substring

```java
public String minWindow(String s, String t) {
    if (s.isEmpty() || t.isEmpty()) return "";

    // Count what we need
    Map<Character, Integer> need = new HashMap<>();
    for (char c : t.toCharArray()) need.put(c, need.getOrDefault(c, 0) + 1);

    int left = 0, formed = 0, required = need.size();
    Map<Character, Integer> window = new HashMap<>();
    int[] ans = {-1, 0, 0};  // [window length, left, right]

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        window.put(c, window.getOrDefault(c, 0) + 1);

        // Check if current character satisfies requirement
        if (need.containsKey(c) && window.get(c).intValue() == need.get(c).intValue()) {
            formed++;
        }

        // Try to shrink window
        while (formed == required) {
            // Update answer if smaller window found
            if (ans[0] == -1 || right - left + 1 < ans[0]) {
                ans[0] = right - left + 1;
                ans[1] = left;
                ans[2] = right;
            }

            char leftChar = s.charAt(left);
            window.put(leftChar, window.get(leftChar) - 1);
            if (need.containsKey(leftChar) && window.get(leftChar) < need.get(leftChar)) {
                formed--;
            }
            left++;
        }
    }

    return ans[0] == -1 ? "" : s.substring(ans[1], ans[2] + 1);
}
```

---

## Pattern 4: Palindrome Problems

### 🧠 Intuition
A palindrome reads the same forward and backward. Key observations:
- Odd-length palindrome has one center character
- Even-length palindrome has two center characters
- Every palindrome can be expanded from its center

### Expand Around Center Approach

```java
// Longest palindromic substring
public String longestPalindrome(String s) {
    if (s.length() < 2) return s;
    int start = 0, maxLen = 1;

    for (int i = 0; i < s.length(); i++) {
        // Odd length palindromes (single center)
        int len1 = expand(s, i, i);
        // Even length palindromes (double center)
        int len2 = expand(s, i, i + 1);

        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            start = i - (len - 1) / 2;
        }
    }

    return s.substring(start, start + maxLen);
}

private int expand(String s, int left, int right) {
    while (left >= 0 && right < s.length()
            && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
    }
    return right - left - 1;  // length of palindrome
}
```

### Dry Run
```
s = "babad"

i=0 'b': odd expand from (0,0):
  (0,0): 'b'=='b' ✅ expand → (-1,1) out of bounds
  len = 1-(-1)-1 = 1
  even expand from (0,1): 'b'≠'a' → len=0

i=1 'a': odd expand from (1,1):
  (1,1):'a'='a' ✅ → (0,2):'b'='b' ✅ → (-1,3) out of bounds
  len = 3-(-1)-1 = 3 → "bab"! maxLen=3, start=0

i=2 'b': odd expand from (2,2):
  (2,2)→(1,3):'a'='a'✅→(0,4):'b'='d'❌
  len = 3-0-1 = ... let me redo
  Actually after (2,2) passes, left=1,right=3: s[1]='a',s[3]='a' ✅
  After that left=0,right=4: s[0]='b',s[4]='d' ❌ stop
  len = 4-0-1 = 3 → "aba", maxLen stays 3

Answer: "bab" (or "aba", both valid) ✅
```

---

## Pattern 5: Anagram Problems

### 🧠 Intuition
Two strings are anagrams if they contain the same characters with the same frequencies.

**The key trick:** Sort both strings — if they're equal, they're anagrams. Or use frequency count.

### Java Code

```java
// Check if two strings are anagrams
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;

    int[] freq = new int[26];
    for (int i = 0; i < s.length(); i++) {
        freq[s.charAt(i) - 'a']++;   // increment for s
        freq[t.charAt(i) - 'a']--;   // decrement for t
    }

    for (int count : freq) {
        if (count != 0) return false;
    }
    return true;
}

// Find all anagram start positions in s (sliding window)
public List<Integer> findAnagrams(String s, String p) {
    List<Integer> result = new ArrayList<>();
    if (s.length() < p.length()) return result;

    int[] pFreq = new int[26];
    int[] windowFreq = new int[26];

    for (char c : p.toCharArray()) pFreq[c - 'a']++;

    for (int right = 0; right < s.length(); right++) {
        windowFreq[s.charAt(right) - 'a']++;

        // Remove element leaving window
        if (right >= p.length()) {
            windowFreq[s.charAt(right - p.length()) - 'a']--;
        }

        if (Arrays.equals(pFreq, windowFreq)) {
            result.add(right - p.length() + 1);
        }
    }

    return result;
}
```

### Dry Run — Find Anagrams
```
s = "cbaebabacd", p = "abc"
pFreq = [1,1,1,0,...] (a=1,b=1,c=1)

right=0 'c': window=[0,0,1,...] right<3
right=1 'b': window=[0,1,1,...] right<3
right=2 'a': window=[1,1,1,...] right=2=p.length-1
  Arrays.equals? YES → add 2-3+1=0 ✅

right=3 'e': add 'e', remove s[0]='c'
  window=[1,1,0,0,1,...] → no match

right=4 'b': add 'b', remove s[1]='b'
  window=[1,1,0,0,1,...] → no match

right=5 'a': add 'a', remove s[2]='a'
  window=[1,1,0,0,1,...] → no match

right=6 'b': add 'b', remove s[3]='e'
  window=[1,2,0,...] → no match

right=7 'a': add 'a', remove s[4]='b'
  window=[2,1,0,...] → no match

right=8 'c': add 'c', remove s[5]='a'
  window=[1,1,1,...] → match! add 8-3+1=6 ✅

right=9 'd': add 'd', remove s[6]='b'
  window=[1,0,1,1,...] → no match

Answer: [0, 6] ✅
```

---

## Pattern 6: Substring Problems

### Java Code — Count Distinct Substrings

```java
// All substrings of a string
public List<String> allSubstrings(String s) {
    List<String> result = new ArrayList<>();
    for (int i = 0; i < s.length(); i++) {
        for (int j = i + 1; j <= s.length(); j++) {
            result.add(s.substring(i, j));
        }
    }
    return result;  // O(n²) substrings
}
```

**Key insight:** For a string of length n:
- Number of substrings = n*(n+1)/2
- Always question if you can do better than O(n²)

---

## Pattern 7: StringBuilder Optimization

### 🧠 Intuition
String concatenation in Java using `+` is O(n) each time because it creates a new String. In a loop of n iterations → O(n²) total. StringBuilder is O(1) for append.

```java
// ❌ SLOW: O(n²)
public String reverseStringSlow(String s) {
    String result = "";
    for (int i = s.length()-1; i >= 0; i--) {
        result += s.charAt(i);  // creates new String each time!
    }
    return result;
}

// ✅ FAST: O(n)
public String reverseStringFast(String s) {
    StringBuilder sb = new StringBuilder();
    for (int i = s.length()-1; i >= 0; i--) {
        sb.append(s.charAt(i));  // O(1) append
    }
    return sb.toString();
}

// Even faster: built-in
public String reverseBuiltIn(String s) {
    return new StringBuilder(s).reverse().toString();
}
```

---

## Pattern 8: HashMap on Strings

### Java Code — Group Anagrams

```java
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();

    for (String s : strs) {
        // Sort characters to get canonical form
        char[] chars = s.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);

        map.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }

    return new ArrayList<>(map.values());
}
```

**Dry Run:**
```
strs = ["eat","tea","tan","ate","nat","bat"]

"eat" → sorted "aet" → key="aet" → map={"aet":["eat"]}
"tea" → sorted "aet" → key="aet" → map={"aet":["eat","tea"]}
"tan" → sorted "ant" → key="ant" → map={"aet":[...],"ant":["tan"]}
"ate" → sorted "aet" → map={"aet":["eat","tea","ate"],...}
"nat" → sorted "ant" → map={...,"ant":["tan","nat"]}
"bat" → sorted "abt" → map={...,"abt":["bat"]}

Result: [["eat","tea","ate"],["tan","nat"],["bat"]] ✅
```

---

## Pattern 9: Pattern Matching

### KMP Algorithm Concept (Simplified)

```java
// Simple contains check (Java built-in O(n*m) worst case)
s.contains(pattern);
s.indexOf(pattern);  // returns starting index or -1

// For interviews, knowing built-in is usually enough
// For hard problems, mention KMP as O(n+m) alternative
```

---

## Pattern 10: Parsing / Tokenization

```java
// Split by delimiter
String[] words = s.split(" ");       // split by space
String[] parts = s.split(",");       // split by comma
String[] lines = s.split("\\n");     // split by newline

// Trim whitespace
s = s.trim();             // removes leading/trailing spaces
s = s.strip();            // Unicode-aware version of trim

// Check character type
Character.isLetter(c)     // a-z, A-Z
Character.isDigit(c)      // 0-9
Character.isLetterOrDigit(c)
Character.isWhitespace(c)
Character.toLowerCase(c)
Character.toUpperCase(c)

// String to number and back
int n = Integer.parseInt("123");
String s = String.valueOf(123);
String s = Integer.toString(123);
```

---

## 10 Real Interview String Problems

---

### Problem 1 — Valid Anagram
Already covered in Pattern 5.

---

### Problem 2 — Longest Substring Without Repeating
Already covered in Pattern 3.

---

### Problem 3 — Valid Palindrome
Already covered in Pattern 2.

---

### Problem 4 — Reverse Words in a String
> "Reverse the order of words in a sentence."

```java
public String reverseWords(String s) {
    String[] words = s.trim().split("\\s+");  // \\s+ handles multiple spaces
    StringBuilder sb = new StringBuilder();

    for (int i = words.length - 1; i >= 0; i--) {
        sb.append(words[i]);
        if (i > 0) sb.append(" ");
    }
    return sb.toString();
}
```

**Dry Run:**
```
s = "  the sky is blue  "
trim → "the sky is blue"
split → ["the","sky","is","blue"]
reverse order: "blue is sky the" ✅
```

---

### Problem 5 — Longest Common Prefix
> "Find longest common prefix among array of strings."

```java
public String longestCommonPrefix(String[] strs) {
    if (strs == null || strs.length == 0) return "";

    String prefix = strs[0];  // assume first string is prefix

    for (int i = 1; i < strs.length; i++) {
        // Shrink prefix until it matches start of strs[i]
        while (!strs[i].startsWith(prefix)) {
            prefix = prefix.substring(0, prefix.length() - 1);
            if (prefix.isEmpty()) return "";
        }
    }
    return prefix;
}
```

---

### Problem 6 — String Compression
> "Compress string aabcccdddd → a2b1c3d4"

```java
public String compress(String s) {
    StringBuilder sb = new StringBuilder();
    int i = 0;
    while (i < s.length()) {
        char c = s.charAt(i);
        int count = 0;
        while (i < s.length() && s.charAt(i) == c) {
            count++;
            i++;
        }
        sb.append(c);
        if (count > 1) sb.append(count);
    }
    String result = sb.toString();
    return result.length() < s.length() ? result : s;
}
```

---

### Problem 7 — Roman to Integer

```java
public int romanToInt(String s) {
    Map<Character, Integer> values = new HashMap<>();
    values.put('I',1); values.put('V',5); values.put('X',10);
    values.put('L',50); values.put('C',100); values.put('D',500);
    values.put('M',1000);

    int result = 0;
    for (int i = 0; i < s.length(); i++) {
        int curr = values.get(s.charAt(i));
        int next = (i+1 < s.length()) ? values.get(s.charAt(i+1)) : 0;

        if (curr < next) result -= curr;  // subtraction case: IV, IX
        else result += curr;
    }
    return result;
}
```

---

### Problem 8 — Count and Say

```java
public String countAndSay(int n) {
    String result = "1";
    for (int i = 1; i < n; i++) {
        StringBuilder sb = new StringBuilder();
        int j = 0;
        while (j < result.length()) {
            char c = result.charAt(j);
            int count = 0;
            while (j < result.length() && result.charAt(j) == c) {
                count++;
                j++;
            }
            sb.append(count).append(c);
        }
        result = sb.toString();
    }
    return result;
}
```

---

### Problem 9 — Decode String
> "s = '3[a]2[bc]' → 'aaabcbc'"

```java
public String decodeString(String s) {
    Stack<Integer> counts = new Stack<>();
    Stack<StringBuilder> builders = new Stack<>();
    StringBuilder current = new StringBuilder();
    int k = 0;

    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) {
            k = k * 10 + (c - '0');  // handle multi-digit numbers
        } else if (c == '[') {
            counts.push(k);
            builders.push(current);
            current = new StringBuilder();
            k = 0;
        } else if (c == ']') {
            int repeat = counts.pop();
            StringBuilder prev = builders.pop();
            for (int i = 0; i < repeat; i++) prev.append(current);
            current = prev;
        } else {
            current.append(c);
        }
    }
    return current.toString();
}
```

---

### Problem 10 — Minimum Window Substring
Already covered in Pattern 3.

---

---

# SECTION 3: HASHMAP / HASHSET MASTER GUIDE 🗺️

---

## What is a HashMap? (Beginner First)

Think of a HashMap as a **dictionary**. You look up a word (key) and instantly get its definition (value). No scanning from page 1 — direct jump to the answer.

```
Key   →   Value
"apple"  →   5
"banana" →   3
"cherry" →   8

map.get("apple")  →  5   (O(1) lookup!)
map.get("mango")  →  null (not found)
```

**In Java:**
```java
HashMap<String, Integer> map = new HashMap<>();

map.put("apple", 5);                    // add/update
map.get("apple");                       // returns 5
map.getOrDefault("mango", 0);          // returns 0 if not found
map.containsKey("apple");              // true
map.containsValue(5);                  // true
map.remove("apple");                   // remove entry
map.size();                            // number of entries
map.isEmpty();                         // true if no entries

// Iterate
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String key = entry.getKey();
    Integer value = entry.getValue();
}
```

---

## What is a HashSet?

A HashSet is like a HashMap but stores **only keys, no values**. Used for fast existence checking.

```java
HashSet<Integer> set = new HashSet<>();

set.add(5);           // add element
set.add(5);           // duplicate → ignored
set.contains(5);      // true → O(1)
set.remove(5);        // remove
set.size();           // number of elements
```

---

## How HashMap Works Internally (Interview Gold)

```
put("apple", 5):
  1. Compute hashCode("apple") → some integer → e.g., 3
  2. index = hashCode % capacity → 3 % 16 = 3
  3. Store ("apple", 5) at bucket[3]

get("apple"):
  1. Compute hashCode("apple") → 3
  2. Go to bucket[3]
  3. Return value 5

Collision (two keys map to same bucket):
  → Java uses Linked List (or Red-Black Tree if > 8 entries in bucket)
  → get() then traverses the list to find exact key
```

**Why O(1) amortized?**
- Average case: hash distributes evenly, each bucket has ~1 entry
- Worst case: all keys hash to same bucket → O(n) (but extremely rare with good hash)

---

## Pattern 1: Frequency Count

### 🧠 Intuition
The most common HashMap use case. Count occurrences of each element.

```java
public Map<Integer, Integer> countFreq(int[] arr) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : arr) {
        freq.put(num, freq.getOrDefault(num, 0) + 1);
    }
    return freq;
}
```

**Interview Pro Tip — `getOrDefault` vs `containsKey`:**
```java
// ❌ Verbose
if (map.containsKey(key)) map.put(key, map.get(key) + 1);
else map.put(key, 1);

// ✅ Concise
map.put(key, map.getOrDefault(key, 0) + 1);

// ✅ Even cleaner (Java 8+)
map.merge(key, 1, Integer::sum);
```

---

## Pattern 2: Lookup Optimization

### 🧠 Intuition
Replace O(n) linear scan with O(1) HashMap lookup. Classic pattern: "Have I seen this before?"

```java
// Two Sum: for each number, check if complement exists
public int[] twoSum(int[] arr, int target) {
    Map<Integer, Integer> seen = new HashMap<>();  // value → index
    for (int i = 0; i < arr.length; i++) {
        int complement = target - arr[i];
        if (seen.containsKey(complement)) {
            return new int[]{seen.get(complement), i};
        }
        seen.put(arr[i], i);
    }
    return new int[]{-1, -1};
}
```

---

## Pattern 3: Duplicate Detection

```java
// Check if array has duplicates
public boolean hasDuplicate(int[] arr) {
    Set<Integer> seen = new HashSet<>();
    for (int num : arr) {
        if (!seen.add(num)) return true;  // add() returns false if duplicate
    }
    return false;
}

// Find all duplicates
public List<Integer> findDuplicates(int[] arr) {
    Set<Integer> seen = new HashSet<>();
    List<Integer> duplicates = new ArrayList<>();
    for (int num : arr) {
        if (!seen.add(num)) duplicates.add(num);
    }
    return duplicates;
}
```

---

## Pattern 4: Pair Sum Problems

### 🧠 Intuition
Store elements you've seen. For each new element, check if its complement was already seen.

```java
// Count pairs with given sum
public int countPairs(int[] arr, int target) {
    Map<Integer, Integer> freq = new HashMap<>();
    int count = 0;

    for (int num : arr) {
        int complement = target - num;
        count += freq.getOrDefault(complement, 0);
        freq.put(num, freq.getOrDefault(num, 0) + 1);
    }
    return count;
}
```

**Dry Run:**
```
arr = [1, 5, 3, 3, 3], target = 6

num=1: complement=5, freq={} → 0 pairs. freq={1:1}
num=5: complement=1, freq has 1 → 1 pair. freq={1:1,5:1}
num=3: complement=3, freq has no 3 → 0. freq={1:1,5:1,3:1}
num=3: complement=3, freq has 3(count=1) → 1 pair. freq={1:1,5:1,3:2}
num=3: complement=3, freq has 3(count=2) → 2 pairs. freq={1:1,5:1,3:3}

Total: 0+1+0+1+2 = 4 pairs ✅
Verify: (1,5),(3,3),(3,3),(3,3) — 4 pairs ✅
```

---

## Pattern 5: Grouping Problems

### 🧠 Intuition
Group elements that share a common property. Use the property as the key, list of elements as value.

```java
// Group anagrams
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> groups = new HashMap<>();
    for (String s : strs) {
        char[] arr = s.toCharArray();
        Arrays.sort(arr);
        String key = new String(arr);  // sorted string as group key
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(groups.values());
}

// Group integers by their remainder
public Map<Integer, List<Integer>> groupByRemainder(int[] arr, int k) {
    Map<Integer, List<Integer>> groups = new HashMap<>();
    for (int num : arr) {
        int rem = num % k;
        groups.computeIfAbsent(rem, r -> new ArrayList<>()).add(num);
    }
    return groups;
}
```

---

## Pattern 6: Counting Distinct

```java
// Count distinct elements
public int countDistinct(int[] arr) {
    return new HashSet<>(Arrays.stream(arr).boxed()
                              .collect(Collectors.toList())).size();
}

// Simpler:
Set<Integer> set = new HashSet<>();
for (int num : arr) set.add(num);
return set.size();
```

---

## Pattern 7: Index Mapping

### 🧠 Intuition
Map value → index. Useful when you need to quickly find WHERE an element is, not just IF it exists.

```java
// Longest consecutive sequence
public int longestConsecutive(int[] arr) {
    Set<Integer> set = new HashSet<>();
    for (int num : arr) set.add(num);

    int maxLen = 0;

    for (int num : set) {
        // Only start counting from sequence beginnings
        if (!set.contains(num - 1)) {
            int current = num;
            int length = 1;

            while (set.contains(current + 1)) {
                current++;
                length++;
            }
            maxLen = Math.max(maxLen, length);
        }
    }
    return maxLen;
}
```

**Dry Run:**
```
arr = [100, 4, 200, 1, 3, 2]
set = {100, 4, 200, 1, 3, 2}

num=100: no 99 in set → start sequence
  101 not in set → length=1, maxLen=1

num=4: no 3? Wait, 3 IS in set → NOT a start, skip

num=200: no 199 → start
  201 not in set → length=1

num=1: no 0 in set → start!
  2 in set → length=2
  3 in set → length=3
  4 in set → length=4
  5 not in set → stop. maxLen=4

Answer: 4 ✅ (sequence 1,2,3,4)
```

---

## Pattern 8: Prefix Sum + HashMap

### 🧠 Intuition
Combine prefix sums with HashMap to count subarrays with exact sum in O(n).

Already covered in Arrays Pattern 4 — Subarray Sum Equals K.

```java
// Count subarrays with sum = k
public int subarraySum(int[] arr, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1);  // crucial: empty subarray has sum 0

    int sum = 0, count = 0;
    for (int num : arr) {
        sum += num;
        count += prefixCount.getOrDefault(sum - k, 0);
        prefixCount.put(sum, prefixCount.getOrDefault(sum, 0) + 1);
    }
    return count;
}
```

**Why `prefixCount.put(0, 1)`?**
```
If sum == k at some index i, then sum-k = 0.
We need prefixCount to contain 0 so we count this subarray.
Without it, we'd miss subarrays starting from index 0.
```

---

## Pattern 9: Cache Concepts (LRU Cache)

```java
// LRU Cache using LinkedHashMap
class LRUCache {
    private final int capacity;
    private final LinkedHashMap<Integer, Integer> cache;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        // accessOrder=true maintains access order (LRU at head)
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;  // auto-remove LRU when over capacity
            }
        };
    }

    public int get(int key) {
        return cache.getOrDefault(key, -1);
    }

    public void put(int key, int value) {
        cache.put(key, value);
    }
}
```

---

## Pattern 10: Set-based Fast Lookup

```java
// Check if array has two elements with absolute difference <= k
public boolean containsNearbyAlmostDuplicate(int[] arr, int k, int t) {
    TreeSet<Long> window = new TreeSet<>();  // sorted set

    for (int i = 0; i < arr.length; i++) {
        long num = (long) arr[i];

        // Find smallest number >= num - t
        Long floor = window.floor(num + t);
        if (floor != null && floor >= num - t) return true;

        window.add(num);
        if (window.size() > k) window.remove((long) arr[i - k]);
    }
    return false;
}
```

---

## 10 Real HashMap/HashSet Interview Problems

---

### Problem 1 — Two Sum
Already covered.

---

### Problem 2 — Longest Consecutive Sequence
Already covered in Pattern 7.

---

### Problem 3 — Top K Frequent Elements
> "Return k most frequent elements."

```java
public int[] topKFrequent(int[] arr, int k) {
    // Step 1: Count frequencies
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : arr) freq.put(num, freq.getOrDefault(num, 0) + 1);

    // Step 2: Min-heap of size k (keeps top k)
    PriorityQueue<Integer> heap = new PriorityQueue<>(
        (a, b) -> freq.get(a) - freq.get(b)  // min-heap by frequency
    );

    for (int num : freq.keySet()) {
        heap.offer(num);
        if (heap.size() > k) heap.poll();  // remove least frequent
    }

    // Step 3: Build result
    int[] result = new int[k];
    for (int i = k-1; i >= 0; i--) result[i] = heap.poll();
    return result;
}
```

---

### Problem 4 — Subarray Sum Equals K
Already covered in Pattern 8.

---

### Problem 5 — Intersection of Two Arrays

```java
public int[] intersection(int[] arr1, int[] arr2) {
    Set<Integer> set1 = new HashSet<>();
    for (int num : arr1) set1.add(num);

    Set<Integer> result = new HashSet<>();
    for (int num : arr2) {
        if (set1.contains(num)) result.add(num);
    }

    return result.stream().mapToInt(i -> i).toArray();
}
```

---

### Problem 6 — Word Frequency Count
> "Count frequency of each word in a sentence."

```java
public Map<String, Integer> wordFrequency(String sentence) {
    Map<String, Integer> freq = new HashMap<>();
    for (String word : sentence.toLowerCase().split("\\s+")) {
        freq.put(word, freq.getOrDefault(word, 0) + 1);
    }
    return freq;
}
```

---

### Problem 7 — Check if Array is Subset
> "Is arr2 a subset of arr1?"

```java
public boolean isSubset(int[] arr1, int[] arr2) {
    Set<Integer> set = new HashSet<>();
    for (int num : arr1) set.add(num);
    for (int num : arr2) {
        if (!set.contains(num)) return false;
    }
    return true;
}
```

---

### Problem 8 — Group Anagrams
Already covered in Pattern 5.

---

### Problem 9 — Ransom Note
> "Can you build the ransom note using letters from the magazine?"

```java
public boolean canConstruct(String note, String magazine) {
    int[] freq = new int[26];
    for (char c : magazine.toCharArray()) freq[c - 'a']++;
    for (char c : note.toCharArray()) {
        if (--freq[c - 'a'] < 0) return false;
    }
    return true;
}
```

---

### Problem 10 — First Missing Positive
> "Find smallest missing positive integer. O(n) time, O(1) space."

```java
// Use the array itself as a HashSet!
public int firstMissingPositive(int[] arr) {
    int n = arr.length;

    // Step 1: Place each number in its correct position
    // arr[i] should be at index arr[i]-1
    for (int i = 0; i < n; i++) {
        while (arr[i] > 0 && arr[i] <= n && arr[arr[i]-1] != arr[i]) {
            int temp = arr[arr[i]-1];
            arr[arr[i]-1] = arr[i];
            arr[i] = temp;
        }
    }

    // Step 2: Find first position where arr[i] != i+1
    for (int i = 0; i < n; i++) {
        if (arr[i] != i+1) return i+1;
    }
    return n+1;
}
```

---

# SECTION 4: PROBLEM-SOLVING FRAMEWORK 🗺️ (GOLD)

---

## The 60-Second Pattern Recognition Checklist

When you read a problem, scan for these keywords:

```
┌─────────────────────────────────┬──────────────────────────────────────────┐
│ KEYWORD(S) IN PROBLEM           │ PATTERN TO USE                           │
├─────────────────────────────────┼──────────────────────────────────────────┤
│ contiguous / subarray           │ Sliding Window or Prefix Sum or Kadane   │
│ maximum subarray sum            │ Kadane's Algorithm                       │
│ pair sum / two numbers sum      │ HashMap (unsorted) or Two Pointers       │
│ frequency / count occurrences   │ HashMap or int[] frequency array         │
│ anagram / same characters       │ Frequency Count (sort or int[26])        │
│ palindrome                      │ Two Pointers or Expand from Center       │
│ duplicate / seen before         │ HashSet                                  │
│ repeated characters             │ HashMap or HashSet                       │
│ sorted array + find target      │ Binary Search                            │
│ sorted array + pair             │ Two Pointers                             │
│ sliding window / k elements     │ Fixed Sliding Window                     │
│ longest / shortest subarray     │ Variable Sliding Window                  │
│ range sum / cumulative          │ Prefix Sum                               │
│ multiple queries on subarray    │ Prefix Sum (precompute)                  │
│ intervals / meetings / overlap  │ Sort + Greedy (Merge Intervals)          │
│ remove duplicates in-place      │ Two Pointers (read/write pointers)       │
│ rotate / shift array            │ Reversal trick                           │
│ in-place / O(1) space           │ In-place modification patterns           │
│ group by property               │ HashMap<Property, List>                  │
│ consecutive sequence            │ HashSet + smart iteration                │
│ top K / K most frequent         │ HashMap + PriorityQueue (heap)           │
│ shortest path / min steps       │ BFS                                      │
│ connected regions / islands     │ DFS / BFS on matrix                      │
└─────────────────────────────────┴──────────────────────────────────────────┘
```

---

## The 5-Step Interview Approach

```
STEP 1: UNDERSTAND (2 min)
  → Repeat the problem in your own words
  → Ask: "Can array have negatives? Duplicates? Is it sorted?"
  → Ask: "What are the constraints (n size)?"
  → Clarify return type and edge cases

STEP 2: EXAMPLES (1 min)
  → Walk through 1-2 examples yourself
  → Include an edge case (empty, single element, all same)

STEP 3: BRUTE FORCE (1 min)
  → State the naive O(n²) or O(n³) solution
  → Say it out loud: "Naive approach would be..."
  → This shows you can think, even if not optimal

STEP 4: OPTIMIZE (3 min)
  → Ask: "What makes this slow?"
  → Apply the pattern checklist
  → Think: "Can I precompute? Can I avoid recomputation?"

STEP 5: CODE + TEST (10 min)
  → Write clean code
  → Handle null/empty input
  → Dry run with your example
  → State time and space complexity
```

---

## Complexity Quick Reference

```
O(1)      → HashMap get/put, array index access
O(log n)  → Binary Search, balanced BST operations
O(n)      → Single pass, linear scan, HashMap build
O(n log n)→ Sorting, heap operations × n
O(n²)     → Nested loops (usually brute force)
O(2^n)    → All subsets (backtracking)
O(n!)     → All permutations

Interview targets:
  If n ≤ 10⁸  → O(n) or better required
  If n ≤ 10⁶  → O(n log n) acceptable
  If n ≤ 10⁴  → O(n²) might be OK
  If n ≤ 100  → O(n³) might be OK
```

---

# SECTION 5: INTERVIEW MODE 🎙️

---

## How to Handle ANY Interview Problem

The interviewer is NOT just checking if you get the right answer.
They are checking:
1. Do you communicate your thinking?
2. Can you handle feedback?
3. Do you think about edge cases?
4. Can you analyze complexity?

---

## Interview Problem 1 (Live)

> 🎤 **Interviewer:** "Given an integer array, find the length of the longest subarray with sum equal to k."

**What to say first:**
```
"Let me make sure I understand. We need a contiguous subarray
whose elements add up to exactly k, and we want the longest one.
Can the array have negative numbers? Can k be negative?"
```

*(Assuming: yes to both)*

**Then:**
```
"Okay. My brute force would be checking all O(n²) subarrays,
computing sum in O(n) each — giving O(n³) total.

I can optimize with prefix sums. If prefix[j] - prefix[i] = k,
then subarray from i to j has sum k. So I store each prefix
sum's earliest index in a HashMap. O(n) time and space."
```

```java
public int longestSubarrayWithSumK(int[] arr, int k) {
    Map<Integer, Integer> firstIndex = new HashMap<>();
    firstIndex.put(0, -1);  // sum=0 at index -1 (before array)

    int sum = 0, maxLen = 0;
    for (int i = 0; i < arr.length; i++) {
        sum += arr[i];

        if (firstIndex.containsKey(sum - k)) {
            maxLen = Math.max(maxLen, i - firstIndex.get(sum - k));
        }

        // Store FIRST occurrence only (we want longest subarray)
        if (!firstIndex.containsKey(sum)) {
            firstIndex.put(sum, i);
        }
    }
    return maxLen;
}
```

**Follow-up the interviewer might ask:**
> "What if all elements are positive?"
→ Then use sliding window (no HashMap needed, simpler O(n) O(1) space)

---

## Interview Problem 2 (Live)

> 🎤 **Interviewer:** "Given a string, find the length of the longest substring where no character repeats."

*(See sliding window Pattern 3 in strings section)*

**Mistake most candidates make:**
```java
// ❌ Using set.clear() or rebuilding from scratch when duplicate found
// → O(n²) instead of O(n)

// ✅ Use lastSeen map → jump left pointer directly to right position
left = Math.max(left, lastSeen.get(c) + 1);
// Note: Math.max is crucial! Don't move left BACKWARD
```

---

## Interview Problem 3 (Live)

> 🎤 **Interviewer:** "Given sorted array rotated at some pivot, find the minimum element."

```
"Since it's sorted then rotated, I can still use binary search.

The key insight: if arr[mid] > arr[right], the minimum is in the right half.
Otherwise it's in the left half (including mid)."
```

```java
public int findMin(int[] arr) {
    int left = 0, right = arr.length - 1;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] > arr[right]) left = mid + 1;
        else right = mid;
    }
    return arr[left];
}
```

---

# SECTION 6: DEBUGGING SKILLS 🐛

---

## Bug 1: Off-By-One Errors

```java
// ❌ Common: wrong loop boundary
for (int i = 0; i <= arr.length; i++)   // crashes at i=arr.length
for (int i = 0; i < arr.length; i++)    // ✅

// ❌ Wrong window size
int len = right - left;      // misses one element
int len = right - left + 1;  // ✅

// ❌ Wrong substring end index
s.substring(i, j)   // end is EXCLUSIVE in Java
// s.substring(0,3) on "hello" → "hel" (indices 0,1,2)
```

## Bug 2: Null Checks

```java
// ❌ NPE on empty input
arr.length  // NPE if arr is null

// ✅ Always guard
if (arr == null || arr.length == 0) return;
if (s == null || s.isEmpty()) return "";
if (map == null) return new HashMap<>();
```

## Bug 3: Integer Overflow

```java
// ❌ Overflow when computing mid
int mid = (left + right) / 2;    // overflows if left+right > MAX_INT

// ✅ Safe
int mid = left + (right - left) / 2;

// ❌ Overflow in prefix sum
int sum = 0;
for (int num : arr) sum += num;  // overflow if sum > MAX_INT

// ✅ Use long
long sum = 0;
```

## Bug 4: Duplicate Handling in Two Pointers

```java
// 3Sum: after finding triplet, skip duplicates
while (left < right && arr[left] == arr[left+1]) left++;
while (left < right && arr[right] == arr[right-1]) right--;
left++;    // ← DON'T forget to still move after skipping
right--;
```

## Bug 5: Forgetting Window Shrink

```java
// ❌ Only expanding, never shrinking → window grows forever
for (int right = 0; right < n; right++) {
    sum += arr[right];
    // forgot: while (sum > target) { sum -= arr[left++]; }
    maxLen = Math.max(maxLen, right - left + 1);  // wrong!
}

// ✅ Always shrink when condition violated
while (sum > target && left <= right) {
    sum -= arr[left++];
}
```

## Bug 6: Wrong HashMap Update Order

```java
// Two Sum: wrong order causes using same index twice
seen.put(arr[i], i);           // ❌ put BEFORE checking complement
if (seen.containsKey(target - arr[i])) ...  // might match itself!

// ✅ Check complement FIRST, then put
if (seen.containsKey(target - arr[i])) return ...;
seen.put(arr[i], i);  // put AFTER
```

## Bug 7: String Immutability

```java
// ❌ Trying to modify string directly
String s = "hello";
s[0] = 'H';  // compile error! String is immutable

// ✅ Convert to char array
char[] chars = s.toCharArray();
chars[0] = 'H';
String result = new String(chars);

// ✅ Or use StringBuilder
StringBuilder sb = new StringBuilder(s);
sb.setCharAt(0, 'H');
String result = sb.toString();
```

## Bug 8: HashMap.get() returning null

```java
// ❌ Unboxing null causes NullPointerException
int count = map.get(key);  // NPE if key not in map!

// ✅ Always use getOrDefault
int count = map.getOrDefault(key, 0);

// ✅ Or check containsKey first
if (map.containsKey(key)) {
    int count = map.get(key);
}
```

---

# SECTION 7: 7-DAY REVISION PLAN 📅

---

## Day 1 — Arrays Foundation (3 hours)

```
Morning (1.5 hr):
  □ Implement traversal, reversal from memory
  □ Implement Two Pointers: Two Sum (sorted), container with water
  □ LeetCode 1: Two Sum
  □ LeetCode 167: Two Sum II

Afternoon (1.5 hr):
  □ Sliding Window: max sum k elements, longest valid subarray
  □ LeetCode 209: Minimum Size Subarray Sum
  □ LeetCode 3: Longest Substring Without Repeating
```

## Day 2 — Arrays Advanced (3 hours)

```
Morning:
  □ Prefix Sum: subarray sum = k, range queries
  □ Kadane's Algorithm: max subarray, circular variant
  □ LeetCode 53: Maximum Subarray
  □ LeetCode 560: Subarray Sum Equals K

Afternoon:
  □ In-place: move zeros, remove duplicates
  □ Merge Intervals
  □ LeetCode 238: Product of Array Except Self
  □ LeetCode 56: Merge Intervals
```

## Day 3 — Strings (3 hours)

```
Morning:
  □ Character frequency count: anagram check, group anagrams
  □ Palindrome: check, longest palindromic substring
  □ LeetCode 242: Valid Anagram
  □ LeetCode 5: Longest Palindromic Substring

Afternoon:
  □ Sliding window on strings: min window, all anagrams
  □ StringBuilder optimization
  □ LeetCode 76: Minimum Window Substring
  □ LeetCode 438: Find All Anagrams in a String
```

## Day 4 — HashMap / HashSet (3 hours)

```
Morning:
  □ Frequency map patterns
  □ Two Sum, pair sum variants
  □ Duplicate detection with HashSet
  □ LeetCode 1: Two Sum
  □ LeetCode 217: Contains Duplicate

Afternoon:
  □ Top K frequent, group by property
  □ Longest consecutive sequence
  □ LeetCode 347: Top K Frequent Elements
  □ LeetCode 128: Longest Consecutive Sequence
```

## Day 5 — Mixed Practice (3 hours)

```
Morning: Timed practice (20 min each problem)
  □ LeetCode 15: 3Sum (Two Pointers + Sorting)
  □ LeetCode 42: Trapping Rain Water
  □ LeetCode 11: Container With Most Water

Afternoon:
  □ LeetCode 49: Group Anagrams
  □ LeetCode 424: Longest Repeating Character Replacement
  □ LeetCode 239: Sliding Window Maximum
```

## Day 6 — Hard Problems + Patterns (3 hours)

```
Morning:
  □ LeetCode 41: First Missing Positive
  □ LeetCode 84: Largest Rectangle in Histogram
  □ LeetCode 239: Sliding Window Maximum

Afternoon: Mock interview style
  □ Pick 3 random medium problems
  □ Solve in 20 min each without looking at solutions
  □ Review: what pattern did you miss?
```

## Day 7 — Final Revision + Interview Strategy (2 hours)

```
Morning:
  □ Review your personal list of tricky edge cases
  □ Review all 10 patterns cheat sheet
  □ Do 5 easy problems under time pressure (10 min each)

Afternoon:
  □ Practice explaining code out loud
  □ Review time complexities for all patterns
  □ Practice your "I'm thinking..." communication
```

---

## Most Repeated Patterns (Ranked by Interview Frequency)

```
Rank 1: HashMap (Two Sum, frequency, grouping)
Rank 2: Sliding Window (longest/shortest subarray/substring)
Rank 3: Two Pointers (pair sum, palindrome, remove duplicates)
Rank 4: Prefix Sum (subarray sum queries)
Rank 5: Kadane's (maximum subarray)
Rank 6: Binary Search (sorted array, rotated, answer search)
Rank 7: In-place modification (space optimization)
Rank 8: Merge Intervals (sorting + greedy)
Rank 9: String + HashMap (anagram, window)
Rank 10: Stack (next greater, valid parentheses)
```

---

## What to Revise 1 Hour Before Interview

```
□ HashMap: getOrDefault, containsKey, entrySet iteration
□ Two pointer template: while(left < right)
□ Sliding window template: for(right) + while(invalid) + left++
□ Prefix sum formula: prefix[i+1] = prefix[i] + arr[i]
□ Kadane's: currentSum = max(arr[i], currentSum + arr[i])
□ Safe mid: left + (right - left) / 2
□ Guard blocks: null check, empty check
□ StringBuilder for string building in loops
□ int[26] for lowercase char frequency
□ Arrays.sort() is O(n log n) in Java
```

---

## How to Communicate During a Coding Round

### Opening
```
"Let me first make sure I understand the problem correctly...
[restate in your words]

Can I assume the array is non-empty? Can it contain negatives?
What should I return if no answer exists?"
```

### While Thinking
```
"My first instinct is a brute force O(n²) approach using nested loops...
But I think we can do better. Let me think...

I notice this involves finding elements that satisfy a sum condition —
this reminds me of the Two Sum pattern where I can use a HashMap
to get O(1) lookup and reduce to O(n)."
```

### While Coding
```
"I'm starting with the guard block for null/empty input...
Now I'll initialize my HashMap...
For each element, I'll check if the complement exists...
Then add the current element to the map..."
```

### After Coding
```
"Let me trace through this example: [2,7,11,15], target=9
 At i=0: complement=7, not in map → add {2:0}
 At i=1: complement=2, found at index 0 → return [0,1]
 Looks correct.

Time complexity: O(n) — single pass through the array.
Space complexity: O(n) — HashMap stores at most n elements.

Edge cases I handled: null input, empty array...
One thing I didn't handle: what if there are multiple valid pairs?
The problem says return one, so this is fine."
```

---

## The Golden Rules of Interviews

```
Rule 1: ALWAYS state the brute force before the optimal.
Rule 2: ALWAYS clarify before coding.
Rule 3: ALWAYS handle edge cases (null, empty, single element).
Rule 4: ALWAYS state time and space complexity when done.
Rule 5: NEVER silently struggle. Say "I'm thinking about..."
Rule 6: NEVER write code without a plan.
Rule 7: ALWAYS test with an example after writing code.
Rule 8: If stuck → think: "What extra information would help?"
         → That's often what data structure to use.
```

---

*Built for cracking Arrays, Strings, and HashMap problems in Java — from zero to interview-ready.*
