# 1. Core Spring Composed Annotations

| Annotation                     | Combination                                                      | Purpose                         | Senior Insight                                         |
| ------------------------------ | ---------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------ |
| `@RestController`              | `@Controller` + `@ResponseBody`                                  | REST APIs returning JSON/XML    | Avoid using `@Controller` + `@ResponseBody` repeatedly |
| `@SpringBootApplication`       | `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` | Bootstraps Spring Boot app      | Most important Boot annotation                         |
| `@ConfigurationPropertiesScan` | `@Import(ConfigurationPropertiesScanRegistrar)`                  | Scans config properties classes | Used in config-driven systems                          |
| `@EnableCaching`               | `@Import(CachingConfigurationSelector)`                          | Enables cache abstraction       | Activates cache AOP                                    |

---

# 2. Stereotype Meta-Annotations

All of these are built on **`@Component`**.

| Annotation        | Combination                                      | Purpose            | Senior Insight                              |
| ----------------- | ------------------------------------------------ | ------------------ | ------------------------------------------- |
| `@Service`        | `@Component`                                     | Service layer bean | Semantic meaning only                       |
| `@Repository`     | `@Component` + Persistence exception translation | DAO layer          | Converts DB exceptions to Spring exceptions |
| `@Controller`     | `@Component`                                     | MVC controller     | Used for view rendering                     |
| `@RestController` | `@Controller` + `@ResponseBody`                  | REST APIs          | Most used in microservices                  |

Senior insight:

```
@Service, @Repository, @Controller are specialization of @Component
```

Spring treats them similarly for bean creation.

---

# 3. Spring Boot Testing Annotations

These are **heavily composed annotations**.

| Annotation        | Combination                                          | Purpose               | Senior Insight                  |
| ----------------- | ---------------------------------------------------- | --------------------- | ------------------------------- |
| `@SpringBootTest` | `@ExtendWith(SpringExtension)` + Boot context loader | Full integration test | Starts entire Spring context    |
| `@WebMvcTest`     | `@ExtendWith(SpringExtension)` + MVC slice config    | Controller layer test | Loads only MVC beans            |
| `@DataJpaTest`    | JPA config + transaction rollback                    | Repository testing    | Uses in-memory DB               |
| `@RestClientTest` | REST client configuration                            | REST client tests     | Useful for external API clients |

Production rule:

```
Avoid @SpringBootTest unless required (slow startup)
```

---

# 4. Spring MVC Mapping Composed Annotations

All are built from **`@RequestMapping`**.

| Annotation       | Combination                        | Equivalent      |
| ---------------- | ---------------------------------- | --------------- |
| `@GetMapping`    | `@RequestMapping(method = GET)`    | GET endpoint    |
| `@PostMapping`   | `@RequestMapping(method = POST)`   | POST endpoint   |
| `@PutMapping`    | `@RequestMapping(method = PUT)`    | Update endpoint |
| `@DeleteMapping` | `@RequestMapping(method = DELETE)` | Delete endpoint |
| `@PatchMapping`  | `@RequestMapping(method = PATCH)`  | Partial update  |

Example:

```java
@GetMapping("/users")
```

equals:

```java
@RequestMapping(value="/users", method=RequestMethod.GET)
```

Senior insight:

```
Always prefer composed mappings
```

Cleaner and readable.

---

# 5. Transaction Related Composed Annotations

| Annotation                     | Combination                                           | Purpose                 |
| ------------------------------ | ----------------------------------------------------- | ----------------------- |
| `@Transactional`               | `@TransactionManagement` + AOP interceptor            | Transaction boundary    |
| `@EnableTransactionManagement` | `@Import(TransactionManagementConfigurationSelector)` | Enables transaction AOP |

Senior insight:

```
@Transactional works via AOP proxy
```

---

# 6. Spring Security Meta Annotations

These are **very common in enterprise projects**.

| Annotation       | Combination                         | Purpose               |
| ---------------- | ----------------------------------- | --------------------- |
| `@PreAuthorize`  | Security expression evaluation      | Method level security |
| `@PostAuthorize` | Security evaluation after execution | Rarely used           |
| `@Secured`       | Role based method security          | Simpler alternative   |

Enablement annotation:

```
@EnableMethodSecurity
```

Which internally imports security configuration.

---

# 7. Scheduling & Async Meta Annotations

| Annotation          | Combination                           | Purpose                        |
| ------------------- | ------------------------------------- | ------------------------------ |
| `@EnableScheduling` | `@Import(SchedulingConfiguration)`    | Enables scheduled tasks        |
| `@Scheduled`        | Task scheduler interceptor            | Cron / fixed delay tasks       |
| `@EnableAsync`      | `@Import(AsyncConfigurationSelector)` | Enables async execution        |
| `@Async`            | Async method execution                | Runs method in separate thread |

Senior insight:

```
@Async also works via proxy
```

---

# 8. Configuration Related Meta Annotations

| Annotation        | Combination                             | Purpose                   |
| ----------------- | --------------------------------------- | ------------------------- |
| `@Configuration`  | `@Component` + configuration processing | Java configuration class  |
| `@PropertySource` | Property loader                         | Loads external properties |
| `@Import`         | Imports config classes                  | Modular configuration     |

Senior insight:

```
@Configuration classes are proxied via CGLIB
```

---

# 9. Conditional Auto Configuration (Spring Boot)

Used heavily inside Boot.

| Annotation                     | Combination           | Purpose                      |
| ------------------------------ | --------------------- | ---------------------------- |
| `@ConditionalOnClass`          | Condition evaluator   | Load config if class present |
| `@ConditionalOnMissingBean`    | Bean condition        | Avoid duplicate beans        |
| `@ConditionalOnProperty`       | Property based config | Feature toggles              |
| `@ConditionalOnWebApplication` | Web app detection     | Web-only configuration       |

These power **Spring Boot AutoConfiguration**.

---

# 10. Request/Response Body Annotations

| Annotation      | Combination             | Purpose                 |
| --------------- | ----------------------- | ----------------------- |
| `@ResponseBody` | HTTP message conversion | JSON/XML response       |
| `@RequestBody`  | HTTP message conversion | Request payload mapping |

When combined with `@Controller` → `@RestController`.

---

# Senior Insight: What Meta-Annotations Actually Mean

Spring supports **annotation composition**.

Example:

```java
@Target(TYPE)
@Retention(RUNTIME)
@Controller
@ResponseBody
public @interface RestController {}
```

So when Spring sees `@RestController`:

```
Spring internally reads all meta annotations
```

This is called:

```
Merged Annotation Model
```

---

# Real Production Tip (Senior Level)

In large companies we often create **custom composed annotations**.

Example:

```java
@Target(TYPE)
@Retention(RUNTIME)
@Service
@Transactional
public @interface DomainService {}
```

Usage:

```java
@DomainService
class PaymentService {}
```

Benefits:

* enforce architecture rules
* reduce repeated annotations
* standardize behavior

---

# Must-Know Composed Annotations (Top 12)

If an engineer remembers these, they understand **most real Spring apps**.

```
@SpringBootApplication
@RestController
@Service
@Repository
@Controller
@GetMapping
@PostMapping
@PutMapping
@DeleteMapping
@Configuration
@EnableAsync
@EnableScheduling
@EnableCaching
```
