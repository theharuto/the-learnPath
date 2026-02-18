## 1. **What is `WeakHashMap`? Internal Working, Why Use It, Differences**

### ✅ What it is

`WeakHashMap<K,V>` is a `Map` where **keys are held using `WeakReference`**.

From JDK docs:

> Entries are removed automatically when keys are no longer strongly reachable.

So:
If **no strong reference exists to a key**, GC can remove it → entry disappears.

---

### ✅ Internal Working

From OpenJDK source + docs:

* Internally:

  ```java
  static class Entry extends WeakReference<Object>
  ```
* Each key is wrapped in `WeakReference`
* A `ReferenceQueue` is used

Flow:

1. You put `(K,V)`
2. Key → wrapped in `WeakReference`
3. If no strong reference to `K` exists
4. GC collects `K`
5. WeakReference enqueued in `ReferenceQueue`
6. `WeakHashMap` cleans entry on next access

⚠️ Cleanup happens **lazily**, not immediately.

---

### ✅ Why Use It (Main Use Case)

**Automatic cache / metadata map**

Example:

```java
Map<Class<?>, Metadata> cache = new WeakHashMap<>();
```

When `Class` unloads → entry disappears.

Used when:

* You want **memory-sensitive caches**
* You don’t control key lifecycle
* You want GC-driven eviction

---

### ✅ Differences vs Other Maps

| Map               | Key Reference | GC Aware | Thread Safe | Use Case        |
| ----------------- | ------------- | -------- | ----------- | --------------- |
| HashMap           | Strong        | ❌        | ❌           | General         |
| WeakHashMap       | Weak          | ✅        | ❌           | Cache           |
| IdentityHashMap   | ==            | ❌        | ❌           | Reference-based |
| ConcurrentHashMap | Strong        | ❌        | ✅           | Concurrency     |

---

### ⚠️ Interview Traps

❌ Trap 1:

```java
map.put(new String("A"), 1);
```

Entry may disappear immediately.

Because no strong reference.

❌ Trap 2: Not deterministic
GC timing ≠ predictable.

❌ Trap 3: Not a replacement for LRU cache.

---

### ✅ Methods

Same as `HashMap` (inherits `AbstractMap`):
`put, get, remove, entrySet, size`

No special APIs.

---

### 🧠 Rule of Thumb

> Use `WeakHashMap` only when key lifecycle is controlled by GC.

Otherwise → dangerous.

---

## 2. **Maven: How to Remove Transitive Dependency**

### ✅ Correct Way: `<exclusions>`

From Maven POM reference.

Example:

