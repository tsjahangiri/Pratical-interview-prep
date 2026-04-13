# Java Interview Preparation Guide (Beginner → Advanced)

> Written for engineers who want to **clear real interviews**, not just memorize theory.
> Focus: WHY + WHEN + HOW, with traps, code, and real usage.

---

## Table of Contents

1. [Core Java Fundamentals](#1-core-java-fundamentals)
2. [Collections Framework](#2-collections-framework)
3. [Multithreading & Concurrency](#3-multithreading--concurrency)
4. [Java 8+ Features](#4-java-8-features)
5. [JVM Internals](#5-jvm-internals)
6. [Exception Handling](#6-exception-handling)
7. [Spring Boot](#7-spring-boot)
8. [System Design Basics](#8-system-design-basics)
9. [Real Interview Coding Problems](#9-real-interview-coding-problems)
10. [Interview Traps & FAQs](#10-interview-traps--faqs)
11. [Cheat Sheet](#11-cheat-sheet)

---

# 1. Core Java Fundamentals

---

## OOP Concepts

### ❓ What are the 4 pillars of OOP?

✅ **Simple Explanation:**
OOP is built on 4 pillars: Encapsulation, Abstraction, Inheritance, Polymorphism.

| Pillar | In one line |
|---|---|
| Encapsulation | Hiding data using access modifiers |
| Abstraction | Hiding implementation, showing only what matters |
| Inheritance | Child class reuses parent class code |
| Polymorphism | Same method name, different behavior |

💡 **Real-world Example:**
- **Encapsulation**: A car's engine is hidden. You just press the accelerator.
- **Abstraction**: You use `list.add()` without knowing the internal array logic.
- **Inheritance**: `ElectricCar extends Car` — gets all car features, adds battery.
- **Polymorphism**: `animal.sound()` calls `Dog.sound()` or `Cat.sound()` based on type.

⚠️ **Interview Trap:**
> "Is Abstraction and Encapsulation the same?"

NO. Encapsulation is about **protecting data** (private fields + getters/setters). Abstraction is about **hiding complexity** (interface/abstract class). You can have one without the other.

💻 **Code Example:**
```java
// Encapsulation
public class BankAccount {
    private double balance; // hidden

    public double getBalance() { return balance; } // controlled access
    public void deposit(double amount) {
        if (amount > 0) balance += amount;
    }
}

// Abstraction
public interface PaymentGateway {
    void processPayment(double amount); // what it does, not HOW
}

public class StripeGateway implements PaymentGateway {
    @Override
    public void processPayment(double amount) {
        // complex Stripe API logic hidden here
    }
}
```

---

## SOLID Principles

### ❓ Explain SOLID with examples.

✅ **S — Single Responsibility Principle (SRP)**
> A class should have only ONE reason to change.

```java
// BAD — does too many things
class OrderService {
    public void saveOrder(Order o) { /* DB logic */ }
    public void sendEmail(Order o) { /* email logic */ }
    public void generateInvoice(Order o) { /* PDF logic */ }
}

// GOOD — each class has one job
class OrderService    { public void saveOrder(Order o) { } }
class EmailService    { public void sendEmail(Order o) { } }
class InvoiceService  { public void generateInvoice(Order o) { } }
```

✅ **O — Open/Closed Principle (OCP)**
> Open for extension, closed for modification.

```java
// BAD — you modify existing code to add new discount types
class DiscountCalculator {
    public double calculate(String type, double price) {
        if (type.equals("STUDENT")) return price * 0.9;
        if (type.equals("SENIOR"))  return price * 0.8;
        // keep modifying this forever...
    }
}

// GOOD — extend without touching existing code
interface Discount {
    double apply(double price);
}
class StudentDiscount implements Discount {
    public double apply(double price) { return price * 0.9; }
}
class SeniorDiscount implements Discount {
    public double apply(double price) { return price * 0.8; }
}
```

✅ **L — Liskov Substitution Principle (LSP)**
> A subclass must be usable in place of its parent without breaking behavior.

```java
// BAD — Square breaks Rectangle behavior
class Rectangle {
    int width, height;
    void setWidth(int w)  { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}
class Square extends Rectangle {
    @Override void setWidth(int w)  { this.width = this.height = w; } // breaks LSP!
}

// GOOD — model them separately
interface Shape { int area(); }
class Rectangle implements Shape { ... }
class Square    implements Shape { ... }
```

✅ **I — Interface Segregation Principle (ISP)**
> Don't force classes to implement methods they don't use.

```java
// BAD — fat interface
interface Worker {
    void work();
    void eat();
    void sleep();
}

// GOOD — split interfaces
interface Workable { void work(); }
interface Eatable  { void eat();  }
class HumanWorker implements Workable, Eatable { ... }
class RobotWorker  implements Workable         { ... } // robots don't eat
```

✅ **D — Dependency Inversion Principle (DIP)**
> High-level modules should not depend on low-level modules. Both should depend on abstractions.

```java
// BAD — OrderService directly depends on MySQLDB
class OrderService {
    MySQLDB db = new MySQLDB(); // tightly coupled
}

// GOOD — depends on abstraction
interface Database { void save(Order o); }
class MySQLDB    implements Database { ... }
class PostgresDB implements Database { ... }

class OrderService {
    private Database db; // depends on interface
    OrderService(Database db) { this.db = db; } // injected
}
```

---

## Interface vs Abstract Class

### ❓ When do you use Interface vs Abstract Class?

✅ **Simple Explanation:**

| Feature | Interface | Abstract Class |
|---|---|---|
| Methods | abstract + default + static | abstract + concrete |
| Variables | `public static final` only | Any type |
| Constructor | No | Yes |
| Multiple inheritance | Yes (implement many) | No (extend one) |
| `IS-A` relationship | Can-do contract | True IS-A |
| Use when | Unrelated classes share behavior | Related classes share state |

⚠️ **Interview Trap:**
> "Can interfaces have constructors?"

NO. Interfaces cannot have constructors. They cannot be instantiated.

> "Can an abstract class have no abstract methods?"

YES. An abstract class can have zero abstract methods. It just can't be instantiated directly.

```java
// Interface — defines a CONTRACT (what a thing can do)
public interface Flyable {
    void fly(); // abstract by default
    default void land() { System.out.println("Landing..."); } // Java 8+
}

// Abstract class — defines a TEMPLATE (what a thing IS)
public abstract class Vehicle {
    String brand;
    int speed;

    Vehicle(String brand) { this.brand = brand; } // constructor allowed

    abstract void start(); // must override

    void stop() { System.out.println("Vehicle stopped"); } // concrete method
}

// A class can do both
class FlyingCar extends Vehicle implements Flyable {
    FlyingCar() { super("FlyingCar Inc"); }
    public void start() { System.out.println("Engine started"); }
    public void fly()   { System.out.println("Taking off!"); }
}
```

---

## Immutable Classes

### ❓ How do you create an immutable class in Java?

✅ **Rules:**
1. Declare class as `final`
2. All fields `private final`
3. No setters
4. Constructor sets all fields
5. If field is a mutable object (like `List`), return a defensive copy from getter

💡 **Real-world example:** `String`, `Integer`, `LocalDate` are all immutable.

⚠️ **Interview Trap:**
> "What if an immutable class has a `List` field?"

If you return the same `List` reference, the caller can mutate it! You must return a copy.

```java
public final class ImmutableEmployee {
    private final String name;
    private final int age;
    private final List<String> skills; // mutable field — careful!

    public ImmutableEmployee(String name, int age, List<String> skills) {
        this.name = name;
        this.age = age;
        this.skills = new ArrayList<>(skills); // defensive copy on intake
    }

    public String getName()       { return name; }
    public int getAge()           { return age; }
    public List<String> getSkills() {
        return Collections.unmodifiableList(skills); // defensive copy on return
    }
}
```

---

## equals() vs hashCode()

### ❓ What is the contract between equals() and hashCode()?

✅ **The Rules:**
1. If `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` MUST be true.
2. If `a.hashCode() == b.hashCode()`, `a.equals(b)` may or may not be true (collision is allowed).
3. If you override `equals()`, you MUST override `hashCode()` too.

⚠️ **Interview Trap:**
```java
// Only override equals — NOT hashCode
class Employee {
    int id;
    @Override
    public boolean equals(Object o) { return ((Employee)o).id == this.id; }
    // hashCode NOT overridden!
}

Map<Employee, String> map = new HashMap<>();
Employee e1 = new Employee(); e1.id = 1;
map.put(e1, "TJ");

Employee e2 = new Employee(); e2.id = 1; // same id = same logical object
System.out.println(map.get(e2)); // prints NULL! because hashCode differs
```
The `HashMap` uses `hashCode()` first to find the bucket. Without a matching `hashCode`, it never even tries `equals()`.

```java
// CORRECT — always override both
class Employee {
    int id;
    String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee)) return false;
        Employee e = (Employee) o;
        return id == e.id && Objects.equals(name, e.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }
}
```

---

## == vs equals()

### ❓ What is the difference between == and equals()?

✅ **Simple rule:**
- `==` compares **memory addresses** (reference equality)
- `equals()` compares **content** (logical equality, if overridden)

⚠️ **Interview Trap — String Pool:**
```java
String a = "hello";          // goes to String pool
String b = "hello";          // reuses same pool object
String c = new String("hello"); // new heap object

System.out.println(a == b);      // true  (same pool reference)
System.out.println(a == c);      // false (different heap object)
System.out.println(a.equals(c)); // true  (same content)
```

---

## final vs finally vs finalize

### ❓ Explain final, finally, and finalize.

These three look similar but are completely unrelated.

✅ **final:**
- `final variable` → value cannot be reassigned
- `final method` → cannot be overridden
- `final class` → cannot be subclassed (e.g., `String`)

✅ **finally:**
- Block in try-catch that ALWAYS executes (even if exception thrown)
- Used to release resources (close DB connection, file handle)

✅ **finalize():**
- Method called by GC before an object is garbage collected
- **Deprecated in Java 9, removed in Java 18.** Don't use it. Use `AutoCloseable` instead.

⚠️ **Interview Trap:**
> "Does finally ALWAYS run?"

Almost always — but NOT if:
1. `System.exit()` is called
2. JVM crashes
3. The thread is killed

```java
// final
final int MAX = 100;
// MAX = 200; // compile error

final class MathUtils { } // cannot be extended
// class ExtendedMath extends MathUtils { } // compile error

// finally
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    System.out.println("Caught: " + e.getMessage());
} finally {
    System.out.println("Always runs"); // always printed
}

// finalize — DON'T USE (just know for interviews)
@Override
protected void finalize() throws Throwable {
    // cleanup before GC — unreliable, deprecated
}
```

---

## Static vs Non-Static

### ❓ What is the difference between static and non-static members?

✅ **Simple Explanation:**
- `static` → belongs to the **class**, shared by all instances
- `non-static` → belongs to the **instance**, each object has its own copy

⚠️ **Interview Trap:**
> "Can you call a non-static method from a static method?"

NO — not directly. You need an instance.

```java
class Counter {
    static int count = 0;  // shared across all Counter objects
    int instanceId;         // each instance has its own

    Counter() {
        count++;
        this.instanceId = count;
    }

    static void showCount() {
        System.out.println("Total: " + count);
        // System.out.println(instanceId); // COMPILE ERROR — no instance context
    }

    void showId() {
        System.out.println("ID: " + instanceId);
        showCount(); // non-static can access static — fine
    }
}
```

---

## Method Overloading vs Method Overriding

### ❓ What is the difference?

| Feature | Overloading | Overriding |
|---|---|---|
| Where | Same class | Parent + Child class |
| Parameters | Must differ | Must be same |
| Return type | Can differ | Must be same (or covariant) |
| Resolved at | Compile-time | Runtime |
| Polymorphism type | Static | Dynamic |

⚠️ **Interview Trap:**
> "Can you override a static method?"

NO — static methods are resolved at compile-time. You can **hide** a static method (define same signature in child), but it's NOT overriding.

```java
// Overloading — same class, different params
class Calculator {
    int add(int a, int b)       { return a + b; }
    double add(double a, double b) { return a + b; }
    int add(int a, int b, int c)  { return a + b + c; }
}

// Overriding — child changes parent behavior
class Animal {
    void sound() { System.out.println("Some sound"); }
}
class Dog extends Animal {
    @Override
    void sound() { System.out.println("Woof!"); } // runtime polymorphism
}

Animal a = new Dog();
a.sound(); // prints "Woof!" — decided at runtime
```

---

## Pass By Value in Java

### ❓ Is Java pass-by-value or pass-by-reference?

✅ **Java is ALWAYS pass-by-value.**

The confusion: when you pass an object, you pass the **value of the reference** (the memory address), not the object itself.

⚠️ **Interview Trap:**
```java
void changeValue(int x) {
    x = 100; // only changes local copy
}
int a = 5;
changeValue(a);
System.out.println(a); // still 5

void changeName(Employee e) {
    e.name = "Changed"; // modifies the OBJECT the reference points to — works!
}
void reassign(Employee e) {
    e = new Employee("New"); // only changes local reference — original untouched
}

Employee emp = new Employee("TJ");
changeName(emp);
System.out.println(emp.name); // "Changed"
reassign(emp);
System.out.println(emp.name); // still "Changed" — reassign didn't affect original
```

---

## String Pool & Immutability

### ❓ Why is String immutable in Java?

✅ **Reasons:**
1. **Security** — passwords, file paths, network connections use Strings. If mutable, could be changed after validation.
2. **String Pool** — immutability allows the JVM to cache Strings. Multiple references can safely point to the same object.
3. **Thread safety** — immutable objects are inherently thread-safe.
4. **hashCode caching** — String caches its hashCode, making it safe for HashMap keys.

```java
String s1 = "hello";
String s2 = "hello"; // reuses s1 from pool
String s3 = new String("hello"); // bypasses pool, new heap object

s1 == s2  // true  (same pool object)
s1 == s3  // false (different objects)
s1.equals(s3) // true (same content)

// String "modification" creates a NEW object
String original = "hello";
String modified = original.toUpperCase();
System.out.println(original); // still "hello"
System.out.println(modified); // "HELLO" — new object
```

---

## Autoboxing & Unboxing

### ❓ What is autoboxing and what are the pitfalls?

✅ **Autoboxing** = automatic conversion from `int` → `Integer`
✅ **Unboxing** = automatic conversion from `Integer` → `int`

⚠️ **Interview Trap — Integer cache:**
```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b); // true — cached range: -128 to 127

Integer c = 128;
Integer d = 128;
System.out.println(c == d); // FALSE — outside cache range, new objects
System.out.println(c.equals(d)); // true — always use equals() for objects

// NullPointerException from unboxing null
Integer x = null;
int y = x; // NullPointerException! — unboxing null throws NPE

// Performance — avoid autoboxing in tight loops
Long sum = 0L;
for (int i = 0; i < 1_000_000; i++) {
    sum += i; // creates 1 million Long objects!
}
// Use long sum = 0L; instead
```

---

## Deep Copy vs Shallow Copy

### ❓ What is the difference?

✅ **Shallow Copy:** Creates a new object but copies references to the same nested objects.
✅ **Deep Copy:** Creates a new object AND recursively copies all nested objects.

```java
class Address {
    String city;
    Address(String city) { this.city = city; }
}

class Employee implements Cloneable {
    String name;
    Address address;

    Employee(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    // Shallow copy — default clone()
    @Override
    public Employee clone() throws CloneNotSupportedException {
        return (Employee) super.clone(); // address reference is SHARED
    }

    // Deep copy — manually clone nested objects
    public Employee deepCopy() {
        return new Employee(this.name, new Address(this.address.city));
    }
}

// Shallow copy demonstration
Employee e1 = new Employee("TJ", new Address("Munich"));
Employee e2 = e1.clone();
e2.address.city = "Berlin";
System.out.println(e1.address.city); // "Berlin" — e1 also changed! (shallow)

// Deep copy demonstration
Employee e3 = e1.deepCopy();
e3.address.city = "Paris";
System.out.println(e1.address.city); // "Berlin" — e1 unchanged (deep)
```

---

## Transient Keyword

### ❓ What does transient mean?

✅ Fields marked `transient` are **skipped during serialization**. They are not saved to the stream.

```java
class User implements Serializable {
    String username;
    transient String password; // will NOT be serialized
    transient int sessionToken; // not saved — regenerated on login
}
```

---

## Default Methods in Interfaces (Java 8+)

### ❓ Why were default methods added?

✅ Before Java 8, adding a new method to an interface broke ALL implementing classes.
Default methods allow **backward-compatible evolution** of interfaces.

```java
interface Greeter {
    void greet(String name); // abstract

    default void greetLoudly(String name) { // default — implementing classes don't have to override
        greet(name.toUpperCase());
    }

    static Greeter formal() { // static factory method in interface
        return name -> System.out.println("Good day, " + name);
    }
}
```

⚠️ **Interview Trap — Diamond problem:**
```java
interface A { default void hello() { System.out.println("A"); } }
interface B { default void hello() { System.out.println("B"); } }

class C implements A, B {
    @Override
    public void hello() { A.super.hello(); } // MUST override and specify which one
}
```

---

# 2. Collections Framework

---

## List vs Set vs Map

| | List | Set | Map |
|---|---|---|---|
| Duplicates | Allowed | NOT allowed | Keys unique, values allowed |
| Order | Insertion order (ArrayList) | No guarantee (HashSet) | No guarantee (HashMap) |
| Null | Multiple nulls | One null (HashSet) | One null key (HashMap) |
| Access | By index | No index | By key |

---

## ArrayList vs LinkedList

| Feature | ArrayList | LinkedList |
|---|---|---|
| Internal structure | Dynamic array | Doubly linked list |
| Random access (get) | O(1) | O(n) |
| Insert/delete at end | O(1) amortized | O(1) |
| Insert/delete at middle | O(n) — shifting | O(n) — traversal |
| Memory | Less (data only) | More (data + 2 pointers) |
| Use when | Frequent reads | Frequent insert/delete at ends |

⚠️ **Interview Trap:**
> "LinkedList is always faster for insertions."

NOT TRUE. For insertion at middle, LinkedList must still traverse O(n) to find the position. Only insertion **at head or tail** is O(1).

```java
// ArrayList — good for indexed access
List<String> list = new ArrayList<>();
list.add("a"); list.add("b"); list.add("c");
String s = list.get(1); // O(1) — direct array access

// LinkedList — good as a queue/deque
Deque<String> deque = new LinkedList<>();
deque.addFirst("a");
deque.addLast("b");
deque.pollFirst(); // O(1)
```

---

## HashMap Internal Working

### ❓ How does HashMap work internally?

✅ **Step-by-step:**

**1. Structure:**
HashMap is an array of "buckets". Each bucket is a linked list (or red-black tree in Java 8+ when bucket size > 8).

```
buckets[0] → null
buckets[1] → [key="cat", value=1] → [key="act", value=2]  (collision)
buckets[2] → [key="dog", value=3]
...
```

**2. PUT operation:**
```
hashMap.put("name", "TJ")

Step 1: Compute hashCode("name") → e.g., 3029967
Step 2: Apply hash spreading: (h ^ (h >>> 16))
Step 3: Calculate bucket index: hash & (capacity - 1)
         e.g., 3029967 & 15 = index 15
Step 4: If bucket empty → insert
         If not empty → check equals() on existing keys
         If key found → update value
         If not found → add to linked list (collision)
```

**3. GET operation:**
```
hashMap.get("name")

Step 1: Compute hashCode("name") → same index 15
Step 2: Go to bucket 15
Step 3: Walk the linked list, check equals() for each key
Step 4: Return matching value
```

**4. Resize (rehashing):**
When `size > capacity × loadFactor (0.75)`, HashMap doubles in size and rehashes all entries.

```java
Map<String, Integer> map = new HashMap<>(16, 0.75f);
// Default: capacity=16, loadFactor=0.75 → resizes at 12 entries

map.put("apple", 1);
map.put("banana", 2);

// Two keys collide if: key1.hashCode() == key2.hashCode() && !key1.equals(key2)
// Java 8+: if bucket size > 8 → converts linked list to Red-Black Tree (O(log n))
```

⚠️ **Interview Trap:**
> "What if hashCode() always returns the same value?"

All keys hash to bucket 0. The entire map becomes one giant linked list. `get()` degrades to O(n). That's why a good `hashCode()` distribution matters.

---

## ConcurrentHashMap

### ❓ Why use ConcurrentHashMap over HashMap?

- `HashMap` is **not thread-safe**. Concurrent modifications can cause infinite loops (Java 7) or data corruption.
- `Hashtable` is thread-safe but uses a single lock on the whole map — very slow.
- `ConcurrentHashMap` is thread-safe using **segment-level locking** (Java 7) or **CAS + bin-level locking** (Java 8+).

```java
// Thread-safe without locking the whole map
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.putIfAbsent("b", 2);  // atomic operation
map.computeIfAbsent("c", k -> expensiveComputation(k)); // atomic
```

---

## Fail-Fast vs Fail-Safe Iterators

| Feature | Fail-Fast | Fail-Safe |
|---|---|---|
| Collections | ArrayList, HashMap | CopyOnWriteArrayList, ConcurrentHashMap |
| On modification | Throws `ConcurrentModificationException` | No exception |
| Works on | Original collection | Copy of collection |
| Memory | Less | More (maintains copy) |

```java
// Fail-fast — throws ConcurrentModificationException
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    if (s.equals("b")) list.remove(s); // THROWS ConcurrentModificationException
}

// Fix 1 — use Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove(); // safe
}

// Fix 2 — use removeIf (Java 8+)
list.removeIf(s -> s.equals("b")); // safe

// Fail-safe
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>(List.of("a","b","c"));
for (String s : cowList) {
    cowList.remove(s); // no exception — iterates over a snapshot
}
```

---

## TreeMap vs HashMap

| Feature | HashMap | TreeMap |
|---|---|---|
| Order | No order | Sorted by natural order or Comparator |
| Performance (get/put) | O(1) average | O(log n) |
| Null keys | 1 null key allowed | NO null keys |
| Internal structure | Hash table | Red-black tree |
| Use when | Fast lookup | Need sorted keys |

```java
TreeMap<String, Integer> tree = new TreeMap<>();
tree.put("banana", 2);
tree.put("apple", 1);
tree.put("cherry", 3);
System.out.println(tree); // {apple=1, banana=2, cherry=3} — sorted!

tree.firstKey(); // "apple"
tree.lastKey();  // "cherry"
tree.headMap("cherry"); // {apple=1, banana=2}
```

---

## When to Use Which Collection

```
Need to store elements in order with duplicates?       → ArrayList
Need fast insert/delete at head/tail?                  → LinkedList / ArrayDeque
Need uniqueness, don't care about order?               → HashSet
Need uniqueness + sorted order?                        → TreeSet
Need uniqueness + insertion order?                     → LinkedHashSet
Need key-value pairs, fast lookup?                     → HashMap
Need key-value sorted by key?                          → TreeMap
Need key-value + insertion order?                      → LinkedHashMap
Need thread-safe key-value?                            → ConcurrentHashMap
Need thread-safe list with many reads, few writes?     → CopyOnWriteArrayList
Need a priority queue (min/max heap)?                  → PriorityQueue
```

---

# 3. Multithreading & Concurrency

---

## Thread Lifecycle

```
NEW → RUNNABLE → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED
```

| State | When |
|---|---|
| NEW | Thread created, not yet started |
| RUNNABLE | Running or ready to run |
| BLOCKED | Waiting to acquire a lock |
| WAITING | Waiting indefinitely (wait(), join()) |
| TIMED_WAITING | Waiting for a time (sleep(), wait(timeout)) |
| TERMINATED | Finished execution |

---

## synchronized vs Lock

```java
// synchronized — simple, automatic release
class Counter {
    private int count = 0;
    public synchronized void increment() { count++; }
}

// Lock — more control: tryLock, timeout, fairness
class Counter {
    private int count = 0;
    private final Lock lock = new ReentrantLock(true); // fair lock

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock(); // MUST unlock in finally
        }
    }

    public boolean tryIncrement() {
        if (lock.tryLock()) { // non-blocking attempt
            try { count++; return true; }
            finally { lock.unlock(); }
        }
        return false;
    }
}
```

---

## volatile Keyword

### ❓ What does volatile do?

✅ `volatile` ensures a variable is **read from and written to main memory directly**, not from a thread's local CPU cache.

```java
class StopTask implements Runnable {
    private volatile boolean running = true; // without volatile, loop may never stop

    public void run() {
        while (running) { /* do work */ }
    }

    public void stop() { running = false; }
}
```

⚠️ **Interview Trap:**
> "Does volatile make operations atomic?"

NO. `volatile` ensures visibility. It does NOT guarantee atomicity. `count++` is NOT atomic even with `volatile`. Use `AtomicInteger` for that.

---

## Race Condition

```java
// BROKEN — multiple threads read-increment-write concurrently
class BrokenCounter {
    int count = 0;
    void increment() { count++; } // read-modify-write: NOT atomic!
}

// FIX 1 — synchronized
synchronized void increment() { count++; }

// FIX 2 — AtomicInteger (preferred — lock-free)
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```

---

## ExecutorService

```java
// DON'T create raw threads — use ExecutorService
ExecutorService pool = Executors.newFixedThreadPool(4);

pool.submit(() -> System.out.println("Task 1"));
pool.submit(() -> System.out.println("Task 2"));

pool.shutdown(); // graceful shutdown — waits for tasks
pool.shutdownNow(); // immediate — interrupts running tasks

// With result
Future<Integer> future = pool.submit(() -> 2 + 2);
Integer result = future.get(); // blocks until done
```

---

## Future vs CompletableFuture

| Feature | Future | CompletableFuture |
|---|---|---|
| Blocking | `get()` blocks | Can be non-blocking with callbacks |
| Chaining | Not possible | `.thenApply()`, `.thenCompose()` |
| Exception handling | Try-catch on get() | `.exceptionally()` |
| Combining | Not possible | `.allOf()`, `.anyOf()` |

```java
// CompletableFuture — non-blocking pipeline
CompletableFuture.supplyAsync(() -> fetchUser(1))       // async fetch
    .thenApply(user -> user.getName().toUpperCase())     // transform
    .thenAccept(name -> System.out.println("User: " + name)) // consume
    .exceptionally(ex -> { System.out.println("Error: " + ex); return null; });

// Combining two futures
CompletableFuture<User> userFuture  = CompletableFuture.supplyAsync(() -> fetchUser(1));
CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(() -> fetchOrder(1));

CompletableFuture.allOf(userFuture, orderFuture)
    .thenRun(() -> {
        User u = userFuture.join();
        Order o = orderFuture.join();
        System.out.println(u.getName() + " → " + o.getTotal());
    });
```

---

## Deadlock

### ❓ What is deadlock? Give an example.

✅ Deadlock occurs when two or more threads are waiting for each other to release locks — **forever**.

```java
Object lockA = new Object();
Object lockB = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lockA) {
        System.out.println("T1 holds lockA, waiting for lockB");
        synchronized (lockB) { System.out.println("T1 holds both"); }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lockB) {
        System.out.println("T2 holds lockB, waiting for lockA");
        synchronized (lockA) { System.out.println("T2 holds both"); }
    }
});

t1.start(); t2.start(); // DEADLOCK
```

✅ **Prevention strategies:**
1. **Lock ordering** — always acquire locks in the same order
2. **tryLock with timeout** — `lock.tryLock(1, TimeUnit.SECONDS)`
3. **Avoid nested locks**
4. **Use higher-level concurrency utilities** (`ConcurrentHashMap`, `Semaphore`)

---

# 4. Java 8+ Features

---

## Lambda Expressions

```java
// Before Java 8 — anonymous class
Runnable r = new Runnable() {
    @Override public void run() { System.out.println("Running"); }
};

// Java 8 Lambda
Runnable r = () -> System.out.println("Running");

// With parameter
Comparator<String> comp = (a, b) -> a.compareTo(b);

// With block body
Comparator<Integer> byLength = (a, b) -> {
    int diff = a - b;
    return diff;
};
```

---

## Functional Interfaces

| Interface | Method | Use case |
|---|---|---|
| `Predicate<T>` | `boolean test(T t)` | Filter — is this true? |
| `Function<T,R>` | `R apply(T t)` | Transform — convert T to R |
| `Consumer<T>` | `void accept(T t)` | Use value, no return |
| `Supplier<T>` | `T get()` | Provide a value, no input |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two inputs, one output |

```java
Predicate<String> isLong = s -> s.length() > 5;
isLong.test("hi");      // false
isLong.test("hello world"); // true

Function<String, Integer> length = String::length;
length.apply("hello"); // 5

Consumer<String> print = System.out::println;
print.accept("Hello"); // prints Hello

Supplier<List<String>> listFactory = ArrayList::new;
List<String> list = listFactory.get(); // new ArrayList each time
```

---

## Streams API

```java
List<Employee> employees = List.of(
    new Employee("TJ",   85000, "IT"),
    new Employee("Anna", 92000, "IT"),
    new Employee("Bob",  70000, "HR"),
    new Employee("Lisa", 95000, "IT")
);

// Filter IT employees earning over 80k, sorted by salary desc, get names
List<String> result = employees.stream()
    .filter(e -> e.getDept().equals("IT"))          // intermediate
    .filter(e -> e.getSalary() > 80000)              // intermediate
    .sorted(Comparator.comparingDouble(Employee::getSalary).reversed()) // intermediate
    .map(Employee::getName)                           // intermediate
    .collect(Collectors.toList());                    // terminal
// ["Lisa", "Anna", "TJ"]

// Aggregation
double avgSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .average()
    .orElse(0.0);

// Grouping
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept));

// Counting by dept
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));
```

---

## map() vs flatMap()

```java
List<List<Integer>> nested = List.of(
    List.of(1, 2, 3),
    List.of(4, 5),
    List.of(6, 7, 8)
);

// map — one stream element → one output element
List<Integer> sizes = nested.stream()
    .map(List::size)          // [3, 2, 3]
    .collect(Collectors.toList());

// flatMap — one element → zero or more elements (flattens)
List<Integer> flat = nested.stream()
    .flatMap(Collection::stream)  // [1, 2, 3, 4, 5, 6, 7, 8]
    .collect(Collectors.toList());

// Real use case — get all unique skills across all employees
List<String> allSkills = employees.stream()
    .flatMap(e -> e.getSkills().stream())
    .distinct()
    .collect(Collectors.toList());
```

---

## Optional

```java
// Instead of returning null — return Optional
Optional<User> findUser(Long id) {
    return userRepository.findById(id); // may be empty
}

// Consuming Optional safely
Optional<User> userOpt = findUser(1L);

// BAD — defeats the purpose
User user = userOpt.get(); // throws NoSuchElementException if empty!

// GOOD patterns
userOpt.ifPresent(u -> System.out.println(u.getName()));

String name = userOpt
    .map(User::getName)
    .orElse("Unknown");

User user = userOpt
    .orElseThrow(() -> new UserNotFoundException("User not found"));

// Chaining
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("No city");
```

---

## Method References

```java
// 4 types of method references

// 1. Static method reference
Function<String, Integer> parseInt = Integer::parseInt;

// 2. Instance method on a specific object
String prefix = "Hello";
Predicate<String> startsWithHello = prefix::startsWith; // broken example for clarity
// Better: Predicate<String> hasHello = "Hello World"::startsWith;

// 3. Instance method on arbitrary object of type
Function<String, String> upper = String::toUpperCase;
List<String> result = names.stream().map(String::toUpperCase).collect(toList());

// 4. Constructor reference
Supplier<ArrayList> listMaker = ArrayList::new;
Function<String, User> userMaker = User::new;
```

---

# 5. JVM Internals

---

## JVM Architecture

```
┌──────────────────────────────────────────────┐
│                  JVM                          │
│                                              │
│  ┌─────────────┐  ┌──────────────────────┐  │
│  │ Class Loader│  │   Runtime Data Areas  │  │
│  │  Subsystem  │  │  ┌────────┬────────┐ │  │
│  │             │  │  │  Heap  │ Stack  │ │  │
│  │  Bootstrap  │  │  ├────────┴────────┤ │  │
│  │  Extension  │  │  │ Method Area     │ │  │
│  │  App Loader │  │  │ PC Register     │ │  │
│  └─────────────┘  │  │ Native Stack    │ │  │
│                   │  └─────────────────┘ │  │
│                   └──────────────────────┘  │
│  ┌─────────────────────────────────────────┐ │
│  │         Execution Engine                 │ │
│  │  Interpreter | JIT Compiler | GC        │ │
│  └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

---

## Heap vs Stack

| Feature | Heap | Stack |
|---|---|---|
| Stores | Objects, instance variables | Method frames, local variables, references |
| Size | Large, configurable | Small, fixed |
| Shared | Yes — all threads | No — each thread has its own |
| Managed by | GC | Auto pop when method returns |
| Error | OutOfMemoryError | StackOverflowError |

```java
void method() {
    int x = 5;                  // Stack — primitive
    String s = "hello";          // Stack — reference; "hello" in String pool (Heap)
    Employee e = new Employee(); // Stack — reference 'e'; Employee object on Heap
}
// when method() returns, x, s, e are popped from stack
// Employee object on heap eventually GC'd (when no references)
```

---

## Garbage Collection

✅ **How it works:**
1. **Mark** — GC marks all live objects (reachable from GC roots)
2. **Sweep** — removes unmarked (unreachable) objects
3. **Compact** — moves surviving objects together to eliminate fragmentation

✅ **Generations:**
- **Young Gen (Eden + Survivor)** — most objects live and die here (Minor GC)
- **Old Gen (Tenured)** — long-lived objects (Major/Full GC)
- **Metaspace** — class metadata (Java 8+ replaced PermGen)

✅ **GC Algorithms:**
- `G1GC` — default Java 9+, good for low-pause applications
- `ZGC` — ultra-low pause, Java 15+
- `Shenandoah` — concurrent, OpenJDK

---

## Memory Leaks in Java

Common causes of memory leaks:
```java
// 1. Static collections holding references
static Map<String, Object> cache = new HashMap<>();
// If you keep adding to this and never remove, it grows forever

// 2. Listeners not removed
button.addActionListener(listener); // must removeActionListener when done

// 3. Unclosed resources
InputStream stream = new FileInputStream("file.txt");
// If exception thrown before close() → stream never released
// FIX: Use try-with-resources
try (InputStream stream = new FileInputStream("file.txt")) {
    // auto-closed
}

// 4. ThreadLocal not removed
ThreadLocal<Object> tl = new ThreadLocal<>();
tl.set(new HeavyObject());
// MUST call tl.remove() when done, especially in thread pools!
```

---

# 6. Exception Handling

---

## Checked vs Unchecked Exceptions

| Feature | Checked | Unchecked |
|---|---|---|
| Extends | `Exception` | `RuntimeException` |
| Must handle? | YES — compile error if not | NO — optional |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `ArrayIndexOutOfBoundsException` |
| When to use | Recoverable conditions | Programming errors |

```java
// Checked — caller MUST handle
public void readFile(String path) throws IOException { // declare it
    Files.readAllBytes(Path.of(path));
}
// Caller:
try {
    readFile("data.txt");
} catch (IOException e) {
    System.out.println("File not found: " + e.getMessage());
}

// Unchecked — no obligation to handle
public void divide(int a, int b) {
    if (b == 0) throw new ArithmeticException("Division by zero"); // unchecked
    return a / b;
}
```

---

## throw vs throws

```java
// throw — actually throws an exception object
throw new IllegalArgumentException("Invalid input");

// throws — declares that a method MAY throw this exception
public void connect(String url) throws IOException, SQLException {
    // ...
}
```

---

## try-catch-finally Deep Explanation

⚠️ **Interview Trap — return in finally:**
```java
int getNumber() {
    try {
        return 1;
    } finally {
        return 2; // finally overrides the try return!
    }
}
// Returns: 2
```

⚠️ **Interview Trap — exception in finally:**
```java
void method() {
    try {
        throw new RuntimeException("from try");
    } finally {
        throw new RuntimeException("from finally"); // SWALLOWS the try exception!
    }
}
```

```java
// try-with-resources — auto closes
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    // both conn and ps closed automatically — even if exception thrown
    ps.executeQuery();
}
```

---

## Custom Exceptions

```java
// Checked custom exception
public class InsufficientFundsException extends Exception {
    private double amount;

    public InsufficientFundsException(double amount) {
        super("Insufficient funds. Short by: " + amount);
        this.amount = amount;
    }

    public double getAmount() { return amount; }
}

// Unchecked custom exception
public class DeviceNotFoundException extends RuntimeException {
    public DeviceNotFoundException(Long id) {
        super("Device not found with ID: " + id);
    }
}

// Usage in Spring Boot
@GetMapping("/devices/{id}")
public Device getDevice(@PathVariable Long id) {
    return deviceRepo.findById(id)
        .orElseThrow(() -> new DeviceNotFoundException(id));
}
```

---

# 7. Spring Boot

---

## Dependency Injection

### ❓ What is DI and how does Spring implement it?

✅ DI is when you **don't create dependencies yourself** — Spring creates and injects them.

```java
// BAD — tight coupling, hard to test
class OrderService {
    PaymentService paymentService = new PaymentService(); // you create it
}

// GOOD — constructor injection (preferred)
@Service
class OrderService {
    private final PaymentService paymentService;

    @Autowired // optional in Spring Boot when single constructor
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

✅ **Constructor injection is preferred over `@Autowired` field injection because:**
1. Dependencies are explicit and required
2. Makes testing easy (just pass a mock)
3. Immutable (`final`)
4. No `NullPointerException` if Spring wiring fails

---

## Bean Lifecycle

```
Container started
      ↓
Bean instantiated (constructor)
      ↓
Dependencies injected
      ↓
@PostConstruct method called
      ↓
Bean ready to use
      ↓
@PreDestroy method called (on shutdown)
      ↓
Bean destroyed
```

```java
@Component
class DatabaseInitializer {

    @PostConstruct
    public void init() {
        System.out.println("DB initializer running after injection");
        // seed data, verify connection, etc.
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Closing resources before shutdown");
    }
}
```

---

## @Component vs @Service vs @Repository vs @Controller

They are all specializations of `@Component`. Spring treats them the same for bean detection, but:

| Annotation | Layer | Extra behavior |
|---|---|---|
| `@Component` | Generic | None |
| `@Service` | Business logic | None (semantic marker) |
| `@Repository` | Data access | Translates DB exceptions → `DataAccessException` |
| `@Controller` | Web MVC | Handles HTTP requests |
| `@RestController` | REST API | `@Controller` + `@ResponseBody` |

---

## REST API Best Practices

```java
@RestController
@RequestMapping("/api/v1/devices")
@RequiredArgsConstructor
public class DeviceController {

    private final DeviceService deviceService;

    @GetMapping                        // GET /api/v1/devices
    public ResponseEntity<List<DeviceDTO>> getAll() {
        return ResponseEntity.ok(deviceService.findAll());
    }

    @GetMapping("/{id}")               // GET /api/v1/devices/1
    public ResponseEntity<DeviceDTO> getById(@PathVariable Long id) {
        return ResponseEntity.ok(deviceService.findById(id));
    }

    @PostMapping                       // POST /api/v1/devices
    public ResponseEntity<DeviceDTO> create(@RequestBody @Valid CreateDeviceRequest req) {
        DeviceDTO created = deviceService.create(req);
        URI location = URI.create("/api/v1/devices/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")               // PUT /api/v1/devices/1
    public ResponseEntity<DeviceDTO> update(
            @PathVariable Long id,
            @RequestBody @Valid UpdateDeviceRequest req) {
        return ResponseEntity.ok(deviceService.update(id, req));
    }

    @DeleteMapping("/{id}")            // DELETE /api/v1/devices/1
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        deviceService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## Global Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(DeviceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(DeviceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(404, ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(400, message));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(500, "Something went wrong"));
    }
}

record ErrorResponse(int status, String message) {}
```

---

## DTO vs Entity

```java
// Entity — mapped to DB table
@Entity
@Table(name = "users")
class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    private String passwordHash; // NEVER expose this in API!
    @CreationTimestamp
    private LocalDateTime createdAt;
}

// DTO — what you expose in the API
record UserDTO(Long id, String name, String email) {}

// Request DTO — what you accept
record CreateUserRequest(
    @NotBlank String name,
    @Email String email,
    @Size(min = 8) String password
) {}

// Mapper using MapStruct (best practice)
@Mapper(componentModel = "spring")
interface UserMapper {
    UserDTO toDTO(User user);
    User toEntity(CreateUserRequest req);
}
```

---

# 8. System Design Basics

---

## Design Scalable APIs

```
Key principles:
1. Stateless — each request contains all information needed
2. Pagination — never return unbounded lists
3. Versioning — /api/v1/ allows evolution without breaking clients
4. Idempotency — PUT/DELETE should be safe to retry
5. Rate limiting — protect from abuse
6. Caching — reduce load on DB

Pagination example:
GET /api/v1/devices?page=0&size=20&sort=createdAt,desc

Response:
{
  "content": [...],
  "page": 0,
  "size": 20,
  "totalElements": 1500,
  "totalPages": 75
}
```

---

## Caching

```java
// Spring Cache — simple annotation-based caching
@Service
public class ProductService {

    @Cacheable("products")  // cache result keyed by id
    public Product findById(Long id) {
        return productRepo.findById(id).orElseThrow(); // only hits DB on cache miss
    }

    @CacheEvict("products") // remove from cache when updated
    public Product update(Long id, UpdateRequest req) {
        Product p = findById(id);
        p.setName(req.getName());
        return productRepo.save(p);
    }

    @CachePut(value = "products", key = "#result.id") // update cache entry
    public Product create(CreateRequest req) {
        return productRepo.save(mapper.toEntity(req));
    }
}
```

✅ **Cache strategies:**
- **Cache-aside (Lazy)** — check cache → miss → load DB → write to cache
- **Write-through** — write to cache AND DB at same time
- **Write-behind** — write to cache immediately, async write to DB
- **TTL (Time-to-live)** — expire entries after N seconds

---

## DB Indexing Basics

```sql
-- Without index: full table scan O(n)
SELECT * FROM orders WHERE customer_id = 100;

-- With index: B-tree lookup O(log n)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- Composite index — order matters! Index on (a, b) helps:
-- WHERE a = ?
-- WHERE a = ? AND b = ?
-- But NOT: WHERE b = ?  (prefix rule)
CREATE INDEX idx_name_dept ON employees(department_id, name);

-- Covering index — query served entirely from index
CREATE INDEX idx_covering ON orders(customer_id, order_date, total);
SELECT order_date, total FROM orders WHERE customer_id = 100;
-- ↑ No heap access needed — all data in index
```

✅ **When NOT to index:**
- Small tables (full scan is faster)
- Columns with very low cardinality (e.g., `gender` with 2 values)
- Columns that are updated very frequently

---

# 9. Real Interview Coding Problems

---

## Problem 1: Two Sum

**❓ Given an array and a target, return indices of two numbers that add to target.**

```java
// Brute force O(n²) — two nested loops
// Optimal: HashMap O(n) — store complement
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>(); // value → index

    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (seen.containsKey(complement)) {
            return new int[]{seen.get(complement), i};
        }
        seen.put(nums[i], i);
    }
    return new int[]{};
}
// [2,7,11,15], target=9 → [0,1]  (2+7=9)
```

---

## Problem 2: Find Duplicates in Array

```java
// Approach: HashSet — O(n) time, O(n) space
public List<Integer> findDuplicates(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    List<Integer> duplicates = new ArrayList<>();

    for (int n : nums) {
        if (!seen.add(n)) { // add returns false if already present
            duplicates.add(n);
        }
    }
    return duplicates;
}
```

---

## Problem 3: Reverse a String Without reverse()

```java
public String reverse(String s) {
    char[] chars = s.toCharArray();
    int left = 0, right = chars.length - 1;

    while (left < right) {
        char temp = chars[left];
        chars[left++] = chars[right];
        chars[right--] = temp;
    }
    return new String(chars);
}
```

---

## Problem 4: Check Anagram

```java
public boolean isAnagram(String s1, String s2) {
    if (s1.length() != s2.length()) return false;

    int[] freq = new int[26];
    for (char c : s1.toCharArray()) freq[c - 'a']++;
    for (char c : s2.toCharArray()) freq[c - 'a']--;

    for (int f : freq) if (f != 0) return false;
    return true;
}
// "listen", "silent" → true
```

---

## Problem 5: Find First Non-Repeating Character

```java
public char firstUnique(String s) {
    Map<Character, Integer> freq = new LinkedHashMap<>(); // preserve insertion order

    for (char c : s.toCharArray()) {
        freq.merge(c, 1, Integer::sum);
    }

    for (Map.Entry<Character, Integer> entry : freq.entrySet()) {
        if (entry.getValue() == 1) return entry.getKey();
    }
    return '_'; // no unique char
}
// "aabcdb" → 'c'
```

---

## Problem 6: Fibonacci (iterative + memoized)

```java
// Iterative O(n) time, O(1) space
public long fib(int n) {
    if (n <= 1) return n;
    long prev = 0, curr = 1;
    for (int i = 2; i <= n; i++) {
        long next = prev + curr;
        prev = curr;
        curr = next;
    }
    return curr;
}

// Memoized O(n) time, O(n) space
private Map<Integer, Long> memo = new HashMap<>();
public long fibMemo(int n) {
    if (n <= 1) return n;
    if (memo.containsKey(n)) return memo.get(n);
    long result = fibMemo(n - 1) + fibMemo(n - 2);
    memo.put(n, result);
    return result;
}
```

---

## Problem 7: Group Anagrams Together

```java
public List<List<String>> groupAnagrams(String[] words) {
    Map<String, List<String>> groups = new HashMap<>();

    for (String word : words) {
        char[] chars = word.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars); // sorted version as key

        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(word);
    }

    return new ArrayList<>(groups.values());
}
// ["eat","tea","tan","ate","nat","bat"]
// → [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

---

## Problem 8: Valid Parentheses

```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');

    for (char c : s.toCharArray()) {
        if ("([{".indexOf(c) >= 0) {
            stack.push(c);
        } else {
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
                return false;
            }
        }
    }
    return stack.isEmpty();
}
// "{[]}" → true,  "{[}" → false
```

---

## Problem 9: Find the Missing Number

```java
// Array contains 0..n with one missing
public int missingNumber(int[] nums) {
    int n = nums.length;
    int expected = n * (n + 1) / 2; // sum formula
    int actual = 0;
    for (int num : nums) actual += num;
    return expected - actual;
}
// [3,0,1] → 2
```

---

## Problem 10: LRU Cache Implementation

```java
class LRUCache {
    private final int capacity;
    private final Map<Integer, Integer> cache; // LinkedHashMap maintains insertion order

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {
            // removeEldestEntry called after each put
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;
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

# 10. Interview Traps & FAQs

---

## Most Common Mistakes

### Trap 1: String comparison with ==
```java
String s1 = new String("hello");
String s2 = new String("hello");
s1 == s2    // false — different objects
s1.equals(s2) // true — ALWAYS use equals() for String comparison
```

### Trap 2: Integer cache boundary
```java
Integer a = 127; Integer b = 127; a == b // true (cached)
Integer c = 128; Integer d = 128; c == d // false (not cached)
// Always use .equals() for Integer comparison
```

### Trap 3: Modifying list while iterating
```java
// Throws ConcurrentModificationException
for (String s : list) { if (s.equals("x")) list.remove(s); }
// Fix: use iterator.remove() or removeIf()
list.removeIf(s -> s.equals("x"));
```

### Trap 4: NullPointerException from unboxing
```java
Map<String, Integer> map = new HashMap<>();
int value = map.get("missing"); // NPE! get() returns null, unboxing null crashes
int value = map.getOrDefault("missing", 0); // safe
```

### Trap 5: Static method "overriding"
```java
class Parent { static void hello() { System.out.println("Parent"); } }
class Child extends Parent { static void hello() { System.out.println("Child"); } }

Parent p = new Child();
p.hello(); // prints "Parent" — static methods are resolved at compile time, NOT overridden
```

### Trap 6: Mutable objects as HashMap keys
```java
// DON'T use mutable objects as keys — if the object changes, you can never find it again
List<String> key = new ArrayList<>(List.of("a", "b"));
Map<List<String>, String> map = new HashMap<>();
map.put(key, "value");
key.add("c"); // mutates the key — hashCode changes
map.get(key); // returns null! key no longer in correct bucket
```

### Trap 7: Thread safety of StringBuilder vs StringBuffer
```java
// StringBuilder — NOT thread safe, faster
// StringBuffer — thread safe (synchronized methods), slower
// In single-threaded code (almost always) use StringBuilder
StringBuilder sb = new StringBuilder();
// In Streams: use Collectors.joining() instead of manual appending
String result = words.stream().collect(Collectors.joining(", "));
```

### Trap 8: @Transactional on private methods
```java
@Service
class OrderService {
    @Transactional
    private void saveOrder(Order o) { // DOES NOTHING — Spring proxy can't intercept private
        orderRepo.save(o);
    }
}
// @Transactional only works on public methods called through Spring proxy
```

### Trap 9: Self-invocation with @Transactional
```java
@Service
class OrderService {
    @Transactional
    public void placeOrder(Order o) {
        this.chargePayment(o); // proxy NOT invoked — self-call bypasses Spring AOP!
    }

    @Transactional(propagation = REQUIRES_NEW) // IGNORED for self-call
    public void chargePayment(Order o) { ... }
}
```

### Trap 10: Checked exceptions and @Transactional rollback
```java
// @Transactional only rolls back on RuntimeException by default!
@Transactional
public void transfer() throws IOException {
    accountRepo.debit(from);
    throw new IOException("Network failed"); // COMMITS anyway! Not rolled back!
}

// Fix:
@Transactional(rollbackFor = Exception.class) // rollback on ALL exceptions
public void transfer() throws IOException { ... }
```

---

## Top Follow-Up Questions to Expect

1. "What happens if you put `null` as a key in HashMap?" → One null key allowed; stored in bucket 0
2. "Can we make a thread safe Singleton?" → Yes — `enum` singleton or double-checked locking with `volatile`
3. "What is the difference between `wait()` and `sleep()`?" → `wait()` releases the lock; `sleep()` does NOT
4. "What happens when main thread throws an exception?" → JVM prints stack trace and terminates
5. "How does `String.intern()` work?" → Adds string to pool, or returns existing pool reference
6. "Can you catch an `Error`?" → Yes technically, but you shouldn't. `OutOfMemoryError`, `StackOverflowError` are fatal
7. "What is the difference between `Comparable` and `Comparator`?" → `Comparable` is natural order (in the class); `Comparator` is external/custom order

---

# 11. Cheat Sheet

---

## Key Difference Tables

### Collections Quick Reference
| | `ArrayList` | `LinkedList` | `HashSet` | `TreeSet` | `HashMap` | `TreeMap` |
|---|---|---|---|---|---|---|
| Order | Insertion | Insertion | None | Sorted | None | Sorted |
| Duplicates | Yes | Yes | No | No | Keys: No | Keys: No |
| get(i) | O(1) | O(n) | — | — | O(1) | O(log n) |
| add | O(1)* | O(1) | O(1) | O(log n) | O(1) | O(log n) |
| Null | Yes | Yes | 1 null | No | 1 null key | No |

### Exception Hierarchy
```
Throwable
├── Error (don't catch — fatal)
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception
    ├── Checked (must handle)
    │   ├── IOException
    │   └── SQLException
    └── RuntimeException (unchecked)
        ├── NullPointerException
        ├── ArrayIndexOutOfBoundsException
        ├── IllegalArgumentException
        └── ClassCastException
```

### Thread States
```
new Thread()  →  start()  →  RUNNABLE  →  run() completes  →  TERMINATED
                                ↕
                         BLOCKED (waiting for lock)
                         WAITING  (wait(), join())
                         TIMED_WAITING (sleep(n), wait(n))
```

### Access Modifiers
| Modifier | Same class | Same package | Subclass | Anywhere |
|---|---|---|---|---|
| `private` | ✅ | ❌ | ❌ | ❌ |
| (default) | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

### Java 8 Functional Interfaces
| Interface | Signature | Use |
|---|---|---|
| `Predicate<T>` | `T → boolean` | filter |
| `Function<T,R>` | `T → R` | map/transform |
| `Consumer<T>` | `T → void` | forEach |
| `Supplier<T>` | `() → T` | factory/lazy init |
| `UnaryOperator<T>` | `T → T` | replaceAll |
| `BinaryOperator<T>` | `(T,T) → T` | reduce |

---

## Interview One-Liners to Remember

| Question | Answer |
|---|---|
| Why is String immutable? | Security, String pool reuse, thread safety, hashCode caching |
| HashMap default capacity? | 16, load factor 0.75, resizes at 12 entries |
| HashMap Java 8 change? | Bucket converts to Red-Black Tree when chain > 8 |
| Can abstract class have constructor? | Yes — called by subclass via `super()` |
| Can interface be instantiated? | No — but can be used as anonymous class / lambda |
| What does `volatile` guarantee? | Visibility — NOT atomicity |
| `wait()` vs `sleep()`? | `wait()` releases lock, needs `notify()`; `sleep()` holds lock |
| `==` for Strings? | Compares references — always use `.equals()` |
| SOLID D = ? | Depend on abstractions, not concretions |
| What triggers GC? | JVM decides; `System.gc()` only suggests |
| fail-fast vs fail-safe? | fail-fast throws CME; fail-safe iterates on copy |
| `Comparable` vs `Comparator`? | Comparable = natural order inside class; Comparator = external, flexible |
| `final` class example? | `String`, `Integer`, `Math` |
| When to use `TreeMap`? | When you need sorted keys |
| Default fetch type for `@ManyToOne`? | `EAGER` |
| Default fetch type for `@OneToMany`? | `LAZY` |

---

## Common Stream Operations Cheat Sheet

```java
stream()                          // create stream from collection
filter(predicate)                 // keep matching elements
map(function)                     // transform each element
flatMap(function)                 // flatten nested streams
sorted()                          // natural order sort
sorted(comparator)                // custom sort
distinct()                        // remove duplicates
limit(n)                          // take first n
skip(n)                           // skip first n
peek(consumer)                    // debug — see element without changing stream
reduce(identity, accumulator)     // fold to single value
collect(Collectors.toList())      // to list
collect(Collectors.toSet())       // to set
collect(Collectors.groupingBy(f)) // group by key
collect(Collectors.joining(", ")) // concat strings
count()                           // number of elements
anyMatch(predicate)               // any element matches?
allMatch(predicate)               // all elements match?
noneMatch(predicate)              // no element matches?
findFirst()                       // first element (Optional)
min(comparator)                   // minimum (Optional)
max(comparator)                   // maximum (Optional)
forEach(consumer)                 // terminal — iterate
```

---

*This guide covers everything from beginner fundamentals to advanced interview scenarios. Focus on understanding the WHY behind each concept — interviewers value depth of reasoning over memorized answers.*
