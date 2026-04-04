# The Diamond Problem in Java

The **diamond problem** is a classic ambiguity that arises in object-oriented programming when a class inherits from two classes that share a common ancestor — creating a diamond-shaped inheritance hierarchy.

```
      A
     / \
    B   C
     \ /
      D
```

## Why It's a Problem

If both `B` and `C` override a method from `A`, and `D` inherits from both, which version does `D` get? This ambiguity is the diamond problem.

---

## Java's Solution: No Multiple Class Inheritance

Java **prevents the diamond problem** with classes by simply **disallowing multiple inheritance**. A class can only extend one other class.

```java
class A { }
class B extends A { }
class C extends A { }

// ❌ COMPILE ERROR — Java doesn't allow this
class D extends B, C { }
```

---

## The Problem Re-emerges with Default Interface Methods (Java 8+)

Java 8 introduced **default methods** in interfaces, which brought the diamond problem back through interfaces.

### Example 1 — Classic Diamond with Interfaces

```java
interface A {
    default void hello() {
        System.out.println("Hello from A");
    }
}

interface B extends A {
    default void hello() {
        System.out.println("Hello from B");
    }
}

interface C extends A {
    default void hello() {
        System.out.println("Hello from C");
    }
}

// ❌ COMPILE ERROR: D inherits unrelated defaults from B and C
class D implements B, C { }
```

The compiler throws:
```
error: class D inherits unrelated defaults for hello() from types B and C
```

### ✅ Fix: Override in the implementing class

```java
class D implements B, C {
    @Override
    public void hello() {
        System.out.println("Hello from D");  // D decides
    }
}
```

---

### Example 2 — Explicitly Calling a Specific Interface's Default

You can choose *which* parent's default to use with the `InterfaceName.super.method()` syntax:

```java
class D implements B, C {
    @Override
    public void hello() {
        B.super.hello();  // Explicitly delegates to B's version
    }
}

public class Main {
    public static void main(String[] args) {
        new D().hello();  // Output: Hello from B
    }
}
```

---

### Example 3 — No Conflict When Only One Interface Overrides

If only one of the two interfaces overrides the method, Java resolves it automatically:

```java
interface A {
    default void greet() {
        System.out.println("Hello from A");
    }
}

interface B extends A {
    default void greet() {
        System.out.println("Hello from B");  // B overrides A
    }
}

interface C extends A {
    // C does NOT override greet() — inherits A's version
}

class D implements B, C { }  // ✅ No conflict — B's version wins (more specific)

public class Main {
    public static void main(String[] args) {
        new D().greet();  // Output: Hello from B
    }
}
```

**Rule:** The **most specific** override wins automatically.

---

### Example 4 — Class Always Wins Over Interface

If the implementing class's own superclass already defines the method, the **class takes priority** over any interface default:

```java
interface A {
    default void speak() {
        System.out.println("Interface A speaking");
    }
}

class Base {
    public void speak() {
        System.out.println("Class Base speaking");
    }
}

class Child extends Base implements A { }  // ✅ No conflict

public class Main {
    public static void main(String[] args) {
        new Child().speak();  // Output: Class Base speaking
    }
}
```

---

## Java's 3 Rules for Resolving Diamond Conflicts

| Priority | Rule |
|---|---|
| **1st** | Classes/superclasses always win over interface defaults |
| **2nd** | The most specific (child) interface wins over a less specific (parent) one |
| **3rd** | If still ambiguous → **must override manually** (compile error otherwise) |

---

## Summary

| Scenario | Java's Behavior |
|---|---|
| Multiple class inheritance | ❌ Not allowed |
| Two interfaces with the same default | ⚠️ Must override manually |
| One interface overrides, one doesn't | ✅ More specific one wins automatically |
| Superclass method vs interface default | ✅ Superclass always wins |
| Manual override with `Interface.super.method()` | ✅ You pick which default to use |

The diamond problem is elegantly handled in Java — it disallows the worst case (multiple class inheritance) and gives you clear, explicit tools to resolve ambiguity at the interface level.
