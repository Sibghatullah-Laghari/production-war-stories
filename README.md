# 🥷 Production War Stories

> **Real outages. Real fixes. Real lessons.**

<!-- ⚠️ REPLACE "YOUR_USERNAME" in the three badge URLs below with your GitHub username or org ⚠️ -->
![GitHub stars](https://img.shields.io/github/stars/YOUR_USERNAME/production-war-stories?style=social)
![GitHub issues](https://img.shields.io/github/issues/YOUR_USERNAME/production-war-stories)
![GitHub pull requests](https://img.shields.io/github/issues-pr/YOUR_USERNAME/production-war-stories)

A curated collection of production war stories, anti-patterns, and ninja tips — focused on **Java**, **databases**, and **backend engineering**. No theory-only fluff. Every entry comes from real incidents, real code reviews, and real 3 AM pages.

---

## 📖 How to Consume This Repo

Different readers, different doors. Pick yours:

- 💥 **Start with [War Stories](./war-stories)** if you want to see how production incidents unfold — the calm, the trigger, the hunt with real commands (`jstack`, `pg_stat_activity`, GC logs), and the fix.
- 🚫 **Study [Anti-Patterns](./anti-patterns)** if you want to fortify your code against common mistakes — flawed code, theoretical fallout, refactored solution, and how to spot each one in a PR.
- 🥷 **Check [Ninja Tips](./ninja-tips)** for quick, actionable daily coding hacks — small, sharp techniques you can apply in your very next commit.

---

## 🗺️ Roadmap

What's coming next:

- [ ] **Week 3:** Architecture Anti-Pattern — *Distributed Transactions* (the two-phase commit trap)
- [ ] **Week 4:** DevOps War Story — *Kubernetes OOM Killer* (the pod that kept dying at 2 AM)

Want to shape the roadmap? Open an issue with a story pitch.

---

---

## 🏛️ The Three Pillars.

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
│   └── api/                # Serialization, timeouts, retries, cascading failures
├── ninja-tips/             # Quick, sharp daily coding hacks
│   └── java/               # JVM-language tips & gotchas
├── templates/              # Copy these when contributing
│   ├── anti-pattern-template.md
│   └── war-story-template.md
├── .github/
│   └── ISSUE_TEMPLATE/     # Pitch a story via structured GitHub issues
│       ├── new-anti-pattern.yml
│       └── new-war-story.yml
├── CONTRIBUTING.md
└── README.md
```

---

## 🤝 Contributing

The fastest way to contribute is through our structured issue templates — pitch your story, and we'll help you shape it into a full entry:

1. Go to **Issues → New Issue** and pick a template:
   - 🚫 **[Submit a New Anti-Pattern](../../issues/new?template=new-anti-pattern.yml)**
   - 💥 **[Submit a Production War Story](../../issues/new?template=new-war-story.yml)**
2. Fill in every field — the issue form mirrors the file templates, so an approved pitch is 90% of the final entry.
3. Once your pitch is approved, copy the matching template from [`/templates`](./templates), write the full entry, and open a PR. **One tip per PR.**

Prefer to skip the pitch and write directly? That's fine too — just follow the templates strictly. Full details in [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## 🌟 Contributors:-

Every contributor is listed here. Ship a story, get your name on the wall.

<!-- Add contributors below this line. -->
....
