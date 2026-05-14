<p align="center">
  <img src="https://em-content.zobj.net/source/apple/391/high-voltage_26a1.png" width="100" />
</p>

<h1 align="center">claude-code-kit</h1>

<p align="center">
  <strong>Skills and rules that turn Claude Code into a principal engineer.</strong>
</p>

<p align="center">
  3 slash-command skills. 15 always-on rules. One install.<br/>
  TDD, structured logging, security, and clean architecture. All enforced automatically.
</p>

<p align="center">
  <a href="https://github.com/faminhub/claude-code-kit/stargazers"><img src="https://img.shields.io/github/stars/faminhub/claude-code-kit?style=flat&color=yellow" alt="Stars"></a>
  <a href="https://github.com/faminhub/claude-code-kit/commits/main"><img src="https://img.shields.io/github/last-commit/faminhub/claude-code-kit?style=flat" alt="Last Commit"></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/faminhub/claude-code-kit?style=flat&color=blue" alt="License"></a>
  <img src="https://img.shields.io/badge/Claude_Code-compatible-blueviolet?style=flat" alt="Claude Code">
  <img src="https://img.shields.io/badge/any_stack-works-brightgreen?style=flat" alt="Any Stack">
</p>

<p align="center">
  <a href="#the-problem">The Problem</a> •
  <a href="#-install">Install</a> •
  <a href="#-skills">Skills</a> •
  <a href="#-rules">Rules</a> •
  <a href="#-enable--disable">Enable / Disable</a>
</p>

---

## The Problem

There is a gap between what AI generates and what actually ships to production.

Claude writes code fast. It knows frameworks, patterns, and best practices. But it does not know your standards. It does not know that your team requires tests before implementation, that services must follow a specific architecture, that every PR needs proper documentation, or that security is non-negotiable on certain paths.

Without that context, Claude fills the gaps with its own defaults. And its defaults are not your standards.

The result is code you trust to run but not to ship. So you correct it. You add what is missing. You restructure what does not fit. You end up spending as much time fixing AI output as you would have spent writing it yourself.

## The Solution

claude-code-kit gives Claude that context upfront.

It is a collection of skills and rules that encode your engineering standards directly into Claude's workflow. Architecture decisions, test requirements, security checks, logging patterns, PR structure. All defined once, applied to every session, every project, without a reminder.

<table>
<tr>
<td width="50%">

### ❌ Without claude-code-kit

Claude is capable but unconstrained. It produces code that works by its own measure, not yours. You catch the gaps after the fact. You correct. You restructure. You re-explain. The output is code that technically runs but is not production ready until you fix it yourself.

</td>
<td width="50%">

### ✅ With claude-code-kit

Claude works within your standards from the start. The output it produces is something you can actually review, not something you have to rewrite first. You stop closing the gap between what AI produced and what your team actually ships.

</td>
</tr>
</table>

---

## 🚀 Install

Clone the repo and copy what you need.

```bash
# macOS / Linux
git clone https://github.com/faminhub/claude-code-kit.git
cd claude-code-kit

# Skills (Claude Code reads from ~/.claude/skills/)
cp -r skills/engineering skills/qa-review skills/testing ~/.claude/skills/

# Rules, global across all your projects
cp -r common typescript ~/.claude/rules/
```

```powershell
# Windows (PowerShell)
git clone https://github.com/faminhub/claude-code-kit.git
cd claude-code-kit

Copy-Item -Recurse skills/engineering, skills/qa-review, skills/testing "$env:USERPROFILE\.claude\skills\"
Copy-Item -Recurse common, typescript "$env:USERPROFILE\.claude\rules\"
```

**Per-project only (rules only, no global effect):**

```bash
mkdir -p .claude/rules
cp -r common typescript .claude/rules/
```

**Verify the install:**
```bash
ls ~/.claude/skills/   # engineering  qa-review  testing
ls ~/.claude/rules/    # common  typescript
```

---

## ⚡ Skills

Skills are on-demand. You invoke them with a slash command, they run for that task, then stop. Nothing activates on its own.

---

### `/engineering`

