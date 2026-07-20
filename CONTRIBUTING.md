# Contributing to Production War Stories

First off — thanks for sharing your scars. Every story here saves someone else a 3 AM page.

---

## 📏 The Golden Rules

### 1. One Tip Per PR
Each pull request must contain **exactly one** anti-pattern, war story, or ninja tip. No bundling. Small, focused PRs get reviewed fast; kitchen-sink PRs get closed.

### 2. Use the Templates
All submissions **must** follow the templates in [`/templates`](./templates) — no exceptions:

- **Anti-Pattern** → [`templates/anti-pattern-template.md`](./templates/anti-pattern-template.md)
- **War Story** → [`templates/war-story-template.md`](./templates/war-story-template.md)

Copy the template, fill in every section, and place the file in the correct category folder using the naming convention:

```
YYYY-MM-DD-a-short-kebab-case-title.md
```

### 3. Code Must Be Real
- All code blocks must specify a language (````java`, ````sql`, ````bash`, etc.).
- Flawed code should compile (or be plausibly real config). Fixed code must be correct.
- Sanitize anything proprietary — change table names, endpoints, and internal hostnames.

---

## 🔍 The Review Process

Every PR is reviewed for three things:

| Criterion | What I Look For |
|---|---|
| **Accuracy** | The theory is correct. The commands actually work. The fix actually fixes it. |
| **Clarity** | A mid-level engineer can follow the story without needing your tribal knowledge. |
| **Working Code** | Code blocks are complete enough to illustrate the point — no `// TODO: magic happens here`. |

Expect feedback within a few days. Don't take revision requests personally — the goal is a repo so sharp it cuts.

---

## 🏆 Attribution

Every merged contribution earns you a spot in the **Contributors** section of the [README](./README.md). Your name, your story, your legacy.

---

## 📜 Code of Conduct

- **Be professional.** Stories are about systems and code, never about blaming people. Anonymize individuals and companies where appropriate.
- **Be respectful.** Disagree with ideas, not with humans. No harassment, no gatekeeping, no "well actually" pile-ons.
- **Be honest.** If a detail is fuzzy, say so. Fabricated war stories destroy the repo's credibility.
- **Assume good intent.** Everyone here was once the person who wrote the flawed code — including the maintainers.

Violations may result in PRs being closed and, for repeated offenses, a ban from contributing.

---

*Now go forth. May your pools be deep and your stack traces short.* 🥷.
