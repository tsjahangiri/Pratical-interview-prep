# OOP Bugs, Traps & Senior Interview Questions

---

## Bug 1 — equals() Without hashCode()

```java
public class Employee {
    private int id;
    private String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee)) return false;
        Employee e = (Employee) o;
        return id == e.id && name.equals(e.name);
    }
    // hashCode() NOT overridden — BUG
}

Set<Employee> set = new HashSet<>();
Employee e1 = new Employee(1, "Alice");
Employee e2 = new Employee(1, "Alice");

set.add(e1);
System.out.println(set.contains(e2)); // FALSE — but they're "equal"!
set.add(e2);
System.out.println(set.size()); // 2 — duplicates in a Set!
```

**The contract:** If `a.equals(b)` then `a.hashCode() == b.hashCode()` must be true.
**Fix:** Always override both. Use `Objects.hash(id, name)` in Java 11+.

---

## Bug 2 — Mutable Object as Map Key

```java
public class Config {
    public String key; // mutable public field

    @Override public int hashCode() { return key.hashCode(); }
    @Override public boolean equals(Object o) { ... }
}

Map<Config, String> map = new HashMap<>();
Config c = new Config("db.url");
map.put(c, "jdbc:postgresql://...");

c.key = "changed"; // mutate after insertion

System.out.println(map.get(c)); // NULL — entry is lost in wrong bucket forever
```

**Rule:** Keys in hash-based collections must be *effectively immutable*.

---

## Bug 3 — Calling Overridable Method from Constructor

```java
public class Base {
    public Base() {
        init(); // calls overridden version before child fields are initialized
    }
    public void init() { System.out.println("Base init"); }
}

public class Child extends Base {
    private String value = "hello";

    @Override
    public void init() {
        System.out.println(value.toUpperCase()); // NullPointerException!
        // value is null — field initializers haven't run yet
    }
}
```

**Rule:** Never call non-final, non-private methods from constructors.

---

## Bug 4 — instanceof Abuse (Open/Closed Violation)

```java
// Grows into a maintenance nightmare as types increase
public double calculateBonus(Employee emp) {
    if (emp instanceof Manager)  return ((Manager) emp).getSalary() * 0.2;
    if (emp instanceof Engineer) return ((Engineer) emp).getSalary() * 0.15;
    if (emp instanceof Intern)   return 500;
    throw new IllegalArgumentException("Unknown type");
}

// Fix — push behavior into the type:
public abstract class Employee {
    public abstract double calculateBonus(); // each type owns its own logic
}
```

---

## Bug 5 — Broken Comparator Contract

```java
// This comparator has a subtle bug:
Comparator<Integer> comp = (a, b) -> {
    if (a > b) return 1;
    return -1; // never returns 0!
};

// TreeMap uses compareTo == 0 to determine key equality
// With this comparator, TreeMap will LOSE entries silently
```

**The contract:** A comparator must return 0 for equal elements.

---

## Inheritance-Specific Traps

---

### Trap 1 — Narrowing a Postcondition (LSP Violation)

```java
public class FileReader {
    // POSTCONDITION: never returns null
    public List<String> readLines(String path) {
        return new ArrayList<>(); // always valid
    }
}

public class CachedFileReader extends FileReader {
    private Map<String, List<String>> cache = new HashMap<>();

    @Override
    public List<String> readLines(String path) {
        return cache.get(path); // returns null on cache miss — NARROWS postcondition
    }
}

FileReader reader = new CachedFileReader();
for (String line : reader.readLines("config.txt")) { // NullPointerException!
}

// Fix:
return cache.getOrDefault(path, Collections.emptyList());
```

---

### Trap 2 — Widening a Precondition (LSP Violation)

```java
public class PaymentProcessor {
    // PRECONDITION: amount > 0
    public void processPayment(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Must be positive");
    }
}

public class PremiumPaymentProcessor extends PaymentProcessor {
    @Override
    public void processPayment(double amount) {
        if (amount < 100) throw new IllegalArgumentException("Minimum is 100"); // WIDENS
        super.processPayment(amount);
    }
}

PaymentProcessor p = new PremiumPaymentProcessor();
p.processPayment(50.0); // parent said this was fine — now it throws
```

---

### Trap 3 — Throwing New Checked Exceptions

```java
// Compiler catches direct violations:
public class Sub extends Base {
    @Override
    public void generate() throws SQLException { } // COMPILE ERROR if parent doesn't declare it
}

// The sneaky version — wrapping as unchecked to bypass compiler:
@Override
public void generate() {
    try {
        connectToDb();
    } catch (SQLException e) {
        throw new RuntimeException(e); // compiles, but surprises callers at runtime
    }
}
```

---

