# [WEEK X] - [Name of Anti-Pattern]

> **Category:** [java | database | architecture]
> **Severity:** 🔴 Critical | 🟠 High | 🟡 Medium | 🔵 Low

---

## ❌ The Anti-Pattern

<!-- 2-3 sentences. What is this pattern, and why does it look deceptively reasonable at first glance? -->

[Describe the anti-pattern here. Explain the seductive logic that makes engineers reach for it.]

---

## 🧨 The Flawed Code

<!-- The code as it typically appears in the wild. It should compile and look plausibly innocent. -->

```java
// Flawed code goes here
```

---

## 📉 The Theoretical Fallout

<!-- Bullet points. Be specific about which axis of quality each consequence hits. -->

- **Performance:** [How does this degrade performance?]
- **Maintainability:** [How does this rot the codebase over time?]
- **Correctness:** [How does this produce wrong behavior or hidden bugs?]

---

## ✅ The Refactored Code

<!-- The corrected version. Must be correct, idiomatic, and complete enough to illustrate the fix. -->

```java
// Refactored code goes here
```

---

## 🧠 The Core Principle

<!-- The underlying CS/design principle this anti-pattern violates. Name it, define it, apply it. -->

[State the principle — e.g., Fail-Fast, Single Responsibility, Least Astonishment — and connect it back to the code above.]

---

## 🔍 How to Spot This in a PR

<!-- Actionable, greppable signals a reviewer can actually look for. -->

- [Signal 1 — e.g., an empty catch block, a TODO that says "should never happen"]
- [Signal 2]
- [Signal 3]

---

*Found this in your own codebase? Good. That means the repo is working.*
