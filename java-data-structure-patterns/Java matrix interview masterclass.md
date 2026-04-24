# 🧩 Java Matrix (2D Array) — Complete Interview Masterclass

> **Your goal:** Recognize the pattern → Apply the template → Ace the interview.
> This guide trains your *thinking process*, not just your memory.

---

## TABLE OF CONTENTS

1. [Matrix Fundamentals](#1-matrix-fundamentals)
2. [Core Matrix Patterns](#2-core-matrix-patterns)
   - [Pattern 1 — Traversal](#pattern-1-traversal)
   - [Pattern 2 — Boundary Traversal](#pattern-2-boundary-traversal)
   - [Pattern 3 — Spiral Traversal](#pattern-3-spiral-traversal)
   - [Pattern 4 — Diagonal Traversal](#pattern-4-diagonal-traversal)
   - [Pattern 5 — Matrix Rotation (90°, 180°)](#pattern-5-matrix-rotation)
   - [Pattern 6 — Transpose](#pattern-6-transpose)
   - [Pattern 7 — Searching in Matrix](#pattern-7-searching-in-matrix)
   - [Pattern 8 — 2D Prefix Sum](#pattern-8-2d-prefix-sum)
   - [Pattern 9 — Flood Fill / DFS / BFS](#pattern-9-flood-fill--dfs--bfs)
   - [Pattern 10 — Island Problems](#pattern-10-island-problems)
3. [Problem-Solving Framework](#3-problem-solving-framework)
4. [Real Interview Problems](#4-real-interview-problems)
5. [Debugging & Edge Cases](#5-debugging--edge-cases)
6. [Interview Traps](#6-interview-traps)
7. [Practice Strategy](#7-practice-strategy)

---

# 1. Matrix Fundamentals

## What is a Matrix in Programming?

A **matrix** is simply a 2D grid of values — rows and columns, like a spreadsheet or chessboard.

```
Matrix (3×4):
      col0  col1  col2  col3
row0 [  1,    2,    3,    4  ]
row1 [  5,    6,    7,    8  ]
row2 [  9,   10,   11,   12  ]
```

- **Rows** run horizontally (left → right)
- **Columns** run vertically (top → bottom)
- Element at row `i`, column `j` → `matrix[i][j]`

---

## How 2D Arrays Work in Java

### Declaration & Initialization

```java
// Method 1: Declare with size
int[][] matrix = new int[3][4]; // 3 rows, 4 columns

// Method 2: Declare with values
int[][] matrix = {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {9, 10, 11, 12}
};

// Key properties
int rows = matrix.length;        // number of rows = 3
int cols = matrix[0].length;     // number of columns = 4
```

### Memory Layout (Very Important!)

Java stores 2D arrays as an **array of arrays** (NOT as a flat block like C).

```
matrix ──► [ ref0 ] ──► [1, 2, 3, 4]
           [ ref1 ] ──► [5, 6, 7, 8]
           [ ref2 ] ──► [9, 10, 11, 12]
```

This means:
- Each row is an independent array on the heap
- Rows CAN have different lengths (jagged arrays) — a common interview trap!
- Always use `matrix[0].length` for columns, not a hardcoded number

### Row-Major vs Column-Major Traversal

```
Row-Major (visit row by row):      Column-Major (visit column by column):
→ 1  → 2  → 3  → 4               ↓ 1    ↓ 5    ↓ 9
→ 5  → 6  → 7  → 8               ↓ 2    ↓ 6    ↓ 10
→ 9  → 10 → 11 → 12              ↓ 3    ↓ 7    ↓ 11
                                  ↓ 4    ↓ 8    ↓ 12
```

**CPU Cache Tip:** Row-major is faster in Java because array elements in the same row are stored contiguously in memory (better cache locality).

---

## Common Beginner Mistakes ❌

| Mistake | Why it's wrong | Fix |
|---|---|---|
| `matrix.length` for columns | Returns row count, not col count | Use `matrix[0].length` |
| Not checking `matrix == null` | NullPointerException | Always null-check first |
| Not checking `matrix.length == 0` | ArrayIndexOutOfBoundsException on `matrix[0]` | Check empty before accessing |
| Hardcoding `n` as both rows and cols | Breaks on non-square matrices | Use separate `rows` and `cols` |
| Modifying array while iterating | Corrupts data | Work on a copy or use markers |
| Using `i` for columns, `j` for rows | Confuses logic | Convention: `i`=row, `j`=col |

---

# 2. Core Matrix Patterns

---

## Pattern 1: Traversal

### 🧠 Intuition

Traversal means visiting every element in a specific order. This is the foundation of ALL matrix problems. Once you understand the loop structure, everything else builds on it.

**Think of it as:** "How do I walk through this grid?"

---

### 1a. Row-Wise Traversal (Most Common)

```
Visit order: (0,0)→(0,1)→(0,2)→(1,0)→(1,1)→(1,2)→(2,0)→(2,1)→(2,2)
```

#### 🔁 Step-by-Step Approach
1. Outer loop → iterate over rows (`i` from 0 to rows-1)
2. Inner loop → iterate over columns (`j` from 0 to cols-1)
3. Access element as `matrix[i][j]`

#### 💻 Java Code

```java
public void rowWiseTraversal(int[][] matrix) {
    int rows = matrix.length;
    int cols = matrix[0].length;

    for (int i = 0; i < rows; i++) {          // row index
        for (int j = 0; j < cols; j++) {      // column index
            System.out.print(matrix[i][j] + " ");
        }
        System.out.println(); // new line after each row
    }
}
```

#### 🧪 Dry Run
```
Input:
[1, 2, 3]
[4, 5, 6]
[7, 8, 9]

i=0: j=0→1, j=1→2, j=2→3  → prints "1 2 3"
i=1: j=0→4, j=1→5, j=2→6  → prints "4 5 6"
i=2: j=0→7, j=1→8, j=2→9  → prints "7 8 9"
```

---

### 1b. Column-Wise Traversal

```
Visit order: (0,0)→(1,0)→(2,0)→(0,1)→(1,1)→(2,1)→...
```

```java
public void columnWiseTraversal(int[][] matrix) {
    int rows = matrix.length;
    int cols = matrix[0].length;

    // ✅ Swap the loops: outer = column, inner = row
    for (int j = 0; j < cols; j++) {
        for (int i = 0; i < rows; i++) {
            System.out.print(matrix[i][j] + " ");
        }
        System.out.println();
    }
}
```

---

### 1c. Reverse Traversal (Right-to-Left, Bottom-to-Top)

```java
// Bottom-to-Top, Right-to-Left
public void reverseTraversal(int[][] matrix) {
    int rows = matrix.length;
    int cols = matrix[0].length;

    for (int i = rows - 1; i >= 0; i--) {      // start from last row
        for (int j = cols - 1; j >= 0; j--) {  // start from last col
            System.out.print(matrix[i][j] + " ");
        }
        System.out.println();
    }
}
```

#### ⚠️ Edge Cases
- **Empty matrix:** `matrix.length == 0` → skip entirely
- **Single element:** `matrix = {{5}}` → should print "5" once
- **Single row:** `matrix = {{1,2,3}}` → treated as 1×3 grid

#### ❌ Common Mistakes
```java
// WRONG: Using matrix.length for both dimensions (fails on non-square)
for (int i = 0; i < matrix.length; i++)
    for (int j = 0; j < matrix.length; j++)  // ❌ should be matrix[0].length

// WRONG: Forgetting to handle null
// Always add: if (matrix == null || matrix.length == 0) return;
```

---

## Pattern 2: Boundary Traversal

### 🧠 Intuition

Print only the **outermost ring** of the matrix — the top row, right column, bottom row, and left column.

**Think of it as:** "Walking around the perimeter of a rectangular frame."

```
For a 4×4 matrix, boundary elements are marked with *:
* * * *
*       *
*       *
* * * *
```

### 🔁 Step-by-Step Approach

Handle 4 sides separately:
1. **Top row:** left → right along `row = 0`
2. **Right column:** top+1 → bottom along `col = cols-1`
3. **Bottom row:** right-1 → left along `row = rows-1` (only if rows > 1)
4. **Left column:** bottom-1 → top+1 along `col = 0` (only if cols > 1)

### ⚠️ Edge Cases (Tricky!)

| Situation | Problem | Fix |
|---|---|---|
| Single row | Bottom row would repeat top row | Skip bottom row traversal |
| Single column | Left col would repeat right col | Skip left col traversal |
| 1×1 matrix | All 4 sides are the same element | Print once only |

### 💻 Java Code

```java
public void boundaryTraversal(int[][] matrix) {
    if (matrix == null || matrix.length == 0) return;

    int rows = matrix.length;
    int cols = matrix[0].length;

    // Special case: single element
    if (rows == 1 && cols == 1) {
        System.out.print(matrix[0][0]);
        return;
    }

    // Top row: (0,0) to (0, cols-1)
    for (int j = 0; j < cols; j++)
        System.out.print(matrix[0][j] + " ");

    // Right column: (1, cols-1) to (rows-1, cols-1)
    for (int i = 1; i < rows; i++)
        System.out.print(matrix[i][cols - 1] + " ");

    // Bottom row (only if more than 1 row): (rows-1, cols-2) to (rows-1, 0)
    if (rows > 1)
        for (int j = cols - 2; j >= 0; j--)
            System.out.print(matrix[rows - 1][j] + " ");

    // Left column (only if more than 1 col): (rows-2, 0) to (1, 0)
    if (cols > 1)
        for (int i = rows - 2; i >= 1; i--)
            System.out.print(matrix[i][0] + " ");
}
```

### 🧪 Dry Run

```
Input:
[1,  2,  3,  4]
[5,  6,  7,  8]
[9, 10, 11, 12]

Top row    → 1 2 3 4
Right col  → 8 12
Bottom row → 11 10 9
Left col   → 5

Output: 1 2 3 4 8 12 11 10 9 5
```

---

## Pattern 3: Spiral Traversal

### 🧠 Intuition

Print elements in a spiral — like peeling layers of an onion, one ring at a time, going clockwise.

**Think of it as:** Boundary traversal, but repeated for each inner layer.

```
Layer 0 (outer):  1  2  3  4  8 12 16 15 14 13  9  5
Layer 1 (inner):  6  7 11 10
```

### 🔁 Step-by-Step Approach

Use 4 pointers: `top`, `bottom`, `left`, `right`
1. Traverse top row (left → right), then increment `top`
2. Traverse right column (top → bottom), then decrement `right`
3. Traverse bottom row (right → left) if `top <= bottom`, then decrement `bottom`
4. Traverse left column (bottom → top) if `left <= right`, then increment `left`
5. Repeat while `top <= bottom` AND `left <= right`

### 💻 Java Code

```java
public List<Integer> spiralOrder(int[][] matrix) {
    List<Integer> result = new ArrayList<>();
    if (matrix == null || matrix.length == 0) return result;

    int top = 0, bottom = matrix.length - 1;
    int left = 0, right = matrix[0].length - 1;

    while (top <= bottom && left <= right) {

        // 1. Traverse top row →
        for (int j = left; j <= right; j++)
            result.add(matrix[top][j]);
        top++;

        // 2. Traverse right column ↓
        for (int i = top; i <= bottom; i++)
            result.add(matrix[i][right]);
        right--;

        // 3. Traverse bottom row ← (guard: avoid re-visiting in single-row case)
        if (top <= bottom) {
            for (int j = right; j >= left; j--)
                result.add(matrix[bottom][j]);
            bottom--;
        }

        // 4. Traverse left column ↑ (guard: avoid re-visiting in single-col case)
        if (left <= right) {
            for (int i = bottom; i >= top; i--)
                result.add(matrix[i][left]);
            left++;
        }
    }

    return result;
}
```

### 🧪 Dry Run

```
Input:
[1,  2,  3,  4]
[5,  6,  7,  8]
[9, 10, 11, 12]

Initial: top=0, bottom=2, left=0, right=3

Round 1:
  Top row    → 1 2 3 4    (top → 1)
  Right col  → 8 12       (right → 2)
  Bottom row → 11 10 9    (bottom → 1)
  Left col   → 5          (left → 1)

Round 2: top=1, bottom=1, left=1, right=2
  Top row    → 6 7        (top → 2)
  Right col  → (top>bottom, skip)
  Bottom row → (top>bottom, skip)
  Left col   → (left>right, skip)

Output: [1,2,3,4,8,12,11,10,9,5,6,7]
```

### ⚠️ Edge Cases

| Input | Expected |
|---|---|
| `[[1]]` | `[1]` |
| `[[1,2,3]]` (single row) | `[1,2,3]` |
| `[[1],[2],[3]]` (single col) | `[1,2,3]` |
| `[[1,2],[3,4]]` | `[1,2,4,3]` |

### ❌ Common Mistakes

```java
// WRONG: Forgetting the guards for bottom and left traversals
// This causes elements to be printed twice in single-row/column cases
if (top <= bottom)  // ← REQUIRED before bottom row traversal
if (left <= right)  // ← REQUIRED before left column traversal
```

---

## Pattern 4: Diagonal Traversal

### 🧠 Intuition

Two main diagonals exist in a matrix:
- **Primary (Main) Diagonal:** top-left to bottom-right → `matrix[i][i]`
- **Secondary (Anti) Diagonal:** top-right to bottom-left → `matrix[i][n-1-i]`

For ALL diagonals, notice that elements on the same diagonal share a relationship between their indices.

```
Primary diagonal:       Secondary diagonal:
[★, ., ., .]           [., ., ., ★]
[., ★, ., .]           [., ., ★, .]
[., ., ★, .]           [., ★, ., .]
[., ., ., ★]           [★, ., ., .]
```

**Key Insight for General Diagonal Traversal:**
- For elements going top-right (↗): `i + j = constant`
- For elements going top-left (↖): `i - j = constant`

### 💻 Java Code — Print Both Main Diagonals

```java
public void printDiagonals(int[][] matrix) {
    int n = matrix.length; // assume square for this example

    System.out.print("Main diagonal: ");
    for (int i = 0; i < n; i++)
        System.out.print(matrix[i][i] + " ");  // row == col

    System.out.print("\nAnti diagonal: ");
    for (int i = 0; i < n; i++)
        System.out.print(matrix[i][n - 1 - i] + " ");  // row + col == n-1
}
```

### 💻 Java Code — Print ALL Diagonals (Top-Right Direction)

```java
// Elements on same diagonal satisfy: i + j = constant (d)
// d ranges from 0 (top-left corner) to (rows+cols-2) (bottom-right corner)
public void printAllDiagonals(int[][] matrix) {
    int rows = matrix.length;
    int cols = matrix[0].length;

    for (int d = 0; d < rows + cols - 1; d++) {
        System.out.print("Diagonal " + d + ": ");
        for (int i = 0; i < rows; i++) {
            int j = d - i;
            if (j >= 0 && j < cols) {
                System.out.print(matrix[i][j] + " ");
            }
        }
        System.out.println();
    }
}
```

### 🧪 Dry Run

```
Input:
[1, 2, 3]
[4, 5, 6]
[7, 8, 9]

d=0 (i+j=0): (0,0)=1
d=1 (i+j=1): (0,1)=2, (1,0)=4
d=2 (i+j=2): (0,2)=3, (1,1)=5, (2,0)=7
d=3 (i+j=3): (1,2)=6, (2,1)=8
d=4 (i+j=4): (2,2)=9
```

### ❌ Common Mistakes

```java
// WRONG: Only iterating square matrices
// ✅ FIX: Always use separate rows/cols variables

// WRONG: Missing bounds check in general diagonal traversal
if (j >= 0 && j < cols)  // ← REQUIRED, don't skip this
```

---

## Pattern 5: Matrix Rotation

### 🧠 Intuition

**90° Clockwise Rotation:**
- The first column becomes the last row
- The last row becomes the last column
- A pixel at `(i, j)` moves to `(j, n-1-i)`

**Mental model:** Tilt the matrix 90° to your right. What was on top is now on the right.

**The Key Trick (In-Place):**
1. **Transpose** the matrix (flip along main diagonal)
2. **Reverse each row**

```
Original → Transpose → Reverse each row = 90° CW rotation

[1, 2, 3]   [1, 4, 7]   [7, 4, 1]
[4, 5, 6] → [2, 5, 8] → [8, 5, 2]
[7, 8, 9]   [3, 6, 9]   [9, 6, 3]
```

### 💻 Java Code — 90° Clockwise (In-Place)

```java
public void rotate90CW(int[][] matrix) {
    int n = matrix.length;

    // Step 1: Transpose (swap matrix[i][j] with matrix[j][i])
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {   // j starts at i+1 to avoid double-swap
            int temp = matrix[i][j];
            matrix[i][j] = matrix[j][i];
            matrix[j][i] = temp;
        }
    }

    // Step 2: Reverse each row
    for (int i = 0; i < n; i++) {
        int left = 0, right = n - 1;
        while (left < right) {
            int temp = matrix[i][left];
            matrix[i][left] = matrix[i][right];
            matrix[i][right] = temp;
            left++;
            right--;
        }
    }
}
```

### 💻 Java Code — 90° Counter-Clockwise (In-Place)

```java
public void rotate90CCW(int[][] matrix) {
    int n = matrix.length;

    // Step 1: Transpose
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++) {
            int temp = matrix[i][j];
            matrix[i][j] = matrix[j][i];
            matrix[j][i] = temp;
        }

    // Step 2: Reverse each COLUMN (instead of each row)
    for (int j = 0; j < n; j++) {
        int top = 0, bottom = n - 1;
        while (top < bottom) {
            int temp = matrix[top][j];
            matrix[top][j] = matrix[bottom][j];
            matrix[bottom][j] = temp;
            top++;
            bottom--;
        }
    }
}
```

### 💻 Java Code — 180° Rotation

```java
public void rotate180(int[][] matrix) {
    // Simply apply 90° CW rotation twice
    rotate90CW(matrix);
    rotate90CW(matrix);

    // OR: Reverse entire matrix (reverse rows, then reverse each row)
}
```

### 🧪 Dry Run — 90° CW

```
Original:         After Transpose:   After Row Reverse:
[1, 2, 3]         [1, 4, 7]          [7, 4, 1]
[4, 5, 6]    →    [2, 5, 8]    →     [8, 5, 2]
[7, 8, 9]         [3, 6, 9]          [9, 6, 3]

Check: element 1 was at (0,0), now at (0,2) ✓ (should move to (0, n-1-0) = (0,2))
Check: element 7 was at (2,0), now at (0,0) ✓
```

### ⚠️ Edge Cases

- Works only for **square matrices** in-place. Non-square needs extra space.
- For non-square: create new `int[cols][rows]` and place `matrix[i][j]` → `result[j][rows-1-i]`

### ❌ Common Mistakes

```java
// WRONG: Starting inner loop at j=0 (swaps elements twice = no change!)
for (int j = 0; j < n; j++)       // ❌
for (int j = i + 1; j < n; j++)   // ✅ start at i+1
```

---

## Pattern 6: Transpose

### 🧠 Intuition

A **transpose** flips a matrix along its main diagonal.
- Rows become columns
- Columns become rows
- Element at `(i, j)` moves to `(j, i)`

```
Original (3×2):     Transposed (2×3):
[1, 2]              [1, 3, 5]
[3, 4]         →    [2, 4, 6]
[5, 6]
```

### 💻 Java Code — In-Place (Square Matrix)

```java
public void transposeInPlace(int[][] matrix) {
    int n = matrix.length;

    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {  // Only upper triangle
            int temp = matrix[i][j];
            matrix[i][j] = matrix[j][i];
            matrix[j][i] = temp;
        }
    }
}
```

### 💻 Java Code — New Matrix (Non-Square)

```java
public int[][] transpose(int[][] matrix) {
    int rows = matrix.length;
    int cols = matrix[0].length;
    int[][] result = new int[cols][rows]; // Note: dimensions swapped!

    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            result[j][i] = matrix[i][j];

    return result;
}
```

### ❌ Common Mistakes

- Forgetting to swap dimensions when creating result array for non-square
- Not starting `j` at `i+1` for in-place square transpose (causes double swap = no change)

---

## Pattern 7: Searching in Matrix

### 🧠 Intuition

There are three key scenarios:

1. **Unsorted Matrix:** Brute force O(n×m)
2. **Row-wise and Column-wise Sorted (but rows not connected):** Binary search on each row O(n log m)
3. **Fully Sorted Matrix (rows connected) or Staircase Sorted:** Smart search O(n + m) or Binary search O(log(n×m))

**The Interview Favorite — "Staircase Search" (Sorted rows + sorted cols):**

Start from **top-right corner**:
- If `target == matrix[i][j]` → found!
- If `target < matrix[i][j]` → move LEFT (eliminate column)
- If `target > matrix[i][j]` → move DOWN (eliminate row)

**Why this works:** Top-right is the max of its row (go left to decrease) and min of its column (go down to increase).

### 💻 Java Code — Staircase Search

```java
// Matrix: rows sorted left→right, columns sorted top→bottom
public boolean searchMatrix(int[][] matrix, int target) {
    if (matrix == null || matrix.length == 0) return false;

    int i = 0;                           // Start at top-right
    int j = matrix[0].length - 1;

    while (i < matrix.length && j >= 0) {
        if (matrix[i][j] == target) return true;
        else if (target < matrix[i][j]) j--;  // go left
        else i++;                              // go down
    }

    return false;
}
// Time: O(n + m), Space: O(1)
```

### 💻 Java Code — Binary Search (Fully Sorted, Rows Connected)

```java
// Matrix where row i's last element < row i+1's first element
// Can be treated as a flattened sorted array
public boolean searchSortedMatrix(int[][] matrix, int target) {
    int rows = matrix.length;
    int cols = matrix[0].length;
    int left = 0, right = rows * cols - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;
        // Convert 1D index to 2D: row = mid/cols, col = mid%cols
        int val = matrix[mid / cols][mid % cols];

        if (val == target) return true;
        else if (val < target) left = mid + 1;
        else right = mid - 1;
    }

    return false;
}
// Time: O(log(n×m)), Space: O(1)
```

### 🧪 Dry Run — Staircase Search

```
Matrix:
[ 1,  4,  7, 11]
[ 2,  5,  8, 12]
[ 3,  6,  9, 16]
[10, 13, 14, 17]

Target = 5

Start at top-right: (0,3) = 11 > 5 → move left
(0,2) = 7 > 5 → move left
(0,1) = 4 < 5 → move down
(1,1) = 5 == 5 → FOUND! ✓
```

---

## Pattern 8: 2D Prefix Sum

### 🧠 Intuition

**Problem:** Sum of elements in a rectangular sub-matrix efficiently.

**Brute Force:** O(r×c) per query → too slow for many queries.

**Better Idea:** Precompute prefix sums so each query takes O(1).

```
prefix[i][j] = sum of all elements in rectangle from (0,0) to (i,j)
```

**Building the prefix:**
```
prefix[i][j] = matrix[i][j]
             + prefix[i-1][j]    (sum above)
             + prefix[i][j-1]    (sum to left)
             - prefix[i-1][j-1]  (was double-counted)
```

**Query (sum from (r1,c1) to (r2,c2)):**
```
sum = prefix[r2][c2]
    - prefix[r1-1][c2]    (remove above)
    - prefix[r2][c1-1]    (remove left)
    + prefix[r1-1][c1-1]  (was double-removed, add back)
```

### 💻 Java Code

```java
class NumMatrix {
    private int[][] prefix;

    public NumMatrix(int[][] matrix) {
        int rows = matrix.length;
        int cols = matrix[0].length;
        // Use 1-indexed prefix to avoid boundary checks (row 0 / col 0 = 0)
        prefix = new int[rows + 1][cols + 1];

        for (int i = 1; i <= rows; i++) {
            for (int j = 1; j <= cols; j++) {
                prefix[i][j] = matrix[i-1][j-1]
                             + prefix[i-1][j]
                             + prefix[i][j-1]
                             - prefix[i-1][j-1];
            }
        }
    }

    // Query: sum from (r1,c1) to (r2,c2) — 0-indexed input
    public int sumRegion(int r1, int c1, int r2, int c2) {
        return prefix[r2+1][c2+1]
             - prefix[r1][c2+1]
             - prefix[r2+1][c1]
             + prefix[r1][c1];
    }
}
// Build: O(n×m), Query: O(1)
```

### 🧪 Dry Run

```
Matrix:
[3, 0, 1, 4]
[5, 6, 3, 2]
[1, 2, 0, 1]

Prefix (1-indexed):
[0,  0,  0,  0,  0]
[0,  3,  3,  4,  8]
[0,  8, 14, 18, 24]
[0,  9, 17, 21, 28]

Query: sum from (1,1) to (2,2) (0-indexed)
= prefix[3][3] - prefix[1][3] - prefix[3][1] + prefix[1][1]
= 21          - 4            - 9            + 3
= 11

Verify manually: 6+3+2+0 = 11 ✓
```

### ⚠️ Edge Cases

- 1-indexed prefix array avoids awkward `if (i > 0 && j > 0)` boundary checks
- Make sure query inputs are valid (0 ≤ r1 ≤ r2 < rows)

---

## Pattern 9: Flood Fill / DFS / BFS in Matrix

### 🧠 Intuition

Some problems require exploring **connected regions** in a grid — moving to adjacent cells (up/down/left/right, and sometimes diagonals).

**Think of it as:** A fire spreading from one cell to its neighbors, and then their neighbors, etc.

**Two approaches:**
- **DFS (Depth-First Search):** Go deep in one direction before backtracking. Uses recursion (or explicit stack).
- **BFS (Breadth-First Search):** Explore all neighbors before going deeper. Uses a queue. Best for **shortest path** problems.

**The 4-Direction Template (Most Important):**
```java
int[] dr = {-1, 1, 0, 0};  // up, down, left, right (row deltas)
int[] dc = {0, 0, -1, 1};  // up, down, left, right (col deltas)

// To visit neighbors of (r, c):
for (int d = 0; d < 4; d++) {
    int nr = r + dr[d];
    int nc = c + dc[d];
    if (nr >= 0 && nr < rows && nc >= 0 && nc < cols) {
        // valid cell (nr, nc)
    }
}
```

### 💻 Java Code — Flood Fill (DFS)

```java
// LeetCode 733: Flood Fill
// Change all connected cells with oldColor to newColor
public int[][] floodFill(int[][] image, int sr, int sc, int newColor) {
    int oldColor = image[sr][sc];
    if (oldColor == newColor) return image;  // avoid infinite loop!
    dfs(image, sr, sc, oldColor, newColor);
    return image;
}

private void dfs(int[][] image, int r, int c, int oldColor, int newColor) {
    // Base cases
    if (r < 0 || r >= image.length) return;
    if (c < 0 || c >= image[0].length) return;
    if (image[r][c] != oldColor) return;      // wrong color, stop

    image[r][c] = newColor;  // paint this cell

    // Explore 4 neighbors
    dfs(image, r - 1, c, oldColor, newColor); // up
    dfs(image, r + 1, c, oldColor, newColor); // down
    dfs(image, r, c - 1, oldColor, newColor); // left
    dfs(image, r, c + 1, oldColor, newColor); // right
}
```

### 💻 Java Code — BFS Template (Shortest Path)

```java
public int shortestPath(int[][] grid, int[] start, int[] end) {
    int rows = grid.length, cols = grid[0].length;
    boolean[][] visited = new boolean[rows][cols];
    Queue<int[]> queue = new LinkedList<>();

    queue.offer(new int[]{start[0], start[1]});
    visited[start[0]][start[1]] = true;
    int steps = 0;

    int[] dr = {-1, 1, 0, 0};
    int[] dc = {0, 0, -1, 1};

    while (!queue.isEmpty()) {
        int size = queue.size();  // process level by level
        for (int k = 0; k < size; k++) {
            int[] curr = queue.poll();
            if (curr[0] == end[0] && curr[1] == end[1]) return steps;

            for (int d = 0; d < 4; d++) {
                int nr = curr[0] + dr[d];
                int nc = curr[1] + dc[d];
                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                        && !visited[nr][nc] && grid[nr][nc] == 0) {
                    visited[nr][nc] = true;
                    queue.offer(new int[]{nr, nc});
                }
            }
        }
        steps++;
    }

    return -1; // no path found
}
```

### ⚠️ Edge Cases

| Issue | Symptom | Fix |
|---|---|---|
| oldColor == newColor | Infinite recursion | Check and return early |
| No visited array | Re-visiting cells, infinite loop | Always mark before enqueuing |
| Stack overflow on large grid | DFS too deep | Use iterative DFS or BFS instead |
| 8-directional vs 4-directional | Wrong connectivity | Confirm in problem statement |

---

## Pattern 10: Island Problems (Connected Components)

### 🧠 Intuition

**"Number of Islands"** is the most popular matrix interview problem. An island is a group of connected `1`s (land) surrounded by `0`s (water).

**Strategy:** Iterate through every cell. When you find an unvisited `1`, trigger DFS/BFS to "sink" (mark visited) the entire island. Count each trigger as one island.

**Key Insight:** After DFS/BFS, the entire island is marked — you'll never count it twice.

### 💻 Java Code — Number of Islands (DFS)

```java
public int numIslands(char[][] grid) {
    if (grid == null || grid.length == 0) return 0;

    int rows = grid.length, cols = grid[0].length;
    int count = 0;

    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (grid[i][j] == '1') {  // found unvisited land
                count++;
                sinkIsland(grid, i, j);  // mark entire island as visited
            }
        }
    }

    return count;
}

private void sinkIsland(char[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length) return;
    if (c < 0 || c >= grid[0].length) return;
    if (grid[r][c] != '1') return;  // water or already visited

    grid[r][c] = '0';  // sink this land cell (mark visited by changing value)

    sinkIsland(grid, r - 1, c);
    sinkIsland(grid, r + 1, c);
    sinkIsland(grid, r, c - 1);
    sinkIsland(grid, r, c + 1);
}
// Time: O(n×m), Space: O(n×m) stack space
```

### 💻 Java Code — Max Area of Island

```java
public int maxAreaOfIsland(int[][] grid) {
    int rows = grid.length, cols = grid[0].length;
    int maxArea = 0;

    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (grid[i][j] == 1) {
                maxArea = Math.max(maxArea, dfsArea(grid, i, j));
            }
        }
    }

    return maxArea;
}

private int dfsArea(int[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length) return 0;
    if (c < 0 || c >= grid[0].length) return 0;
    if (grid[r][c] != 1) return 0;

    grid[r][c] = 0; // mark visited

    return 1 + dfsArea(grid, r-1, c)
             + dfsArea(grid, r+1, c)
             + dfsArea(grid, r, c-1)
             + dfsArea(grid, r, c+1);
}
```

### 🧪 Dry Run — Number of Islands

```
Grid:
['1','1','0','0','0']
['1','1','0','0','0']
['0','0','1','0','0']
['0','0','0','1','1']

Scan: hit (0,0)='1' → count=1, DFS sinks all connected 1s:
  (0,0),(0,1),(1,0),(1,1) all become '0'

Scan continues to (2,2)='1' → count=2, DFS sinks just (2,2)

Scan continues to (3,3)='1' → count=3, DFS sinks (3,3),(3,4)

Answer: 3 islands ✓
```

### ⚠️ Edge Cases

- All water (`grid` full of `0`s) → return 0
- All land → return 1
- Grid with single cell → handle normally
- Diagonal connectivity: confirm if diagonals count (usually they don't)

---

# 3. Problem-Solving Framework

## 🥇 The GOLD Checklist — Identify the Pattern in 60 Seconds

When you see a matrix problem, mentally go through this checklist in order:

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: UNDERSTAND THE GRID                                     │
│  □ What does each cell represent? (0/1, char, int, color?)      │
│  □ Is it sorted? (by row? by row+col? fully sorted?)            │
│  □ Square or rectangular?                                       │
│  □ Any special cells? (obstacles, source, target?)              │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ STEP 2: WHAT DOES THE PROBLEM ASK?                              │
│                                                                 │
│  "Print/collect elements"        → TRAVERSAL                    │
│  "Print the border/perimeter"    → BOUNDARY TRAVERSAL           │
│  "Print in spiral/layers"        → SPIRAL TRAVERSAL             │
│  "Print diagonal / anti-diag"    → DIAGONAL TRAVERSAL           │
│  "Rotate / flip the matrix"      → ROTATION / TRANSPOSE         │
│  "Find a value"                  → SEARCHING                    │
│  "Sum of submatrix / rectangle"  → 2D PREFIX SUM                │
│  "Spread / fill / change color"  → FLOOD FILL (DFS/BFS)         │
│  "Count groups / regions"        → ISLAND / CONNECTED COMP.     │
│  "Shortest path from A to B"     → BFS (level-order)            │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ STEP 3: WHAT CONSTRAINTS MATTER?                                │
│  □ n × m size? (10^3 × 10^3 → O(n×m) is fine, O(n²×m²) is not)│
│  □ Multiple queries? → Precompute (Prefix Sum)                  │
│  □ Need in-place? → Can't create new array                      │
│  □ Need O(1) space? → Must reuse grid (mark visited in-place)   │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ STEP 4: CHOOSE & APPLY TEMPLATE                                 │
│  □ Write the standard template first                            │
│  □ Verify with a small example (2×2 or 3×3)                     │
│  □ Handle edge cases (empty, single row/col, 1×1)               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Problem Statement → Pattern Translation

| Keywords in Problem | Pattern to Use |
|---|---|
| "print all elements", "row by row" | Simple Traversal |
| "boundary", "border", "perimeter", "frame" | Boundary Traversal |
| "spiral", "layer by layer", "onion" | Spiral Traversal |
| "diagonal", "anti-diagonal", "top-right to bottom-left" | Diagonal Traversal |
| "rotate 90°", "rotate clockwise" | Rotation (Transpose + Reverse) |
| "flip", "mirror", "reflect" | Transpose |
| "find target in sorted matrix" | Binary Search / Staircase Search |
| "sum of submatrix", "range sum query" | 2D Prefix Sum |
| "spread", "paint", "fill", "color" | Flood Fill / DFS |
| "count regions", "number of islands", "connected" | Island / BFS / DFS |
| "shortest path", "minimum steps", "level" | BFS |
| "all paths", "can you reach" | DFS with backtracking |

---

## Optimization Hints

| Brute Force | Optimized | Technique |
|---|---|---|
| O(n×m) per query | O(1) per query | 2D Prefix Sum |
| O(n×m) search | O(n+m) search | Staircase Search |
| BFS with visited array | Mark in-place | Reduce Space |
| Multiple DFS calls | Union-Find | Advanced Island Problems |

---

# 4. Real Interview Problems

> **How to use this section:** Read the problem, stop, think for 5–10 minutes on your own, THEN read the solution. Simulate real interview conditions.

---

## Problem 1: Set Matrix Zeroes

> **Interviewer asks:**
> "Given an m×n matrix, if an element is 0, set its entire row and column to 0. Do it **in-place**. Can you do it with O(1) extra space?"

*(Think for a moment...)*

---

**Approach:**

**Naive:** Use a copy matrix → O(n×m) space — interviewer will immediately ask to optimize.

**Better:** Record which rows and cols have zeros using two sets → O(n+m) space.

**Best (Tricky!):** Use the **first row and first column as markers**:
1. Check if row 0 or col 0 originally had a zero (save this separately)
2. For every `matrix[i][j] == 0` (i>0, j>0), mark `matrix[i][0] = 0` and `matrix[0][j] = 0`
3. Use those markers to zero out rows and columns (iterate backwards to avoid corrupting markers)
4. Handle row 0 and col 0 separately

```java
public void setZeroes(int[][] matrix) {
    int rows = matrix.length, cols = matrix[0].length;
    boolean firstRowZero = false, firstColZero = false;

    // Check if first row has zero
    for (int j = 0; j < cols; j++)
        if (matrix[0][j] == 0) firstRowZero = true;

    // Check if first column has zero
    for (int i = 0; i < rows; i++)
        if (matrix[i][0] == 0) firstColZero = true;

    // Use first row/col as markers
    for (int i = 1; i < rows; i++)
        for (int j = 1; j < cols; j++)
            if (matrix[i][j] == 0) {
                matrix[i][0] = 0;  // mark row
                matrix[0][j] = 0;  // mark col
            }

    // Zero out marked rows and cols
    for (int i = 1; i < rows; i++)
        for (int j = 1; j < cols; j++)
            if (matrix[i][0] == 0 || matrix[0][j] == 0)
                matrix[i][j] = 0;

    // Zero first row if needed
    if (firstRowZero)
        for (int j = 0; j < cols; j++) matrix[0][j] = 0;

    // Zero first col if needed
    if (firstColZero)
        for (int i = 0; i < rows; i++) matrix[i][0] = 0;
}
// Time: O(n×m), Space: O(1)
```

**Follow-up questions:**
- Why process from (1,1) and not (0,0)?
- What if multiple zeros exist in row 0 originally?

---

## Problem 2: Rotate Image (90° Clockwise)

> **Interviewer asks:**
> "You are given an n×n 2D matrix representing an image. Rotate the image by 90° clockwise **in-place**."

*(Think for a moment...)*

---

**Approach:** Transpose + Reverse each row (see Pattern 5)

```java
public void rotate(int[][] matrix) {
    int n = matrix.length;
    // Step 1: Transpose
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++) {
            int tmp = matrix[i][j];
            matrix[i][j] = matrix[j][i];
            matrix[j][i] = tmp;
        }
    // Step 2: Reverse each row
    for (int i = 0; i < n; i++) {
        int l = 0, r = n - 1;
        while (l < r) {
            int tmp = matrix[i][l];
            matrix[i][l++] = matrix[i][r];
            matrix[i][r--] = tmp;
        }
    }
}
```

**Follow-up questions:**
- How to rotate counter-clockwise?
- How to rotate 180°?
- What if the matrix is not square?

---

## Problem 3: Search a 2D Matrix II

> **Interviewer asks:**
> "Write an efficient algorithm to search for a target value in an m×n matrix. Each row is sorted left to right, each column is sorted top to bottom."

*(Think for a moment...)*

---

**Approach:** Staircase Search from top-right corner.

```java
public boolean searchMatrix(int[][] matrix, int target) {
    int i = 0, j = matrix[0].length - 1;
    while (i < matrix.length && j >= 0) {
        if (matrix[i][j] == target) return true;
        else if (matrix[i][j] > target) j--;
        else i++;
    }
    return false;
}
// Time: O(n + m), Space: O(1)
```

**Follow-up questions:**
- What if the matrix is fully sorted (row ends connect to row starts)? → Binary search
- What is the brute force? → O(n×m)

---

## Problem 4: Number of Islands

> **Interviewer asks:**
> "Given a 2D grid of '1's (land) and '0's (water), count the number of islands. An island is surrounded by water and is formed by connecting adjacent lands horizontally or vertically."

*(Think for a moment...)*

---

**Approach:** DFS — for each unvisited '1', count it and DFS to mark the whole island.

*(See full code in Pattern 10)*

**Follow-up questions:**
- What if diagonals also count as connected?
- What if you can't modify the input grid? (Use a visited boolean array)
- How to find the largest island? (Track max during DFS)
- BFS approach?

---

## Problem 5: Spiral Matrix

> **Interviewer asks:**
> "Given an m×n matrix, return all elements in spiral order."

*(Think for a moment...)*

---

*(See full code in Pattern 3)*

**Follow-up questions:**
- Can you generate a spiral matrix given n? (Fill in spiral order instead of reading)
- Space complexity? Can you do it O(1) extra space?

---

## Problem 6: Word Search

> **Interviewer asks:**
> "Given an m×n grid of characters and a word, return true if the word exists in the grid. The word can be constructed from letters of sequentially adjacent cells (horizontally or vertically adjacent)."

*(Think for a moment...)*

---

**Approach:** DFS with backtracking from each starting cell.

```java
public boolean exist(char[][] board, String word) {
    int rows = board.length, cols = board[0].length;
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            if (dfs(board, word, i, j, 0)) return true;
    return false;
}

private boolean dfs(char[][] board, String word, int r, int c, int idx) {
    if (idx == word.length()) return true;  // found all characters
    if (r < 0 || r >= board.length) return false;
    if (c < 0 || c >= board[0].length) return false;
    if (board[r][c] != word.charAt(idx)) return false;  // mismatch

    char temp = board[r][c];
    board[r][c] = '#';  // mark visited (in-place, no extra space)

    boolean found = dfs(board, word, r-1, c, idx+1)
                 || dfs(board, word, r+1, c, idx+1)
                 || dfs(board, word, r, c-1, idx+1)
                 || dfs(board, word, r, c+1, idx+1);

    board[r][c] = temp;  // backtrack: restore original character
    return found;
}
// Time: O(n×m×4^L) where L = word length
```

**Follow-up questions:**
- Can you use the same cell twice? (Usually no)
- What if you need to find all words? (Use Trie — Word Search II)

---

## Problem 7: Surrounded Regions

> **Interviewer asks:**
> "Given an m×n matrix with 'X' and 'O'. Capture all regions of 'O' that are surrounded by 'X'. A region is captured by flipping all 'O's into 'X's. Regions connected to the border are NOT captured."

*(Think for a moment...)*

---

**Approach (Reverse Thinking — Interview Gold!):**
1. Any 'O' connected to the border is SAFE. Mark them as 'S' using DFS.
2. After DFS, flip all remaining 'O' (surrounded) to 'X'.
3. Flip 'S' back to 'O'.

```java
public void solve(char[][] board) {
    int rows = board.length, cols = board[0].length;

    // Step 1: Mark safe 'O's (border-connected) as 'S'
    for (int i = 0; i < rows; i++) {
        if (board[i][0] == 'O') dfs(board, i, 0);
        if (board[i][cols-1] == 'O') dfs(board, i, cols-1);
    }
    for (int j = 0; j < cols; j++) {
        if (board[0][j] == 'O') dfs(board, 0, j);
        if (board[rows-1][j] == 'O') dfs(board, rows-1, j);
    }

    // Step 2 & 3: Flip and restore
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++) {
            if (board[i][j] == 'O') board[i][j] = 'X';  // surrounded
            else if (board[i][j] == 'S') board[i][j] = 'O'; // safe, restore
        }
}

private void dfs(char[][] board, int r, int c) {
    if (r < 0 || r >= board.length) return;
    if (c < 0 || c >= board[0].length) return;
    if (board[r][c] != 'O') return;
    board[r][c] = 'S';
    dfs(board, r-1, c); dfs(board, r+1, c);
    dfs(board, r, c-1); dfs(board, r, c+1);
}
```

**Follow-up questions:**
- Why start from the border instead of the inside?
- BFS approach?

---

## Problem 8: Rotting Oranges

> **Interviewer asks:**
> "In a grid: 0 = empty, 1 = fresh orange, 2 = rotten orange. Every minute, any fresh orange 4-directionally adjacent to a rotten orange becomes rotten. Return the minimum number of minutes for all oranges to rot. Return -1 if impossible."

*(Think for a moment...)*

---

**Approach:** Multi-source BFS. Start all rotten oranges simultaneously in the queue.

```java
public int orangesRotting(int[][] grid) {
    int rows = grid.length, cols = grid[0].length;
    Queue<int[]> queue = new LinkedList<>();
    int fresh = 0;

    // Find all initially rotten oranges and count fresh ones
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++) {
            if (grid[i][j] == 2) queue.offer(new int[]{i, j});
            else if (grid[i][j] == 1) fresh++;
        }

    if (fresh == 0) return 0;  // No fresh oranges

    int minutes = 0;
    int[] dr = {-1, 1, 0, 0};
    int[] dc = {0, 0, -1, 1};

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int k = 0; k < size; k++) {
            int[] curr = queue.poll();
            for (int d = 0; d < 4; d++) {
                int nr = curr[0] + dr[d];
                int nc = curr[1] + dc[d];
                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                        && grid[nr][nc] == 1) {
                    grid[nr][nc] = 2;
                    fresh--;
                    queue.offer(new int[]{nr, nc});
                }
            }
        }
        minutes++;
    }

    return fresh == 0 ? minutes - 1 : -1;
}
// Time: O(n×m), Space: O(n×m)
```

**Follow-up questions:**
- Why BFS and not DFS here? (BFS gives shortest/minimum time naturally)
- What's the "minutes - 1" at the end? (Last BFS round processes but doesn't need one more minute)

---

## Problem 9: Pacific Atlantic Water Flow

> **Interviewer asks:**
> "Given an m×n island with heights. Rain water flows to adjacent cells with equal or lower height. Find all cells from which water can flow to BOTH the Pacific ocean (top/left border) and Atlantic ocean (bottom/right border)."

*(Think for a moment...)*

---

**Approach (Reverse Thinking again!):** Instead of flowing from each cell outward, flow INWARD from ocean borders. BFS/DFS from Pacific borders, mark reachable cells. BFS/DFS from Atlantic borders, mark reachable cells. Answer = intersection.

```java
public List<List<Integer>> pacificAtlantic(int[][] heights) {
    int rows = heights.length, cols = heights[0].length;
    boolean[][] pacific = new boolean[rows][cols];
    boolean[][] atlantic = new boolean[rows][cols];

    // BFS from Pacific borders (top row + left col)
    Queue<int[]> pQueue = new LinkedList<>();
    Queue<int[]> aQueue = new LinkedList<>();

    for (int i = 0; i < rows; i++) {
        pQueue.offer(new int[]{i, 0});       pacific[i][0] = true;
        aQueue.offer(new int[]{i, cols-1}); atlantic[i][cols-1] = true;
    }
    for (int j = 0; j < cols; j++) {
        pQueue.offer(new int[]{0, j});       pacific[0][j] = true;
        aQueue.offer(new int[]{rows-1, j}); atlantic[rows-1][j] = true;
    }

    bfs(heights, pQueue, pacific);
    bfs(heights, aQueue, atlantic);

    List<List<Integer>> result = new ArrayList<>();
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            if (pacific[i][j] && atlantic[i][j])
                result.add(Arrays.asList(i, j));

    return result;
}

private void bfs(int[][] h, Queue<int[]> q, boolean[][] visited) {
    int[] dr = {-1, 1, 0, 0}, dc = {0, 0, -1, 1};
    while (!q.isEmpty()) {
        int[] curr = q.poll();
        for (int d = 0; d < 4; d++) {
            int nr = curr[0] + dr[d], nc = curr[1] + dc[d];
            if (nr >= 0 && nr < h.length && nc >= 0 && nc < h[0].length
                    && !visited[nr][nc] && h[nr][nc] >= h[curr[0]][curr[1]]) {
                visited[nr][nc] = true;
                q.offer(new int[]{nr, nc});
            }
        }
    }
}
```

**Follow-up questions:**
- Why reverse direction (flow uphill from ocean)?
- Time complexity? O(n×m)

---

## Problem 10: Maximal Rectangle

> **Interviewer asks:**
> "Given a binary matrix filled with 0s and 1s, find the largest rectangle containing only 1s and return its area."

*(Think for a moment...)*

---

**Approach:** Reduce to "Largest Rectangle in Histogram" for each row.
For each row, compute heights array (consecutive 1s above each cell). Then apply the histogram stack algorithm.

```java
public int maximalRectangle(char[][] matrix) {
    if (matrix == null || matrix.length == 0) return 0;
    int cols = matrix[0].length;
    int[] heights = new int[cols];
    int maxArea = 0;

    for (char[] row : matrix) {
        // Build histogram
        for (int j = 0; j < cols; j++)
            heights[j] = (row[j] == '1') ? heights[j] + 1 : 0;
        // Find max rectangle in histogram
        maxArea = Math.max(maxArea, largestRectangleInHistogram(heights));
    }

    return maxArea;
}

private int largestRectangleInHistogram(int[] heights) {
    Stack<Integer> stack = new Stack<>();
    int maxArea = 0;
    int n = heights.length;

    for (int i = 0; i <= n; i++) {
        int h = (i == n) ? 0 : heights[i];
        while (!stack.isEmpty() && h < heights[stack.peek()]) {
            int height = heights[stack.pop()];
            int width = stack.isEmpty() ? i : i - stack.peek() - 1;
            maxArea = Math.max(maxArea, height * width);
        }
        stack.push(i);
    }

    return maxArea;
}
// Time: O(n×m), Space: O(m)
```

---

# 5. Debugging & Edge Cases

## The Essential Guard Block

**ALWAYS start every matrix function with this:**

```java
public void solve(int[][] matrix) {
    // Guard block — add BEFORE any logic
    if (matrix == null || matrix.length == 0) return;
    if (matrix[0] == null || matrix[0].length == 0) return;

    int rows = matrix.length;
    int cols = matrix[0].length;

    // ... rest of logic
}
```

---

## Common Edge Cases & How to Handle Them

### Empty Matrix

```java
// 3 ways a matrix can be "empty":
matrix == null           // null reference
matrix.length == 0       // zero rows
matrix[0].length == 0    // zero columns
```

### Single Row / Single Column

```java
int[][] singleRow = {{1, 2, 3}};  // rows=1, cols=3
int[][] singleCol = {{1}, {2}, {3}}; // rows=3, cols=1

// Spiral: skip bottom-row and left-col traversal
// Boundary: top row == bottom row, must not double-count
// Rotation: trivially handled by general code if guards are correct
```

### Non-Square Matrix

```java
// WRONG: Assuming square
for (int i = 0; i < matrix.length; i++)
    for (int j = 0; j < matrix.length; j++) // ❌ rows != cols

// CORRECT: Use separate variables
int rows = matrix.length;
int cols = matrix[0].length;
for (int i = 0; i < rows; i++)
    for (int j = 0; j < cols; j++) // ✅
```

### Index Out of Bounds

```java
// Happens when: going outside grid during DFS/BFS
// Fix: ALWAYS check bounds before accessing
if (r >= 0 && r < rows && c >= 0 && c < cols && /* condition */) {
    // safe to access matrix[r][c]
}
// Check ORDER matters: short-circuit evaluation protects against ArrayIndexOutOfBounds
```

### Infinite Loops

```java
// Causes:
// 1. Forgetting to mark cells visited in DFS/BFS
// 2. Using == check instead of >= for loop termination
// 3. Spiral traversal: missing guards for inner boundary checks

// Fixes:
// 1. Mark visited immediately upon adding to queue (not when polling)
// 2. Use visited[][] array OR mark in-place
// 3. Add if (top <= bottom) before bottom-row traversal in spiral
```

---

## Debugging Checklist

```
□ Does your code handle null input?
□ Does it handle empty matrix (0 rows, 0 cols)?
□ Does it handle 1×1 matrix?
□ Does it handle single row?
□ Does it handle single column?
□ Does it handle non-square matrix?
□ Are you using matrix.length (rows) vs matrix[0].length (cols) correctly?
□ Is your DFS/BFS marking cells visited before recursing?
□ Are your loop bounds inclusive/exclusive correct? (< vs <=)
□ In spiral: are the inner guards (top<=bottom, left<=right) present?
□ In rotation: is your inner loop starting at j = i+1?
```

---

# 6. Interview Traps

## Trap 1: "Do it in-place"

**What the interviewer wants:** O(1) extra space, no creating a new matrix.

**Common techniques:**
```java
// 1. Use first row/col as a marker array (Set Matrix Zeroes)
// 2. Change cell value in-place (Flood Fill: '1' → '0')
// 3. Use math trick: store two values in one cell
//    e.g., matrix[i][j] *= -1 to mark negative (restore with Math.abs)
matrix[i][j] *= -1;  // mark visited
// ... later
if (matrix[i][j] < 0) // was visited
matrix[i][j] = Math.abs(matrix[i][j]); // restore
```

---

## Trap 2: "Avoid Extra Space"

**Watch for:**
- Using visited array when you can modify the grid itself
- Creating result array when you can sort/modify in-place
- BFS queue size: can be O(n×m) in worst case — interviewer may ask about this

---

## Trap 3: "Optimize Time Complexity"

| Situation | Trap | Optimization |
|---|---|---|
| Many range sum queries | O(n×m) per query = O(Q×n×m) total | 2D Prefix Sum → O(1) per query |
| Search in sorted matrix | Binary search entire n×m | Staircase → O(n+m) |
| Finding all islands | DFS from every cell repeatedly | DFS/BFS with in-place marking |

---

## Trap 4: Large Matrix Constraints

```
If n,m ≤ 10^3: O(n×m) = 10^6 → safe
If n,m ≤ 10^3 with Q queries: O(Q×n×m) = 10^9 → too slow, use prefix sum
If DFS recursion depth = n×m = 10^6 → stack overflow! Use iterative DFS or BFS
```

---

## Trap 5: Tricky Rotation Formula

```
90° CW:  (i, j) → (j, n-1-i)
90° CCW: (i, j) → (n-1-j, i)
180°:    (i, j) → (n-1-i, n-1-j)

Memorize: Transpose then reverse for CW.
          Reverse then Transpose for CCW.
```

---

## Trap 6: "Can you reach from A to B?"

- **Yes/No reachability:** DFS is fine
- **Shortest path / minimum steps:** MUST use BFS (DFS doesn't guarantee shortest)
- **All possible paths:** DFS with backtracking

---

## Trap 7: Modifying Grid During Iteration

```java
// WRONG: Modifying while iterating can cause issues
for (int i = 0; i < rows; i++)
    for (int j = 0; j < cols; j++)
        if (condition) grid[i][j] = 0;  // might affect later cells in same pass

// CORRECT approach options:
// 1. Use a separate boolean[][] visited array
// 2. Mark with a temporary value (#, -1, etc.) then clean up
// 3. Collect changes in a list, apply after iteration
```

---

## Trap 8: Diagonal Traversal Direction

```
"Print top-right to bottom-left diagonals" ≠ "Print top-left to bottom-right diagonals"

Top-right diagonals: i + j = constant (fixed sum)
Top-left diagonals:  i - j = constant (fixed difference)

Always draw a 3×3 example to verify your formula before coding!
```

---

# 7. Practice Strategy

## 5-Day Intensive Practice Plan

### Day 1 — Foundations (2–3 hours)
```
Morning:
  □ Implement row-wise, column-wise, reverse traversal from scratch
  □ Implement boundary traversal (handle all edge cases manually)
  □ LeetCode 54: Spiral Matrix ★★

Afternoon:
  □ LeetCode 48: Rotate Image ★★
  □ LeetCode 867: Transpose Matrix ★
  □ Dry run each solution on 3×3 and 4×4 inputs by hand
```

### Day 2 — Search & Transform (2–3 hours)
```
Morning:
  □ LeetCode 74: Search a 2D Matrix ★★ (Binary Search approach)
  □ LeetCode 240: Search a 2D Matrix II ★★ (Staircase approach)

Afternoon:
  □ LeetCode 73: Set Matrix Zeroes ★★ (Focus: O(1) space solution)
  □ LeetCode 59: Spiral Matrix II ★★ (Fill spiral, not just read)
```

### Day 3 — DFS & Flood Fill (2–3 hours)
```
Morning:
  □ LeetCode 733: Flood Fill ★ (warm-up)
  □ LeetCode 200: Number of Islands ★★ (core pattern)

Afternoon:
  □ LeetCode 695: Max Area of Island ★★
  □ LeetCode 130: Surrounded Regions ★★ (reverse thinking)
```

### Day 4 — BFS & Harder DFS (2–3 hours)
```
Morning:
  □ LeetCode 994: Rotting Oranges ★★ (multi-source BFS)
  □ LeetCode 542: 01 Matrix ★★ (BFS from zeros)

Afternoon:
  □ LeetCode 79: Word Search ★★ (DFS + backtracking)
  □ LeetCode 417: Pacific Atlantic Water Flow ★★★
```

### Day 5 — Hard Problems & Review (2–3 hours)
```
Morning:
  □ LeetCode 304: Range Sum Query 2D (Prefix Sum)
  □ LeetCode 85: Maximal Rectangle ★★★

Afternoon:
  □ Timed practice: Pick any 3 random problems from Days 1–4
  □ For each, aim to solve in < 20 minutes
  □ Review any patterns you felt shaky on
```

---

## Revision Pattern Map

Revisit these concepts in this order if you have limited time:

```
1. Traversal (foundation of everything)
2. Spiral (most common "trick" question)
3. Rotation (in-place, Transpose + Reverse)
4. DFS template (boundary checks, marking visited)
5. BFS template (queue, level-order, shortest path)
6. Island counting (core of all connected-component problems)
7. Prefix Sum 2D (for range query problems)
8. Binary Search in matrix (sorted matrix optimization)
```

---

## Thinking Under Pressure in Interviews

### The 60-Second Framework

```
0–10 sec: Read problem twice, identify key words
10–30 sec: Map keywords to pattern (use the checklist)
30–50 sec: Think of brute force first (state it aloud)
50–60 sec: Think of optimization
```

### What to Say When Stuck

```
✅ "Let me start with a brute force approach and then optimize."
✅ "Can I modify the input matrix, or do I need to preserve it?"
✅ "What are the constraints on n and m?"
✅ "Let me trace through this example step by step."
✅ "I think this is similar to [pattern], let me apply that here."
```

### Avoid These Mistakes Under Pressure

```
❌ Jumping to code before thinking
❌ Silently struggling for 5+ minutes
❌ Forgetting to handle null/empty input
❌ Not testing on a small example before submitting
❌ Hardcoding n for both rows and cols
```

### The "Explain Your Thinking" Template

```
"This looks like a [PATTERN] problem because [REASON].
 My approach is:
   Step 1: [...]
   Step 2: [...]
   Step 3: [...]
 Time complexity: O(...), Space complexity: O(...)
 Edge cases I need to handle: null input, empty matrix, single row..."
```

---

## Quick Reference Card (Memorize These)

```java
// ✅ GUARD (always first)
if (matrix == null || matrix.length == 0 || matrix[0].length == 0) return;
int rows = matrix.length, cols = matrix[0].length;

// ✅ 4-DIRECTION TEMPLATE
int[] dr = {-1, 1, 0, 0};
int[] dc = {0, 0, -1, 1};
for (int d = 0; d < 4; d++) {
    int nr = r + dr[d], nc = c + dc[d];
    if (nr >= 0 && nr < rows && nc >= 0 && nc < cols) {
        // safe
    }
}

// ✅ TRANSPOSE (square, in-place)
for (int i = 0; i < n; i++)
    for (int j = i + 1; j < n; j++) { int t=m[i][j]; m[i][j]=m[j][i]; m[j][i]=t; }

// ✅ ROTATE 90° CW = Transpose + Reverse each row

// ✅ SPIRAL POINTER INIT
int top=0, bottom=rows-1, left=0, right=cols-1;
while (top<=bottom && left<=right) { ... }

// ✅ INDEX CONVERSION (1D ↔ 2D for binary search)
int row = mid / cols;
int col = mid % cols;

// ✅ 2D PREFIX SUM
prefix[i][j] = matrix[i-1][j-1] + prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1];
// Query:
sum = prefix[r2+1][c2+1] - prefix[r1][c2+1] - prefix[r2+1][c1] + prefix[r1][c1];
```

---

*Built for cracking ANY matrix problem in Java — from fundamentals to FAANG-level. Pattern recognition over memorization.*
