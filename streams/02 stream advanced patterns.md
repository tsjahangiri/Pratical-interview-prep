# Java Stream API — Part 2: Advanced Patterns, Real-World Use & Interview

---

## Section 1: Real-World Patterns

---

### Pattern 1: DTO Transformation Pipeline

Converting domain objects to DTOs is the most common stream use in backend Java.

```java
// Domain model
@Data
public class Employee {
    private Long id;
    private String name;
    private String email;
    private double salary;
    private String department;
    private boolean active;
    private LocalDate hireDate;
    private List<String> skills;
}

// DTO for API response
@Data
public class EmployeeDTO {
    private Long id;
    private String name;
    private String maskedEmail;  // only first letter shown
    private String salaryBand;   // "Junior", "Mid", "Senior", "Lead"
    private int yearsAtCompany;
}

// Transformation
public List<EmployeeDTO> toDTO(List<Employee> employees) {
    return employees.stream()
        .filter(Employee::isActive)                      // only active employees
        .map(e -> {
            EmployeeDTO dto = new EmployeeDTO();
            dto.setId(e.getId());
            dto.setName(e.getName());
            dto.setMaskedEmail(maskEmail(e.getEmail()));
            dto.setSalaryBand(getSalaryBand(e.getSalary()));
            dto.setYearsAtCompany(
                (int) ChronoUnit.YEARS.between(e.getHireDate(), LocalDate.now())
            );
            return dto;
        })
        .sorted(Comparator.comparing(EmployeeDTO::getName))
        .collect(Collectors.toList());
}

private String maskEmail(String email) {
    int atIdx = email.indexOf('@');
    return email.charAt(0) + "***" + email.substring(atIdx);
}

private String getSalaryBand(double salary) {
    if (salary < 50_000) return "Junior";
    if (salary < 80_000) return "Mid";
    if (salary < 120_000) return "Senior";
    return "Lead";
}
```

---

### Pattern 2: Frequency Count and Grouping

```java
// Word frequency counter
public Map<String, Long> wordFrequency(String text) {
    return Arrays.stream(text.toLowerCase().split("\\W+"))
        .filter(w -> !w.isEmpty())
        .collect(Collectors.groupingBy(
            w -> w,                      // group by the word itself
            Collectors.counting()        // count occurrences
        ));
}

// Top 5 most frequent words
public List<String> topWords(String text, int n) {
    return Arrays.stream(text.toLowerCase().split("\\W+"))
        .filter(w -> !w.isEmpty())
        .collect(Collectors.groupingBy(w -> w, Collectors.counting()))
        .entrySet().stream()
        .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
        .limit(n)
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
}

// Department report: department → average salary → count
public void departmentReport(List<Employee> employees) {
    employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.teeing(                               // Java 12+
                Collectors.averagingDouble(Employee::getSalary),
                Collectors.counting(),
                (avg, count) -> String.format("avg=%.0f count=%d", avg, count)
            )
        ))
        .forEach((dept, stats) ->
            System.out.println(dept + ": " + stats)
        );
}
```

---

### Pattern 3: Nested Collection Processing (flatMap)

```java
// E-commerce: find all products across all orders in a date range
public List<String> getProductsInRange(
        List<Order> orders, LocalDate from, LocalDate to) {
    return orders.stream()
        .filter(o -> !o.getDate().isBefore(from) && !o.getDate().isAfter(to))
        .flatMap(o -> o.getItems().stream())
        .map(OrderItem::getProductName)
        .distinct()
        .sorted()
        .collect(Collectors.toList());
}

// Get all unique skills across a department
public Set<String> getDepartmentSkills(List<Employee> employees, String dept) {
    return employees.stream()
        .filter(e -> e.getDepartment().equals(dept))
        .flatMap(e -> e.getSkills().stream())
        .collect(Collectors.toSet());
}

// Build an inverted index: skill → list of employees with that skill
public Map<String, List<String>> skillToEmployees(List<Employee> employees) {
    return employees.stream()
        .flatMap(e -> e.getSkills().stream()
            .map(skill -> Map.entry(skill, e.getName())))   // skill → name pair
        .collect(Collectors.groupingBy(
            Map.Entry::getKey,
            Collectors.mapping(Map.Entry::getValue, Collectors.toList())
        ));
}
```

---

### Pattern 4: Map Processing

