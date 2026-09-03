# AGENTS - Repository Rules

**Effective:** 2026-09-03
**Owner:** prcoe1
**Parent (upstream, read-only):** HUSRCF/amdtop (`origin` -> https://github.com/HUSRCF/amdtop.git)
**Own fork (allowed):** prcoe1/amdtop (`fork` -> https://github.com/prcoe1/amdtop.git)

## Rule: Never touch upstream

- DO NOT `git commit` to parent
- DO NOT `git push` to `origin` (HUSRCF/amdtop)
- DO NOT create Issues on HUSRCF/amdtop
- DO NOT create Pull Requests on HUSRCF/amdtop
- All commits, pushes, issues, and PRs must target **only** `prcoe1/amdtop` (`fork` remote)

## Allowed workflows

```bash
# read from upstream
git fetch origin

# work and push only to fork
git checkout -b my-feature
git push fork my-feature
gh issue create --repo prcoe1/amdtop ...
gh pr create --repo prcoe1/amdtop ...
```

## Enforcement for AI agents

Before any `gh` or `git` operation, verify `--repo` / remote is `prcoe1/amdtop` or `fork`. If a command would target `HUSRCF/amdtop` or `origin`, abort and ask for confirmation.

This file is the source of truth for this constraint. Do not override without explicit user instruction to update this file.