### Trap 4 — super.method() Required for Correctness (Fragile Base Class)

```java
public class SessionManager {
    private boolean initialized = false;

    public void initialize() { this.initialized = true; } // must be called

    public void doWork() {
        if (!initialized) throw new IllegalStateException("Not initialized!");
    }
}

public class SecureSessionManager extends SessionManager {
    @Override
    public void initialize() {
        setupSecurity(); // forgot super.initialize() — no compiler warning
    }
}

new SecureSessionManager().initialize();
new SecureSessionManager().doWork(); // IllegalStateException — silent failure

// Fix: Template Method Pattern
public abstract class SessionManager {
    public final void initialize() { // final — can't be bypassed
        performCoreInit(); // base class handles required state
        onInitialize();    // hook for subclasses
    }
    private void performCoreInit() { this.initialized = true; }
    protected void onInitialize() { } // override this, not initialize()
}
```

---

### Trap 5 — Inheritance for Code Reuse (Not IS-A)

```java
// Wrong: UserRepository IS-NOT-A ArrayList
public class UserRepository extends ArrayList<User> {
    public User findByEmail(String email) { ... }
    // Now callers can call .remove(0), .trimToSize(), .set() — wrong abstraction
}

// Correct: Composition
public class UserRepository {
    private final List<User> users = new ArrayList<>();
    public void add(User user) { users.add(user); }
    public User findByEmail(String email) { ... }
    // Only expose what makes sense for a repository
}
```

---

### Trap 6 — Fragile Base Class (Internal Implementation Dependency)

```java
public class Counter {
    private int count = 0;
    public void increment() { count++; }
    public void incrementByN(int n) {
        for (int i = 0; i < n; i++) increment(); // calls own method
    }
}

public class LoggingCounter extends Counter {
    private int logged = 0;
    @Override
    public void increment() { super.increment(); logged++; }
}

// Works now. But if parent "optimizes":
public void incrementByN(int n) { count += n; } // no longer calls increment()

// LoggingCounter silently breaks — logged stays 0
```

---

### Trap 7 — Constructor and Inheritance Ordering

```java
// Java constructor execution order:
// 1. super() runs first (always)
// 2. Child field initializers run
// 3. Child constructor body runs

public class Dog extends Animal {
    private String breed = "Labrador";

    @Override
    public String getType() {
        return "Dog:" + breed; // breed is NULL when called from Animal()!
    }
}
// Animal() calls getType() → Dog.getType() runs → breed is null → "Dog:null"
```

---

### Trap 8 — Static Method Hiding vs Instance Overriding

```java
public class Vehicle {
    public static String getCategory() { return "Vehicle"; }
    public        String getType()     { return "Vehicle"; }
}

public class Car extends Vehicle {
    public static String getCategory() { return "Car"; } // HIDES, not overrides
    @Override
    public        String getType()     { return "Car"; } // OVERRIDES
}

Vehicle v = new Car();
System.out.println(v.getCategory()); // "Vehicle" — static: compile-time resolution
System.out.println(v.getType());     // "Car"     — instance: runtime dispatch
```

**Static methods belong to the class, not the object. You cannot override them.**

---

### Trap 9 — Overloading Across Inheritance

```java
public class Printer {
    public void print(Object o) { System.out.println("Object: " + o); }
    public void print(String s) { System.out.println("String: " + s); }
}

public class FancyPrinter extends Printer {
    @Override
    public void print(Object o) { System.out.println("Fancy Object: " + o); }
}

Printer p = new FancyPrinter();
p.print("hello");         // "String: hello"       — overload picked at compile time
p.print((Object)"hello"); // "Fancy Object: hello" — cast forces Object overload
```

Overloading resolution is **compile-time** by reference type.
Overriding dispatch is **runtime** by actual object type.

---

## Senior Interview Questions

**Design questions:**
- "When would you choose composition over inheritance? Give a real example."
- "What's the difference between an interface and abstract class in Java 11+?"
- "How does Java implement polymorphism under the hood?" (vtable / virtual dispatch)
- "What does it mean for a class to be immutable? List all the steps."

**Code review questions:**
- "Review this class — what invariants could be violated?"
- "This equals() method was written by a junior — what's wrong?"
- "Why might extending this class in the future break existing code?"

**Deep knowledge questions:**
- "Explain the difference between `==` and `equals()` for Strings — and why `String.intern()` changes `==` behavior."
- "What is the fragile base class problem and how do you defend against it?"
- "What does `final` do on a class, method, and field — why are they conceptually different?"

---

## The Senior Mindset

> A senior developer doesn't just make code *work* — they design classes so that **incorrect use is impossible or immediately obvious**, and so that adding new features doesn't require touching existing, tested code.
