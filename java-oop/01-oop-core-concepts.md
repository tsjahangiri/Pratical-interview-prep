# OOP Core Concepts — Java Senior Developer Guide

---

## The 4 Pillars

---

### 1. Encapsulation

**The concept:** Bundling data and the methods that operate on it, while restricting direct access to internal state.

**Junior answer:** "Use private fields with getters/setters."

**Senior answer:** Encapsulation is about *invariant protection*. A class owns its state and guarantees its consistency. Getters/setters are often a sign of *broken* encapsulation — you're just making private fields public with extra steps.

```java
// BAD — this is not encapsulation, it's just syntax
public class BankAccount {
    private double balance;
    public void setBalance(double balance) { this.balance = balance; } // anyone can set anything
}

// GOOD — the class protects its own invariant
public class BankAccount {
    private double balance;

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        this.balance -= amount;
    }

    public double getBalance() { return balance; } // read-only, no setter
}
```

**Senior trap:** Returning a mutable object from a getter breaks encapsulation:

```java
public class Schedule {
    private List<String> appointments = new ArrayList<>();

    public List<String> getAppointments() {
        return appointments; // BROKEN — caller can call .clear() and destroy state
    }

    // Fix:
    public List<String> getAppointmentsSafe() {
        return Collections.unmodifiableList(appointments);
    }
}
```

---

### 2. Inheritance

**The concept:** A subclass acquires the properties and behaviors of a parent class.

**Senior reality:** Inheritance is the most *misused* OOP feature. Prefer composition. Inheritance only makes sense when the IS-A relationship is permanent and the subclass truly extends (not restricts) the parent.

**The Liskov Substitution Principle (LSP):**

> A subclass must be substitutable for its parent without breaking the program.

```java
// Classic LSP violation — Rectangle/Square problem
public class Rectangle {
    protected int width, height;
    public void setWidth(int w)  { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w)  { this.width = w; this.height = w; }
    @Override
    public void setHeight(int h) { this.width = h; this.height = h; }
}

Rectangle r = new Square();
r.setWidth(5);
r.setHeight(3);
System.out.println(r.area()); // Expected: 15 → Actual: 9 — LSP violated
```

---

### 3. Polymorphism

**Two types seniors must distinguish:**

| Type | Resolved at | Mechanism |
|---|---|---|
| **Runtime (dynamic)** | Runtime | Method overriding, virtual dispatch |
| **Compile-time (static)** | Compile time | Method overloading |

```java
// Fields are NEVER polymorphic — only methods are
public class Parent { public String name = "Parent"; }
public class Child extends Parent { public String name = "Child"; }

Parent p = new Child();
System.out.println(p.name);       // "Parent" — field, compile-time resolution
System.out.println(p.getName());  // "Child"  — method, runtime dispatch
```

---

### 4. Abstraction

**Abstract class vs Interface — the senior distinction:**

| | Abstract Class | Interface |
|---|---|---|
| State | Can have instance fields | No instance state |
| Constructor | Yes | No |
| Use when | Sharing implementation + IS-A | Defining a capability/contract |
| Multiple inheritance | No | Yes |

**Diamond problem with default methods (Java 8+):**

```java
interface A { default String hello() { return "Hello from A"; } }
interface B extends A { default String hello() { return "Hello from B"; } }

class C implements A, B {} // COMPILE ERROR — must resolve

class C implements A, B {
    @Override
    public String hello() { return B.super.hello(); } // explicit resolution required
}
```

---

## Immutable Class Checklist

A senior must know all six steps:

1. Declare class `final`
2. All fields `private final`
3. No setters
4. Defensive copies in constructor for mutable inputs
5. Defensive copies in getters for mutable fields
6. No methods that modify state
