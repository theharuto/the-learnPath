## What the Factory Pattern Does (in simple words)

*   **Encapsulates object creation**: Client asks a factory to create objects instead of calling `new` directly.
*   **Decouples client from concrete classes**: Client relies on an interface/abstract class.
*   **Centralizes creation logic**: Validation, defaults, caching, and selection of concrete types live in one place.
*   **Open/Closed Principle**: Add new products with minimal changes to client code.

> Common variants:
>
> *   **Simple Factory**: one class with a `create(type)` method (not a GoF pattern but widely used).
> *   **Factory Method**: let subclasses decide which product to create.
> *   **Abstract Factory**: create **families** of related objects.

***

## When to Use

*   You have **multiple implementations** of an interface and you want the client to depend only on the interface.
*   You expect **product types to grow over time**.
*   You need **conditional or complex construction** (e.g., configuration-driven, environment-driven).
*   You want **testability** (mock factories or products in unit tests).

***

## Minimal Example (Simple Factory)

### 1) Product interface and concrete products

```java
public interface Shape {
    void draw();
}

public class Circle implements Shape {
    private final int radius;
    public Circle(int radius) { this.radius = radius; }
    @Override public void draw() { System.out.println("Drawing Circle with r=" + radius); }
}

public class Rectangle implements Shape {
    private final int width, height;
    public Rectangle(int width, int height) {
        this.width = width; this.height = height;
    }
    @Override public void draw() { System.out.println("Drawing Rectangle " + width + "x" + height); }
}
```

### 2) Factory (centralizes creation logic)

```java
import java.util.Map;

public class ShapeFactory {
    public enum Type { CIRCLE, RECTANGLE }

    // Overloads for convenience; you can also pass a config object or Map
    public Shape create(Type type, Map<String, Integer> params) {
        switch (type) {
            case CIRCLE:
                int r = getOrDefault(params, "radius", 10);
                return new Circle(r);
            case RECTANGLE:
                int w = getOrDefault(params, "width", 20);
                int h = getOrDefault(params, "height", 10);
                return new Rectangle(w, h);
            default:
                throw new IllegalArgumentException("Unknown type: " + type);
        }
    }

    private int getOrDefault(Map<String, Integer> m, String key, int def) {
        return m != null && m.containsKey(key) ? m.get(key) : def;
    }
}
```

### 3) Client usage

```java
import java.util.Map;

public class Client {
    public static void main(String[] args) {
        ShapeFactory factory = new ShapeFactory();

        Shape circle = factory.create(ShapeFactory.Type.CIRCLE, Map.of("radius", 15));
        Shape rect   = factory.create(ShapeFactory.Type.RECTANGLE, Map.of("width", 30, "height", 10));

        circle.draw();
        rect.draw();
    }
}
```

***

## Factory Method Variant (polymorphic creation)

Use when **subclasses** decide which product to create:

```java
// Product
interface Transport { void deliver(); }
class Truck implements Transport { public void deliver() { System.out.println("Road delivery"); } }
class Ship  implements Transport { public void deliver() { System.out.println("Sea delivery"); } }

// Creator
abstract class Logistics {
    public void planDelivery() {
        Transport t = createTransport(); // factory method
        t.deliver();
    }
    protected abstract Transport createTransport();
}

class RoadLogistics extends Logistics {
    @Override protected Transport createTransport() { return new Truck(); }
}
class SeaLogistics extends Logistics {
    @Override protected Transport createTransport() { return new Ship(); }
}

// Client
class App {
    public static void main(String[] args) {
        Logistics logistics = new RoadLogistics(); // swap to SeaLogistics without changing client flow
        logistics.planDelivery();
    }
}
```

***

## Abstract Factory (families of related products)

Use when you need **consistent variants** across multiple product types:

```java
// Products: Button + Checkbox families
interface Button { void render(); }
interface Checkbox { void render(); }
class WinButton implements Button { public void render() { System.out.println("WinButton"); } }
class WinCheckbox implements Checkbox { public void render() { System.out.println("WinCheckbox"); } }
class MacButton implements Button { public void render() { System.out.println("MacButton"); } }
class MacCheckbox implements Checkbox { public void render() { System.out.println("MacCheckbox"); } }

// Abstract Factory
interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}
// Concrete factories
class WinFactory implements GUIFactory {
    public Button createButton() { return new WinButton(); }
    public Checkbox createCheckbox() { return new WinCheckbox(); }
}
class MacFactory implements GUIFactory {
    public Button createButton() { return new MacButton(); }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Client
class UIFactoryClient {
    private final Button button;
    private final Checkbox checkbox;

    UIFactoryClient(GUIFactory factory) {
        this.button = factory.createButton();
        this.checkbox = factory.createCheckbox();
    }

    void renderUI() { button.render(); checkbox.render(); }

    public static void main(String[] args) {
        GUIFactory factory = System.getProperty("os.name").startsWith("Windows")
                ? new WinFactory() : new MacFactory();
        new UIFactoryClient(factory).renderUI();
    }
}
```

***

## Pros & Cons (quick)

**Pros**

*   Decouples creation from usage.
*   Centralizes complex construction logic (validation, caching).
*   Improves testability and adherence to SOLID.

**Cons**

*   Can introduce **class explosion** (many factories/products).
*   Indirection can be overkill for simple cases.

***

## Practical Tips & Variations

*   **Enum-based factory** (clean switch on enum) – shown above.
*   **Map-based registry** for pluggability:
    ```java
    import java.util.Map;
    import java.util.function.Function;

    public class PaymentFactory {
        private final Map<String, Function<Map<String, Object>, Payment>> providers;

        public PaymentFactory(Map<String, Function<Map<String, Object>, Payment>> providers) {
            this.providers = Map.copyOf(providers);
        }

        public Payment create(String type, Map<String, Object> config) {
            var ctor = providers.get(type);
            if (ctor == null) throw new IllegalArgumentException("Unknown payment type: " + type);
            return ctor.apply(config);
        }
    }
    ```
*   **Thread-safety**: Factories are typically stateless and thus thread-safe. If you cache instances (flyweight/singletons), use concurrent structures or proper synchronization.
*   **Avoid reflection** for safety and clarity unless you need plugin discovery.
*   With **Spring**, prefer **dependency injection** over manual factories. Factories still useful for conditional logic or external config.

***

## Tiny JUnit Test Example (verifying factory)

```java
import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;
import java.util.Map;

class ShapeFactoryTest {
    @Test
    void createsCircleWithDefaultRadius() {
        ShapeFactory f = new ShapeFactory();
        Shape s = f.create(ShapeFactory.Type.CIRCLE, Map.of()); // no radius provided
        assertTrue(s instanceof Circle);
    }

    @Test
    void createsRectangleWithParams() {
        ShapeFactory f = new ShapeFactory();
        Shape s = f.create(ShapeFactory.Type.RECTANGLE, Map.of("width", 5, "height", 2));
        assertTrue(s instanceof Rectangle);
    }
}
```

***

## Cheat Sheet (memory hooks)

*   **Simple Factory**: `Factory.create("type")`.
*   **Factory Method**: `Creator.createProduct()` overridden by subclasses.
*   **Abstract Factory**: `Factory.createA()` + `Factory.createB()` for **related** products.

***