**Principal Engineer.** Activates a principal-level engineer that enforces production standards on every decision before writing a single line.

#### How to use

```
/engineering <your task>
```

```
/engineering build the user authentication module
/engineering refactor the notifications service
/engineering design the API for file uploads
```

#### What happens when you invoke it

1. States the requirement in one sentence before touching code
2. Checks if a pattern already exists in the codebase
3. Plans a vertical slice so the full feature path is built together, not layer by layer
4. Implements bottom-up, each layer complete before the next
5. Handles error paths, not just the happy path
6. Adds structured logging with request ID on all paths
7. Paginates list endpoints and uses transactions for multi-write operations
8. Opens a PR with Summary, Why, and Test plan sections

#### How to stop

The skill ends naturally when the task is done. To cancel mid-task just say:

```
stop
```

#### Is it required?

No. Invoke it when writing new features, refactoring, or designing APIs. Skip it for trivial changes like typo fixes or config updates.

#### Disable permanently

```bash
rm -rf ~/.claude/skills/engineering
```

---

### `/qa-review`

**8-Axis Code Reviewer.** Reviews code across 8 axes: correctness, security, performance, observability, architecture, readability, data integrity, and queue/async safety.

#### How to use

Run it after implementing a feature, before opening a PR, or when auditing existing code.

```
/qa-review
```

Point it at specific files or modules:

```
/qa-review the auth module
/qa-review the notifications service
```

#### What it checks

| Axis | What it catches |
|------|---------|
| Correctness | Edge cases, null paths, race conditions |
| Security | Timing attacks, hardcoded secrets, injection vulnerabilities, auth gaps |
| Performance | N+1 queries, unbounded fetches, blocking operations |
| Observability | Missing request IDs, unredacted secrets in logs |
| Architecture | Business logic in wrong layer, circular dependencies |
| Readability | God functions, dead code, unclear names |
| Data Integrity | Status regression, missing idempotency, missing transactions |
| Queue / Async | Messages deleted before confirmed processing, missing dead-letter queue |

#### Output format

```
## Review: <what changed in one sentence>

### Verdict: Request Changes

### Critical
- auth/service.ts:47 - Token compared with ===. Use constant-time comparison.

### High
- users/service.ts:89 - Query inside loop. Batch the calls instead.

### Medium
- users/controller.ts:34 - Raw database error returned to client. Wrap it.
```

#### How to stop

It finishes after producing the report. To cancel early:

```
stop
```

#### Is it required?

No. Strongly recommended before every PR merge. Treat it as mandatory for security-sensitive code like auth, user data, and file handling.

#### Disable permanently

```bash
rm -rf ~/.claude/skills/qa-review
```

---

### `/testing`

**TDD Enforcer.** Enforces red-green-refactor. It will not let you fix a bug without first proving it with a failing test.

#### How to use

```
/testing
```

Then describe what to implement or fix:

```
/testing add email validation to the registration flow
/testing fix the bug where deleted items still appear in search results
```

#### The cycle it enforces

```
RED      write a test that FAILS first (proves the test works)
GREEN    write minimal code to make it PASS
REFACTOR clean up, run tests again
REPEAT
```

**Bug fix pattern:**
```
Bug arrives > write failing test > implement fix > test passes > run full suite
```

It will never fix a bug without first reproducing it in a test.

#### Coverage targets

| Scope | Target |
|------|--------|
| All code | 80%+ statements and branches |
| Auth and data mutations | 95%+ |

#### How to stop

```
stop
```

#### Is it required?

No. Invoke it for any new feature, bug fix, or behavior change. Skip it for documentation, config changes, or purely structural refactors with no logic change.

#### Disable permanently

```bash
rm -rf ~/.claude/skills/testing
```

---

## 📋 Rules

Rules are always on. Claude reads them at the start of every session and applies them silently. No slash command needed.

---

### Common Rules

These apply to all files and all languages.

---

#### `agents.md`: Agent Orchestration

**What it does:** Tells Claude which agent to use for which task and enforces parallel execution for independent work.

**Activates:** Every session, automatically.

