In real production projects using Spring Boot with Hibernate (JPA implementation) and the Jakarta Persistence API, entities usually contain **several groups of annotations**:

1. **Basic JPA mapping annotations**
2. **Relationship annotations**
3. **Validation annotations**
4. **Column configuration**
5. **Lifecycle & auditing**
6. **Serialization / utility annotations**
7. **Performance / advanced mapping**

Below is a **complete practical list used in production systems**.

---

# 1️⃣ Basic Entity Annotations (Mandatory / Core)

### `@Entity`

Marks the class as a JPA entity.

```java
@Entity
```

### `@Table`

Maps entity to database table.

```java
@Table(name = "users")
```

### `@Id`

Primary key.

```java
@Id
```

### `@GeneratedValue`

Auto-generate ID.

```java
@GeneratedValue(strategy = GenerationType.IDENTITY)
```

Strategies:

* `IDENTITY`
* `AUTO`
* `SEQUENCE`
* `TABLE`

Example:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

---

# 2️⃣ Column Mapping Annotations

### `@Column`

Customize column details.

```java
@Column(name="email", nullable=false, unique=true, length=100)
```

Common attributes:

| Attribute  | Purpose           |
| ---------- | ----------------- |
| name       | column name       |
| nullable   | allow null        |
| unique     | unique constraint |
| length     | varchar length    |
| updatable  | update allowed    |
| insertable | insert allowed    |

Example:

```java
@Column(nullable = false, unique = true)
private String email;
```

---

# 3️⃣ Relationship Mapping

Used when entities relate to each other.

### One-to-One

```java
@OneToOne
@JoinColumn(name="profile_id")
```

### One-to-Many

```java
@OneToMany(mappedBy="user")
```

### Many-to-One

```java
@ManyToOne
@JoinColumn(name="user_id")
```

### Many-to-Many

```java
@ManyToMany
@JoinTable(
    name="user_roles",
    joinColumns=@JoinColumn(name="user_id"),
    inverseJoinColumns=@JoinColumn(name="role_id")
)
```

Common options:

```java
fetch = FetchType.LAZY
cascade = CascadeType.ALL
```

---

# 4️⃣ Validation Annotations (Very Important in Production)

These come from **Jakarta Validation**.

Common ones:

### Not Null

```java
@NotNull
```

### Not Empty

```java
@NotEmpty
```

### Not Blank

```java
@NotBlank
```

### Size

```java
@Size(min=3,max=50)
```

### Email

```java
@Email
```

### Pattern

```java
@Pattern(regexp="^[A-Za-z]+$")
```

### Min / Max

```java
@Min(18)
@Max(60)
```

Example:

```java
@NotBlank
@Size(min = 3, max = 50)
private String name;
```

---

# 5️⃣ Date and Time Annotations

```java
@Temporal(TemporalType.DATE)
```

Modern Java usually uses:

```java
LocalDate
LocalDateTime
```

Hibernate timestamps:

```java
@CreationTimestamp
@UpdateTimestamp
```

Example:

```java
@CreationTimestamp
private LocalDateTime createdAt;
```

---

# 6️⃣ Enum Mapping

```java
@Enumerated(EnumType.STRING)
```

Example:

```java
public enum Role {
    ADMIN,
    USER
}
```

```java
@Enumerated(EnumType.STRING)
private Role role;
```

---

# 7️⃣ Transient Fields

Not stored in database.

```java
@Transient
private String temporaryData;
```

---

# 8️⃣ Auditing Annotations (Production systems)

Used with Spring Data JPA auditing.

```java
@CreatedDate
@LastModifiedDate
@CreatedBy
@LastModifiedBy
```

Example:

```java
@CreatedDate
private LocalDateTime createdAt;
```

Requires enabling auditing:

```java
@EnableJpaAuditing
```

---

# 9️⃣ JSON / Serialization Annotations

When entities are returned in APIs.

From Jackson.

### Ignore fields

```java
@JsonIgnore
```

### Prevent infinite recursion

```java
@JsonManagedReference
@JsonBackReference
```

or

```java
@JsonIgnoreProperties
```

---

# 🔟 Lombok Annotations (Very common in production)

From Project Lombok.

Instead of writing boilerplate code:

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
```

Example:

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
```

---

# 1️⃣1️⃣ Indexes and Constraints

```java
@Table(
    name="users",
    indexes = {
        @Index(name="idx_email", columnList="email")
    }
)
```

Unique constraint:

```java
@Table(
    uniqueConstraints = {
        @UniqueConstraint(columnNames="email")
    }
)
```

---

# 🔥 Real Production Example Entity

```java
@Entity
@Table(name="users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Size(min=3,max=50)
    @Column(nullable=false)
    private String name;

    @Email
    @Column(nullable=false,unique=true)
    private String email;

    @NotBlank
    private String password;

    @Enumerated(EnumType.STRING)
    private Role role;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

---

# 💡 Production Annotation Categories

| Category       | Examples                      |
| -------------- | ----------------------------- |
| Entity         | `@Entity`, `@Table`           |
| Primary Key    | `@Id`, `@GeneratedValue`      |
| Column mapping | `@Column`                     |
| Relationships  | `@OneToMany`, `@ManyToOne`    |
| Validation     | `@NotNull`, `@Email`, `@Size` |
| Date handling  | `@CreationTimestamp`          |
| Enums          | `@Enumerated`                 |
| JSON           | `@JsonIgnore`                 |
| Utility        | Lombok                        |
| Auditing       | `@CreatedDate`                |

---

✅ **Industry tip:**
Real production entities usually combine:

* **JPA annotations**
* **Validation annotations**
* **Lombok**
* **Jackson annotations**
* **Auditing**

---
If you'd like, I can also show **the *perfect production entity template* most Spring Boot companies use**, which includes **20+ best practices** (this is very useful for interviews and real projects).
