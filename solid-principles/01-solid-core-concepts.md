# SOLID Principles — Core Concepts (Senior Java Developer Guide)

SOLID is an acronym for five design principles that make software easier to maintain, extend, and understand. They were introduced by Robert C. Martin (Uncle Bob) and are the foundation of good object-oriented design.

---

## S — Single Responsibility Principle (SRP)

> **"A class should have only one reason to change."**

Every class should do exactly one thing and own exactly one part of the system's responsibility. "One reason to change" means one stakeholder, one concern, one axis of variation.

### What it looks like violated:

```java
// This class has FOUR reasons to change:
// 1. Business rules change (how invoice is calculated)
// 2. Output format changes (how it's printed)
// 3. Persistence changes (how it's saved to DB)
// 4. Email rules change (when/how it's sent)
public class Invoice {
    private Order order;
    private double taxRate;

    public double calculateTotal() {
        return order.getAmount() + (order.getAmount() * taxRate);
    }

    public void printInvoice() {
        System.out.println("Invoice for: " + order.getCustomer());
        System.out.println("Total: " + calculateTotal());
        // formatting logic embedded here
    }

    public void saveToDatabase() {
        // SQL logic embedded in a business object
        String sql = "INSERT INTO invoices VALUES (...)";
        // execute sql...
    }

    public void sendByEmail() {
        // SMTP logic embedded here
        // email formatting, connection, retry logic...
    }
}
```

### Correctly split by responsibility:

```java
// 1. Business logic only
public class Invoice {
    private Order order;
    private double taxRate;

    public double calculateTotal() {
        return order.getAmount() * (1 + taxRate);
    }

    public String getCustomerName() { return order.getCustomer(); }
}

// 2. Presentation only
public class InvoicePrinter {
    public void print(Invoice invoice) {
        System.out.println("Invoice for: " + invoice.getCustomerName());
        System.out.println("Total: " + invoice.calculateTotal());
    }
}

// 3. Persistence only
public class InvoiceRepository {
    public void save(Invoice invoice) {
        // SQL logic lives here and only here
    }
}

// 4. Notification only
public class InvoiceEmailSender {
    public void send(Invoice invoice, String recipientEmail) {
        // SMTP logic lives here and only here
    }
}
```

### The subtle SRP trap — "cohesion" doesn't mean "one method":

SRP is NOT about having one method per class. A class can have many methods if they all serve the same single responsibility. `Invoice` above has multiple methods — all of them are about invoice business logic, so SRP is satisfied.

---

## O — Open/Closed Principle (OCP)

> **"Software entities should be open for extension, but closed for modification."**

You should be able to add new behavior without changing existing, tested code. New features are added by writing new classes, not by editing old ones.

### What it looks like violated:

```java
// Every new shape requires modifying this class — fragile, risky
public class AreaCalculator {
    public double calculate(Object shape) {
        if (shape instanceof Circle) {
            Circle c = (Circle) shape;
            return Math.PI * c.getRadius() * c.getRadius();
        } else if (shape instanceof Rectangle) {
            Rectangle r = (Rectangle) shape;
            return r.getWidth() * r.getHeight();
        } else if (shape instanceof Triangle) {
            // add another branch every time a new shape appears
        }
        throw new IllegalArgumentException("Unknown shape");
    }
}
```

### Correctly closed for modification, open for extension:

```java
// Define the extension point — this never changes
public interface Shape {
    double area();
}

// Add new shapes by writing new classes — existing code untouched
public class Circle implements Shape {
    private double radius;
    @Override
    public double area() { return Math.PI * radius * radius; }
}

public class Rectangle implements Shape {
    private double width, height;
    @Override
    public double area() { return width * height; }
}

// Triangle added later — zero changes to existing classes
public class Triangle implements Shape {
    private double base, height;
    @Override
    public double area() { return 0.5 * base * height; }
}

// Calculator never needs to change again
public class AreaCalculator {
    public double calculate(Shape shape) {
        return shape.area(); // polymorphism does the work
    }

    public double calculateAll(List<Shape> shapes) {
        return shapes.stream().mapToDouble(Shape::area).sum();
    }
}
```