**Effect:**
```
Complex feature        → planner agent runs automatically
Code just written      → code-reviewer agent runs automatically
Bug fix                → tdd-guide agent runs automatically
Architectural decision → architect agent runs automatically
```

**How to disable:** Delete `~/.claude/rules/common/agents.md`

**Required?** Optional. High value if you use Claude Code agents regularly.

---

#### `development-workflow.md`: Feature Pipeline

**What it does:** Enforces the Research → Plan → TDD → Review → Commit order. Claude will not start implementing until research and planning are done.

**Activates:** Every session, automatically.

**Effect:** Claude will search for existing solutions, check documentation, and plan before writing code, even if you do not ask it to.

**How to disable:** Delete `~/.claude/rules/common/development-workflow.md`

**Required?** Optional. Highly recommended for non-trivial features.

---

#### `testing.md`: Test Standards

**What it does:** Makes TDD mandatory, sets 80% coverage minimum, enforces the Arrange-Act-Assert test structure, and requires end-to-end tests for critical flows.

**Activates:** Every session, automatically.

**How to disable:** Delete `~/.claude/rules/common/testing.md`

**Required?** Recommended. Remove only if your project has no test suite at all.

---

#### `code-review.md`: Review Standards

**What it does:** Triggers a mandatory review after every code change. Defines severity levels (CRITICAL, HIGH, MEDIUM, LOW) and what action each level requires.

**Activates:** Every session, automatically.

**How to disable:** Delete `~/.claude/rules/common/code-review.md`

**Required?** Recommended.

---

#### `security.md`: Security Checklist

**What it does:** Runs a pre-commit security checklist on every change. Covers hardcoded secrets, injection vulnerabilities, input validation, CSRF protection, and rate limiting.

**Activates:** Every session, automatically.

**How to disable:** Delete `~/.claude/rules/common/security.md`

**Required?** Yes, keep this one. Removing it means Claude will not catch security issues proactively.

---

#### `coding-style.md`: Style Standards

**What it does:** Enforces immutability, KISS/DRY/YAGNI principles, file size limits (800 lines max), function size limits (50 lines max), and consistent naming conventions.

**Activates:** Every session, automatically.

**How to disable:** Delete `~/.claude/rules/common/coding-style.md`

**Required?** Optional. Remove if your project has its own style guide you do not want overridden.

---

#### `git-workflow.md`: Commit and PR Standards

**What it does:** Enforces conventional commits format. Every PR body must have three sections: Summary, Why, and Test plan.

**Activates:** Every session, automatically.

**How to disable:** Delete `~/.claude/rules/common/git-workflow.md`

**Required?** Optional. Remove if your team uses a different commit convention.

---

#### `performance.md`: Model and Context Standards

**What it does:** Guides Claude on which model to pick for which type of work and how to manage the context window across long sessions.

**Activates:** Every session, automatically.

**How to disable:** Delete `~/.claude/rules/common/performance.md`

**Required?** Optional. Most useful if you work with multiple Claude models.

---

#### `patterns.md`: Design Patterns

**What it does:** Enforces the repository pattern, a consistent API response structure, and a proven project scaffold strategy.

**Activates:** Every session, automatically.

**How to disable:** Delete `~/.claude/rules/common/patterns.md`

**Required?** Optional.

---

#### `hooks.md`: Hook System

**What it does:** Guides Claude on configuring pre-tool, post-tool, and stop hooks, and on using task tracking effectively across multi-step work.

**Activates:** Every session, automatically.

**How to disable:** Delete `~/.claude/rules/common/hooks.md`

**Required?** Optional.

---

### Language-Specific Rules

The kit ships with rules for TypeScript and JavaScript today. More languages are on the way. These rules activate only on files that match their language and have no effect on anything else.

---

#### `typescript/coding-style.md`

**What it does:** Enforces strict type safety, schema-based input validation, immutable updates, and explicit types on all public APIs.

**Activates:** Automatically, only on TypeScript and JavaScript files.

**How to disable:** Delete `~/.claude/rules/typescript/coding-style.md`

**Required?** Recommended for TypeScript and JavaScript projects.

---

#### `typescript/security.md`

