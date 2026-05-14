# Git Workflow

## Commit Message Format
```
<type>: <description>

<optional body>
```

Types: feat, fix, refactor, docs, test, chore, perf, ci

## Pull Request Workflow

When creating PRs:
1. Analyze full commit history (not just latest commit)
2. Use `git diff [base-branch]...HEAD` to see all changes
3. Draft comprehensive PR summary
4. Include test plan with TODOs
5. Push with `-u` flag if new branch

PR body MUST always include three sections in this order:

```
## Summary
<1-3 bullet points — what changed>

## Why
<The motivation — constraint, bug, stakeholder requirement, or risk being addressed.
Explain WHY this change exists, not what it does.>

## Test plan
- [ ] <specific scenario to verify>
- [ ] <edge case or regression check>
```

Use a HEREDOC when passing body to gh:
```
gh pr create --title "..." --body "$(cat <<'EOF'
## Summary
...

## Why
...

## Test plan
- [ ] ...
EOF
)"
```

> For the full development process (planning, TDD, code review) before git operations,
> see [development-workflow.md](./development-workflow.md).
