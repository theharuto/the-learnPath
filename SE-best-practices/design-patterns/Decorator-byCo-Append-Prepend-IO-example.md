## Part 1 — Flip the order: **append** vs **prepend**

We’ll use the same `Coffee` setup but show two decorator styles:

*   **Append style**: `return wrapped + ", MyPart"` → Output order follows **wrapping order** (inner first, outer last).
*   **Prepend style**: `return "MyPart, " + wrapped` → Output order flips (outer first, inner last).

### Minimal setup

```java
interface Coffee { String getDescription(); double cost(); }

class SimpleCoffee implements Coffee {
    public String getDescription() { return "Simple Coffee"; }
    public double cost() { return 10.0; }
}

abstract class CoffeeDecorator implements Coffee {
    protected final Coffee coffee;
    protected CoffeeDecorator(Coffee coffee) { this.coffee = coffee; }
}
```

***

### A) **Append** decorators (usual style)

```java
class MilkAppend extends CoffeeDecorator {
    public MilkAppend(Coffee coffee) { super(coffee); }
    public String getDescription() { return coffee.getDescription() + ", Milk"; }
    public double cost() { return coffee.cost() + 2.0; }
}

class SugarAppend extends CoffeeDecorator {
    public SugarAppend(Coffee coffee) { super(coffee); }
    public String getDescription() { return coffee.getDescription() + ", Sugar"; }
    public double cost() { return coffee.cost() + 1.0; }
}
```

**Client:**

```java
Coffee c = new SugarAppend(new MilkAppend(new SimpleCoffee()));
System.out.println(c.getDescription()); // ?
System.out.println(c.cost());           // ?
```

**Flow (append):**

    SugarAppend.getDescription()
      -> MilkAppend.getDescription()
           -> SimpleCoffee.getDescription() => "Simple Coffee"
           -> + ", Milk"                    => "Simple Coffee, Milk"
      -> + ", Sugar"                        => "Simple Coffee, Milk, Sugar"

**Output:**

    Simple Coffee, Milk, Sugar
    13.0

**Diagram (append):**

    [ Sugar ] -> [ Milk ] -> [ Simple ]
    desc builds as: Simple → +Milk → +Sugar

***

### B) **Prepend** decorators (flipped order)

```java
class MilkPrepend extends CoffeeDecorator {
    public MilkPrepend(Coffee coffee) { super(coffee); }
    public String getDescription() { return "Milk, " + coffee.getDescription(); }
    public double cost() { return 2.0 + coffee.cost(); }
}

class SugarPrepend extends CoffeeDecorator {
    public SugarPrepend(Coffee coffee) { super(coffee); }
    public String getDescription() { return "Sugar, " + coffee.getDescription(); }
    public double cost() { return 1.0 + coffee.cost(); }
}
```

**Client:**

```java
Coffee c = new SugarPrepend(new MilkPrepend(new SimpleCoffee()));
System.out.println(c.getDescription()); // ?
System.out.println(c.cost());           // ?
```

**Flow (prepend):**

    SugarPrepend.getDescription()
      -> "Sugar, " + MilkPrepend.getDescription()
          -> "Milk, " + SimpleCoffee.getDescription()  => "Milk, Simple Coffee"
      => "Sugar, Milk, Simple Coffee"

**Output:**

    Sugar, Milk, Simple Coffee
    13.0

**Diagram (prepend):**

    [ Sugar ] -> [ Milk ] -> [ Simple ]
    desc builds as: Sugar → +Milk → +Simple

> ✅ Key takeaway: **Append** → inner-first order; **Prepend** → outer-first order.  
> Cost adds up the same in both because it’s just numbers being summed.

***

## Part 2 — Real JDK I/O Decorators

**`InputStream`** is the component interface (abstract class).  
**`FileInputStream`** is a **concrete component**.  
**`FilterInputStream`** is the **base decorator**, and **`BufferedInputStream`** is a **concrete decorator** that adds buffering (fewer disk reads, supports `mark/reset`).

### UML-ish sketch

           +---------------------+
           |   InputStream       |  <- component (abstract)
           +---------------------+
                    ▲
                    |
           +---------------------+
           |  FileInputStream    |  <- concrete component
           +---------------------+

           +---------------------+
           |  FilterInputStream  |  <- base decorator (wraps InputStream)
           +---------------------+
                    ▲
                    |
           +---------------------+
           | BufferedInputStream |  <- concrete decorator (adds buffer)
           +---------------------+

