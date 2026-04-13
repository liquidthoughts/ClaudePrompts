---
name: repo-onboard
description: Explore a repo and produce project_repo_reference.md in .claude/. Run when developer first opens a repo or asks to be onboarded. Outputs a dense reference file committed with the repo — all team members and future agents benefit automatically.
---

# repo-onboard

Onboard a developer to this repo. Explore the codebase and write:

1. `<repo-root>/.claude/project_repo_reference.md` — dense reference for future agents
2. Register it in `<repo-root>/.claude/CLAUDE.md` so Claude auto-loads it

Do NOT perform a code review. Do NOT list bugs. Do NOT suggest fixes. Document only.

---

## Step 1 — Find the repo root and check if already onboarded

Use git — works on all OS, no path encoding needed:

```bash
CLAUDE_DIR="$(git rev-parse --show-toplevel)/.claude"
```

**Early exit check — run this first:**

```bash
test -f "$CLAUDE_DIR/project_repo_reference.md" && echo "EXISTS"
```

If the file exists: **stop immediately.** Report:
> "Repo already onboarded. `project_repo_reference.md` exists at `<path>`. Nothing to do."

Do not re-explore, do not overwrite. If the user explicitly asks to refresh or re-run, they must pass the flag `--refresh` (e.g., `/repo-onboard --refresh`), in which case skip this check and proceed.

Create the `.claude/` directory if absent:

```bash
mkdir -p "$CLAUDE_DIR"
```

### Why `.claude/` in the repo

Files here are committed with the repo. Every developer who clones gets the reference immediately — no need to re-run onboarding. Claude Code auto-loads `CLAUDE.md` from this directory for all users on all platforms.

---

## Step 2 — Create .claude/settings.json

Do this before any file reads or writes so permissions are in place for all remaining steps.

Check if `$CLAUDE_DIR/settings.json` already exists:

```bash
test -f "$CLAUDE_DIR/settings.json" && echo "EXISTS"
```

If it exists: **skip this step entirely.** Do not overwrite. Report it was skipped.

If it does not exist: create `$CLAUDE_DIR/settings.json` with the following content exactly:

```json
{
  "permissions": {
    "allow": [
      "Read(./**)",
      "Write(.claude/**)",
      "Edit(./**)",
      "Glob(./**)",
      "Grep(./**)",
      "WebSearch",
      "Bash(git *)",
      "Bash(npm *)",
      "Bash(npm install*)",
      "Bash(node *)",
      "Bash(find . *)",
      "Bash(find ./*)",
      "Bash(grep *)",
      "Bash(sed *)",
      "Bash(awk *)",
      "Bash(cat ./*)",
      "Bash(ls *)",
      "Bash(ls)",
      "Bash(mkdir *)",
      "Bash(mkdir -p *)",
      "Bash(mkdir -p ./*)",
      "Bash(cp ./* ./*)",
      "Bash(mv ./* ./*)",
      "Bash(jq *)",
      "Bash(sort *)",
      "Bash(uniq *)",
      "Bash(head *)",
      "Bash(tail *)",
      "Bash(wc *)",
      "Bash(xargs *)",
      "Bash(echo *)",
      "Bash(pwd)",
      "Bash(which *)",
      "Bash(where *)",
      "Bash(touch ./*)",
      "Bash(curl -s *)",
      "mcp__claude_ai_Atlassian__atlassianUserInfo",
      "mcp__claude_ai_Atlassian__getAccessibleAtlassianResources",
      "mcp__claude_ai_Atlassian__getVisibleJiraProjects",
      "mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql",
      "mcp__claude_ai_Atlassian__getJiraIssue",
      "mcp__claude_ai_Atlassian__editJiraIssue",
      "mcp__claude_ai_Atlassian__createJiraIssue",
      "mcp__claude_ai_Atlassian__addCommentToJiraIssue",
      "mcp__claude_ai_Atlassian__getTransitionsForJiraIssue",
      "mcp__claude_ai_Atlassian__transitionJiraIssue",
      "mcp__claude_ai_Atlassian__getJiraIssueTypeMetaWithFields",
      "mcp__claude_ai_Atlassian__getJiraProjectIssueTypesMetadata",
      "mcp__claude_ai_Atlassian__getConfluencePage",
      "mcp__claude_ai_Atlassian__getConfluenceSpaces",
      "mcp__claude_ai_Atlassian__getPagesInConfluenceSpace",
      "mcp__claude_ai_Atlassian__getConfluencePageDescendants",
      "mcp__claude_ai_Atlassian__getConfluencePageFooterComments",
      "mcp__claude_ai_Atlassian__getConfluencePageInlineComments",
      "mcp__claude_ai_Atlassian__createConfluencePage",
      "mcp__claude_ai_Atlassian__updateConfluencePage",
      "mcp__claude_ai_Atlassian__searchConfluenceUsingCql",
      "mcp__claude_ai_Atlassian__searchAtlassian",
      "mcp__MCP_DOCKER__browser_navigate"
    ]
  }
}
```

> Note: `settings.json` takes effect on the **next** session load. Permissions for the current run still require user approval for any tool not already allowed globally.

---

## Step 3 — Explore repo structure

```bash
find . -type f | grep -vE "node_modules|\.git|dist|build|\.class$|\.pyc$" | sort
```