```xml
<dependency>
  <groupId>A</groupId>
  <artifactId>B</artifactId>
  <version>1.0</version>

  <exclusions>
    <exclusion>
      <groupId>X</groupId>
      <artifactId>Y</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

This blocks `X:Y` coming via `B`.

---

### ✅ Find Transitives First

```bash
mvn dependency:tree
```

Mandatory before exclusion.

---

### ⚠️ Common Mistakes

❌ Excluding wrong artifact
❌ Excluding in wrong dependency
❌ Assuming exclusion is global (it’s local)

---

### 🧠 Production Tip

If many conflicts → use `<dependencyManagement>`.

---

## 3. **Nested try-catch: Inner Doesn’t Catch Exception**

### Example

```java
try {
  try {
    risky();
  } catch (IOException e) {
  }
} catch (Exception e) {
}
```

If `risky()` throws `SQLException`.

---

### ✅ What Happens

From JLS §14.20:

1. Exception thrown
2. Nearest matching `catch` searched
3. Inner catch doesn’t match
4. Propagates outward
5. Outer catch handles it

So → **outer try-catch handles it**

If no outer → thread terminates.

---

### ⚠️ Trap

If `finally` throws → original exception lost.

```java
finally {
   throw new RuntimeException();
}
```

Overrides original.

---

### 🧠 Rule

> Exceptions bubble up until a compatible catch is found.

---

## 4. **`finalize()` — What Is It? Who Calls It?**

### ✅ What it is

```java
protected void finalize() throws Throwable
```

Called by GC **before reclaiming object**.

---

### ✅ Who Calls It?

**Garbage Collector Thread**

Not user thread.

Not deterministic.

---

### ✅ Status (Important!)

Since Java 9:

> Deprecated.

Java 18+: disabled by default.

Official docs:

> Finalization is inherently unreliable.

---

### ⚠️ Problems

| Issue        | Why               |
| ------------ | ----------------- |
| No guarantee | May never run     |
| Performance  | GC slowdown       |
| Resurrection | Object revived    |
| Deadlocks    | Runs in GC thread |

---

### ✅ Correct Replacement

Use:

* `try-with-resources`
* `AutoCloseable`
* `Cleaner`

---

### 🧠 Rule

> Never use finalize in production.

If you do → you’re wrong.

---

## 5. **Adding to List While Iterating (Iterator vs For-loop)**

---

### Case 1: Using Iterator

```java
Iterator it = list.iterator();
while(it.hasNext()) {
   list.add(x);
}
```

### ❌ Result: `ConcurrentModificationException`

Why?

* `ArrayList` uses `modCount`
* Iterator checks `expectedModCount`
* Structural change → mismatch

Fail-Fast behavior.

---

### Case 2: Using Index-based for-loop

```java
for(int i=0;i<list.size();i++){
   list.add(x);
}
```

### ❌ Result: Infinite loop / OOM

Because `size()` keeps growing.

---

### Case 3: Correct Way

```java
ListIterator it = list.listIterator();
it.add(x);
```

Only safe way.

---

### ⚠️ Trap

Fail-fast ≠ guaranteed.
Best-effort only.

---

### 🧠 Rule

> Modify via iterator, or don’t modify.

---

## 6. **What Are Records? How Do You Access Fields?**

### ✅ What They Are (Java 16+)

From JEP 395:

> Immutable data carrier class.

Example:

```java
public record Person(String name, int age) {}
```

Compiler generates:

* final fields
* constructor
* equals/hashCode
* toString
* accessors

---

### ✅ Access Fields

Not getters.

```java
Person p = new Person("A",10);

p.name(); // accessor
p.age();
```

Not `getName()`.

---

### ⚠️ Restrictions

Records:

* Are final
* No setters
* Cannot extend classes
* Fields are private final

---

### 🧠 Mental Model

> Record = named immutable tuple.

---

## 7. **Decorator vs Strategy**

### ✅ Strategy Pattern

Goal: **Change behavior**

```java
interface PaymentStrategy {
   pay();
}
```

Choose algorithm at runtime.

---

### ✅ Decorator Pattern

Goal: **Add behavior**

Wraps object.

```java
class LoggingService implements Service {
   Service base;
}
```

---

### ✅ Comparison

| Aspect   | Strategy      | Decorator        |
| -------- | ------------- | ---------------- |
| Purpose  | Replace logic | Extend logic     |
| Focus    | Algorithm     | Responsibility   |
| Wrapping | ❌             | ✅                |
| Example  | Sort strategy | Logging, Caching |

---

### ⚠️ Trap

Many confuse:

Decorator ≠ Strategy + Wrapper

Strategy replaces.
Decorator stacks.

---

### 🧠 Rule

> Strategy = “which way?”
> Decorator = “plus what?”

---

## 8. **Count Utility Method Calls in Tests Without Modifying Code**

### ✅ Correct Way: Mocking / Spying

Using Mockito (standard practice):

```java
Utility util = Mockito.spy(new Utility());

service.call(util);

Mockito.verify(util, times(3)).doWork();
```

---

### ✅ If Method is Static (Harder)

Use Mockito-inline (Java agent):

```java
try (MockedStatic<Util> mock = mockStatic(Util.class)) {
   mock.verify(() -> Util.doWork(), times(2));
}
```

---

### ✅ If No Mocking Allowed

Use AspectJ / Java Agent / ByteBuddy

But in interviews: **Mockito verify**.

---

### ⚠️ Trap

❌ Using counters in production code
❌ Adding logs
❌ Modifying logic

All bad.

---

### 🧠 Rule

> Tests observe behavior, not modify it.

---

## Final Senior-Level Takeaways

1️⃣ `WeakHashMap` = GC-driven cache (dangerous if misused)
2️⃣ Maven exclusions are **local and precise**
3️⃣ Exceptions bubble outward
4️⃣ `finalize()` is obsolete — never use
5️⃣ Modifying during iteration → CME / infinite loop
6️⃣ Records = immutable tuples
7️⃣ Strategy replaces, Decorator augments
8️⃣ Use mocking to observe calls
