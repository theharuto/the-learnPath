# 📘 JPA Many-to-Many — Mistakes, Rules & Senior Insights

---

# 🔴 1. Wrong Annotation for Collection

## ❌ Mistake

```java
@ManyToOne
Set<Course> courses;
```

## 💥 Problem

* `@ManyToOne` expects a **single entity**, not a collection
* Causes:

```
targets the type 'java.util.Set' which is not an '@Entity'
```

## ✅ Rule

> Collection → use `@OneToMany` or `@ManyToMany`

## ✅ Fix

```java
@ManyToMany
Set<Course> courses;
```

---

# 🔴 2. Wrong `mappedBy` Value

## ❌ Mistake

```java
@ManyToMany(mappedBy = "students")
```

## 💥 Problem

* `mappedBy` must match **field name in owning entity**
* Not class name

## ✅ Rule

> `mappedBy` = exact field name on owning side

## ✅ Fix

```java
@ManyToMany(mappedBy = "courses")
Set<Student> students;
```

---

# 🔴 3. Confusion About Owning Side

## ❌ Mistake

Thinking:

> Owning side = where foreign key exists

## 💥 Reality

* In `@ManyToMany`, **no FK in entity tables**
* FK exists in **join table**

## ✅ Rule

> Owning side = side that defines `@JoinTable` and controls updates

## ✅ Example

```java
@ManyToMany
@JoinTable(...)
Set<Course> courses; // owning side
```

---

# 🔴 4. Missing Helper Methods (Object Graph Bug)

## ❌ Mistake

```java
student.getCourses().add(course);
```

## 💥 Problem

* Only one side updated
* Object graph becomes inconsistent

## ✅ Rule

> Always sync both sides in bidirectional relationships

## ✅ Fix

```java
public void addCourse(Course course) {
    this.courses.add(course);
    course.students.add(this);
}
```

---

# 🔴 5. Lombok `@Builder` Nulling Collections

## ❌ Mistake

```java
@Builder
Set<Course> courses = new HashSet<>();
```

## 💥 Problem

* Builder ignores field initialization
* Leads to:

```
NullPointerException
```

## ✅ Rule

> Always use `@Builder.Default` for collections

## ✅ Fix

```java
@Builder.Default
Set<Course> courses = new HashSet<>();
```

---

# 🔴 6. Using Getter Inside Helper (Immutable Trap)

## ❌ Mistake

```java
course.getStudents().add(this);
```

With:

```java
return Collections.unmodifiableSet(students);
```

## 💥 Problem

```
UnsupportedOperationException
```

## ✅ Rule

> Inside entity → access field directly, not getter

## ✅ Fix

```java
course.students.add(this);
```

---

# 🔴 7. Not Saving Referenced Entities (Transient Issue)

## ❌ Mistake

```java
studentRepository.save(student);
```

(with unsaved courses)

## 💥 Problem

```
TransientObjectException
```

## ✅ Rule

> Persist related entities OR use cascade

## ✅ Fix

```java
courseRepository.save(course);
studentRepository.save(student);
```

---

# 🔴 8. Duplicate Data in Many-to-Many

## ❌ Mistake

Saving same logical entity multiple times

## 💥 Problem

* Duplicate rows in `course` table

## ✅ Rule

> Reuse managed entities, don’t recreate them

---

# 🔴 9. Wrong Derived Query Method

## ❌ Mistake

```java
findAllByCourseTitle(...)
```

## 💥 Problem

* Uses class name instead of field name

## ✅ Rule

> Use **field names**, not class names

## ✅ Fix

```java
findAllByCoursesTitle(String title);
```

---

# 🔴 10. LazyInitializationException (VERY IMPORTANT)

## ❌ Mistake

```java
System.out.println(student);
```

With:

```java
@ToString
```

## 💥 Problem

* `toString()` triggers lazy loading
* Session already closed

```
LazyInitializationException
```

---

## ❗ Root Cause

> Accessing LAZY field outside Hibernate session

---

## ✅ Rule (CRITICAL)

> NEVER include LAZY collections in `toString()`, `equals()`, `hashCode()`

---

## ✅ Fix 1 (Best)

```java
@ToString(exclude = "courses")
```

---

## ✅ Fix 2

```java
@Transactional
```

---

## ✅ Fix 3 (Best for queries)

```java
@Query("SELECT s FROM Student s JOIN FETCH s.courses WHERE LOWER(c.title) = LOWER(:title)")
```

---

# 🔴 11. Wrong Assumption About Fetch Type

## ❌ Mistake

> ManyToMany is eager by default

## ✅ Reality

| Relation   | Default |
| ---------- | ------- |
| ManyToOne  | EAGER   |
| OneToMany  | LAZY    |
| ManyToMany | LAZY    |

---

# 🔴 12. Join Table Misunderstanding

## ❌ Mistake

Thinking FK is in entity

## ✅ Rule

> ManyToMany → FK lives in join table

---

## ✅ Example

```java
@JoinTable(
    name = "student_course",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id")
)
```

---

# 🧠 Senior-Level Insights

---

## 💡 1. Default Strategy

> Prefer **unidirectional** unless you NEED reverse navigation

---

## 💡 2. Avoid EAGER

> EAGER = hidden performance killer

---

## 💡 3. Entity ≠ DTO

> Never expose entities directly in APIs
> → causes recursion + lazy issues

---

## 💡 4. Think in Object Graphs

> JPA is about **navigating objects**, not tables

---

## 💡 5. Helper Methods Are Mandatory

> Not for JPA — for **your sanity**

---

## 💡 6. Debug Rule

When you see:

```
LazyInitializationException
```

👉 Ask:

> “Am I outside a transaction?”

---

## 💡 7. Golden Rule

> Owning side controls persistence
> Both sides must stay in sync in memory

---

# ✅ Final Clean Version (Reference)

```java
@Entity
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ToString(exclude = "courses")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Builder.Default
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();

    public void addCourse(Course course) {
        this.courses.add(course);
        course.students.add(this);
    }
}
```

---

```java
@Entity
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ToString(exclude = "students")
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @Builder.Default
    @ManyToMany(mappedBy = "courses")
    Set<Student> students = new HashSet<>();
}
```
