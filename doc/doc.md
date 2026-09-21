# commit-generate - Documentation

> Back to [README](../README.md)

## Prerequisites

- An agent harness that loads `SKILL.md` skills and can run shell commands
- Git (with `git diff --cached`, `git status --short`, and `git branch --show-current`)
- The target project must be a Git repository

## Installation

`<skills-dir>` is the skill directory your harness scans.

### Clone from GitHub

```bash
git clone https://github.com/agenvoy/skill-commit-generate.git \
    <skills-dir>/commit-generate
```

### Verify Installation

```bash
ls <skills-dir>/commit-generate/SKILL.md
```

Invoke it from your harness with `/commit-generate`.

## Usage

### Basic

```bash
# 1. Stage the files you want in the commit
git add <file1> <file2>

# 2. Invoke the skill from your harness
/commit-generate
```

Example output:

```
feat: Add Docker environment auto-detection and database path switching
feat: 新增 Docker 環境自動偵測與資料庫路徑切換機制
```

The skill only prints the message; it never runs `git commit`.

### Unstaged File Reminder

Files in the same module as the staged ones but still at ` M` are listed above the message:

```
⚠️ Unstaged files in the same module: internal/auth/token.go

fix: Fix token expiration not handled correctly during user login
fix: 修正使用者登入時 token 過期未正確處理的問題
```

The message still describes only the staged content.

### Multi-Topic Changes

When the staged diff spans unrelated modules or touches 2+ primary tags:

```
⚠️ Multi-topic change detected; consider splitting into multiple commits:
- Auth module refactor: internal/auth/*.go
- UI style tweaks: web/styles/*.css

If merging anyway, the rollup description is:
refactor: Split auth module and adjust UI styles
refactor: 重構認證模組並調整 UI 樣式
```

### No Staged Changes

```
No staged changes. Run `git add` first to select files to commit.
```

## Configuration Reference

### Data Sources

| Command | Purpose | Scope of Influence |
|---------|---------|--------------------|
| `git diff --cached` | Changes to describe | Sole content source |
| `git status --short` | Find same-module files left unstaged | Reminder only |
| `git log --oneline -10` | Existing tag vocabulary and granularity | Wording only |
| `git branch --show-current` | Intent carried by the branch name | Wording only |

The four reads are independent and run in parallel. `git log` and the branch name never change the tag choice; the rules win whenever they conflict.

### Classification Tags

| Tag | Use Case |
|-----|----------|
| `feat` | New feature, endpoint, or component |
| `fix` | Bug fix, error handling, nil check |
| `update` | Adjust existing behavior or parameters |
| `add` | New file or resource (non-feature) |
| `remove` | Delete code or files |
| `refactor` | Refactor without behavior change |
| `perf` | Performance optimization |
| `style` | Formatting and layout |
| `doc` | Documentation and comments |
| `test` | Test-related changes |
| `chore` | CI/CD, tooling, dependency management |
| `security` | Security patch |
| `breaking` | Breaking change |

### Tag Priority

```
BREAKING > FEAT > FIX > SECURITY > UPDATE > REFACTOR > PERF > others
```

### Breaking Signals (hit → `breaking`)

| Category | Signal |
|----------|--------|
| Symbol removal | Delete any exported or public function, class, method, type, constant, enum value |
| Signature change | Parameter type, order, new required parameter, return type change |
| HTTP API | Endpoint deletion, URL change, response schema field removed or type changed, status code change |
| Configuration | Delete existing key, add required key, type or format change on existing key |
| Migration | `DROP COLUMN`, required column added to populated table, type narrowing |
| CLI | Flag deletion, new required positional argument, semantic change to existing flag |
| Imports | Package or module restructure that breaks existing references |

### Security Signals (hit → `security`, unless Breaking also hits)

| Category | Signal |
|----------|--------|
| Injection | SQL / XSS / Command / LDAP / Template injection |
| AuthZ | Missing auth check, privilege escalation, JWT validation gap |
| Sensitive data | Remove secrets, tokens, or PII from log / response / error message |
| Hardcoding | Remove hardcoded secrets, tokens, default passwords |
| Web security | CSRF / CORS / CSP / HSTS patches |
| CVE | Dependency CVE patches (upgraded from `chore: upgrade` to `security`) |

### Multi-Topic Criteria

Any one of the following triggers detection:

- Diff touches 2+ primary tag categories (`feat` / `fix` / `refactor` / `breaking` / `security`)
- Changed files span 2+ semantically unrelated modules (top-level directory or package boundary)
- A single commit contains 3+ unrelated topics

Formatting, comment-only tweaks, and dependency bumps can ride along with the primary commit and do not count as multi-topic.

### Output Format Rules

| Rule | Description |
|------|-------------|
| Single tag | Pick the tag that best represents the core intent |
| English subject | Imperative mood, at most 72 characters |
| Traditional Chinese body | At most 50 characters, verb-first (新增 / 修正 / 重構 / 移除 / 優化) |
| Merge related changes | Roll small edits into a single description |
| Single topic first | Warn and suggest a split before any multi-topic rollup |
