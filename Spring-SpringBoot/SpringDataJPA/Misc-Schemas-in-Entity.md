# Schemas in Spring Data JPA

This document explains how **schemas work in databases** and how they are used in **Spring Data JPA** when mapping entities to database tables.

---

# 1. What is a Schema?

A **schema** is a logical container inside a database that holds objects such as:

* Tables
* Views
* Indexes
* Functions
* Sequences

You can think of a schema as a **folder inside a database**.

Example structure:

```
Database: app_db

Schema: public
    users
    orders

Schema: admin
    users
    audit_logs
```

Here we have two tables named `users`, but they belong to different schemas:

```
public.users
admin.users
```

Because the schema is part of the identifier, these are **two different tables**.

---

# 2. Schemas in Spring Data JPA

In Spring Data JPA, schemas are specified using the `@Table` annotation.

Example:

```java
@Entity
@Table(
    name = "users",
    schema = "public"
)
public class User {

    @Id
    private Long id;

    private String name;
}
```

This maps the entity to the database table:

```
public.users
```

---

# 3. SQL Equivalent

The JPA mapping above corresponds to SQL like:

```sql
SELECT * FROM public.users;
```

---

# 4. Database Hierarchy

The hierarchy of database objects typically looks like this:

```
Database Server
    └── Database (Catalog)
            └── Schema
                    └── Table
```

Example:

```
Server
    └── app_db
            └── public
                    └── users
```

So the fully qualified table name becomes:

```
app_db.public.users
```

---

# 5. `@Table` Annotation Fields

The `@Table` annotation may contain three key attributes:

```java
@Table(
    name = "users",
    schema = "public",
    catalog = "app_db"
)
```

### name

Refers to the **table name**.

```
users
```

### schema

Refers to the **schema inside the database**.

```
public.users
```

### catalog

Refers to the **database itself**.

```
app_db.public.users
```

---

# 6. Default Schema

Many databases have a default schema.

For example, in PostgreSQL the default schema is:

```
public
```

So this works even without specifying a schema:

```java
@Table(name = "users")
```

Spring will automatically look for:

```
public.users
```

---

# 7. When You Need to Specify Schema

You should specify the schema when:

### Multiple schemas exist

Example:

```
public.users
admin.users
analytics.users
```

Then your entity mapping must specify which schema to use.

Example:

```java
@Table(name = "users", schema = "admin")
```

---

# 8. Multiple Schemas with Same Table Name

Databases allow tables with the **same name in different schemas**.

Example:

```
public.users
admin.users
analytics.users
```

Each table can have different columns and purposes.

---

# 9. Mapping Multiple Schemas in JPA

You can map different schemas using separate entity classes.

Example:

### Public Users

```java
@Entity
@Table(name = "users", schema = "public")
public class PublicUser {

    @Id
    private Long id;

    private String name;
}
```

### Admin Users

```java
@Entity
@Table(name = "users", schema = "admin")
public class AdminUser {

    @Id
    private Long id;

    private String role;
}
```

These map to:

```
public.users
admin.users
```

---

# 10. Java Class Name Limitations

In Java, two classes **cannot have the same name inside the same package**.

Invalid:

```
com.app.entity.User
com.app.entity.User
```

However, there are two solutions.

---

# 11. Solution 1: Different Class Names (Common Practice)

Rename the entity classes.

Example:

```
PublicUser
AdminUser
AnalyticsUser
```

Structure:

```
entity/
    PublicUser.java
    AdminUser.java
```

This is the most common approach in real projects.

---

# 12. Solution 2: Different Packages

Java allows classes with the same name if they are in different packages.

Example:

```
com.app.publicschema.User
com.app.adminschema.User
```

Structure:

```
com.app.publicschema
    User.java

com.app.adminschema
    User.java
```

---

# 13. Repositories for Each Entity

Each entity normally has its own repository.

Example:

```java
public interface PublicUserRepository
        extends JpaRepository<PublicUser, Long> {
}
```

```java
public interface AdminUserRepository
        extends JpaRepository<AdminUser, Long> {
}
```

---

# 14. Database Differences

Different databases treat schemas and catalogs differently.

| Database             | Catalog          | Schema        |
| -------------------- | ---------------- | ------------- |
| PostgreSQL           | Database         | Namespace     |
| MySQL                | Same as database | Often ignored |
| Oracle Database      | Rarely used      | User schema   |
| Microsoft SQL Server | Database         | Schema        |

---

# 15. Real-World Usage

Schemas are often used for:

* Separating application modules
* Managing permissions
* Organizing large databases
* Multi-tenant architectures

Example:

```
auth.users
payments.transactions
analytics.events
```

---

# Key Takeaway

Simple mental model:

```
Server
 └ Database (Catalog)
      └ Schema
           └ Table
```

Example:

```
app_db.public.users
```

Where:

* **app_db** → database (catalog)
* **public** → schema
* **users** → table
