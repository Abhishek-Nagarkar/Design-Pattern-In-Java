# Key Design Patterns in Java

Design patterns are reusable fixes for problems that keep coming up in code design. Many of the classic Gang of Four patterns look different in modern Java, because lambdas, enums and records replace a lot of the old boilerplate. I'll cover the five that come up most often in real code and in interviews, in the old style where it helps, then the modern version.

---

## 1. Builder (Creational)

**Problem:** A class has many fields, several of them optional. Constructors with long parameter lists are hard to read and easy to get wrong:

```java
new Pizza("large", true, false, true, false, "thin"); // which boolean is cheese?
```

The usual workaround, a separate constructor for each combination of fields (called telescoping constructors), doesn't scale. Setters avoid that, but then the object is mutable and can be half-built at any moment.

**Solution:** A static nested `Builder` class collects the values step by step and then creates the final object in one go. The finished object can be immutable.

```java
public final class Pizza {
    private final String size;        // required
    private final boolean cheese;     // optional
    private final boolean mushrooms;  // optional
    private final String crust;       // optional

    private Pizza(Builder b) {
        this.size = b.size;
        this.cheese = b.cheese;
        this.mushrooms = b.mushrooms;
        this.crust = b.crust;
    }

    public static Builder builder(String size) { return new Builder(size); }

    public static final class Builder {
        private final String size;
        private boolean cheese;
        private boolean mushrooms;
        private String crust = "regular";   // default value

        private Builder(String size) { this.size = size; }

        public Builder cheese()            { this.cheese = true; return this; }
        public Builder mushrooms()         { this.mushrooms = true; return this; }
        public Builder crust(String crust) { this.crust = crust; return this; }

        public Pizza build() {
            if (size == null) throw new IllegalStateException("size required");
            return new Pizza(this);   // validate here, before the object exists
        }
    }
}

Pizza p = Pizza.builder("large").cheese().crust("thin").build();
```

**Key points:**
- Each setter returns `this`, which is what lets you chain the calls (a "fluent" interface).
- `build()` is the one place to check that the object is valid.
- You'll find builders in the JDK too: `StringBuilder`, `HttpRequest.newBuilder()` and `Stream.builder()`.
- With records, Lombok's `@Builder` is often what teams use in practice. If a record only has 2–3 fields, its canonical constructor is usually enough and you don't need a builder.

---

## 2. Factory (Creational)

**Problem:** Calling code shouldn't have to know which concrete class to create. If `new EmailNotifier()` is scattered everywhere, adding an SMS option means changing all of those places.

### Static factory method (the most common form in modern Java)

```java
public sealed interface Notifier permits EmailNotifier, SmsNotifier, PushNotifier {
    void send(String msg);

    static Notifier of(String channel) {
        return switch (channel) {
            case "email" -> new EmailNotifier();
            case "sms"   -> new SmsNotifier();
            case "push"  -> new PushNotifier();
            default      -> throw new IllegalArgumentException("Unknown: " + channel);
        };
    }
}

Notifier n = Notifier.of("sms");   // caller depends only on the interface
```

**Why static factories often beat constructors** (this is Effective Java, Item 1):
- They have **names**, e.g. `Optional.empty()`, `List.of()`, `Duration.ofSeconds()`.
- They can return a **cached instance** instead of a new one each time. `Integer.valueOf(5)` does this, and `new Integer(5)` can't.
- They can return a **subtype**. `List.of()` hands back different internal classes depending on how many elements you pass.

### Factory Method pattern (the classic GoF form)

Here the choice of which class to create is left to **subclasses**:

```java
abstract class Dialog {
    abstract Button createButton();        // the "factory method"
    void render() { createButton().draw(); }
}
class WindowsDialog extends Dialog { Button createButton() { return new WindowsButton(); } }
class WebDialog     extends Dialog { Button createButton() { return new HtmlButton(); } }
```

**Interview tip:** Be ready to explain the difference between a *Simple Factory* (one method with a switch), the *Factory Method* (subclasses decide) and an *Abstract Factory* (a factory that creates whole families of related objects, such as a `UIFactory` that makes a matching button, checkbox and menu).

---

## 3. Strategy (Behavioral)

**Problem:** You have several interchangeable algorithms, such as ways to pay or apply a discount, and you want to choose one at runtime without a growing `if/else` chain.

### Classic form

```java
interface DiscountStrategy {
    double apply(double price);
}

class NoDiscount implements DiscountStrategy {
    public double apply(double p) { return p; }
}
class FestiveDiscount implements DiscountStrategy {
    public double apply(double p) { return p * 0.8; }
}

class Checkout {
    private final DiscountStrategy strategy;
    Checkout(DiscountStrategy strategy) { this.strategy = strategy; }
    double total(double price) { return strategy.apply(price); }
}
```

### Modern form: a strategy is often just a lambda

Because `DiscountStrategy` has a single abstract method, it is a **functional interface**, so you don't need a class per strategy:

```java
Checkout c1 = new Checkout(p -> p);           // no discount
Checkout c2 = new Checkout(p -> p * 0.8);     // festive
Checkout c3 = new Checkout(p -> Math.max(0, p - 100)); // flat ₹100 off
```

You could even skip the custom interface and use `UnaryOperator<Double>` or `DoubleUnaryOperator`.

### Enum-based strategy, for a fixed set of options

```java
enum Discount implements DiscountStrategy {
    NONE    { public double apply(double p) { return p; } },
    FESTIVE { public double apply(double p) { return p * 0.8; } };
}
```

