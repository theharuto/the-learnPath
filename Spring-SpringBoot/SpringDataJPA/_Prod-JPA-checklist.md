# 🚀 Production-Grade JPA Checklist

---

# 🔹 1. Entity Design Rules

## ✅ Always initialize collections

```java
@Builder.Default
private Set<Course> courses = new HashSet<>();
```

✔ Prevents `NullPointerException`
✔ Works with Lombok builder

---

## ❌ Never expose mutable collections

```java
public Set<Course> getCourses() {
    return Collections.unmodifiableSet(courses);
}
```

✔ Protects internal state
✔ Prevents accidental modification

---

## ❌ Never include relationships in `toString()`

```java
@ToString(exclude = "courses")
```

✔ Prevents:

* Lazy loading issues
* Infinite recursion

---

## ❌ Never include relationships in `equals()` / `hashCode()`

👉 Use only ID:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Student)) return false;
    return id != null && id.equals(((Student) o).id);
}

@Override
public int hashCode() {
    return getClass().hashCode();
}
```

✔ Prevents:

* StackOverflowError
* Broken `Set` behavior

---

# 🔹 2. Relationship Rules

---

## ✅ Always define owning side clearly

```java
@ManyToMany
@JoinTable(...)
private Set<Course> courses;
```

✔ Only this side updates DB

---

## ❌ Never rely on inverse side for persistence

```java
course.getStudents().add(student); // ❌ ignored
```

---

## ✅ Always use helper methods

```java
public void addCourse(Course course) {
    this.courses.add(course);
    course.students.add(this);
}
```

✔ Keeps object graph consistent

---

## ❌ Never call getters inside helper methods

```java
course.students.add(this); // ✅
course.getStudents().add(this); // ❌
```

---

# 🔹 3. Fetching Strategy (CRITICAL)

---

## ❌ Never change everything to EAGER

```java
@ManyToMany(fetch = FetchType.EAGER) // ❌ avoid
```

✔ Causes performance issues (N+1, huge joins)

---

## ✅ Keep default LAZY

```java
@ManyToMany // LAZY by default
```

---

## ✅ Use `JOIN FETCH` when needed

```java
@Query("""
    SELECT s FROM Student s
    JOIN FETCH s.courses
    WHERE LOWER(c.title) = LOWER(:title)
""")
List<Student> findWithCourses(String title);
```

✔ Fetch only when required

---

# 🔹 4. Transaction Rules

---

## ❗ Golden Rule

> LAZY fields need an **active transaction**

---

## ✅ Use `@Transactional` at service layer

```java
@Transactional
public List<Student> getStudents() {
    return studentRepository.findAll();
}
```

---

## ❌ Avoid using it in controllers blindly

👉 Keep transactions in **service layer only**

---

# 🔹 5. Query Design

---

## ✅ Use correct field names in derived queries

```java
findAllByCoursesTitle(String title);
```

✔ Based on **field path**, not class name

---

## ✅ Prefer explicit queries for complex joins

```java
@Query("SELECT s FROM Student s JOIN s.courses c WHERE c.title = :title")
```

---

# 🔹 6. Persistence Rules

---

## ❌ Don’t save transient references

```java
student.addCourse(new Course(...)); // ❌ risky
```

---

## ✅ Save or reuse managed entities

```java
courseRepository.save(course);
student.addCourse(course);
```

---

## ⚠️ Be careful with cascade

```java
@ManyToMany(cascade = CascadeType.PERSIST)
```

✔ Use only when you fully control lifecycle

---

# 🔹 7. API Layer (VERY IMPORTANT)

---

## ❌ Never return entities directly

```java
return studentRepository.findAll(); // ❌
```

---

## ✅ Use DTOs

```java
record StudentDTO(Long id, String name) {}
```

✔ Prevents:

* Lazy loading issues
* Infinite recursion
* Over-fetching

---

# 🔹 8. Debugging Checklist

---

## 🔥 When you see:

### ❗ `LazyInitializationException`

Ask:

* Am I outside transaction?
* Is `toString()` accessing lazy field?

---

### ❗ `NullPointerException` on collection

Ask:

* Did I use `@Builder.Default`?

---

### ❗ Duplicate data

Ask:

* Am I reusing same entity instance?
* Am I saving twice?

---

### ❗ Relationship not saved

Ask:

* Am I updating owning side?

---

# 🔹 9. Performance Awareness

---

## ❗ Watch for N+1 problem

Example:

```java
students.forEach(s -> s.getCourses()); // 💣 multiple queries
```

---

## ✅ Fix with fetch join

```java
SELECT s FROM Student s JOIN FETCH s.courses
```

---

# 🔹 10. Clean Architecture Rule

---

## 🧠 Golden Separation

| Layer      | Responsibility                |
| ---------- | ----------------------------- |
| Entity     | DB mapping                    |
| Repository | DB access                     |
| Service    | Business logic + transactions |
| Controller | HTTP layer                    |

---

# 🔹 11. Final Mental Models

---

## 🧠 Model 1: JPA ≠ Database

> JPA manages **object graph**, not just tables

---

## 🧠 Model 2: Owning Side Rule

> Only owning side writes relationship

---

## 🧠 Model 3: LAZY Rule

> LAZY = “only works inside transaction”

---

## 🧠 Model 4: Collections Rule

> Collections must always be:

* initialized
* synchronized (both sides)

---

# 🏁 Final Senior Advice

---

## 💡 1. Start simple

👉 Use **unidirectional** unless needed

---

## 💡 2. Add bidirectional only when necessary

👉 Complexity grows fast

---

## 💡 3. Always think:

> “Will this break when scaled?”

---

## 💡 4. If confused:

👉 Log SQL:

```properties
spring.jpa.show-sql=true
```

---

# 🚀 Final Thought

What you just went through covers:

* Mapping errors ✅
* Lifecycle issues ✅
* Lazy loading traps ✅
* Query mistakes ✅
* Lombok pitfalls ✅

👉 This is literally **80% of real-world JPA problems**

---

If you want next level, I can give:
👉 “Top 10 JPA interview traps (with real-world scenarios)”
👉 or “How senior engineers design relationships (DDD style)”
**
