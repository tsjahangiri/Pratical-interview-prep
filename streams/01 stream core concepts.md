# Java Stream API — Part 1: Core Concepts & Operations

---

## What Is a Stream?

A Stream is a **pipeline for processing data**. It is not a data structure — it holds no data. It describes a sequence of operations to apply to a source (array, list, set, map, file, etc.).

Think of it as an assembly line in a factory:
```
Raw material → Station 1 → Station 2 → Station 3 → Final product
   (source)   (filter)    (transform)    (sort)      (collect)
```

```
List<Employee> employees → filter(active) → map(getName) → sorted() → collect(toList())
```

Three key characteristics:
- **Lazy** — intermediate operations do nothing until a terminal operation runs
- **Single-use** — a stream can only be consumed once
- **Non-destructive** — the original source is never modified

---

## Stream Pipeline Structure

Every stream pipeline has exactly three parts:

```
SOURCE          →    INTERMEDIATE OPS    →    TERMINAL OP
(produces stream)    (transform stream)       (produces result)

List.stream()   →    .filter().map()     →    .collect()
                     ↑ lazy, chainable         ↑ triggers execution
```

```java
List<String> result = employees.stream()          // SOURCE
    .filter(e -> e.getSalary() > 50000)           // INTERMEDIATE (lazy)
    .map(Employee::getName)                        // INTERMEDIATE (lazy)
    .sorted()                                      // INTERMEDIATE (lazy)
    .collect(Collectors.toList());                 // TERMINAL (executes everything)
```

Nothing runs until `.collect()` is called. This is laziness — and it matters for performance.

---

## Creating Streams

```java
// From a Collection
List<String> names = List.of("Alice", "Bob", "Charlie");
Stream<String> s1 = names.stream();
Stream<String> s2 = names.parallelStream(); // parallel execution

// From an array
int[] numbers = {1, 2, 3, 4, 5};
IntStream s3 = Arrays.stream(numbers);           // primitive IntStream
Stream<Integer> s4 = Arrays.stream(numbers)
                            .boxed();            // IntStream → Stream<Integer>

// From values directly
Stream<String> s5 = Stream.of("a", "b", "c");
Stream<String> s6 = Stream.empty();              // empty stream

// From a range (IntStream)
IntStream range      = IntStream.range(1, 6);    // 1,2,3,4  (exclusive end)
IntStream rangeClosed = IntStream.rangeClosed(1, 5); // 1,2,3,4,5 (inclusive end)

// Infinite streams (must be limited!)
Stream<Integer> naturals = Stream.iterate(1, n -> n + 1);  // 1,2,3,4,...
Stream<Double>  randoms  = Stream.generate(Math::random);  // infinite randoms
Stream<Integer> evens    = Stream.iterate(0, n -> n + 2)
                                 .limit(10);               // 0,2,4,...,18

// Java 9+ iterate with predicate (like a for-loop)
Stream<Integer> countdown = Stream.iterate(10, n -> n > 0, n -> n - 1); // 10,9,...,1

// From a String
IntStream chars = "Hello".chars();               // stream of char values (as int)
Stream<String> lines = "Hello\nWorld".lines();  // stream of lines

// From a Map
Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87);
Stream<Map.Entry<String, Integer>> entryStream = scores.entrySet().stream();
Stream<String> keyStream   = scores.keySet().stream();
Stream<Integer> valueStream = scores.values().stream();
```

---

## Intermediate Operations

These transform the stream. They are **lazy** — nothing runs until a terminal operation is called.

---

### filter() — Keep elements matching a condition

```java
List<Employee> employees = getEmployees();

// Basic filter
List<Employee> seniors = employees.stream()
    .filter(e -> e.getAge() > 35)
    .collect(Collectors.toList());

// Multiple conditions — chain filters or use &&
List<Employee> seniorEngineers = employees.stream()
    .filter(e -> e.getAge() > 35)
    .filter(e -> e.getDepartment().equals("Engineering"))
    .collect(Collectors.toList());

// Equivalent — single filter with &&
List<Employee> same = employees.stream()
    .filter(e -> e.getAge() > 35 && e.getDepartment().equals("Engineering"))
    .collect(Collectors.toList());

// Filter nulls out (defensive programming)
List<String> nonNull = Stream.of("Alice", null, "Bob", null, "Charlie")
    .filter(Objects::nonNull)
    .collect(Collectors.toList());
// Result: ["Alice", "Bob", "Charlie"]

// Filter with method reference
List<String> nonEmpty = names.stream()
    .filter(Predicate.not(String::isEmpty)) // Java 11+
    .collect(Collectors.toList());
```

---

### map() — Transform each element into something else