Note: top-level dirs, source files, test files, config files, infra files.

---

## Step 4 — Read these files (in order, skip if absent)

| Priority | What to read |
|----------|-------------|
| High | All files in `src/` |
| High | `package.json` / `*.csproj` / `go.mod` / `requirements.txt` — runtime + deps |
| High | `.env.example`, `template.yaml`, `serverless.yml`, `*.tfvars` — env vars |
| Medium | `infra/*.tf` — Terraform resources, IAM, env var wiring |
| Medium | `.gitlab-ci.yml` / `.github/workflows/*.yml` — pipeline + packaging |
| Medium | `*.ocl` — deployment config |
| Low | `README.md` |

Do not read `node_modules/`, `.git/`, build output, or lock files.

---

## Step 5 — Extract the following

### File Map
One-line description per file: what it does, what it exports/handles.

### Entry Point & Flow
What triggers the code (HTTP, event, queue, cron). Step-by-step execution sequence: what calls what, in order.

### External APIs / Services
For each: name, base URL source (env var / hardcoded), auth method, endpoints used (method + path), request shape, response shape.

### Environment Variables
Table: variable name | which file reads it | what it controls.

### Secrets
How secrets are fetched (Secrets Manager, SSM, env var, vault). Secret names and keys within each secret.

### Input Shape
Expected input structure (event body, API payload, CLI args). Optional vs required fields, defaults.

### Output / Return Value
What the entry point returns on success and on error.

### HTTP Client Pattern
If HTTP calls are made: what library/module, and the standard boilerplate pattern used in this codebase.

### Hardcoded Values
Magic strings, placeholder values, feature flags, GUIDs, lookup maps — anything that looks like it could/should be configurable.

### Naming Conventions
Inconsistencies: casing mismatches between export names and internal functions, mixed `require`/`import` path styles, etc.

### Infrastructure Summary
Runtime, handler, timeout, memory, VPC/networking, IAM permissions, log retention, CI/CD toolchain.

### Known Quirks
Anything non-obvious that would surprise a developer: unusual ID formats, API response shapes needing transformation, ordering dependencies, hardcoded placeholders, missing features (e.g., no tests).

---

## Step 6 — Write project_repo_reference.md

Write to `$CLAUDE_DIR/project_repo_reference.md`. Use this structure:

```markdown
# <repo-name> — Repo Reference

> Auto-generated by repo-onboard. Update when APIs, env vars, or structure change.

## What This Does
[2-3 sentences: purpose, trigger, what it orchestrates]

## File Map
| File | Purpose |
|------|---------|
...

## Entry Point & Execution Flow
1. ...

## External APIs

### <API Name>
- Hostname: `env.VAR_NAME`
- Auth: x-api-key header, key from Secrets Manager `KEY_NAME`
- Endpoints:
  - `METHOD /path` — description, returns `{shape}`

## Environment Variables
| Variable | File | Purpose |
|----------|------|---------|
...

## Secrets
- Fetched via: ...
- Secret name: `env.SECRET_NAME`
- Keys: `KEY1` (used by X), `KEY2` (used by Y)

## Input Shape
```json
{ ... }
```

## Output
- Success: ...
- Error: ...

## HTTP Client Pattern
```js
// standard boilerplate used in this codebase
```

## Hardcoded Values
| Value | Location | Notes |
|-------|----------|-------|
...

## Naming Conventions
- ...

## Infrastructure
| Property | Value |
|----------|-------|
| Runtime | ... |
| Handler | ... |
| Timeout | ... |
| Memory | ... |
| VPC | ... |
| IAM | ... |
| Logs | ... |
| CI/CD | ... |

## Known Quirks
- ...
```

Be dense. No filler. Future agents read this instead of re-reading source files.

---

## Step 7 — Register in CLAUDE.md

Check if `$CLAUDE_DIR/CLAUDE.md` exists.

**If it exists:** check whether it already imports `project_repo_reference.md`. If not, prepend this block at the top (do not duplicate if already present):
```
@project_repo_reference.md

> Do not use worktree isolation (isolation: "worktree") when launching agents in this repo.
```

**If it does not exist:** create it with:
```markdown
@project_repo_reference.md

> Do not use worktree isolation (isolation: "worktree") when launching agents in this repo.
```

---

## Step 8 — Suppress worktrees in .gitignore

```bash
REPO_ROOT="$(git rev-parse --show-toplevel)"
test -f "$REPO_ROOT/.gitignore" && echo "EXISTS"
```

**If it exists:** check whether `.claude/worktrees/` is already present. If not, append:
```
# Claude Code agent worktrees (ephemeral, do not commit)
.claude/worktrees/
```

**If it does not exist:** create `$REPO_ROOT/.gitignore` with:
```
# Claude Code agent worktrees (ephemeral, do not commit)
.claude/worktrees/
```

Clean up any existing worktrees:
```bash
git worktree list
```
For each listed under `.claude/worktrees/`:
```bash
git worktree remove --force "<path>"
```
Then:
```bash
git worktree prune
```

---

## Done

Report:
- Repo root used
- Sections written to `project_repo_reference.md`
- Whether `CLAUDE.md` was created or updated
- Whether `settings.json` was created or skipped (already existed)
- Whether `.claude/` dir was created or already existed
- Whether `.gitignore` was created or updated
