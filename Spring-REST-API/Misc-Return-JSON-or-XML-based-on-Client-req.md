# 📝 Notes: Returning XML in Spring Boot

### 1️⃣ Dependency

To enable XML support:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

* Without this dependency → XML conversion will **not work**.

---

### 2️⃣ Controller Return Types

Spring **only converts objects** (POJOs) to XML/JSON automatically.

| Return Type                     | JSON         | XML          | Notes                              |
| ------------------------------- | ------------ | ------------ | ---------------------------------- |
| `String`                        | ✅ raw string | ✅ raw string | Not converted, treated as final    |
| Primitive (`int`, `long`, etc.) | ✅ raw value  | ✅ raw value  | Not wrapped, no tags               |
| Wrapper (`Integer`, `Long`)     | ✅ raw value  | ✅ raw value  | Same as primitives                 |
| POJO (`User`, `NumberResponse`) | ✅ structured | ✅ structured | **Recommended for XML**            |
| Collections (`List`, `Map`)     | ✅ structured | ✅ structured | XML root may default to class name |

> 💡 **Rule:** Anything that is not a POJO is **not automatically structured for XML**.

---

### 3️⃣ Optional: Customize root tag

Use `@JacksonXmlRootElement` to control XML root:

```java
@JacksonXmlRootElement(localName = "user")
public class User {
    private String name;
    private int age;
    // getters & setters
}
```

* Without it → root defaults to class name.

---

### 4️⃣ Produces / Accept Header

Spring picks response format based on **client request**:

```java
@GetMapping(value = "/user", produces = {"application/json","application/xml"})
```

* Client sends `Accept: application/xml` → XML returned
* Client sends `Accept: application/json` → JSON returned

> ✅ Browser usually defaults to JSON. Use Postman or curl to test XML.

---

### 5️⃣ Best Practice

1. **Always return POJOs** instead of String/primitive for structured XML.
2. **Use wrapper POJOs** for single primitive values:

```java
public class NumberResponse {
    private int value;
    public NumberResponse(int value) { this.value = value; }
    // getters & setters
}
```

3. **Use `@JacksonXmlRootElement`** only if you need a custom root name.
4. **Accept header controls the format**, not the return type.

---

### 6️⃣ Example

```java
@GetMapping("/user")
public User getUser() {
    return new User("John", 25);
}
```

**Postman Header**: `Accept: application/xml`

**Response**:

```xml
<User>
    <name>John</name>
    <age>25</age>
</User>
```

---

✅ **Key Takeaways**

* Dependency → required
* POJO → required for XML structure
* String / primitive → not converted
* Accept header → controls JSON vs XML
