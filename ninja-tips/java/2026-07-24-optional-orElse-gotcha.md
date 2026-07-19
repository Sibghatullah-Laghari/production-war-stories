# Optional.orElse() - The Hidden Performance Trap

> **Category:** ninja-tip / java
> **Severity:** 🟡 Medium (death by a thousand cuts — until the day it isn't)

---

## 🎯 The Problem

Developers reach for this pattern constantly:

```java
Optional.ofNullable(config).orElse(loadDefaultConfig());
```

It reads beautifully. It *looks* like `loadDefaultConfig()` is a fallback that only runs when `config` is absent. **It is not.** `orElse()` takes a plain value — which means Java evaluates `loadDefaultConfig()` **eagerly, every single time**, whether the `Optional` is empty or not. If that fallback method does real work — a database query, an HTTP call, a heavy computation — you pay that cost on *every* invocation, then throw the result away 99% of the time.

---

## ✅ The Solution

Use `orElseGet()`, which takes a `Supplier` and only invokes it when the `Optional` is actually empty:

```java
Optional.ofNullable(config).orElseGet(() -> loadDefaultConfig());
```

Now `loadDefaultConfig()` runs **only** when needed. The `Supplier` is lazily evaluated — that's the entire difference, and it's the whole game.

---

## 🔬 Why It Works

```java
// java.util.Optional (simplified signatures):

// Takes a VALUE — already evaluated by the caller before this method runs.
public T orElse(T other) {
    return value != null ? value : other;  // 'other' was computed no matter what
}

// Takes a SUPPLIER — invoked only inside the else branch.
public T orElseGet(Supplier<? extends T> supplier) {
    return value != null ? value : supplier.get();  // called only when empty
}
```

Same fluent style, fundamentally different evaluation semantics. `orElse` = eager. `orElseGet` = lazy.

---

## 💻 Code Example

```java
@Service
public class FeatureFlagService {

    private final FeatureFlagRepository repository;

    public FeatureFlagService(FeatureFlagRepository repository) {
        this.repository = repository;
    }

    // Simulates an expensive fallback: DB round trip, ~15 ms.
    private FeatureFlag loadFromDatabase(String key) {
        return repository.findByKey(key)
            .orElseThrow(() -> new IllegalStateException("No flag: " + key));
    }

    public boolean isEnabled_wrong(String key, FeatureFlag cached) {
        // ❌ loadFromDatabase(key) runs EVERY time — even when 'cached' exists.
        // 10,000 checks/sec with a 95% cache hit rate = 10,000 wasted DB queries/sec.
        FeatureFlag flag = Optional.ofNullable(cached)
                                   .orElse(loadFromDatabase(key));
        return flag.isEnabled();
    }

    public boolean isEnabled_right(String key, FeatureFlag cached) {
        // ✅ loadFromDatabase(key) runs ONLY when 'cached' is null.
        // Same 10,000 checks/sec = ~500 DB queries/sec (the genuine misses).
        FeatureFlag flag = Optional.ofNullable(cached)
                                   .orElseGet(() -> loadFromDatabase(key));
        return flag.isEnabled();
    }
}
```

The waste is invisible in code review unless you know to look for it — the query doesn't appear in the `if` branch of any conditional. It hides inside an argument list.

### ⚠️ The one exception

If the fallback is a **constant or an already-computed value**, `orElse` is fine and even preferred (no lambda allocation):

```java
Optional.ofNullable(name).orElse("anonymous");       // fine — constant
Optional.ofNullable(user).orElse(GUEST_USER);        // fine — pre-built constant
Optional.ofNullable(user).orElseGet(() -> load());   // required — method call
```

Rule of thumb: **`orElse` for values, `orElseGet` for computations.**

---

## 🔍 PR Review Signal

- **Flag any `orElse()` whose argument is a method call** — `orElse(foo())`, `orElse(this.getDefault())`, `orElse(repo.find(...))`. Suggest `orElseGet(() -> foo())` in the review comment.
- **Extra-critical in hot paths:** inside loops, streams, request handlers, or anything called per-request. An eager DB call in a `.map()` over a stream is a hidden N+1 generator.
- **Grep-friendly:** searching your codebase for `.orElse(` followed by anything containing `(` is a 30-second audit that almost always finds victims.
- **Watch for side effects too:** eager evaluation isn't just slow — if `foo()` mutates state or writes logs, `orElse(foo())` does it unconditionally, which can produce phantom log entries ("Loading default...") on every single call even when nothing was loaded.

---

*Small tip. Real savings. The best optimizations are the ones that delete work that should never have existed.*