```java
// Transform Employee → String (name)
List<String> names = employees.stream()
    .map(Employee::getName)        // method reference
    .collect(Collectors.toList());

// Transform String → Integer (length)
List<Integer> lengths = names.stream()
    .map(String::length)
    .collect(Collectors.toList());

// Transform with logic
List<String> upperNames = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// Transform to a new object (DTO pattern — very common in production)
List<EmployeeDTO> dtos = employees.stream()
    .map(e -> new EmployeeDTO(e.getId(), e.getName(), e.getSalary()))
    .collect(Collectors.toList());

// Chain map operations
List<String> result = employees.stream()
    .map(Employee::getDepartment)
    .map(String::toLowerCase)
    .map(dept -> dept.replace(" ", "-"))
    .collect(Collectors.toList());
```

---

### flatMap() — Transform and flatten (one-to-many)

`map()` produces one output per input. `flatMap()` produces zero-or-more outputs per input and flattens them into a single stream.

```
map():     [A, B, C] → [[1,2], [3], [4,5]] (Stream<List<Integer>>)
flatMap(): [A, B, C] → [1, 2, 3, 4, 5]    (Stream<Integer>)
```

```java
// Each order has multiple items — get all items across all orders
List<Order> orders = getOrders();

List<OrderItem> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream()) // Order → Stream<Item>
    .collect(Collectors.toList());

// Split sentences into words
List<String> sentences = List.of("hello world", "foo bar baz");

List<String> words = sentences.stream()
    .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
    .collect(Collectors.toList());
// Result: ["hello", "world", "foo", "bar", "baz"]

// Remove nested Optional (flatMap on Optional)
List<String> departments = employees.stream()
    .map(Employee::getDepartmentOptional) // returns Optional<String>
    .flatMap(Optional::stream)            // Java 9+: empty Optional → empty stream
    .collect(Collectors.toList());
// Only employees WITH a department are included

// Get all unique skills across all employees
List<String> allSkills = employees.stream()
    .flatMap(e -> e.getSkills().stream())
    .distinct()
    .collect(Collectors.toList());
```

---

### distinct() — Remove duplicates

```java
List<Integer> nums = List.of(1, 2, 2, 3, 3, 3, 4);

List<Integer> unique = nums.stream()
    .distinct()
    .collect(Collectors.toList());
// Result: [1, 2, 3, 4]

// Works on objects too — uses equals() and hashCode()
List<String> uniqueDepts = employees.stream()
    .map(Employee::getDepartment)
    .distinct()
    .sorted()
    .collect(Collectors.toList());
```

---

### sorted() — Sort elements

```java
// Natural order (Comparable)
List<String> sorted = names.stream()
    .sorted()
    .collect(Collectors.toList());

// Custom comparator
List<Employee> byAge = employees.stream()
    .sorted(Comparator.comparingInt(Employee::getAge))
    .collect(Collectors.toList());

// Descending
List<Employee> byAgeDesc = employees.stream()
    .sorted(Comparator.comparingInt(Employee::getAge).reversed())
    .collect(Collectors.toList());

// Multi-level sort — department ascending, then salary descending
List<Employee> multiSort = employees.stream()
    .sorted(Comparator.comparing(Employee::getDepartment)
                      .thenComparingDouble(Employee::getSalary).reversed())
    .collect(Collectors.toList());

// Sort with null safety
List<String> withNulls = Arrays.asList("Charlie", null, "Alice", null, "Bob");
List<String> safeSort = withNulls.stream()
    .sorted(Comparator.nullsLast(Comparator.naturalOrder()))
    .collect(Collectors.toList());
// Result: ["Alice", "Bob", "Charlie", null, null]
```

---

### limit() and skip() — Pagination

```java
List<Employee> topPaid = employees.stream()
    .sorted(Comparator.comparingDouble(Employee::getSalary).reversed())
    .limit(10)                           // take first 10
    .collect(Collectors.toList());

// Pagination — page 3, page size 10
int pageSize = 10, page = 3;
List<Employee> paginated = employees.stream()
    .sorted(Comparator.comparing(Employee::getName))
    .skip((long)(page - 1) * pageSize)   // skip first (page-1)*size elements
    .limit(pageSize)                     // take next pageSize elements
    .collect(Collectors.toList());
```

---

### peek() — Debug without disrupting the pipeline

```java
// peek() sees each element without modifying the stream
// Use ONLY for debugging — never for side effects in production pipelines
List<Employee> result = employees.stream()
    .filter(e -> e.getSalary() > 50000)
    .peek(e -> System.out.println("After filter: " + e.getName()))  // debug
    .map(e -> applyBonus(e))
    .peek(e -> System.out.println("After bonus: " + e.getSalary())) // debug
    .collect(Collectors.toList());

// WARNING: peek() is lazy — it only fires when terminal op pulls elements
// In production: remove peek() calls — they add overhead and can hide bugs
```

---

