# 🥷 Production War Stories

> **Real outages. Real fixes. Real lessons.**

![GitHub stars](https://img.shields.io/github/stars/YOUR_USERNAME/production-war-stories?style=social)
![GitHub issues](https://img.shields.io/github/issues/YOUR_USERNAME/production-war-stories)
![GitHub pull requests](https://img.shields.io/github/issues-pr/YOUR_USERNAME/production-war-stories)

A curated collection of production war stories, anti-patterns, and ninja tips — focused on **Java**, **databases**, and **backend engineering**. No theory-only fluff. Every entry comes from real incidents, real code reviews, and real 3 AM pages.

---

## 🏛️ The Three Pillars

### 🚫 Anti-Patterns — *Weekly*
Code that looks harmless, compiles fine, passes review… and then quietly ruins your system six months later. Each entry dissects one anti-pattern: the flawed code, the theoretical fallout, the refactor, and how to spot it in a PR.

### 💥 War Stories — *Bi-Weekly*
Post-mortems from real production incidents. The calm before the storm, the flawed code or config that triggered it, the debugging hunt (with actual commands), the fix, and the golden rule you walk away with.

### 🥷 Ninja Tips
Short, sharp, battle-tested techniques. The kind of thing a senior engineer mentions once in passing and you never forget.

---

## 📂 Folder Structure

```
production-war-stories/
├── anti-patterns/          # Weekly anti-pattern breakdowns
│   ├── java/               # Java language & concurrency anti-patterns
│   ├── database/           # SQL, ORM, and connection-handling anti-patterns
│   └── architecture/       # System design & distributed-systems anti-patterns
├── war-stories/            # Bi-weekly production incident post-mortems
│   ├── concurrency/        # Race conditions, pool exhaustion, deadlocks
│   ├── memory/             # Leaks, GC pauses, OOM kills
│   └── api/                # Timeouts, retries, cascading failures
├── templates/              # Copy these when contributing
│   ├── anti-pattern-template.md
│   └── war-story-template.md
├── CONTRIBUTING.md
└── README.md
```

---

## 🤝 How to Contribute

1. Pick the right template from [`/templates`](./templates).
2. Follow the structure **strictly** — every section exists for a reason.
3. Open a PR. **One tip per PR.**

Full details in [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## 🌟 Contributors

Every contributor is listed here. Ship a story, get your name on the wall.

<!-- Add contributors below this line -->