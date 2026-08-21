# 2026-07-23 - The JSON Bloat Disaster

> **Stack:** Java 17 | Spring Boot 3 | Jackson | REST API | 1 GB heap
> **Outage Duration:** 35 minutes
> **Impact:** Memory spikes triggering stop-the-world Full GC pauses for ~20% of users; p99 latency ballooned from 120 ms to 9+ seconds

---

## 🌩️ The Calm

Our user service had a workhorse endpoint: `GET /users/{id}`. It returned a user's profile — name, email, last login — and had served millions of requests at a steady 120 ms p99. The service ran happily on a modest 1 GB heap across three pods.

The endpoint did one thing that felt convenient: it returned the JPA `User` entity **directly** from the controller. No DTO, no projection. "Why maintain two classes for the same data?" was the reasoning. For two years, that reasoning held — because the `User` entity was small and self-contained.

Then a compliance feature landed on a Thursday afternoon: every user needed an audit trail. A new field appeared on the entity — `List<AuditLog> auditLogs`, bidirectionally mapped, with power users accumulating **500+ audit entries** each. The developer tested it with their local account (3 audit logs). Response looked fine. It shipped Friday morning.

By Friday at lunch, 20% of requests — the ones hitting users with long histories — were stalling for seconds at a time while the JVM stopped the world to dig itself out of heap exhaustion.

---

## 💥 The Trigger

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    private Long id;

    private String name;
    private String email;
    private Instant lastLogin;

    // NEW FIELD — 500+ entries per power user, bidirectional.
    // No @JsonIgnore. Jackson will serialize ALL of it.
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<AuditLog> auditLogs = new ArrayList<>();

    // getters/setters omitted
}
```

```java
@Entity
@Table(name = "audit_logs")
public class AuditLog {

    @Id
    private Long id;

    private String action;
    private Instant timestamp;
    private String ipAddress;

    // The back-reference that closes the loop: User -> AuditLog -> User -> ...
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    // getters/setters omitted
}
```

```java
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserRepository userRepository;

    public UserController(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // Returning the raw entity. Jackson now owns what crosses the wire.
        return userRepository.findById(id).orElseThrow();
    }
}
```

Two compounding bombs in one return statement:

1. **The cycle:** Jackson serializes `User` → hits `auditLogs` → serializes each `AuditLog` → hits its `user` back-reference → serializes that `User` again → hits *its* `auditLogs` → ... Jackson eventually throws its `InvalidDefinitionException: Infinite recursion` guard, which Spring catches and converts — but not before building massive intermediate object graphs for *every* request.
2. **The bloat:** even where serialization aborted, the damage was done upstream. Hundreds of `AuditLog` objects (each carrying strings, timestamps, and the parent reference) were materialized into the young generation per request. On a 1 GB heap serving hundreds of concurrent requests, the object churn was relentless.

---

## 🕵️ The Hunt

**Step 1: The JVM is thrashing — GC logs**

We had GC logging enabled via JVM flags (do this on every service; it costs nothing):

```bash
-Xlog:gc*:file=/var/log/app/gc.log:time,level,tags
```

The log told the story immediately — Full GC cycles every few seconds, each one a stop-the-world pause:

```
[2026-07-23T12:14:03.412+0000][info][gc] Pause Full (Allocation Failure) 1023M->971M(1024M) 1842.336ms
[2026-07-23T12:14:07.891+0000][info][gc] Pause Full (Allocation Failure) 1022M->982M(1024M) 1903.114ms
```

Heap pinned at ~95-1023 MB, reclaimed almost nothing (~50 MB), repeated every 4 seconds. **Allocation Failure** Full GCs that barely recover memory = the heap is full of *live* objects. Something is holding gigabytes of references. This wasn't a leak in the classic sense — the memory was genuinely reachable.

**Step 2: Heap dump autopsy with Eclipse MAT**

We captured a dump from a suffering pod:

```bash
jmap -dump:live,format=b,file=heap.hprof <pid>
```

Opening it in **Eclipse Memory Analyzer (MAT)** and running the *Dominator Tree* report, the top retained-heap offenders were unmistakable:

```
Class Name                  | Objects  | Shallow Heap | Retained Heap
----------------------------+----------+--------------+--------------
java.util.HashMap$Node      | 4,812,003|   153 MB     |   481 MB
java.util.ArrayList         | 1,204,551|    38 MB     |   512 MB
com.acme.user.AuditLog      |   986,220|    47 MB     |   388 MB
com.acme.user.User          |   312,840|    15 MB     |   291 MB
```

Over **300k `User` instances** in a heap dump of a service that had served maybe 2,000 distinct users since boot. Users were being re-materialized *inside* other users' audit logs, recursively — the same few hundred power users exploding into hundreds of thousands of object references. The `HashMap$Node` army was Jackson's intermediate serialization state.

**Step 3: The "aha" — follow the retained set to the controller**

MAT's *Path to GC Roots* on a random `AuditLog` instance walked straight up through a `MappingJackson2HttpMessageConverter` write in progress — to this line:

```java
return userRepository.findById(id).orElseThrow();
```

The controller was handing the live entity graph to Jackson and saying "you decide." Jackson decided to serialize everything, recursively. No DTO boundary, no `@JsonIgnore`, no payload budget. The fix wrote itself.

**Step 4: Stop the bleeding**

We rolled back the audit-log field from serialization first (one-line `@JsonIgnore` hot-patch) while building the real fix. GC pressure vanished within one deploy cycle.

---

## 🔧 The Cure

**1. A DTO that declares exactly what crosses the wire:**

```java
public record UserProfileDTO(
        Long id,
        String name,
        String email,
        Instant lastLogin
) {}
```

**2. MapStruct to do the mapping (compile-time, zero reflection cost):**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {

    UserProfileDTO toProfileDTO(User user);
}
```

**3. The controller now returns the DTO — never the entity:**

```java
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserRepository userRepository;
    private final UserMapper userMapper;

    public UserController(UserRepository userRepository, UserMapper userMapper) {
        this.userRepository = userRepository;
        this.userMapper = userMapper;
    }

    @GetMapping("/{id}")
    public UserProfileDTO getUser(@PathVariable Long id) {
        User user = userRepository.findById(id).orElseThrow();
        return userMapper.toProfileDTO(user);
    }
}
```

**4. Break the cycle at the entity level too** — defense in depth, so no future endpoint can repeat the mistake:

```java
@Entity
@Table(name = "audit_logs")
public class AuditLog {

    @Id
    private Long id;

    private String action;
    private Instant timestamp;
    private String ipAddress;

    // Never serialize the parent from the child side.
    @JsonIgnore
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
}
```

Response payload for a power user went from **~4 MB (attempted)** to **~300 bytes**. GC pressure back to baseline: young-gen collections only, no Full GCs in the following 30 days.

---

## 📜 The Golden Rule

> **Never, ever expose JPA entities directly in REST APIs. Use DTOs to control serialization payload size explicitly — the entity models your database, the DTO models your contract, and confusing the two is an outage waiting for a field to be added.**

---

*Names and details sanitized to protect the guilty. The pain was real.*
....
