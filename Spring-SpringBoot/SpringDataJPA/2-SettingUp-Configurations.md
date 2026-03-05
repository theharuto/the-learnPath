# 📘 Spring Data JPA – Topic 2: Setup & Configuration

## 1️⃣ Adding Spring Data JPA Dependency

Spring Boot provides a starter dependency that includes everything required for JPA integration.

### Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

### Gradle

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

---

## What This Starter Includes

`spring-boot-starter-data-jpa` automatically brings:

* **Spring Data JPA**
* **Spring ORM**
* **Spring Transaction Management**
* **Hibernate ORM (default JPA provider)**
* **Jakarta Persistence API**
* **Spring JDBC**
* **HikariCP (default connection pool)**

So adding this starter sets up the entire **ORM infrastructure**.

---

# 2️⃣ Database Driver Dependency

Spring Boot does **not include database drivers automatically**.

Example: PostgreSQL

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

Example: MySQL

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
</dependency>
```

---

# 3️⃣ Configuring Database Connection

Spring Boot reads database configuration from:

* `application.properties`
* `application.yml`

---

## application.properties Example

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/appdb
spring.datasource.username=appuser
spring.datasource.password=secret
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

---

## application.yml Example

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/appdb
    username: appuser
    password: secret

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

---

# 4️⃣ Minimum Required Properties

At minimum:

```properties
spring.datasource.url
spring.datasource.username
spring.datasource.password
```

Spring Boot then automatically configures:

* DataSource
* EntityManagerFactory
* TransactionManager
* Repository infrastructure

---

# 5️⃣ Core Infrastructure Components

Spring Boot automatically creates the following beans.

---

## DataSource

Responsibility:

* Manages database connections.
* Provides connection pooling.

Default pool used by Spring Boot:
**HikariCP**

Think of DataSource as:

```
Application → DataSource → Database Connection
```

---

## EntityManagerFactory

Defined by **JPA specification**.

Responsibilities:

* Creates `EntityManager` objects
* Holds entity metadata
* Integrates JPA with Hibernate

Spring Boot uses:

```
LocalContainerEntityManagerFactoryBean
```

---

## EntityManager

JPA interface used for:

* Persisting entities
* Fetching entities
* Updating entities
* Removing entities

Example operations:

```
persist()
find()
merge()
remove()
```

Spring Data repositories internally use `EntityManager`.

---

## TransactionManager

Spring manages transactions using:

```
JpaTransactionManager
```

Responsibilities:

* Begin transaction
* Commit transaction
* Rollback transaction
* Synchronize persistence context

Works with:

```
@Transactional
```

---

# 6️⃣ Hibernate Schema Generation

Property:

```properties
spring.jpa.hibernate.ddl-auto=
```

Options:

| Value       | Behavior                            |
| ----------- | ----------------------------------- |
| none        | No schema changes                   |
| validate    | Validate schema against entities    |
| update      | Update schema automatically         |
| create      | Drop and recreate schema            |
| create-drop | Create on startup, drop on shutdown |

### Production Rule

Avoid using:

```
create
update
```

Instead use database migration tools like:

* Flyway
* Liquibase

---

# 7️⃣ Spring Boot Auto Configuration

When Spring Boot starts and detects:

* JPA libraries
* A configured DataSource

It automatically configures:

1. DataSource
2. EntityManagerFactory
3. TransactionManager
4. JPA repositories
5. Entity scanning

No manual configuration required.

---

# 8️⃣ Entity & Repository Scanning

Spring Boot automatically scans:

* `@Entity` classes
* `@Repository` interfaces

Scanning starts from the package where the **main application class** exists.

Example:

```java
@SpringBootApplication
public class Application {}
```

All sub-packages will be scanned.

---

# 9️⃣ Common Configuration Mistakes

### ❌ Missing Database Driver

Application fails with driver class error.

Always add database dependency.

---

### ❌ Entities Outside Scan Package

If entities are outside main package:

```java
@EntityScan("com.example.domain")
```

---

### ❌ Multiple DataSources Without Configuration

Spring Boot auto-configures **only one DataSource**.

Multiple DB setups require manual configuration.

---

### ❌ show-sql Enabled in Production

```properties
spring.jpa.show-sql=true
```

This can:

* expose sensitive data
* degrade performance

Disable in production.

---

# 🔟 Startup Flow (Mental Model)

When Spring Boot application starts:

```
1. Read application properties
2. Create DataSource
3. Configure EntityManagerFactory
4. Create TransactionManager
5. Scan entities
6. Initialize repositories
7. Application ready
```

---

# ✅ Final Summary

To configure Spring Data JPA in Spring Boot:

1. Add `spring-boot-starter-data-jpa`
2. Add database driver dependency
3. Configure `spring.datasource` properties
4. Configure optional `spring.jpa` settings
5. Let Spring Boot auto-configure JPA infrastructure

This provides a fully functional **ORM setup with minimal configuration**.
