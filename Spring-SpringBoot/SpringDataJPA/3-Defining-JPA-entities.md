# 📘 Spring Data JPA – Topic 3: Defining JPA Entities

## 1️⃣ What is a JPA Entity?

A **JPA Entity** is a Java class mapped to a database table.

Each **entity instance represents one row** in the database table.

Example mapping:

```
Java Object (User)  →  Database Row (users table)
```

Entities are defined using **Jakarta Persistence (JPA) annotations**.

---

# 2️⃣ `@Entity`

Marks a class as a **JPA entity**.

Example:

```java
@Entity
public class User {

    @Id
    private Long id;

    private String name;
}
```

### JPA Requirements for Entity Classes

According to the JPA specification:

* Must have a **no-argument constructor**
* Must have a **primary key**
* Must be a **top-level class**
* Should not be **final**
* Persistent fields should not be **final**

---

# 3️⃣ `@Table`

Specifies the **database table name**.

If not specified, JPA uses the **entity class name**.

Example:

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    private Long id;

    private String name;
}
```

### Useful Attributes

```java
@Table(
    name = "users",
    schema = "public"
)
```

### Best Practice

Always define table names explicitly in production systems.

---

# 4️⃣ `@Id`

Defines the **primary key of the entity**.

Example:

```java
@Id
private Long id;
```

JPA uses this field to uniquely identify an entity within the **persistence context**.

Without `@Id`, the application will fail at startup.

---

# 5️⃣ `@GeneratedValue`

Automatically generates primary key values.

Example:

```java
@Id
@GeneratedValue
private Long id;
```

### Generation Strategies

| Strategy | Usage                                      |
| -------- | ------------------------------------------ |
| AUTO     | JPA chooses the strategy                   |
| IDENTITY | Auto-increment columns (common with MySQL) |
| SEQUENCE | Database sequences (PostgreSQL, Oracle)    |
| TABLE    | Uses a separate table for id generation    |

Example:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

### Best Practice

Always prefer **generated IDs** instead of manually assigning them.

---

# 6️⃣ `@Column`

Maps a field to a database column.

Example:

```java
@Column(name = "email")
private String email;
```

### Common Attributes

```java
@Column(
    name = "email",
    nullable = false,
    unique = true,
    length = 100
)
```

| Attribute  | Meaning                       |
| ---------- | ----------------------------- |
| name       | Column name                   |
| nullable   | Allows NULL values            |
| unique     | Adds unique constraint        |
| length     | Maximum string length         |
| insertable | Included in INSERT statements |
| updatable  | Included in UPDATE statements |

Example:

```java
@Column(nullable = false)
private String name;
```

---

# 7️⃣ `@Embeddable` (Value Objects)

Used to represent **value objects embedded within the same table**.

Example:

```java
@Embeddable
public class Address {

    private String street;
    private String city;
    private String zipCode;
}
```

This class does **not create a separate table**.

---

# 8️⃣ `@Embedded`

Used inside an entity to include an embedded object.

Example:

```java
@Entity
public class User {

    @Id
    private Long id;

    @Embedded
    private Address address;
}
```

Database table columns:

```
street
city
zip_code
```

All fields are stored in the **same table**.

### Use Cases

Embedded objects represent **value types without identity**, such as:

* Address
* Money
* Coordinates
* Audit information

---

# 9️⃣ `@Enumerated`

Used to persist **Java enum values** in the database.

Example enum:

```java
public enum UserStatus {
    ACTIVE,
    INACTIVE,
    BLOCKED
}
```

Mapping in entity:

```java
@Enumerated(EnumType.STRING)
private UserStatus status;
```

---

## Enum Storage Options

### STRING (Recommended)

Stored as:

```
ACTIVE
INACTIVE
BLOCKED
```

### ORDINAL (Risky)

Stored as:

```
0
1
2
```

Problem:

If enum order changes, stored values become incorrect.

### Best Practice

Always use:

```java
EnumType.STRING
```

---

# 🔟 Complete Example Entity

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(unique = true)
    private String email;

    @Enumerated(EnumType.STRING)
    private UserStatus status;

    @Embedded
    private Address address;
}
```

---

# ⚠️ Common Mistakes

### Missing `@Id`

Application fails at startup.

---

### No Default Constructor

JPA cannot instantiate the entity.

---

### Using `EnumType.ORDINAL`

Leads to data corruption if enum order changes.

---

### Using `final` Fields

Hibernate cannot modify them during persistence.

---

# 🧠 Mental Model

```
Java Entity Object
        ↓
JPA Mapping Annotations
        ↓
ORM Provider (Hibernate)
        ↓
Database Table Row
```

Annotations define **how Java objects map to database structure**.