**What it does:** Bans hardcoded secrets and enforces environment variable usage with startup validation.

**Activates:** Automatically, only on TypeScript and JavaScript files.

**How to disable:** Delete `~/.claude/rules/typescript/security.md`

**Required?** Yes, keep it alongside `common/security.md`.

---

#### `typescript/testing.md`

**What it does:** Sets the preferred end-to-end testing framework and patterns for critical user flows.

**Activates:** Automatically, only on TypeScript and JavaScript files.

**How to disable:** Delete `~/.claude/rules/typescript/testing.md`

**Required?** Optional.

---

#### `typescript/hooks.md`

**What it does:** Configures automatic formatting, type checking after edits, and detection of debug statements left in code.

**Activates:** Automatically, only on TypeScript and JavaScript files.

**How to disable:** Delete `~/.claude/rules/typescript/hooks.md`

**Required?** Optional.

---

#### `typescript/patterns.md`

**What it does:** Enforces typed API response structures, reusable hook patterns, and a typed repository interface.

**Activates:** Automatically, only on TypeScript and JavaScript files.

**How to disable:** Delete `~/.claude/rules/typescript/patterns.md`

**Required?** Optional.

---

## 🔧 Enable / Disable Reference

| Item | Type | Default | How to disable |
|------|------|---------|----------------|
| `/engineering` | Skill | Off (invoke manually) | `rm -rf ~/.claude/skills/engineering` |
| `/qa-review` | Skill | Off (invoke manually) | `rm -rf ~/.claude/skills/qa-review` |
| `/testing` | Skill | Off (invoke manually) | `rm -rf ~/.claude/skills/testing` |
| `agents.md` | Rule | On (auto) | Delete the file |
| `development-workflow.md` | Rule | On (auto) | Delete the file |
| `testing.md` | Rule | On (auto) | Delete the file |
| `code-review.md` | Rule | On (auto) | Delete the file |
| `security.md` | Rule | On (auto) | Delete the file (not recommended) |
| `coding-style.md` | Rule | On (auto) | Delete the file |
| `git-workflow.md` | Rule | On (auto) | Delete the file |
| `performance.md` | Rule | On (auto) | Delete the file |
| `patterns.md` | Rule | On (auto) | Delete the file |
| `hooks.md` | Rule | On (auto) | Delete the file |
| `typescript/coding-style.md` | Rule, TS/JS only | On (auto) | Delete the file |
| `typescript/security.md` | Rule, TS/JS only | On (auto) | Delete the file |
| `typescript/testing.md` | Rule, TS/JS only | On (auto) | Delete the file |
| `typescript/hooks.md` | Rule, TS/JS only | On (auto) | Delete the file |
| `typescript/patterns.md` | Rule, TS/JS only | On (auto) | Delete the file |

Skills are manual. You invoke them per task and they stop when done.
Rules are automatic. They load every session and apply silently.

---

## 📁 Structure

```
claude-code-kit/
├── skills/
│   ├── engineering/SKILL.md    ← /engineering
│   ├── qa-review/SKILL.md      ← /qa-review
│   └── testing/SKILL.md        ← /testing
├── common/                     ← always on, all languages
│   ├── agents.md
│   ├── code-review.md
│   ├── coding-style.md
│   ├── development-workflow.md
│   ├── git-workflow.md
│   ├── hooks.md
│   ├── patterns.md
│   ├── performance.md
│   ├── security.md
│   └── testing.md
└── typescript/                 ← TS/JS files only
    ├── coding-style.md
    ├── hooks.md
    ├── patterns.md
    ├── security.md
    └── testing.md
```

---

## 🤝 Contributing

PRs welcome.

- **New skill:** add `skills/<name>/SKILL.md` with frontmatter (`name`, `description`, `allowed-tools`)
- **New language rules:** add a folder named after your language alongside `common/` and `typescript/`
- **New rule:** add a `.md` file in `common/` or in a language folder
- **Improvement:** open a PR explaining what changed and why it matters

---

## ⭐ Star This Repo

If this saved you from repeating yourself to Claude, leave a star. ⭐

---

## License

MIT
