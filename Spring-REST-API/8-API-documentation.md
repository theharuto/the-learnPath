# 📦 Module 10: API Documentation (Swagger / OpenAPI)

---

# 🔹 0. Big Picture (Why This Exists)

Imagine your API without documentation:

```text
Client → "What endpoints exist?"
Client → "What request format?"
Client → "What response?"
```

👉 They must read code or ask developers

---

## With API Docs

```text
Client → Opens Swagger UI
→ Sees endpoints
→ Tries API directly
→ Understands request/response instantly
```

---

## Core Idea

> API documentation = **machine-readable contract + interactive UI**

---

# 🔹 1. What is OpenAPI / Swagger?

---

## OpenAPI

> Standard specification to describe REST APIs

---

## Swagger

> Tools built on top of OpenAPI:

* Swagger UI (interactive UI)
* Swagger Editor
* Swagger Codegen

---

## In Spring Boot

We use:

👉 **SpringDoc OpenAPI**

---

---

# 🔹 2. Setup (Spring Boot)

---

## Dependency (MUST KNOW)

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.x.x</version>
</dependency>
```

---

## What it does

* Scans controllers
* Generates OpenAPI spec
* Provides Swagger UI

---

---

## After Running App

Open:

```text
http://localhost:8080/swagger-ui.html
```

or

```text
http://localhost:8080/swagger-ui/index.html
```

---

---

# 🔹 3. Auto Documentation (Default Behavior)

---

## Example Controller

```java
@RestController
@RequestMapping("/resources")
public class ResourceController {

    @GetMapping("/{id}")
    public Resource get(@PathVariable Long id) {
        return new Resource(id, "data");
    }
}
```

---

## What Swagger Generates

* Endpoint: GET /resources/{id}
* Input: path variable id
* Output: Resource JSON

---

## Internal Working

```text
Spring scans annotations
→ Builds OpenAPI model
→ Exposes JSON spec (/v3/api-docs)
→ Swagger UI renders it
```

---

---

# 🔹 4. `@Operation` (Core Annotation)

---

## Purpose

> Describes what an API does

---

## Example

```java
@Operation(
    summary = "Get resource by ID",
    description = "Fetches a resource using its unique identifier"
)
@GetMapping("/{id}")
public Resource get(@PathVariable Long id) {
    return new Resource(id, "data");
}
```

---

## Output in Swagger UI

* Title
* Description
* Clear documentation

---

---

# 🔹 5. `@ApiResponses` and `@ApiResponse`

---

## Purpose

> Define possible HTTP responses

---

## Example

```java
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Success"),
    @ApiResponse(responseCode = "404", description = "Resource not found"),
    @ApiResponse(responseCode = "400", description = "Invalid input")
})
@GetMapping("/{id}")
public Resource get(@PathVariable Long id) {
    return new Resource(id, "data");
}
```

---

## Why Important

* Clients know all possible outcomes
* Improves API reliability

---

---

# 🔹 6. Documenting Request Body

---

## Example

```java
@Operation(summary = "Create resource")
@PostMapping
public Resource create(
        @RequestBody Resource request) {
    return request;
}
```

---

## Swagger shows:

* JSON schema
* Required fields
* Data types

---

---

# 🔹 7. Documenting Parameters

---

## Path Variable

```java
@Parameter(description = "ID of resource")
@PathVariable Long id
```

---

## Request Param

```java
@Parameter(description = "Page number")
@RequestParam int page
```

---

---

# 🔹 8. Full Example (Production-Style)

---

```java
@RestController
@RequestMapping("/resources")
public class ResourceController {

