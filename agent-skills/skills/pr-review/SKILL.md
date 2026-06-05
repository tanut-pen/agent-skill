---
name: code-pr-review
description: Perform thorough, structured code pull request (PR) reviews with a strong focus on syntax quality, code duplication (DRY), performance, and security. Use this skill whenever a user asks to review a PR, code diff, patch, or set of changed files — even if phrased casually like "look at my changes", "check this diff", "what do you think of this PR", "review my code", "give feedback on these changes", or "does this look good". Also trigger when the user pastes a git diff, shares a GitHub/GitLab PR link, uploads changed files, or asks for code feedback in the context of merging or approving work.
---

# Code PR Review Skill

Produce thorough, actionable PR reviews laser-focused on four pillars: **syntax quality**, **no repeated code (DRY)**, **performance**, and **security**. Reviews should feel like they came from a senior engineer who is helpful, direct, and specific — not a linter or a checklist robot.

---

## Input Formats

Accept any of these:

- **Raw git diff** (pasted inline or in a file)
- **GitHub / GitLab PR URL** — fetch the diff via web_fetch if available
- **Uploaded files** — read changed files directly
- **Inline code snippets** described as "changes" or "before/after"
- **Partial context** — review what's available; flag if more context would change the verdict

---

## Review Structure

Always produce a review in this order. Use markdown headers.

### 1. Summary (2–4 sentences)
What does this PR do? State the purpose in plain language. Note if the scope seems larger or smaller than the title/description suggests.

### 2. Verdict
One of:
- ✅ **Approve** — ready to merge as-is
- 🔶 **Approve with suggestions** — merge is fine, but improvements noted
- 🚫 **Request changes** — must-fix issues found before merging

Brief rationale (1–2 sentences).

### 3. Security Issues 🔒
**Always the first content section.** Security bugs are blocking by default.

For each issue:
- **File + line reference** (if available)
- **Vulnerability type** (e.g. SQL injection, XSS, broken auth)
- **Why it's dangerous**
- **Concrete fix** (code snippet preferred)

Key things to catch:
- SQL/NoSQL injection — any string interpolation into queries
- XSS — unescaped user input rendered in HTML/JS
- Command injection — user input passed to shell, subprocess, `eval`
- Broken authentication or missing authorization checks
- Hardcoded secrets, API keys, passwords (even in comments or tests)
- Insecure defaults: `verify=False`, `DEBUG=True`, wildcard CORS, open redirects
- Sensitive data in logs, error messages, stack traces
- Insecure deserialization (`pickle`, `eval`, `yaml.load` without `Loader`)
- Missing rate limiting on sensitive endpoints
- Cryptographic misuse (MD5/SHA1 for passwords, weak random for tokens)

If none: write "None found."

### 4. Performance Issues ⚡
Non-blocking unless on a critical hot path.

For each issue:
- **What the bottleneck is**
- **Why it's slow or wasteful**
- **How to fix it** (with before/after code when helpful)

Key things to catch:
- **N+1 queries** — loop issuing a DB call per iteration; suggest eager loading / batching
- **Unbounded fetches** — `SELECT *` or `.all()` with no limit on large tables
- **Missing indexes** — new `WHERE`/`JOIN`/`ORDER BY` columns not covered by indexes
- **Redundant computation** — same value computed multiple times in a loop; hoist it out
- **Synchronous I/O on hot paths** — blocking calls that should be async
- **Memory waste** — loading entire dataset into memory to use one field
- **Premature string concatenation** — use `join()` or template strings instead
- **Heavy work in tight loops** — regex compilation, object instantiation, imports inside loops

If none: write "None found."

### 5. Code Duplication (DRY) 🔁
Non-blocking, but flag clearly — repeated code becomes a maintenance trap.

For each violation:
- **Where the duplication is** (files/functions/lines)
- **What should be extracted** (function, constant, mixin, base class)
- **Sketch of the refactor** (short code example if it clarifies)

Key patterns to catch:
- Identical or near-identical logic copy-pasted across functions/files
- Same magic number or string literal appearing multiple times (extract to constant)
- Parallel if/else or switch blocks that could be a lookup table or strategy pattern
- Multiple functions that do 90% the same thing — extract shared logic, parameterize the diff
- Repeated error-handling boilerplate — consider a decorator or middleware
- Config values duplicated in code and in config files

