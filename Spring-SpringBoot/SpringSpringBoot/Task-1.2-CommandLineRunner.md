If you implement `CommandLineRunner`, you **still need the main method**. Spring Boot must start somehow.

Your class should look like this:

```java
@SpringBootApplication
public class Application implements CommandLineRunner {

    private final CalculatorService calculatorService;

    // Constructor Injection (preferred)
    public Application(CalculatorService calculatorService) {
        this.calculatorService = calculatorService;
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Override
    public void run(String... args) {
        System.out.println("Addition Result: " + calculatorService.add(5, 3));
    }
}
```

---

## What happens here?

1. `main()` starts the Spring container.
2. Spring scans for beans (`@Component`, `@Service`, etc.).
3. Spring creates `CalculatorService`.
4. Spring creates `Application` and injects `CalculatorService` into its constructor.
5. After context startup, Spring calls `run()` automatically.

---

## Important Concept

`CommandLineRunner` does **not replace main**.

It just gives you a hook that runs *after* the application context is ready.

---

## When to use `CommandLineRunner`

Use it when:

* You want to run startup logic
* You’re building a CLI app
* You need to test DI without REST

Do **not** use it for web-based controllers unless the task explicitly requires injection into the main class (like yours).

---

## For Your Task Specifically

Since the task says:

> inject it into the main application class

Using `CommandLineRunner` is probably what they expect.

If the task was about REST, your controller approach would be better.

---

## Clean Summary

You need:

✔ `main()`
✔ `SpringApplication.run(...)`
✔ constructor injection
✔ `run()` implementation

All together.

---

## More on CommandLineRunner

In **Spring Boot**, `CommandLineRunner` is used to:

> Run some code automatically **after the application starts**.

---

## 🔥 When Does It Execute?

Spring Boot starts in this order:

1. Creates application context
2. Creates all beans
3. Injects dependencies
4. ✅ Calls `CommandLineRunner.run()`
5. Application is fully ready

So your `run()` method executes **after Spring finishes startup**.

---

## 🎯 Why Is It Useful?

### 1️⃣ Run startup logic

Example:

* Insert initial data
* Call a service
* Print something
* Trigger background process

---

### 2️⃣ Good for non-web apps

If you're building:

* Console apps
* Batch jobs
* CLI tools

`CommandLineRunner` is perfect.

---

### 3️⃣ Access command-line arguments

```bash
java -jar app.jar hello world
```

Then:

```java
run(String... args)
```

`args` will contain:

```
["hello", "world"]
```

---

## 🧠 In Your Example

```java
@Override
public void run(String... args) {
    System.out.println("Addition Result: " + calculatorService.add(5, 3));
}
```

After Spring starts:

* `CalculatorService` is injected
* Then `run()` executes
* It prints the result

---

## 🔄 Difference From `main()`

* `main()` → Starts Spring
* `CommandLineRunner.run()` → Runs after Spring is ready

---

## 🚀 Real Use Case Example

Insert default data:

```java
@Override
public void run(String... args) {
    userRepository.save(new User("admin"));
}
```

---

## 📌 Important

You can define multiple `CommandLineRunner` beans.

If multiple exist:
You can control order using:

```java
@Order(1)
```

---

## 🎯 Short Summary

`CommandLineRunner` is used to execute code **once automatically after the Spring Boot application fully starts**.

---

If you want, I can also explain the difference between:

* `CommandLineRunner`
* `ApplicationRunner`
* `@PostConstruct`

Those differences matter in interviews.


