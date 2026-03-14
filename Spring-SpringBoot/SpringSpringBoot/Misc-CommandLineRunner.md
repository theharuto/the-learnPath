In Spring Boot, **`CommandLineRunner`** is used to run code **after the Spring application context is fully started**. It’s commonly used for:

* Initializing data
* Running startup logic
* Testing services at startup
* Printing configuration info

---

# 1️⃣ What is `CommandLineRunner`?

`CommandLineRunner` is an interface in Spring Boot.

```java
public interface CommandLineRunner {
    void run(String... args) throws Exception;
}
```

Spring Boot automatically calls the **`run()` method after the application starts**.

---

# 2️⃣ When Does `run()` Execute?

Execution order in Spring Boot:

1. `main()` method starts
2. `SpringApplication.run()` starts the application
3. Spring creates all **beans**
4. Application context is ready
5. **`CommandLineRunner.run()` executes**

So:

```
main() → Spring Boot starts → Beans created → run() executes
```

---

# 3️⃣ Where to Place `run()` Method

The **`run()` method is NOT inside `main()`**.

It must be **inside a class that implements `CommandLineRunner`**.

Example:

```java
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication implements CommandLineRunner {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        System.out.println("Application started!");
    }
}
```

Structure:

```
main()  → starts Spring Boot
run()   → executed automatically after startup
```

---

# 4️⃣ Better Practice (Recommended)

Instead of putting it in the **main class**, create a **separate component**.

```java
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class StartupRunner implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        System.out.println("Running startup logic...");
    }
}
```

Spring Boot will automatically detect it because of `@Component`.

---

# 5️⃣ Example Use Case

Example: Insert default data into a database.

```java
@Component
public class DataLoader implements CommandLineRunner {

    private final UserRepository userRepository;

    public DataLoader(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public void run(String... args) {
        userRepository.save(new User("John"));
        userRepository.save(new User("Alice"));
    }
}
```

---

# 6️⃣ Multiple `CommandLineRunner`s

If multiple runners exist, you can control order using:

```java
@Order(1)
```

Example:

```java
@Component
@Order(1)
public class FirstRunner implements CommandLineRunner
```

---

# 7️⃣ Alternative: `ApplicationRunner`

Spring Boot also provides:

* ApplicationRunner

Difference:

| Feature         | CommandLineRunner | ApplicationRunner    |
| --------------- | ----------------- | -------------------- |
| Arguments       | String[]          | ApplicationArguments |
| Structured args | ❌                 | ✅                    |

---

✅ **Summary**

* `run()` is **not inside `main()`**
* It is **inside a class implementing `CommandLineRunner`**
* It executes **after Spring Boot fully starts**
* Used for **startup logic**

```
main() → SpringApplication.run()
      → Spring Context Ready
      → CommandLineRunner.run()
```

---

If you'd like, I can also show **3 very common real-world uses of `CommandLineRunner` in Spring Boot projects (used in interviews)**.