```java
Map<String, List<Integer>> scores = Map.of(
    "Alice", List.of(85, 92, 78, 96),
    "Bob",   List.of(70, 65, 80),
    "Charlie", List.of(95, 98, 100)
);

// Average score per student — Map<String, Double>
Map<String, Double> averages = scores.entrySet().stream()
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        e -> e.getValue().stream()
               .mapToInt(Integer::intValue)
               .average()
               .orElse(0.0)
    ));

// Top scorer
Optional<String> topScorer = scores.entrySet().stream()
    .max(Comparator.comparingDouble(e ->
        e.getValue().stream().mapToInt(Integer::intValue).average().orElse(0)))
    .map(Map.Entry::getKey);

// Filter map entries (Java idiom — stream entrySet, filter, re-collect)
Map<String, List<Integer>> highScorers = scores.entrySet().stream()
    .filter(e -> e.getValue().stream().allMatch(s -> s >= 80))
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

// Transform map values
Map<String, Integer> totalScores = scores.entrySet().stream()
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        e -> e.getValue().stream().mapToInt(Integer::intValue).sum()
    ));
```

---

### Pattern 5: Pagination and Chunking

```java
// Paginate a sorted stream
public <T> List<T> paginate(List<T> items, int page, int pageSize) {
    return items.stream()
        .skip((long)(page - 1) * pageSize)
        .limit(pageSize)
        .collect(Collectors.toList());
}

// Split a list into batches of size N (for bulk DB inserts, API rate limiting)
public <T> List<List<T>> partition(List<T> list, int batchSize) {
    AtomicInteger counter = new AtomicInteger(0);
    return new ArrayList<>(
        list.stream()
            .collect(Collectors.groupingBy(item -> counter.getAndIncrement() / batchSize))
            .values()
    );
}

// Usage: insert in batches of 1000
List<List<Employee>> batches = partition(employees, 1000);
batches.forEach(batch -> employeeRepository.saveAll(batch));
```

---

### Pattern 6: Statistics and Aggregation

```java
List<Double> prices = List.of(10.5, 20.0, 15.75, 30.0, 25.5);

// Full statistics in one pass
DoubleSummaryStatistics stats = prices.stream()
    .mapToDouble(Double::doubleValue)
    .summaryStatistics();

System.out.println("Count: " + stats.getCount());    // 5
System.out.println("Sum:   " + stats.getSum());      // 101.75
System.out.println("Min:   " + stats.getMin());      // 10.5
System.out.println("Max:   " + stats.getMax());      // 30.0
System.out.println("Avg:   " + stats.getAverage());  // 20.35

// Percentile approximation (sort then take index)
public double percentile(List<Double> values, double percentile) {
    List<Double> sorted = values.stream().sorted().collect(Collectors.toList());
    int index = (int) Math.ceil(percentile / 100.0 * sorted.size()) - 1;
    return sorted.get(Math.max(0, index));
}

// Compute salary distribution
Map<String, Long> distribution = employees.stream()
    .collect(Collectors.groupingBy(
        e -> {
            double s = e.getSalary();
            if (s < 50_000)  return "< 50k";
            if (s < 80_000)  return "50k-80k";
            if (s < 120_000) return "80k-120k";
            return "> 120k";
        },
        Collectors.counting()
    ));
```

---

### Pattern 7: Joining Streams and Optional Chains

```java
// Merge two streams
Stream<Employee> active   = getActiveEmployees().stream();
Stream<Employee> contract = getContractors().stream();
Stream<Employee> all      = Stream.concat(active, contract);

// Multiple streams
Stream<Employee> combined = Stream.of(
    getActiveEmployees().stream(),
    getContractors().stream(),
    getInterns().stream()
).flatMap(s -> s);

// Safe method chaining with Optional replacing null checks
public String getCityOfEmployee(Long employeeId) {
    return Optional.ofNullable(employeeRepository.findById(employeeId))
        .map(Employee::getAddress)       // null-safe map
        .map(Address::getCity)           // null-safe map
        .map(String::toUpperCase)
        .orElse("UNKNOWN");
}

// Chain of Optional alternatives (try cache → DB → default)
public User findUser(String id) {
    return Optional.ofNullable(userCache.get(id))
        .or(() -> userRepository.findById(id))
        .or(() -> Optional.of(User.anonymous()))
        .get();
}
```

---

## Section 2: Collectors Deep Dive

### Custom Collector