    @Operation(summary = "Get resource by ID")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Success"),
        @ApiResponse(responseCode = "404", description = "Not found")
    })
    @GetMapping("/{id}")
    public Resource get(@PathVariable Long id) {
        return new Resource(id, "data");
    }

    @Operation(summary = "Create new resource")
    @ApiResponse(responseCode = "201", description = "Created")
    @PostMapping
    public ResponseEntity<Resource> create(
            @RequestBody Resource request) {

        return ResponseEntity.status(201).body(request);
    }
}
```

---

---

# 🔹 9. Swagger UI (What You See)

![Image](https://images.openai.com/static-rsc-4/rOHieC27XdFbwQoTiMa5OGKHFYUD-jfRCwP1sN1rgoOM_h4qxiKK8V79n0ejoLxQa1jMmGMojCGtfvWvzLCx9aJzhB3V_TCYWOuOM0wGQmZawHMeKGwmWnVuom9w0cY5w7XJ0ZbSAhEhqTGQ75-qYO3OPHjdIqAxRydSGkWqvOlCMml1ZNSYCmLn8L8Mmdi1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/o2j6OO7KZ4YCXsFob_4FHxab3d6Oiwo6bvWG6zMp2YBNtE9WfS7FX-gLCQJPS8sXEfh8MIre0LQeKHbfDuIXACPRTMDewn3EVDx0iP4oLT06BydVzdpL8PaJd7z0tnnSpaNpchN1jikA9Ec7YhEcJRY8IDdHx-30vpF3o_4p83G7dOMbXpNvrqNHuJXktb0o?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/g9_jHaBrf4Sh5I00NJRX22C9veJsoMmlN43tl1olLJEHViwW_mNaJpgmWv_KcbghVPlcsBE3oqjuMkrDwLDifrvwhUaJBxB5brbhhbflHM17jxXRHV63qoNgB6GW1oePUZofOUlms5r1lkz9E9kBy-waZgjeF-8XbOYk8Svau_y0LDI7nj6kag5GYBGhlL1H?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/r0J_lVdVpQqZmKe5tp9gYyQImEGINjk2WKIc41RIUVVZOqf5dSH9-2I-IvLItKanXGjn_wwK5SzzkMefRbz_hqsmpRZR7SSj4Mv2703EBqFk_9p-eeYyj7hWudTf3m8C3JkWG2dGMhK0ivTNAy4Jjwn9izXVMtFCMSTuKmVFv8lMic_FvqnEnKkzF9Ay1dBg?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/MBNnH2s2yv5rT3AEgMPaVTaY4I1m69wJ_GxvH3j8T6QatDHrt16YsDbixflTceKZ5nb6fprZjzT0flhRUv_LVb165SYcWQuYKd1osq8J3NVv4q8riu0uwOJrZvGm_svm7_TrT1epNTXED2So7RC5SWthjZuJbykvpCSYsHJAi-4_TKRYqBb4SvvRu7b33pD1?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-kOMsCo-h3HGt9GZAQ8VmBX7BZX8tYA_4FeChxTu0BnSY_vAngXuWQgtcKRRDjcb63hjT-wCmPAAzdNvLl5nITS4XHhiOUP-0tOHnQTwrCj2tmVZAOY6We8at0VfqLCy1iM9z3mVvOLX4oPs7VgKTnWwTtN1u2AzjfEpcm2M_Og69IxVwYFlFau9kEaigeAM?purpose=fullsize)

---

## Features

* List all endpoints
* Expand each API
* Try requests directly
* See request/response schema

---

---

# 🔹 10. Advanced Customization

---

## Global Info

```java
@OpenAPIDefinition(
    info = @Info(
        title = "My API",
        version = "1.0",
        description = "API Documentation"
    )
)
```

---

## Grouping APIs

```java
@Tag(name = "Resource APIs")
```

---

---

# 🔹 11. Production-Level Concerns

---

## 1. Security

❌ Don’t expose Swagger in production publicly

---

## Solution

* Disable in prod OR
* Protect with authentication

---

---

## 2. Accuracy

Docs must match actual behavior:

❌ Wrong status codes
❌ Missing error responses

---

---

## 3. Versioning

```text
/v1/api-docs
/v2/api-docs
```

---

---

## 4. DTO Documentation

Always document DTOs:

* Required fields
* Formats

---

---

# 🔹 12. Common Mistakes

---

❌ Not documenting APIs
❌ Missing error responses
❌ Outdated docs
❌ Exposing internal models
❌ Ignoring request/response examples

---

---

# 🔹 13. Final Mental Model

```text
API Documentation =
Contract between backend and client
```

---

```text
Swagger =
Interactive way to explore that contract
```

---

---

# 📘 GitHub Notes (Markdown)

---

# 📘 API Documentation (Swagger / OpenAPI)

---

## 🔹 OpenAPI

* Standard for describing REST APIs

---

## 🔹 Swagger

* Tools for OpenAPI
* Provides UI for API testing

---

## 🔹 SpringDoc

* Integrates OpenAPI with Spring Boot
* Auto-generates API docs

---

## 🔹 Key Annotations

### @Operation

* Describes API purpose

### @ApiResponse

* Defines response codes

### @ApiResponses

* Multiple responses

---

## 🔹 Features

* Auto API documentation
* Interactive UI
* Request/response schema

---

## 🔹 Swagger UI

* Test APIs directly
* View endpoints
* Explore request formats

---

## 🔹 Best Practices

* Document all endpoints
* Include error responses
* Keep docs updated
* Use DTOs

---

## 🔹 Production Concerns

* Secure Swagger UI
* Avoid exposing sensitive data

---

## 🔹 Key Takeaways

* Documentation improves usability
* Swagger simplifies API exploration
* OpenAPI defines the contract


---

* Global OpenAPI metadata (`@OpenAPIDefinition`)
* Grouping endpoints using `@Tag`

---

# 🔹 1. Global API Info (Configuration Class)

```java
import io.swagger.v3.oas.annotations.OpenAPIDefinition;
import io.swagger.v3.oas.annotations.info.Info;
import org.springframework.context.annotation.Configuration;