### mapToInt / mapToLong / mapToDouble — Avoid boxing

```java
// Avoid this — maps to Stream<Integer> which boxes each int
employees.stream()
    .map(Employee::getAge)            // Stream<Integer> — boxing!
    .reduce(0, Integer::sum);

// Better — mapToInt creates IntStream (no boxing)
int totalAge = employees.stream()
    .mapToInt(Employee::getAge)       // IntStream — no boxing
    .sum();

double avgSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .average()
    .orElse(0.0);

long totalSalary = employees.stream()
    .mapToLong(e -> (long) e.getSalary())
    .sum();

// IntStream has specialized terminal ops: sum, average, min, max, count
IntSummaryStatistics stats = employees.stream()
    .mapToInt(Employee::getAge)
    .summaryStatistics();

System.out.println("Min age: "  + stats.getMin());
System.out.println("Max age: "  + stats.getMax());
System.out.println("Avg age: "  + stats.getAverage());
System.out.println("Sum age: "  + stats.getSum());
System.out.println("Count: "    + stats.getCount());
```

---

## Terminal Operations

These trigger execution of the entire pipeline and produce a result.

---

### collect() — Most Powerful Terminal Operation

```java
// Collect to List (most common)
List<String> nameList = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());          // mutable list
    // OR Java 16+:
    .toList();                              // immutable list

// Collect to Set (removes duplicates)
Set<String> deptSet = employees.stream()
    .map(Employee::getDepartment)
    .collect(Collectors.toSet());

// Collect to specific List/Set type
List<String> linkedList = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toCollection(LinkedList::new));

TreeSet<String> treeSet = employees.stream()
    .map(Employee::getDepartment)
    .collect(Collectors.toCollection(TreeSet::new));

// Collect to Map
Map<Integer, String> idToName = employees.stream()
    .collect(Collectors.toMap(
        Employee::getId,      // key extractor
        Employee::getName     // value extractor
    ));

// Map with merge function (handles duplicate keys)
Map<String, Double> deptAvgSalary = employees.stream()
    .collect(Collectors.toMap(
        Employee::getDepartment,
        Employee::getSalary,
        (existing, newVal) -> (existing + newVal) / 2 // merge on collision
    ));

// Collect to string
String namesCsv = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", "));
// "Alice, Bob, Charlie"

String withBrackets = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Charlie]"
```

---

### Grouping and Partitioning

```java
// groupingBy — group into Map<K, List<V>>
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));
// {"Engineering": [Alice, Bob], "Sales": [Charlie], ...}

// groupingBy with downstream collector
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()          // downstream: count per group
    ));
// {"Engineering": 15, "Sales": 8, ...}

Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

Map<String, Optional<Employee>> highestPaidByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))
    ));

// Multi-level grouping — department → seniority → list
Map<String, Map<String, List<Employee>>> nested = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getSeniority)
    ));

// partitioningBy — splits into exactly two groups (true/false)
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 70000));

List<Employee> highPaid = partitioned.get(true);
List<Employee> lowPaid  = partitioned.get(false);
```

---

### forEach() — Perform an action for each element

```java
// Simple iteration
employees.stream()
    .filter(e -> e.isActive())
    .forEach(e -> System.out.println(e.getName()));

// forEach with side effects (save to DB, send notification)
employees.stream()
    .filter(e -> e.needsReview())
    .forEach(reviewService::initiateReview);

// WARNING: forEach with parallel streams — order not guaranteed
// Use forEachOrdered() if order matters with parallel streams
employees.parallelStream()
    .forEach(e -> log(e));           // order unpredictable
employees.parallelStream()
    .forEachOrdered(e -> log(e));    // maintains encounter order — but slower
```

---

### reduce() — Aggregate to a single value

```java
// Sum with identity value (no Optional)
int sum = IntStream.rangeClosed(1, 100)
    .reduce(0, Integer::sum);           // 5050

// Without identity — returns Optional (stream might be empty)
Optional<Integer> product = Stream.of(1, 2, 3, 4, 5)
    .reduce((a, b) -> a * b);           // Optional[120]

// Reduce employees to the one with highest salary
Optional<Employee> topEarner = employees.stream()
    .reduce((e1, e2) -> e1.getSalary() > e2.getSalary() ? e1 : e2);
// Equivalent (cleaner):
Optional<Employee> topEarner = employees.stream()
    .max(Comparator.comparingDouble(Employee::getSalary));

// Concatenate strings (prefer Collectors.joining() for strings)
String combined = Stream.of("a", "b", "c")
    .reduce("", (s1, s2) -> s1 + s2);  // "abc" — but O(n²)! Use joining() instead.
```

---

### count, min, max, sum, average

