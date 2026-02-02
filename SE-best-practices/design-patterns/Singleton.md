## CREATIONAL PATTERNS

### **Singleton Pattern (From Junior → Senior → Architect)**

**Reference anchor (conceptual):**

* Refactoring Guru

---

## 1. Why Singleton Exists (Not the Definition Yet)

### Real-world requirement

> “We need a **single configuration manager** that loads app config once and is accessible everywhere.”

Key words:

* single
* shared
* expensive to create
* consistent state

This is where Singleton is **tempting**.

---

## 2. The Naive Approach (What Most Juniors Do)

```java
public class AppConfig {

    public AppConfig() {
        loadFromFile();
    }
}
```

Used like:

```java
AppConfig config = new AppConfig();
```

### Problem (real-world)

* Every `new` reloads config
* Different parts may see different state
* Startup cost repeated
* Bugs appear under load

---

## 3. First “Fix” (Still Wrong)

```java
public class AppConfig {
    public static AppConfig INSTANCE = new AppConfig();

    private AppConfig() {
        loadFromFile();
    }
}
```

Used as:

```java
AppConfig.INSTANCE.get("db.url");
```

### Why this *seems* fine

* Only one instance
* Simple
* Works in single-thread demos

### Why seniors are uneasy

* Eager initialization
* No lazy loading
* Hard to test
* Hard dependency (global state)

This is **where interviews stop**.
But **production does not**.

---

## 4. What Singleton *Really* Solves

> Singleton is about **controlling instance count**,
> **not about global access**.

Global access is a **side effect**, not the goal.

---

## 5. Correct Mental Model (Important)

Ask **three questions** before Singleton:

1. Is **exactly one instance required**?
2. Is creation **expensive or stateful**?
3. Is the lifecycle **application-wide**?

If any answer is “no” → **don’t use Singleton**.

---

## 6. The Classic Lazy Singleton (Broken in Production)

```java
public class Logger {

    private static Logger instance;

    private Logger() {}

    public static Logger getInstance() {
        if (instance == null) {
            instance = new Logger();
        }
        return instance;
    }
}
```

### Fatal problem

❌ **Not thread-safe**

In real servers:

* Two threads enter `getInstance`
* Two instances created
* Bug appears once in a million requests
* Nightmare to debug

---

## 7. Thread-Safe Singleton (Synchronized)

```java
public class Logger {

    private static Logger instance;

    private Logger() {}

    public static synchronized Logger getInstance() {
        if (instance == null) {
            instance = new Logger();
        }
        return instance;
    }
}
```

### Works, but…

* Synchronization cost on every call
* Performance hit
* Not ideal under high throughput

---

## 8. Double-Checked Locking (Correct, But Subtle)

```java
public class Logger {

    private static volatile Logger instance;

    private Logger() {}

    public static Logger getInstance() {
        if (instance == null) {
            synchronized (Logger.class) {
                if (instance == null) {
                    instance = new Logger();
                }
            }
        }
        return instance;
    }
}
```

### Why `volatile` matters (senior detail)

* Prevents instruction reordering
* Ensures fully constructed object visibility
* Without it → **partially constructed object bug**

This is **interview gold**.

---

## 9. Best Singleton in Java (Senior Choice)

```java
public class Logger {

    private Logger() {}

    private static class Holder {
        private static final Logger INSTANCE = new Logger();
    }

    public static Logger getInstance() {
        return Holder.INSTANCE;
    }
}
```

### Why this is excellent

* Lazy-loaded
* Thread-safe
* No synchronization overhead
* JVM guarantees safety

This is **production-grade Singleton**.

---

## 10. Enum Singleton (Most Robust)

```java
public enum AppConfig {
    INSTANCE;

    public String get(String key) {
        return "value";
    }
}
```

### Why seniors love this

* Thread-safe by JVM
* Serialization-safe
* Reflection-safe
* Simple

### When NOT to use

* When lazy initialization is required
* When you need inheritance (rare)

---

## 11. Anime Analogy — *One Punch Man (Saitama)*

Saitama:

* Exists only once
* Overpowered
* Centralized strength

Imagine:

* Multiple Saitamas
* Different power levels
* Chaos

Singleton ensures:

> “There is only one Saitama.”

---

## 12. Real-World Use Cases (Correct)

✅ Logging
✅ Configuration
✅ Metrics registry
✅ Cache manager

❌ Database connection (use pool)
❌ Business services
❌ Domain objects

---

## 13. The Dark Side of Singleton (Senior Warnings)

### 1. Hidden Dependencies

* Looks innocent
* Actually global state

### 2. Testing Hell

* State leaks between tests
* Requires reset hacks

### 3. Violates DIP if abused

* Business logic depends on concrete singleton

### 4. Often replaces proper DI

> **Singleton is often a smell disguised as convenience.**

---

## 14. Singleton vs Dependency Injection (Important)

| Singleton               | DI                    |
| ----------------------- | --------------------- |
| Controls instance count | Controls wiring       |
| Often global            | Explicit dependencies |
| Harder to test          | Test-friendly         |

**Senior rule:**

> Prefer DI-managed singletons over manual ones.

(Spring singletons ≠ Singleton pattern)

---

## 15. Interview Traps (Be Ready)

**Q:** Is Singleton an anti-pattern?
**A:** It becomes one when used as global mutable state.

**Q:** Is Spring `@Singleton` same as Singleton pattern?
**A:** No. Scope ≠ design pattern.