@Configuration
@OpenAPIDefinition(
    info = @Info(
        title = "Resource Management API",
        version = "1.0",
        description = "APIs for managing resources in the system"
    )
)
public class OpenApiConfig {
}
```

---

## 🔹 What this does

* Sets global metadata visible in Swagger UI:

  * Title at the top
  * Version
  * Description

👉 Think:

```text
This describes your entire API system
```

---

# 🔹 2. Grouping APIs using `@Tag`

```java
import io.swagger.v3.oas.annotations.tags.Tag;
import io.swagger.v3.oas.annotations.Operation;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/resources")
@Tag(name = "Resource APIs", description = "Operations related to resources")
public class ResourceController {

    @Operation(summary = "Get resource by ID")
    @GetMapping("/{id}")
    public String get(@PathVariable Long id) {
        return "Resource " + id;
    }

    @Operation(summary = "Create a new resource")
    @PostMapping
    public String create(@RequestBody String body) {
        return "Created";
    }
}
```

---

## 🔹 What this does

* Groups endpoints under **"Resource APIs"** section in Swagger UI
* Improves readability when many controllers exist

👉 Without tags:

```text
All endpoints appear in one flat list ❌
```

👉 With tags:

```text
Grouped by domain/module ✔
```

---

# 🔹 How it Looks (Conceptually)

![Image](https://images.openai.com/static-rsc-4/f8Wnt57CzUkLTBznvxrxTxTwrpgob1mUhYTgEhkTFvsf7gv5wJuS4qoeeBXf0h_iX8TBBWqxaZLzm1ZUpr2TMb_iQkNPKSWQc5NALrOM272kcASgb2LlIYF8BMnFWYb0tqmanrtgAoaxGb22ymWjIpHRTQ5izMezNc7jR3eP_YtzLBMJ0f7gxYwFbHnnWtYk?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/iVnSMRO04xwKBDcsH71HtN0vNgUVH3Ma0JYkt3jaq77BhZqSE1CxHz5zHxaKv6x1u8X-skOsLFdUzwkVrXSsq6OEw1sz3DlblIT0CAR7lHnvOTgAjVsGWStSAoG3Iq2PG6wRmTVi4KcuO1vPUCRUp2Btt8kMgU-Qq7wtIlBo_nN7k_XaMIdabbAiEMOU1MTS?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/A16NX4LqT7KownCyjCNMa_CCFgWJecC8eKpVxBDZrlEGaT0XS1B-8Yy0hr7-7sU-y6oms3NVKJZKiVEk0TROm1E6e6MWyV9g1Gxx0IgSNkGdBfv0jnsqOjt44IBM_I9GySR3rUf3shD3__3FkGVjS6yUBYpEO0MWb9HLCGzcMCNt0hIo0-otVIGgpt8WLFwF?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/oeheidm4DepFY_I976emU_EO2jiS9L-68vtOxAsEpeP5gk7-7wDeV-ss6L5q03wlDGJbFTfcwg98NGkkCD1dJ6ILvdrL1C8k61NzEcSPYhyqAQc3SVw613FT7FZgRto-dhP4jZxE5Alw10pmTi-DSn5z__FqQbvS_6vVsQ7ZU-_J_4RQcntbyStGb6idEACC?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/s49nZN8OhSdFSfsUrb2k_aV-d2bpF76_9eYjHtj-IIFWD0c5eHlIbjoDfyp6FlFSO9NXvg-7wshN5y7c4U-l4Rl9OSM3DJhkS6xCZmQ3YJtbKKZwy1NA7P6nns27eG_C2iZi2BWC3z3vqSt0QYtFRMb8r3JgbDo_icPQPbcKNR2HUrwrqdud_pjDMcqAVc7Z?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vF8lc_7uQZViXU7ZQMIC8DatSnaZptXqx7h2cOeRSJDD-3FdSLahToWqCSGh_DP6MaJPCVZFseR_uzRsUH0kPeSjgPg4nGmPDkKvQ1SJwfBy33iw8WTgnXDMEl8GMG0-u8ZScClF67FHCBdLwGusAPQlM3t2VN8aPA0AKP6j1E4KEotUlgntPQ0qZWNuioHu?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/AGCz-R9-UE-R_G8Dyknq772dU7S1ww13VSUZUbqjc23wc3xtgjZqJ-eFRkTsOu7a_NUQwsuL25vNFQzeYpvb6LSD6Cuz_fheVUu_96AIayJysN7IMALyDw4bMGbW5IxYWuIC-oJHzKPaQ4j_3CGhyRMrd9tkvTx7K5CdxYpE8rRy8nGVcHFRmst25CHI2hBa?purpose=fullsize)

---

# 🔹 When to Use This in Production

---

## Use `@OpenAPIDefinition` when:

* You want clear API identity
* Multiple teams consume your API

---

## Use `@Tag` when:

* You have multiple modules:

  * User APIs
  * Order APIs
  * Payment APIs

---

# 🔹 Final Mental Model

```text
@OpenAPIDefinition → describes the whole API system

@Tag → organizes endpoints into logical groups
```

---

If you want, next we can:
👉 Add **security (JWT / auth headers) into Swagger**
👉 Or move to **Filtering & Dynamic Queries (Specifications)**