```java
long count = employees.stream()
    .filter(Employee::isActive)
    .count();                           // O(n)

Optional<Employee> youngest = employees.stream()
    .min(Comparator.comparingInt(Employee::getAge));

Optional<Employee> oldest = employees.stream()
    .max(Comparator.comparingInt(Employee::getAge));

// For primitives — use IntStream/DoubleStream directly
int minAge = employees.stream()
    .mapToInt(Employee::getAge)
    .min()
    .orElseThrow();

double totalSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .sum();

OptionalDouble avgSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .average();
```

---

### findFirst(), findAny() — Short-circuit search

```java
// findFirst — guaranteed to return first in encounter order
Optional<Employee> firstEngineer = employees.stream()
    .filter(e -> e.getDepartment().equals("Engineering"))
    .findFirst();

// findAny — may return any matching element (faster in parallel streams)
Optional<Employee> anyEngineer = employees.parallelStream()
    .filter(e -> e.getDepartment().equals("Engineering"))
    .findAny();

// These are SHORT-CIRCUIT — stop processing as soon as found
// For large lists, they are much faster than collecting then getting first

// Safe usage with Optional
firstEngineer.ifPresent(e -> System.out.println("Found: " + e.getName()));
String name = firstEngineer.map(Employee::getName).orElse("Not found");
Employee e  = firstEngineer.orElseThrow(() -> new NotFoundException("No engineers"));
```

---

### anyMatch, allMatch, noneMatch — Short-circuit boolean checks

```java
boolean anyActive   = employees.stream().anyMatch(Employee::isActive);     // at least one
boolean allVerified = employees.stream().allMatch(Employee::isVerified);   // all match
boolean noneDeleted = employees.stream().noneMatch(Employee::isDeleted);   // none match

// All short-circuit — stop as soon as the answer is known
// anyMatch  stops at first TRUE
// allMatch  stops at first FALSE
// noneMatch stops at first TRUE

// Practical use
if (employees.stream().anyMatch(e -> e.getSalary() > 200_000)) {
    System.out.println("Has executives");
}
```

---

## Optional — Stream's Companion

Optional wraps a value that might be absent. It forces you to handle the "no result" case.

```java
// Creating Optionals
Optional<String> present = Optional.of("Alice");
Optional<String> empty   = Optional.empty();
Optional<String> nullable = Optional.ofNullable(null); // empty if null

// Extracting values
String name = present.get();                      // throws if empty — avoid
String safe = present.orElse("Default");          // value or default
String lazy = present.orElseGet(() -> compute()); // value or compute lazily
String must = present.orElseThrow();              // Java 10+, throws NoSuchElementException
String err  = present.orElseThrow(() -> new NotFoundException("not found"));

// Transforming
Optional<Integer> length = present.map(String::length);       // Optional[5]
Optional<String>  upper  = present.map(String::toUpperCase);  // Optional["ALICE"]

// Filtering
Optional<String> longName = present.filter(s -> s.length() > 3); // Optional["Alice"]
Optional<String> shortName = present.filter(s -> s.length() > 10); // Optional.empty

// FlatMap for Optional-returning functions
Optional<String> dept = Optional.of(employee)
    .flatMap(e -> Optional.ofNullable(e.getDepartment())); // avoids Optional<Optional<>>

// Chaining (replaces null checks)
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("Unknown");

// ifPresent — perform action only if value exists
present.ifPresent(s -> System.out.println("Found: " + s));

// ifPresentOrElse — Java 9+
present.ifPresentOrElse(
    s -> System.out.println("Found: " + s),
    ()  -> System.out.println("Not found")
);

// or() — Java 9+ — provide alternative Optional
Optional<Employee> result = findInCache(id)
    .or(() -> findInDatabase(id))  // only called if cache miss
    .or(() -> findInArchive(id));  // only called if DB miss
```

---

## Primitive Streams — IntStream, LongStream, DoubleStream

Use these instead of `Stream<Integer>` whenever possible — no boxing, specialized methods.

```java
// Generate range
IntStream.range(0, 5)         // 0,1,2,3,4
IntStream.rangeClosed(1, 5)   // 1,2,3,4,5

// Specialized terminal ops (not on Stream<Integer>)
IntStream ages = employees.stream().mapToInt(Employee::getAge);
ages.sum();               // total
ages.average();           // OptionalDouble
ages.min();               // OptionalInt
ages.max();               // OptionalInt
ages.count();             // long
ages.summaryStatistics(); // all at once

// Convert between stream types
IntStream      intS    = IntStream.of(1, 2, 3);
Stream<Integer> boxed  = intS.boxed();           // IntStream → Stream<Integer>
LongStream     longS   = intS.asLongStream();    // IntStream → LongStream
DoubleStream   doubleS = intS.asDoubleStream();  // IntStream → DoubleStream

// Array from IntStream
int[] arr = IntStream.range(0, 10).toArray(); // [0,1,2,...,9]
```