```java
// Build a custom Collector that creates an unmodifiable sorted list
public static <T extends Comparable<T>> Collector<T, List<T>, List<T>> toSortedUnmodifiableList() {
    return Collector.of(
        ArrayList::new,                       // supplier: create new list
        List::add,                            // accumulator: add element
        (left, right) -> {                    // combiner: merge (for parallel)
            left.addAll(right);
            return left;
        },
        list -> {                             // finisher: sort and lock
            Collections.sort(list);
            return Collections.unmodifiableList(list);
        }
    );
}

// Usage
List<Integer> sortedImmutable = Stream.of(3, 1, 4, 1, 5, 9)
    .collect(toSortedUnmodifiableList());
```

### Collectors.teeing() — Java 12+

Applies two collectors to the same stream and merges their results.

```java
// Single pass: get both min and max salary
record SalaryRange(double min, double max) {}

SalaryRange range = employees.stream()
    .collect(Collectors.teeing(
        Collectors.minBy(Comparator.comparingDouble(Employee::getSalary)),
        Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary)),
        (min, max) -> new SalaryRange(
            min.map(Employee::getSalary).orElse(0.0),
            max.map(Employee::getSalary).orElse(0.0)
        )
    ));

// Single pass: count and sum at the same time
record CountAndSum(long count, double sum) {}

CountAndSum result = employees.stream()
    .collect(Collectors.teeing(
        Collectors.counting(),
        Collectors.summingDouble(Employee::getSalary),
        CountAndSum::new
    ));

double average = result.sum() / result.count();
```

---

## Section 3: Parallel Streams

### When to Use (and When Not To)

```java
// GOOD for parallel:
// - Large collections (> 10,000 elements as a rough guide)
// - CPU-intensive operations (parsing, computation, encryption)
// - Stateless, independent operations

List<String> processedLarge = largeList.parallelStream()
    .filter(this::isValid)           // independent per element
    .map(this::expensiveTransform)   // CPU-intensive
    .collect(Collectors.toList());   // thread-safe collector

// BAD for parallel:
// - Small collections (overhead dominates)
// - I/O operations (threads block, no CPU gain)
// - Operations with shared mutable state (race conditions)
// - Ordered operations where order matters (findFirst, forEachOrdered are slower)

// WRONG — shared mutable state with parallel stream
List<Integer> results = new ArrayList<>();
numbers.parallelStream()
    .map(n -> n * 2)
    .forEach(results::add);   // ArrayList is NOT thread-safe — data corruption!

// CORRECT — collect to thread-safe result
List<Integer> results = numbers.parallelStream()
    .map(n -> n * 2)
    .collect(Collectors.toList()); // Collectors handle parallelism safely
```

### Parallel Stream Performance Reality

```java
// BENCHMARK: sum of 10 million integers
List<Integer> bigList = IntStream.range(0, 10_000_000)
    .boxed()
    .collect(Collectors.toList());

// Sequential: ~60ms on typical laptop
long seqSum = bigList.stream()
    .mapToLong(Integer::longValue)
    .sum();

// Parallel: ~20ms on 4-core machine (3x speedup)
long parSum = bigList.parallelStream()
    .mapToLong(Integer::longValue)
    .sum();

// But for 100 elements:
// Sequential: ~0.001ms
// Parallel:   ~0.5ms  ← 500x SLOWER due to thread pool overhead!
```

### Thread Safety in Parallel Streams

```java
// WRONG: shared counter (race condition)
int[] count = {0};
employees.parallelStream()
    .filter(Employee::isActive)
    .forEach(e -> count[0]++);  // NOT ATOMIC — wrong result

// CORRECT: use atomic or count()
long activeCount = employees.parallelStream()
    .filter(Employee::isActive)
    .count();  // count() is thread-safe

// Or use AtomicInteger
AtomicInteger count = new AtomicInteger(0);
employees.parallelStream()
    .filter(Employee::isActive)
    .forEach(e -> count.incrementAndGet()); // thread-safe

// Control the thread pool (prevent blocking ForkJoinPool.commonPool)
ForkJoinPool customPool = new ForkJoinPool(4);
List<String> results = customPool.submit(() ->
    employees.parallelStream()
        .map(this::callExternalService)  // isolated in custom pool
        .collect(Collectors.toList())
).get();
customPool.shutdown();
```

---

## Section 4: Stream Performance and Common Mistakes

### Mistake 1: Collect then Stream Again