**You already use this pattern:** `Comparator` is a strategy. `list.sort(Comparator.comparing(User::age))` passes the sorting algorithm in as a strategy.

**Main benefit:** It follows the **Open/Closed Principle**. You add new behavior by adding a new strategy, without editing existing code.

---

## 4. Singleton (Creational), and its pitfalls

**Goal:** There should be exactly one instance of the class, with a global access point to it, for things like a config registry or a connection pool.

This pattern comes up in interviews mostly so you can explain **what goes wrong** with it.

### ❌ Naive version: not thread-safe

```java
public class Config {
    private static Config instance;
    private Config() {}
    public static Config getInstance() {
        if (instance == null) instance = new Config();  // two threads can both see null
        return instance;
    }
}
```

### ⚠️ Double-checked locking: works only with `volatile`

```java
public class Config {
    private static volatile Config instance;   // volatile is essential
    private Config() {}
    public static Config getInstance() {
        if (instance == null) {
            synchronized (Config.class) {
                if (instance == null) instance = new Config();
            }
        }
        return instance;
    }
}
```

**Why `volatile` is needed:** `instance = new Config()` is not atomic. It allocates memory, runs the constructor, then assigns the reference, and the CPU or JIT may **reorder** those steps. Without `volatile`, another thread can see a non-null reference to an object that hasn't finished being constructed. `volatile` sets up a *happens-before* relationship that prevents this.

### ✅ Holder idiom: lazy, thread-safe, no locks

```java
public class Config {
    private Config() {}
    private static class Holder {
        static final Config INSTANCE = new Config();
    }
    public static Config getInstance() { return Holder.INSTANCE; }
}
```

The JVM loads `Holder` only the first time `getInstance()` is called, and the JVM guarantees that class initialization is thread-safe.

### ✅✅ Enum singleton: recommended by Joshua Bloch

```java
public enum Config {
    INSTANCE;
    public String get(String key) { /* ... */ return null; }
}
```

An enum singleton is safe against **reflection attacks**: calling `setAccessible(true)` on a private constructor won't let you create a second instance. It also handles **serialization** correctly, whereas a normal class would create a new instance on deserialization unless you implement `readResolve()`.

### Why many people consider Singleton an anti-pattern
- It is **global mutable state** under another name.
- It is **hard to test**, because you can't easily replace it with a mock.
- It **hides dependencies**, since classes reach for it instead of declaring what they need.

**Modern practice:** Let a DI container manage the lifecycle. Spring beans are singleton-scoped by default, and you **inject** them, so you get one instance without the global-access problems.

---

## 5. Observer (Behavioral)

**Problem:** When one object changes, several other objects need to react, and the object that changes shouldn't know their concrete types.

```java
interface OrderListener {
    void onOrderPlaced(Order order);
}

class OrderService {
    private final List<OrderListener> listeners = new CopyOnWriteArrayList<>();

    void subscribe(OrderListener l)   { listeners.add(l); }
    void unsubscribe(OrderListener l) { listeners.remove(l); }

    void placeOrder(Order order) {
        // ... save order ...
        listeners.forEach(l -> l.onOrderPlaced(order));
    }
}

record Order(String id, double amount) {}

// Subscribers, written as lambdas
OrderService svc = new OrderService();
svc.subscribe(o -> System.out.println("Email sent for " + o.id()));
svc.subscribe(o -> System.out.println("Inventory updated for " + o.id()));
svc.placeOrder(new Order("A-101", 2500));
```

**Details worth noticing:**
- **`CopyOnWriteArrayList`** prevents a `ConcurrentModificationException` when a listener unsubscribes while notifications are being sent. It suits lists that are read far more often than they're changed.
- **Memory leaks:** If a listener never unsubscribes, the subject keeps a reference to it and it can never be garbage-collected. This is the "lapsed listener" problem.
- **Exception isolation:** If one listener throws, the listeners after it are never notified. In production code, wrap each call in a try/catch.
- `java.util.Observable` is **deprecated** (since Java 9), so don't use it.

**Where you'll see it in practice:** Spring's `ApplicationEventPublisher` and `@EventListener`, GUI event handlers, `PropertyChangeListener`, and, at a larger scale, Kafka and pub/sub systems. Reactive Streams (`Flow.Publisher`, Project Reactor, RxJava) are an Observer with backpressure added.

---

## Quick reference

| Pattern | Category | Problem it solves | Modern Java twist |
|---|---|---|---|
| Builder | Creational | Many optional parameters, immutability | Records and Lombok `@Builder` |
| Factory | Creational | Hide which concrete class gets created | Static factories, sealed types, switch expressions |
| Strategy | Behavioral | Swap algorithms at runtime | Lambdas and functional interfaces |
| Singleton | Creational | Exactly one instance | Enum singleton, or a DI container |
| Observer | Behavioral | Notify many objects when one changes | Lambdas as listeners, Spring events, reactive streams |

Other patterns worth knowing are **Decorator** (`BufferedReader` wrapping a `FileReader` is the classic JDK example), **Adapter** (`Arrays.asList()`), **Template Method** (`AbstractList`), **Proxy** (Spring AOP and `@Transactional`) and **Command** (`Runnable` tasks given to an executor).

---

## 🧠 Check yourself

1. Why does the double-checked locking Singleton break without `volatile`?
2. You have 6 tax rules, and the business adds new ones every quarter. Which pattern fits, and how would you write it in modern Java?
3. Name two advantages that static factory methods have over constructors.
4. What can cause a memory leak in the Observer pattern, and how do you prevent it?
