# 🧠 What happens in your code

```java
@GetMapping("/{id}")
public ResponseEntity<Resource> get() {
    Resource res = new Resource(1, "data");

    return ResponseEntity.ok()
        .header("X-Request-Id", "abc-123")
        .header("X-Source", "internal-service")
        .header("Cache-Control", "no-cache")
        .body(res);
}
```

1. `X-Request-Id` and `X-Source` → custom headers, nothing to do with caching.
2. `Cache-Control: no-cache` → **tells the client (browser or proxy) not to use a cached copy without revalidation**.

---

# 🔹 Important nuance: `no-cache` vs `no-store`

| Directive   | Meaning                                                                                                        |
| ----------- | -------------------------------------------------------------------------------------------------------------- |
| `no-cache`  | The client **must revalidate** with the server before using cached data. The browser may store it temporarily. |
| `no-store`  | The client **must not store** the response **anywhere** (memory, disk, etc.). Use this for sensitive data.     |
| `max-age=0` | The cached response is immediately stale. The client may still revalidate.                                     |

---

### ✅ If you want the browser **not to cache at all**:

```java
return ResponseEntity.ok()
    .header("Cache-Control", "no-store, no-cache, must-revalidate")
    .header("Pragma", "no-cache")
    .header("Expires", "0")
    .body(res);
```

* `Pragma` → legacy HTTP 1.0 support
* `Expires=0` → ensures older browsers don’t cache

---

### 💡 Example behavior

* With your current `no-cache` only:

  * Browser may store the response in memory/disk
  * But **will check with server** before using it

* With `no-store`:

  * Browser **won’t save the response** at all

---

### ⚡ TL;DR

* `Cache-Control: no-cache` → **revalidate first** before using cache
* `Cache-Control: no-store` → **never cache**
* For sensitive or dynamic data (like a resource download), prefer `no-store`