```java
// WRONG — creates intermediate list unnecessarily
employees.stream()
    .filter(Employee::isActive)
    .collect(Collectors.toList())  // materializes to list
    .stream()                      // creates new stream
    .map(Employee::getName)
    .collect(Collectors.toList());

// CORRECT — one continuous pipeline
employees.stream()
    .filter(Employee::isActive)
    .map(Employee::getName)
    .collect(Collectors.toList());
```

### Mistake 2: Reusing a Stream

```java
Stream<Employee> stream = employees.stream().filter(Employee::isActive);

List<String> names = stream.map(Employee::getName).collect(Collectors.toList());
long count = stream.count(); // IllegalStateException: stream has already been operated upon or closed!

// CORRECT — each terminal op needs a new stream
List<String> names = employees.stream().filter(Employee::isActive)
    .map(Employee::getName).collect(Collectors.toList());
long count = employees.stream().filter(Employee::isActive).count();
```

### Mistake 3: Modifying Source During Stream

```java
// WRONG — modifying source while streaming it
employees.stream()
    .filter(e -> !e.isActive())
    .forEach(e -> employees.remove(e)); // ConcurrentModificationException!

// CORRECT
employees.removeIf(e -> !e.isActive()); // direct collection method

// Or collect what to remove, then remove
Set<Employee> toRemove = employees.stream()
    .filter(e -> !e.isActive())
    .collect(Collectors.toSet());
employees.removeAll(toRemove);
```

### Mistake 4: Using peek() for Side Effects

```java
// WRONG — using peek() as forEach() substitute (lazy!)
employees.stream()
    .filter(Employee::isActive)
    .peek(e -> emailService.send(e)); // DANGEROUS: may not execute if terminal op short-circuits
    // .count() only counts, doesn't need element values → peek may not fire at all!

// CORRECT — use forEach() for side effects
employees.stream()
    .filter(Employee::isActive)
    .forEach(e -> emailService.send(e));
```

### Mistake 5: Ignoring Boxing Cost

```java
// WRONG — boxing on every element
OptionalDouble avg = employees.stream()
    .map(Employee::getAge)           // Stream<Integer> — boxing!
    .reduce(0, Integer::sum);        // more boxing

// CORRECT — stay primitive
OptionalDouble avg = employees.stream()
    .mapToInt(Employee::getAge)      // IntStream — no boxing
    .average();                      // specialized method

// In hot paths (called millions of times), this difference is measurable
```

### Mistake 6: String Concatenation in Stream

```java
// WRONG — O(n²) due to String immutability
String result = employees.stream()
    .map(Employee::getName)
    .reduce("", (a, b) -> a + ", " + b); // creates new String object each time

// CORRECT — O(n) with joining collector
String result = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", "));
```

---

## Section 5: Interview Questions and Model Answers

---

### Q1: "What is the difference between map() and flatMap()?"

```
map():     one input → one output        (1:1)
flatMap(): one input → zero or more outputs (1:many), then flattens

map() wraps in a stream:
  Stream.of(List.of(1,2), List.of(3,4))
        .map(list -> list.stream())        // Stream<Stream<Integer>>

flatMap() unwraps and merges:
  Stream.of(List.of(1,2), List.of(3,4))
        .flatMap(List::stream)             // Stream<Integer>: 1,2,3,4

When to use flatMap:
  - Input element is a collection/array/Optional
  - You want to work with the inner elements, not the containers
```

---

### Q2: "What is lazy evaluation in streams? Why does it matter?"

```java
// Lazy evaluation demo
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// This does NOT execute until findFirst():
Optional<Integer> first = nums.stream()
    .filter(n -> {
        System.out.println("Checking: " + n);
        return n % 2 == 0;
    })
    .findFirst(); // short-circuit terminal

// Output: 
// Checking: 1
// Checking: 2   ← stops here! 2 is even, findFirst returns it
// Does NOT check 3 through 10

// Why it matters:
// 1. Short-circuit ops (findFirst, anyMatch, limit) can avoid processing entire stream
// 2. Operations are fused: filter + map applied to each element together,
//    not filter ALL then map ALL — avoids intermediate collections
// 3. Expensive operations (I/O, DB calls) in map() only run for elements that survive filter
```

---

### Q3: "When would you NOT use streams?"

