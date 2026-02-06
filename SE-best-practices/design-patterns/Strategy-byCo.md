## 🚀 **What the Strategy Pattern Is (Simple Definition)**

The **Strategy Pattern** lets you **change the behavior (algorithm) of an object at runtime** by encapsulating the behavior into separate classes.

*   You define **a family of algorithms**
*   Put each algorithm in a **separate class**
*   Make them **interchangeable at runtime**

This avoids messy `if-else` or `switch` statements.

***

# 🧱 **Layer 1: Strategy Interface (The Behavior Contract)**

This is the **parent** of all strategies.  
It defines *what* the algorithm must do, not *how*.

### UML‑ish Diagram

    +-------------------+
    |   Strategy        |
    +-------------------+
    | + doOperation()   |
    +-------------------+

### Java Example

```java
public interface PaymentStrategy {
    void pay(int amount);
}
```

***

# 🧱 **Layer 2: Concrete Strategies (Different Implementations)**

Each strategy provides its **own version** of the algorithm.

### UML‑ish

    +-------------------+         +---------------------+
    |   Strategy        |<>------>| ConcreteStrategyA   |
    +-------------------+         +---------------------+
    | + doOperation()   |         | + doOperation()     |
    +-------------------+         +---------------------+

                               +---------------------+
                               | ConcreteStrategyB   |
                               | + doOperation()     |
                               +---------------------+

### Java Example

```java
public class CreditCardPayment implements PaymentStrategy {
    public void pay(int amount) {
        System.out.println("Paid " + amount + " using Credit Card.");
    }
}

public class PayPalPayment implements PaymentStrategy {
    public void pay(int amount) {
        System.out.println("Paid " + amount + " using PayPal.");
    }
}
```

***

# 🧱 **Layer 3: Context (The Object That Uses the Strategy)**

The **Context** contains a reference to a `Strategy`.  
It does **not** know the algorithm details — just calls the strategy.

### UML‑ish

    +----------------------+
    |       Context        |
    +----------------------+
    | - strategy:Strategy  |
    +----------------------+
    | + setStrategy()      |
    | + execute()          |
    +----------------------+

### Java Example

```java
public class PaymentContext {
    private PaymentStrategy strategy;

    public void setStrategy(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public void pay(int amount) {
        strategy.pay(amount);
    }
}
```

***

# 🧱 **Layer 4: Client (Choosing the Strategy at Runtime)**

### UML‑ish (Full Diagram)

                      +----------------------+
                      |      Context         |
                      +----------------------+
                      | - strategy:Strategy  |
                      +----------------------+
                      | + setStrategy()      |
                      | + execute()          |
                      +----------^-----------+
                                 |
            ---------------------------------------------
            |                                           |
    +--------------------+                     +----------------------+
    | ConcreteStrategyA  |                     | ConcreteStrategyB    |
    +--------------------+                     +----------------------+
    | + doOperation()    |                     | + doOperation()      |
    +--------------------+                     +----------------------+

### Java Example

```java
public class Main {
    public static void main(String[] args) {

        PaymentContext context = new PaymentContext();

        context.setStrategy(new CreditCardPayment());
        context.pay(500);

        context.setStrategy(new PayPalPayment());
        context.pay(800);
    }
}
```

***

# 🧩 **Layer 5: Where Strategy Pattern Is Used**

Very commonly used in:

*   Sorting algorithms (`Comparator`)
*   Payment systems
*   Compression algorithms
*   Input validation rules
*   Logging rules
*   Game movement behavior (AI strategies)

***

# 🎯 **Layer 6: When to Use Strategy Pattern (Simple Points)**

✔ When you have many IF/ELSE for choosing behaviors  
✔ When behavior must change at runtime  
✔ When you want clean, maintainable, testable code  
✔ When algorithms are independent and reusable

***

# 🆚 **Strategy vs State Pattern (Quick clarity)**

Both use composition and delegation.

| Strategy                      | State                                 |
| ----------------------------- | ------------------------------------- |
| User selects behavior         | Object transitions itself             |
| Behaviors are interchangeable | Behaviors represent state transitions |
| Focus on algorithm            | Focus on state                        |

***

# 🎁 Final Summary (In Simple Points)

*   Strategy = **Behavior as an object**
*   Helps avoid **if-else hell**
*   Can change behavior **dynamically**
*   Uses **interface + multiple implementations**
*   Context holds strategy and calls it

***
✅ **Real-world examples**  
✅ **Advanced diagrams**  
✅ **Interview questions**  
✅ **Comparison with other patterns**  
