# 📘 Spring Data JPA – Topic 4: Relationships Between Entities

In relational databases, tables are connected using **foreign keys**.
JPA models these relationships between entities using specific annotations.

Example database relation:

```text
User (1)  ────── (*)  Order
```

One **User** can have many **Orders**.

---

# 1️⃣ Types of Entity Relationships in JPA

JPA defines four main relationship mappings.

| Relationship  | Meaning                                  |
| ------------- | ---------------------------------------- |
| `@OneToOne`   | One entity relates to exactly one entity |
| `@OneToMany`  | One entity relates to many entities      |
| `@ManyToOne`  | Many entities relate to one entity       |
| `@ManyToMany` | Many entities relate to many entities    |

These relationships are implemented using **foreign keys or join tables**.

---

# 2️⃣ One-to-One (`@OneToOne`)

One record relates to exactly **one other record**.

Example:

```text
User  →  Passport
```

Each user has one passport.

### Entity Example

```java
@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    @OneToOne
    @JoinColumn(name = "passport_id")
    private Passport passport;
}
```

Passport entity:

```java
@Entity
public class Passport {

    @Id
    @GeneratedValue
    private Long id;

    private String passportNumber;
}
```

### Database Structure

```text
users
-----
id
passport_id
```

`passport_id` is a **foreign key referencing passport.id**.

---

# 3️⃣ One-to-Many (`@OneToMany`)

One parent entity relates to **multiple child entities**.

Example:

```text
User → Orders
```

One user can place many orders.

### Example

User entity:

```java
@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}
```

Order entity:

```java
@Entity
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne
    private User user;
}
```

### Database Structure

```text
orders
------
id
user_id
```

The **foreign key lives in the child table (`orders`)**.

---

# 4️⃣ Many-to-One (`@ManyToOne`)

Many records reference a **single parent entity**.

Example:

```text
Many Orders → One User
```

Example mapping:

```java
@ManyToOne
@JoinColumn(name = "user_id")
private User user;
```

This is the **most common relationship in real-world applications**.

---

# 5️⃣ Many-to-Many (`@ManyToMany`)

Many records relate to many records.

Example:

```text
Students ↔ Courses
```

A student can enroll in multiple courses, and a course can have multiple students.

### Example

```java
@Entity
public class Student {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses;
}
```

### Database Tables

```text
students
courses
student_courses
```

Join table:

```text
student_courses
---------------
student_id
course_id
```

---

# 6️⃣ Fetch Types (Lazy vs Eager)

Fetch type defines **when related entities are loaded from the database**.

---

## Lazy Fetch (`FetchType.LAZY`)

Related entities are **loaded only when accessed**.

Example:

```java
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;
```

Behavior:

```text
Load User
Orders NOT loaded yet
Orders loaded when accessed
```

---

## Eager Fetch (`FetchType.EAGER`)

Related entities are **loaded immediately with the parent entity**.

Example:

```java
@OneToOne(fetch = FetchType.EAGER)
private Passport passport;
```

Behavior:

```text
Load User
Passport loaded immediately
```

---

## Default Fetch Types (JPA Specification)

| Relationship  | Default Fetch |
| ------------- | ------------- |
| `@OneToMany`  | LAZY          |
| `@ManyToMany` | LAZY          |
| `@ManyToOne`  | EAGER         |
| `@OneToOne`   | EAGER         |

---

# 7️⃣ Cascade Types

Cascade defines how operations on a parent entity affect related entities.

Example:

```java
@OneToOne(cascade = CascadeType.ALL)
private Address address;
```

If the parent is saved or deleted, the child is also affected.

---

## Cascade Options

| Cascade Type | Description                            |
| ------------ | -------------------------------------- |
| `PERSIST`    | Persist child entity                   |
| `MERGE`      | Merge updates                          |
| `REMOVE`     | Delete child entity                    |
| `REFRESH`    | Refresh entity state                   |
| `DETACH`     | Detach entity from persistence context |
| `ALL`        | Apply all cascade operations           |

Example:

```java
@OneToMany(cascade = CascadeType.PERSIST)
private List<Order> orders;
```

Saving the user also saves the orders.

---

# 8️⃣ Unidirectional vs Bidirectional Relationships

---

## Unidirectional Relationship

Only **one entity knows about the relationship**.

Example:

```java
@OneToMany
private List<Order> orders;
```

Orders do not reference User.

---

## Bidirectional Relationship

Both entities reference each other.

Example:

User entity:

```java
@OneToMany(mappedBy = "user")
private List<Order> orders;
```

Order entity:

```java
@ManyToOne
private User user;
```

The `mappedBy` attribute indicates **which entity owns the relationship**.

---

# 9️⃣ Owning Side of Relationship

In JPA, one side must control the **foreign key**.

The owning side:

* Contains the `@JoinColumn`
* Controls updates to the relationship

Example:

```java
@ManyToOne
@JoinColumn(name = "user_id")
private User user;
```

This entity **owns the relationship**.

---

# ⚠️ Common Mistakes

### Missing `mappedBy`

Creates unnecessary join tables.

---

### Using `FetchType.EAGER` everywhere

Leads to excessive queries and performance problems.

---

### Overusing `@ManyToMany`

Often better to create an explicit **join entity**.

---

### Incorrect cascade configuration

Child entities may not persist automatically.

---

# 🧠 Mental Model

Relationships represent **foreign key connections between tables**.

```text
Entity A
   ↓
Foreign Key
   ↓
Entity B
```

JPA annotations define how these relationships are mapped between **Java objects and database tables**.