```
Don't use streams when:

1. You need to modify index while iterating:
   for (int i = 0; i < list.size(); i++) {
       list.set(i, transform(list.get(i), i)); // index needed
   }
   
2. You need to check/modify elements relative to each other:
   for (int i = 1; i < nums.length; i++) {
       if (nums[i] > nums[i-1]) { } // comparing adjacent elements
   }

3. You need mutable local state with complex update logic:
   // A for-loop with a running total and running max and an early break
   // is often clearer than a stream

4. Performance is critical and the loop is tight:
   // Traditional for-loop is often faster for small arrays (< 100 elements)
   // due to no stream overhead

5. Exception handling is complex:
   // Checked exceptions in lambdas are verbose and awkward
   // A for-loop handles try-catch more naturally
```

---

### Q4: "Write a stream to find the second highest salary"

```java
// Approach 1: sort descending, skip 1, find first
Optional<Double> second = employees.stream()
    .map(Employee::getSalary)
    .distinct()                                           // remove duplicates
    .sorted(Comparator.reverseOrder())
    .skip(1)
    .findFirst();

// Approach 2: collect to sorted set, then get second
TreeSet<Double> salaries = employees.stream()
    .map(Employee::getSalary)
    .collect(Collectors.toCollection(TreeSet::new));
Double secondHighest = salaries.size() >= 2 
    ? salaries.lower(salaries.last()) 
    : null;

// Interview answer: state both, explain trade-offs
// Approach 1: cleaner, O(n log n) sort
// Approach 2: O(n log n) TreeSet insert but no sort step, direct navigation
```

---

### Q5: "Explain Collectors.groupingBy() with a downstream collector"

```java
// groupingBy returns Map<K, List<V>> by default
// downstream collector transforms the List<V> into something else

// Pattern: .collect(Collectors.groupingBy(classifier, downstreamCollector))

// Count per group (not list)
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,     // how to group
        Collectors.counting()        // what to do with each group
    ));

// Sum salaries per department
Map<String, Double> totalSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.summingDouble(Employee::getSalary)
    ));

// Collect names per department (mapping downstream)
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));

// Join names per department into a string
Map<String, String> nameCsvByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.joining(", "))
    ));
```

---

### Q6: "What's the difference between findFirst() and findAny()?"

```
findFirst(): always returns the first element in encounter order
             works correctly in both sequential and parallel streams
             parallel: must coordinate to find first → slower
             
findAny():   may return ANY element (undefined which one)
             sequential: typically returns first (but not guaranteed by spec)
             parallel: returns whichever thread finishes first → faster
             
Use findFirst() when: order matters, deterministic behavior needed
Use findAny() when: you just need "any" match, parallel stream, maximum speed
```

---

### Q7: "How do you handle checked exceptions inside a stream?"

```java
// PROBLEM: checked exceptions can't be thrown from lambda directly
employees.stream()
    .map(e -> parseJson(e.getData())) // parseJson throws IOException — won't compile!
    .collect(Collectors.toList());

// SOLUTION 1: Wrap in unchecked exception
employees.stream()
    .map(e -> {
        try {
            return parseJson(e.getData());
        } catch (IOException ex) {
            throw new UncheckedIOException(ex); // wrap as unchecked
        }
    })
    .collect(Collectors.toList());

// SOLUTION 2: Helper method to wrap the function
@FunctionalInterface
interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;
}

static <T, R> Function<T, R> wrap(ThrowingFunction<T, R> fn) {
    return t -> {
        try { return fn.apply(t); }
        catch (Exception e) { throw new RuntimeException(e); }
    };
}

// Clean usage
employees.stream()
    .map(wrap(e -> parseJson(e.getData())))  // clean!
    .collect(Collectors.toList());

// SOLUTION 3: Handle in method, return Optional
employees.stream()
    .map(e -> {
        try { return Optional.of(parseJson(e.getData())); }
        catch (IOException ex) {
            log.warn("Failed to parse {}", e.getId(), ex);
            return Optional.<JsonNode>empty();
        }
    })
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());
```

---

## Section 6: Advanced One-Liners (Production Patterns)