### Typical wrapping

```java
try (InputStream in = new BufferedInputStream(new FileInputStream("data.txt"))) {
    int b;
    while ((b = in.read()) != -1) {
        // process byte b
    }
}
```

**What happens:**

*   Client calls `BufferedInputStream.read()`.
*   It fills an internal byte array buffer by delegating to the **wrapped** `FileInputStream` in chunks (e.g., 8 KB) instead of one byte at a time.
*   Subsequent `read()` calls are served from the buffer, reducing system calls → **performance boost**.
*   `BufferedInputStream` also implements `mark()`/`reset()` behavior using its buffer.

### Tiny demo: count raw disk reads vs buffered reads (conceptual)

```java
import java.io.*;

class CountingFileInputStream extends FileInputStream {
    private int reads = 0;
    public CountingFileInputStream(String name) throws FileNotFoundException { super(name); }
    @Override public int read(byte[] b, int off, int len) throws IOException {
        reads++;
        return super.read(b, off, len);
    }
    @Override public int read() throws IOException {
        reads++;
        return super.read();
    }
    public int getReads() { return reads; }
}

public class BufferDemo {
    public static void main(String[] args) throws Exception {
        // Prepare a sample file "data.txt" with some content beforehand.

        // 1) No buffering: many OS reads
        CountingFileInputStream raw = new CountingFileInputStream("data.txt");
        int c1;
        while ((c1 = raw.read()) != -1) { /* consume */ }
        System.out.println("Raw reads: " + raw.getReads());
        raw.close();

        // 2) With buffering: far fewer OS reads
        CountingFileInputStream base2 = new CountingFileInputStream("data.txt");
        BufferedInputStream buf = new BufferedInputStream(base2, 8192);
        int c2;
        while ((c2 = buf.read()) != -1) { /* consume */ }
        buf.close();
        System.out.println("With buffer reads: " + base2.getReads()); // typically << raw
    }
}
```

> Expect **“With buffer reads”** to be **much smaller** because `BufferedInputStream` reads large chunks into memory and serves many single-byte `read()` calls from that chunk.

### Showcasing `mark` / `reset`

```java
try (BufferedInputStream in = new BufferedInputStream(new FileInputStream("data.txt"))) {
    if (in.markSupported()) {
        in.mark(100);                     // remember position for next 100 bytes
        int a = in.read();                // read some bytes
        int b = in.read();
        in.reset();                       // go back to mark
        int a2 = in.read();               // a2 == a
        int b2 = in.read();               // b2 == b
        System.out.printf("a=%d a2=%d, b=%d b2=%d%n", a, a2, b, b2);
    }
}
```

### ASCII call flow (like your coffee chain)

    Client.read()
      -> BufferedInputStream.read()
           -> (fills buffer) FileInputStream.read(byte[], off, len)
           <- returns bytes into internal buffer
         -> returns data from buffer to client

***

## Key parallels to the Decorator pattern

*   **Same interface**: `BufferedInputStream` is an `InputStream` → can be used anywhere an `InputStream` is expected.
*   **Composition**: `BufferedInputStream` **wraps** another `InputStream` (e.g., `FileInputStream`).
*   **Adds behavior**: buffering + `mark/reset` without changing the original `FileInputStream` class.
*   **Stackable**: You can wrap multiple decorators, e.g., `DataInputStream(new BufferedInputStream(new FileInputStream(...)))`.

***

## Quick recap

*   **Append vs Prepend** changes only **where** you add your contribution relative to the delegated call:
    *   Append → `"inner, ..., outer"` order in strings.
    *   Prepend → `"outer, ..., inner"` order.
*   **JDK I/O** uses the same pattern:
    *   `FileInputStream` = base component.
    *   `BufferedInputStream` = decorator adding buffering and features, delegating to the wrapped stream.

***

If you want, I can:

*   Combine multiple I/O decorators (e.g., `BufferedInputStream` + `DataInputStream`) and show how methods like `readInt()` still delegate down to the file.
*   Do a **sequence diagram** for one `read()` call end-to-end.
