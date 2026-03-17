# 🧱 Step 1 — Entity Lifecycle in Hibernate

Every entity in Hibernate can be in **one of three states**:

| State                    | Description                             | How to get there                            | DB interaction                                 |
| ------------------------ | --------------------------------------- | ------------------------------------------- | ---------------------------------------------- |
| **Transient**            | New object, not in DB, not tracked      | `new Author("X")`                           | No DB entry yet                                |
| **Managed / Persistent** | Object is attached to Hibernate session | `repo.save(entity)` or `repo.findById()`    | Changes automatically tracked                  |
| **Detached**             | Previously managed, session closed      | After transaction ends or session is closed | No automatic DB updates, Lazy collections fail |

### Example:

```java
// TRANSIENT
Author a = new Author();
a.setName("Orwell"); // Just a Java object, no DB

// MANAGED
Author saved = authorRepository.save(a); // now Hibernate session manages it

// DETACHED
// After method ends, 'saved' is detached. You can't lazy-load anymore.
```

✅ **Key insight:** Once the session closes, **all objects retrieved from the DB are detached**. Any lazy-loaded collection or proxy will throw `LazyInitializationException` if accessed outside a session.

---

# 🔄 Step 2 — How Session Works

Think of **Session** as a **lifetime of DB tracking**:

* Session starts → JPA opens EntityManager
* Inside session:

  * All `find`, `save`, `update` calls happen here
  * Lazy collections/proxies can be initialized
* Session ends → EntityManager closed → objects become detached

In Spring Data JPA:

```text
@Repository methods are transactional by default (readOnly=false)
→ Session opens for that repository call
→ Session closes when method returns
```

---

# 🧩 Step 3 — Lazy vs Eager

### 1. Lazy (`fetch = FetchType.LAZY`)

* Hibernate creates a **proxy** for the collection
* Actual DB query happens **only when you access it**
* **Requires an open session**

```java
Author a = authorRepository.findById(1).get(); // books not loaded
a.getBooks(); // triggers query
```

💥 If session is closed → `LazyInitializationException`

### 2. Eager (`fetch = FetchType.EAGER`)

* Hibernate loads collection immediately
* Works even if session is closed
* Bad for big collections (memory + performance hit)

### 3. Best practice

* Use **lazy loading**
* Explicitly fetch collections inside transaction:

  * `@Transactional`
  * Or custom query with `JOIN FETCH`

---

# 🛠 Step 4 — Real World Flow

Let’s model your `Author` → `Book` example

```text
1️⃣ Create author
   Author a = new Author("Orwell"); // transient
   authorRepository.save(a);        // now managed

2️⃣ Add book to author
   Book b = new Book("1984", "1234");
   b.setAuthor(a);
   bookRepository.save(b);          // now book is managed

3️⃣ Access books later
   Author author = authorRepository.findById(a.getId());
   author.getBooks();               // ❌ Lazy fails if session closed
```

✅ **Solution:** keep session open while accessing books or use `JOIN FETCH`:

```java
@Query("SELECT a FROM Author a JOIN FETCH a.books WHERE a.id = :id")
Author findByIdWithBooks(@Param("id") Long id);
```

---

# 💡 Step 5 — Detachment Confusion

Your previous error:

```text
author.getBooks() → LazyInitializationException
```

Why?

1. `savedAuthors = authorService.getAllAuthors()`

   * Repository call ends → session closes → authors become **detached**
2. `savedAuthors.forEach(author -> author.getBooks())`

   * Trying to access lazy collection → session is gone → Hibernate fails

**Important:** Just calling `author.getBooks()` is **DB access**, even if it feels like a simple Java call.

---

# 🔑 Step 6 — Rule of Thumb for Lazy

Whenever you see **`@OneToMany(fetch=LAZY)`**:

1. Access only inside a **transaction**
2. Or initialize explicitly:

   * `Hibernate.initialize(author.getBooks())`
   * `authorRepository.findByIdWithBooks(id)` (join fetch)
3. Never assume lazy is “just a list” in detached objects

---

# 🧠 Step 7 — Ownership Matters in Relationships

```java
class Author {
    @OneToMany(mappedBy="author")
    List<Book> books;
}

class Book {
    @ManyToOne
    Author author;
}
```

✅ `Book` is the **owning side** (`author_id` FK exists in `Book`)

Rule:

* Changes to `author.getBooks()` alone **do NOT** update DB
* You must set `book.setAuthor(author)` and save `book`
* Cascade helps, but you must know which side is owning

---

# 🏗 Step 8 — How to safely add a book to an author

```java
@Transactional
public Book addBookToAuthor(Long authorId, Book book) {
    Author author = authorRepository.findById(authorId)
                     .orElseThrow(() -> new RuntimeException("Not found"));

    book.setAuthor(author);
    return bookRepository.save(book); // now DB is updated
}
```

* `@Transactional` keeps session open
* Saves book to DB
* Lazy collections can be accessed **inside transaction**

---

# 🧩 Step 9 — What NOT to do

```java
Author author = authorRepository.findById(id).get();
author.getBooks().add(new Book(...)); // ❌ does NOT save book unless saved explicitly
```

* Adding to `List` alone is **just Java memory**
* DB doesn’t know → you get detached errors

---

# ✅ Step 10 — Your Mental Map

```
Transient → Managed → Detached
      ↑         ↑
      | save    | session closed
      |         |
    DB insert   Lazy fails outside session
```

* **Lazy** = proxy, only works when managed
* **Owning side** = only side that updates DB
* **Session** = temporary life of DB-tracked object