```java
// 1. Null-safe list processing
List<String> safe = Optional.ofNullable(maybeNullList)
    .orElseGet(Collections::emptyList)
    .stream()
    .filter(Objects::nonNull)
    .collect(Collectors.toList());

// 2. Map inversion: Map<K,V> → Map<V, List<K>>
Map<String, List<Long>> roleToIds = idToRole.entrySet().stream()
    .collect(Collectors.groupingBy(
        Map.Entry::getValue,
        Collectors.mapping(Map.Entry::getKey, Collectors.toList())
    ));

// 3. Index elements (add position to each element)
List<String> indexed = IntStream.range(0, names.size())
    .mapToObj(i -> i + ": " + names.get(i))
    .collect(Collectors.toList());
// ["0: Alice", "1: Bob", "2: Charlie"]

// 4. Zip two lists (no built-in zip in Java streams)
List<String> zipped = IntStream.range(0, Math.min(keys.size(), values.size()))
    .mapToObj(i -> keys.get(i) + "=" + values.get(i))
    .collect(Collectors.toList());

// 5. Running total (scan / prefix sum)
int[] nums = {1, 2, 3, 4, 5};
int[] prefixSum = IntStream.range(0, nums.length)
    .map(i -> IntStream.rangeClosed(0, i).map(j -> nums[j]).sum())
    .toArray();
// More efficient with arrays directly, but shows the pattern

// 6. Cartesian product
List<String> cartesian = Stream.of("A", "B", "C")
    .flatMap(l -> Stream.of("1", "2", "3").map(r -> l + r))
    .collect(Collectors.toList());
// [A1, A2, A3, B1, B2, B3, C1, C2, C3]

// 7. Check all required fields are non-null
boolean isComplete = Stream.of(
    user.getName(), user.getEmail(), user.getPhone()
).allMatch(Objects::nonNull);

// 8. Deep max (max across nested collections)
Optional<Integer> overallMax = matrix.stream()
    .flatMapToInt(row -> Arrays.stream(row))
    .max();

// 9. Group consecutive duplicates (sliding window via reduce)
// [1,1,2,3,3,3] → [[1,1],[2],[3,3,3]]
// (Requires reduce with accumulator tracking last group — advanced pattern)

// 10. Collect to Map, last value wins on duplicate key
Map<String, Employee> latestByEmail = employees.stream()
    .collect(Collectors.toMap(
        Employee::getEmail,
        e -> e,
        (existing, replacement) -> replacement  // last write wins
    ));
```

---

## Quick Reference Card

```
INTERMEDIATE (lazy)         │ What it does
────────────────────────────┼─────────────────────────────────────────
filter(predicate)           │ keep elements matching condition
map(function)               │ transform each element (1:1)
flatMap(function)           │ transform + flatten (1:many)
distinct()                  │ remove duplicates (uses equals/hashCode)
sorted()                    │ natural order sort
sorted(comparator)          │ custom order sort
limit(n)                    │ take first n elements
skip(n)                     │ skip first n elements
peek(consumer)              │ debug only — DO NOT use for side effects
mapToInt/Long/Double        │ convert to primitive stream (avoid boxing)
                            │
TERMINAL (eager, executes)  │ What it produces
────────────────────────────┼─────────────────────────────────────────
collect(collector)          │ List, Set, Map, String, or custom
forEach(consumer)           │ void — side effects per element
count()                     │ long — number of elements
min(comparator)             │ Optional<T> — minimum element
max(comparator)             │ Optional<T> — maximum element
findFirst()                 │ Optional<T> — first element (ordered)
findAny()                   │ Optional<T> — any element (fastest in parallel)
anyMatch(predicate)         │ boolean — at least one matches (short-circuit)
allMatch(predicate)         │ boolean — all match (short-circuit)
noneMatch(predicate)        │ boolean — none match (short-circuit)
reduce(identity, op)        │ T — fold all elements into one
toArray()                   │ Object[] — stream to array
                            │
COLLECTORS (toXxx)          │ Produces
────────────────────────────┼─────────────────────────────────────────
toList()                    │ List<T> (Java 16+ immutable)
toUnmodifiableList()        │ Unmodifiable List<T>
toSet()                     │ Set<T>
toMap(k, v)                 │ Map<K,V>
toMap(k, v, merge)          │ Map<K,V> with collision handling
joining(sep)                │ String with separator
joining(sep, pre, suf)      │ String with prefix and suffix
groupingBy(classifier)      │ Map<K, List<T>>
groupingBy(c, downstream)   │ Map<K, R> with transformed values
partitioningBy(predicate)   │ Map<Boolean, List<T>>
counting()                  │ Long — count per group (downstream)
summingInt/Long/Double      │ numeric sum (downstream)
averagingInt/Long/Double    │ Double average (downstream)
mapping(fn, downstream)     │ transform then collect (downstream)
teeing(c1, c2, merger)      │ apply 2 collectors, merge results (Java 12+)
```