**Q:** Can Singleton break SOLID?
**A:** Yes — SRP, DIP, testability.

---

## 16. One-Line Memory Anchor

> **Singleton controls existence, not access.**


<details>
  <summary>
    Doubts
  </summary>


# 1️⃣ `readResolve()` — Why It Exists and When You Need It

### The real problem it solves (not the definition)

**Serialization can break Singleton.**

Example:

```java
Logger s1 = Logger.getInstance();

// serialize s1 to file
// deserialize from file

Logger s2 = deserializedObject;

System.out.println(s1 == s2); // ❌ false
```

Why?

* Deserialization **creates a new instance**
* JVM does NOT call your constructor
* Singleton guarantee is violated

---

## How `readResolve()` fixes this

```java
private Object readResolve() {
    return getInstance();
}
```

### What actually happens in JVM

1. JVM deserializes object
2. JVM sees `readResolve()`
3. JVM **replaces** the deserialized object with returned object
4. Garbage-collects the new instance

So:

```java
s1 == s2 // ✅ true
```

---

## When you NEED `readResolve()`

| Singleton Type         | Needs `readResolve()`    |
| ---------------------- | ------------------------ |
| Classic Singleton      | ✅ YES                    |
| Double-checked locking | ✅ YES                    |
| Static inner class     | ✅ YES                    |
| Enum Singleton         | ❌ NO (JVM guarantees it) |

**Enum singletons are serialization-safe by design.**

---

## Senior rule

> If your Singleton is `Serializable` and not an enum,
> **you MUST implement `readResolve()`**.

This is a **common interview trap**.

---

# 2️⃣ Why the Inner Holder Class Is `private`

You asked:

> “Why not public instead of private static inner class?”

### Short answer

**Because it leaks implementation details and breaks encapsulation.**

---

## Correct implementation

```java
public class Logger {

    private Logger() {}

    private static class Holder {
        private static final Logger INSTANCE = new Logger();
    }

    public static Logger getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## What happens if `Holder` is `public`

```java
public static class Holder {
    public static final Logger INSTANCE = new Logger();
}
```

### Problems introduced

1. External code can access:

   ```java
   Logger.Holder.INSTANCE
   ```

   → bypassing your API

2. You lose control over:

   * initialization strategy
   * lazy loading contract
   * future refactoring

3. You expose **how** Singleton works, not **what** it provides

---

## Senior design principle involved

This is **encapsulation**, not Singleton itself.

> Public classes define contracts.
> Private classes define implementation.

The holder is an **implementation detail**.

---

# 3️⃣ Can Static Inner Class Singleton Be Broken by Reflection?

### Yes.

**Any non-enum Singleton can be broken by reflection**.

Example:

```java
Constructor<Logger> c =
    Logger.class.getDeclaredConstructor();

c.setAccessible(true);

Logger s1 = Logger.getInstance();
Logger s2 = c.newInstance();

System.out.println(s1 == s2); // ❌ false
```

---

## How seniors mitigate this (if they must)

### Defensive constructor

```java
private static boolean initialized = false;

private Logger() {
    if (initialized) {
        throw new RuntimeException("Use getInstance()");
    }
    initialized = true;
}
```

### But…

* Still hackable via deep reflection
* Still not bulletproof

---

## The ONLY reflection-proof Singleton

```java
public enum Logger {
    INSTANCE;
}
```

Why?

* JVM **forbids reflective enum instantiation**
* Even `setAccessible(true)` fails
* Serialization also safe

---

## Senior rule (important)

> If reflection/serialization safety matters → **use enum Singleton**
> Otherwise → static holder is acceptable

---

# 4️⃣ Is Static Inner Class Singleton Eager or Lazy?

This is where **many explanations are wrong**.

### ❌ Common myth

> Static inner class causes eager loading

### ✅ Reality (JVM spec)

* **Outer class loading does NOT load inner classes**
* Inner class loads **only when referenced**

In this line:

```java
return Holder.INSTANCE;
```

➡️ `Holder` is loaded **at that moment**, not before.

---

## Memory impact (actual behavior)

| Approach           | Memory allocated when? |
| ------------------ | ---------------------- |
| Eager Singleton    | Class load time        |
| Static inner class | First access           |
| Enum Singleton     | Class load time        |

So:

* **Static holder = lazy**
* **No wasted memory**
* **No premature initialization**

---

## Senior nuance

Yes, the **class metadata** exists, but:

* No instance created
* No heavy object allocation
* Negligible overhead

This is NOT a practical concern.

---

# 5️⃣ Putting It All Together (Decision Table)

| Requirement                                  | Best Choice                    |
| -------------------------------------------- | ------------------------------ |
| Absolute safety (reflection + serialization) | Enum Singleton                 |
| Lazy + performant                            | Static inner class             |
| Legacy codebase                              | Double-checked locking         |
| Spring / DI framework                        | Let container manage singleton |
| Business logic                               | ❌ Avoid Singleton              |

---

# 6️⃣ The Real Senior Insight (Most Important)

> Singleton is NOT about “one object”.
> It’s about **controlling lifecycle**.

And most of the time:

* DI containers do this better
* Singleton pattern becomes unnecessary

---

## Final One-Liners (Memorize)

* `readResolve()` → **serialization safety**
* `private static inner class` → **lazy + encapsulated**
* Reflection breaks most Singletons → **enum is safest**
* Static holder is NOT eager → **lazy by JVM guarantee**
* Singleton ≠ good design by default
</details>