### OCP with Strategy Pattern — runtime extension:

```java
// Discount calculation that grows without modification
public interface DiscountStrategy {
    double apply(double price);
}

public class NoDiscount       implements DiscountStrategy { public double apply(double p) { return p; } }
public class PercentDiscount  implements DiscountStrategy {
    private double percent;
    public double apply(double p) { return p * (1 - percent / 100); }
}
public class SeasonalDiscount implements DiscountStrategy { ... } // new — no existing code touched

public class PriceCalculator {
    private DiscountStrategy strategy;

    public PriceCalculator(DiscountStrategy strategy) { this.strategy = strategy; }

    public double finalPrice(double price) { return strategy.apply(price); }
}
```

---

## L — Liskov Substitution Principle (LSP)

> **"Objects of a subclass should be substitutable for objects of the parent class without altering the correctness of the program."**

If you have code that works with a `Parent`, it must work correctly with any `Child` — without knowing which one it has. Any violation means your inheritance hierarchy is wrong.

### What it looks like violated:

```java
public class Bird {
    public void fly() { System.out.println("Flying"); }
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!");
    }
}

// Caller written against Bird's contract:
public void makeBirdFly(Bird bird) {
    bird.fly(); // throws for Penguin — LSP violated
}
```

### The fix — restructure the hierarchy to reflect reality:

```java
public abstract class Bird {
    public abstract void eat();
    public abstract void move();
}

public interface Flyable {
    void fly();
}

public class Sparrow extends Bird implements Flyable {
    public void eat()  { System.out.println("Sparrow eating"); }
    public void move() { fly(); }
    public void fly()  { System.out.println("Sparrow flying"); }
}

public class Penguin extends Bird {
    public void eat()  { System.out.println("Penguin eating"); }
    public void move() { System.out.println("Penguin swimming"); }
    // No fly() — penguins don't claim to fly
}

// Code that needs flying only works with Flyable:
public void makeFly(Flyable f) { f.fly(); } // Penguin can never be passed here
```

### LSP rules recap for seniors:

| Rule | Meaning |
|---|---|
| **Preconditions** | Subclass must accept at least what parent accepts — cannot demand stricter input |
| **Postconditions** | Subclass must deliver at least what parent promises — cannot return less |
| **Invariants** | Subclass must preserve all parent invariants |
| **Exceptions** | Subclass cannot throw new checked exceptions not declared in parent |
| **History constraint** | Subclass cannot allow state changes the parent would prohibit |

---

## I — Interface Segregation Principle (ISP)

> **"Clients should not be forced to depend on interfaces they do not use."**

Fat interfaces force implementors to provide empty or broken implementations of methods they don't need. Split large interfaces into smaller, focused ones.

### What it looks like violated:

```java
// One fat interface forces every implementor to deal with everything
public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeReport();
}

// A robot can work but doesn't eat, sleep, attend meetings, or write reports
public class Robot implements Worker {
    public void work() { System.out.println("Robot working"); }

    public void eat()           { throw new UnsupportedOperationException(); }
    public void sleep()         { throw new UnsupportedOperationException(); }
    public void attendMeeting() { throw new UnsupportedOperationException(); }
    public void writeReport()   { throw new UnsupportedOperationException(); }
    // Forced to stub 4 methods that make no sense — ISP violated
}
```

### Correctly segregated:

```java
public interface Workable    { void work(); }
public interface Feedable    { void eat(); }
public interface Restable    { void sleep(); }
public interface Reportable  { void writeReport(); }
public interface MeetingGoer { void attendMeeting(); }

// Human implements what makes sense for a human
public class Human implements Workable, Feedable, Restable, MeetingGoer, Reportable {
    public void work()          { System.out.println("Human working"); }
    public void eat()           { System.out.println("Human eating"); }
    public void sleep()         { System.out.println("Human sleeping"); }
    public void attendMeeting() { System.out.println("In meeting"); }
    public void writeReport()   { System.out.println("Writing report"); }
}

// Robot only implements what makes sense for a robot
public class Robot implements Workable {
    public void work() { System.out.println("Robot working"); }
    // No forced stubs, no UnsupportedOperationExceptions
}
```