If none: write "None found."

### 6. Syntax & Code Quality 🧹
Covers correctness, clarity, and idiomatic style. Severity varies — flag blockers (crashes, bugs) separately from style suggestions.

For each issue:
- **What's wrong**
- **Why it matters** (bug vs. style vs. readability)
- **Suggested fix**

Key things to catch:
- **Bugs hiding in syntax**: wrong operator precedence, missing `await`, off-by-one, incorrect boolean logic
- **Null/undefined hazards**: unguarded `.foo` access, missing default values, absent null checks
- **Dead code**: unreachable branches, unused variables, commented-out code blocks
- **Overly complex expressions**: deeply nested ternaries, one-liners that should be broken up
- **Inconsistent naming**: mixing camelCase/snake_case, vague names (`data`, `temp`, `x`)
- **Type mismatches or unsafe casts**: implicit coercions, ignored return types, `any` overuse
- **Improper error handling**: bare `except`, swallowed exceptions, `console.log` as error handling
- **Missing edge cases**: empty input, zero, negative numbers, empty list, concurrent access
- **Idiomatic violations**: non-idiomatic patterns for the language (see Language Awareness below)

### 7. What's Done Well ✨
1–3 specific things the author did right. Not filler — signals good patterns to keep.

---

## Review Depth Guidelines

| Diff size | Approach |
|-----------|----------|
| < 50 lines | Full line-by-line |
| 50–300 lines | Section-by-section, flag notable lines |
| 300–1000 lines | Architectural + spot-check critical paths |
| > 1000 lines | High-level review + note that splitting is recommended |

---

## Tone Guidelines

- **Be specific.** "This could be simplified" is not helpful. Show how.
- **Be direct but not harsh.** "This will panic on nil input" is better than "you forgot nil handling".
- **Explain the why** for non-obvious issues.
- **Don't pile on nits.** If there are 15 style issues, group them or pick the top 3.
- **Assume good intent.** Frame issues as improvements, not failures.
- **Use code blocks** for suggested fixes when they're short (< 20 lines).

---

## Language Awareness

Apply language-specific instincts for the four pillars:

| Language | Syntax/Quality | DRY | Performance | Security |
|----------|---------------|-----|-------------|----------|
| **Python** | Type hints, f-strings, comprehension misuse, bare `except` | Decorators, mixins, `functools` | Generator vs list, `__slots__`, avoid recompiling regex | `pickle`, `yaml.load`, `eval`, `subprocess` with shell=True |
| **JavaScript/TypeScript** | `any` overuse, missing `await`, `==` vs `===` | Shared hooks, utility modules | Avoid re-renders, debounce, lazy imports | XSS via `innerHTML`, prototype pollution, `eval` |
| **Go** | Error ignoring, short variable shadowing | Interface extraction, embedding | Goroutine leaks, unnecessary allocations, sync.Pool | `fmt.Sprintf` in SQL, unsafe pointer use |
| **Java/Kotlin** | Null safety, unchecked casts, resource leaks | Inheritance vs composition, shared utils | Stream misuse, boxing overhead, connection pool sizing | Deserialization, XXE in XML parsers, Spring misconfig |
| **SQL** | Implicit type coercion, ambiguous column refs | CTEs and views instead of copy-paste | Missing indexes, `SELECT *`, no `LIMIT` | String interpolation in queries — always parameterize |
| **Infra (TF/K8s/Docker)** | Hardcoded values, missing variable descriptions | Modules, shared configs, DRY variable blocks | Resource limits/requests, image layer caching | Least-privilege IAM, secrets in env vars, open ports |

---

## When Context Is Missing

If you can only see part of the change (e.g. no surrounding code, no tests, no PR description):

- Review what's available
- Explicitly note what you couldn't assess: "I can't see the callers of this function, so I can't verify the interface change is backward-compatible."
- Don't invent issues you can't substantiate from the diff

---

## Output Length

- Small PRs (< 100 lines): aim for concise, focused review (~200–400 words)
- Medium PRs (100–500 lines): full structured review (~400–800 words)
- Large PRs (> 500 lines): structured review + note on size (~600–1200 words)

Don't pad. If a section has nothing worth saying, write "None" or omit it.