### ISP with Java 8 default methods — the nuance:

```java
// Default methods can provide optional behavior without forcing all implementors
public interface Exportable {
    byte[] toBytes(); // required

    default String toBase64() { // optional — implementors can override if needed
        return Base64.getEncoder().encodeToString(toBytes());
    }

    default void exportToFile(String path) { // optional convenience
        Files.write(Paths.get(path), toBytes());
    }
}
// Implementors only override what they need to change
```

---

## D — Dependency Inversion Principle (DIP)

> **"High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions."**

The high-level "what" should not be coupled to the low-level "how". Depend on interfaces, not concrete classes.

### What it looks like violated:

```java
// High-level module (OrderService) directly depends on low-level module (MySQLOrderRepository)
public class OrderService {
    private MySQLOrderRepository repository; // concrete dependency — tightly coupled

    public OrderService() {
        this.repository = new MySQLOrderRepository(); // creates its own dependency
    }

    public void placeOrder(Order order) {
        // business logic...
        repository.save(order); // if we ever switch to PostgreSQL or MongoDB — this class changes
    }
}

public class MySQLOrderRepository {
    public void save(Order order) { /* MySQL-specific SQL */ }
}
```

**Problems:**
- Can't test `OrderService` without a real MySQL database
- Switching to PostgreSQL requires modifying `OrderService`
- High-level business logic is coupled to low-level infrastructure

### Correctly inverted:

```java
// Abstraction — neither high-level nor low-level owns this, both depend on it
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(long id);
}

// Low-level detail — depends on the abstraction
public class MySQLOrderRepository implements OrderRepository {
    @Override
    public void save(Order order) { /* MySQL SQL */ }
    @Override
    public Optional<Order> findById(long id) { /* MySQL query */ }
}

// Easy to add new implementations — no existing code changes
public class MongoOrderRepository implements OrderRepository {
    @Override
    public void save(Order order) { /* MongoDB driver */ }
    @Override
    public Optional<Order> findById(long id) { /* MongoDB query */ }
}

// In-memory implementation for tests — no database needed
public class InMemoryOrderRepository implements OrderRepository {
    private Map<Long, Order> store = new HashMap<>();
    @Override
    public void save(Order order) { store.put(order.getId(), order); }
    @Override
    public Optional<Order> findById(long id) { return Optional.ofNullable(store.get(id)); }
}

// High-level module — depends on abstraction, never on concrete classes
public class OrderService {
    private final OrderRepository repository; // interface, not a class

    // Dependency injected from outside — not created here
    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

    public void placeOrder(Order order) {
        validate(order);
        repository.save(order); // works with MySQL, Mongo, or in-memory — doesn't care
    }
}

// Wiring happens at the composition root (main, Spring config, etc.)
OrderService service = new OrderService(new MySQLOrderRepository()); // production
OrderService testService = new OrderService(new InMemoryOrderRepository()); // test
```

### DIP + Spring (real-world):

```java
@Service
public class OrderService {
    private final OrderRepository repository;

    @Autowired // Spring injects the concrete implementation
    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}

@Repository
public class MySQLOrderRepository implements OrderRepository {
    // Spring registers this as the concrete bean
}
```

---

## How the Principles Relate

```
SRP  → Each class has one job
 └─► OCP  → Add new jobs via new classes, not editing old ones
      └─► LSP  → New classes must honor existing contracts
           └─► ISP  → Contracts stay small and focused
                └─► DIP  → Classes depend on those contracts, not each other directly
```

They reinforce each other. Violate one and you almost always end up violating others.

---

## The One-Line Test for Each Principle

| Principle | Ask yourself |
|---|---|
| **SRP** | "What is the ONE reason this class would change?" If you think of two — split it. |
| **OCP** | "To add this new feature, do I have to edit existing classes?" If yes — redesign. |
| **LSP** | "If I swap this for its parent type, does everything still work?" If no — fix the hierarchy. |
| **ISP** | "Does this implementor have any method that makes no sense for it?" If yes — split the interface. |
| **DIP** | "Does this class `new` up its own dependencies or import concrete infrastructure classes?" If yes — inject instead. |
