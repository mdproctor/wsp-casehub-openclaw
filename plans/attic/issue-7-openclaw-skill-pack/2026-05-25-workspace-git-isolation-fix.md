# Workspace Git Isolation Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give life and openclaw their own isolated workspace git repos, restore the parent workspace to a clean state, and fix new-repo-checklist.md so this cannot recur.

**Architecture:** Four parts with a hard order constraint — Part A (parent session) must complete before Parts B and C. Part D (GitHub issue) can run in parallel with anything. Parts B and C are independent of each other once A is done.

**Tech Stack:** git, GitHub CLI (`gh`), bash

---

## Session ownership

| Part | Run in | Blocked by |
|------|--------|------------|
| A — Restore parent workspace | parent session | — |
| B — Init openclaw workspace | openclaw session (this one) | A |
| C — Init life workspace | life session | A |
| D — GitHub issue for checklist fix | openclaw session (this one) | — |

---

## Part A — Restore parent workspace
*(Execute in the parent session — `/Users/mdproctor/claude/public/casehub`)*

### Task A1: Switch parent workspace to main

- [ ] **Verify current state**

```bash
git -C /Users/mdproctor/claude/public/casehub branch --show-current
git -C /Users/mdproctor/claude/public/casehub ls-files life/ openclaw/
```

Expected output — branch: `issue-2-layer1-naive-domain`; tracked files:
```
life/HANDOFF.md
life/IDEAS.md
life/design/.meta
life/design/JOURNAL.md
openclaw/HANDOFF.md
openclaw/IDEAS.md
```

- [ ] **Switch to main**

```bash
git -C /Users/mdproctor/claude/public/casehub checkout main
```

Expected: git removes `life/design/.meta` and `life/design/JOURNAL.md` from working tree (they were only on the branch). No conflicts — untracked files are untouched.

- [ ] **Verify design files are gone, content files remain**

```bash
ls /Users/mdproctor/claude/public/casehub/life/design/ 2>/dev/null || echo "design/ absent"
ls /Users/mdproctor/claude/public/casehub/life/HANDOFF.md
ls /Users/mdproctor/claude/public/casehub/openclaw/HANDOFF.md
```

Expected: `design/ absent`, both HANDOFF.md files present.

---

### Task A2: Untrack life and openclaw from parent git

- [ ] **Add to parent workspace `.gitignore`**

Open `/Users/mdproctor/claude/public/casehub/.gitignore` and add after the last existing entry:

```
/eidos
/life
/openclaw
```

- [ ] **Untrack the directories**

```bash
git -C /Users/mdproctor/claude/public/casehub rm -r --cached life/ openclaw/
```

Expected output lists files being removed:
```
rm 'life/HANDOFF.md'
rm 'life/IDEAS.md'
rm 'openclaw/HANDOFF.md'
rm 'openclaw/IDEAS.md'
```
Physical files remain on disk throughout.

- [ ] **Verify files still exist on disk**

```bash
ls /Users/mdproctor/claude/public/casehub/life/HANDOFF.md
ls /Users/mdproctor/claude/public/casehub/openclaw/HANDOFF.md
```

Expected: both present.

- [ ] **Commit**

```bash
git -C /Users/mdproctor/claude/public/casehub add .gitignore
git -C /Users/mdproctor/claude/public/casehub commit -m "chore: untrack life and openclaw workspaces — now own repos; ignore eidos, life, openclaw"
```

- [ ] **Push**

```bash
git -C /Users/mdproctor/claude/public/casehub push
```

---

### Task A3: Delete stale life branch

- [ ] **Delete local branch**

```bash
git -C /Users/mdproctor/claude/public/casehub branch -D issue-2-layer1-naive-domain
```

- [ ] **Delete remote branch**

```bash
git -C /Users/mdproctor/claude/public/casehub push origin --delete issue-2-layer1-naive-domain
```

- [ ] **Verify branch is gone**

```bash
git -C /Users/mdproctor/claude/public/casehub branch -a | grep issue-2-layer1-naive-domain || echo "branch gone"
```

Expected: `branch gone`

---

### Task A4: Verify parent workspace is clean

- [ ] **Check git status**

```bash
git -C /Users/mdproctor/claude/public/casehub status --short
```

Expected: no output (clean) or only the known untracked plan files (`plans/2026-05-19-*.md`) — not `life/` or `openclaw/`.

- [ ] **Confirm life and openclaw are ignored**

```bash
git -C /Users/mdproctor/claude/public/casehub check-ignore -v /Users/mdproctor/claude/public/casehub/life /Users/mdproctor/claude/public/casehub/openclaw
```

Expected: both listed as ignored by `.gitignore`.

- [ ] **Confirm no tracked files remain in life/ or openclaw/**

```bash
git -C /Users/mdproctor/claude/public/casehub ls-files life/ openclaw/
```

Expected: no output.

✅ **Part A complete. Parts B and C may now proceed.**

---

## Part D — File GitHub issue for checklist fix
*(Execute in openclaw session — can run before or after Part B)*

### Task D1: File issue on casehubio/parent

- [ ] **Create the issue**

```bash
gh issue create --repo casehubio/parent \
  --title "fix(new-repo-checklist): Step 14 missing git init — workspaces inherit parent repo" \
  --body "$(cat <<'EOF'
## Problem

Step 14 of `docs/new-repo-checklist.md` creates workspace directories and symlinks but never
runs `git init`. The final bullet incorrectly commits the new subdirectory into the parent
workspace git repo instead of creating an isolated repo.

This caused `life/` and `openclaw/` workspaces to inherit the parent workspace's git repo,
entangling their branches and artifacts with the parent's history.

Additionally, the entire step (and several others) is hardcoded to a single developer's
machine paths and GitHub username, making the checklist unusable by other contributors.

## Fix required

See spec: committed to `casehub-openclaw` workspace at
`specs/2026-05-25-workspace-git-isolation-fix.md`

### 1. Add a Conventions preamble before Step 1

Define these variables used throughout:

| Variable | Meaning | Example |
|----------|---------|---------|
| `$GITHUB_USER` | Developer's personal GitHub username | `mdproctor` |
| `$CASEHUB_LOCAL` | Local root for project repos | `~/claude/casehub` |
| `$CASEHUB_WORKSPACE` | Local root for workspace repos | `~/claude/public/casehub` |

Workspace repos are named `wsp-casehub-<name>`, private, under `$GITHUB_USER`.

### 2. Step 1 — de-personalise

Replace `mdproctor` with `$GITHUB_USER` (4 occurrences: fork command, remote add, two
explanation lines).

### 3. Step 14 — full replacement

Replace all current bullets with:

```
- [ ] Create workspace dir:
      mkdir -p $CASEHUB_WORKSPACE/<name>/{adr,blog,plans,snapshots,specs}
- [ ] Create proj symlink:
      ln -s $CASEHUB_LOCAL/<name> $CASEHUB_WORKSPACE/<name>/proj
- [ ] Create CLAUDE.md symlink:
      ln -s $CASEHUB_LOCAL/<name>/CLAUDE.md $CASEHUB_WORKSPACE/<name>/CLAUDE.md
- [ ] Create wksp symlink in project:
      ln -s $CASEHUB_WORKSPACE/<name> $CASEHUB_LOCAL/<name>/wksp
- [ ] Create HANDOFF.md stub in workspace dir
- [ ] Create IDEAS.md stub in workspace dir
- [ ] Create INDEX.md stubs in each artifact subdir
      (adr/ blog/ plans/ snapshots/ specs/) — git cannot track empty dirs
- [ ] Create .gitignore in workspace dir:
      proj
      CLAUDE.md
      .DS_Store
- [ ] gh repo create $GITHUB_USER/wsp-casehub-<name> --private
- [ ] git init $CASEHUB_WORKSPACE/<name>
- [ ] git -C $CASEHUB_WORKSPACE/<name> remote add origin \
          https://github.com/$GITHUB_USER/wsp-casehub-<name>.git
- [ ] git -C $CASEHUB_WORKSPACE/<name> add .
- [ ] git -C $CASEHUB_WORKSPACE/<name> commit -m "init: workspace scaffold for casehub-<name>"
- [ ] git -C $CASEHUB_WORKSPACE/<name> push -u origin main
- [ ] Add /<name> to parent workspace .gitignore ($CASEHUB_WORKSPACE/.gitignore)
- [ ] Commit and push the parent workspace .gitignore update
```

### 4. Step 19 — fix push target

Replace:
```
git -C /Users/mdproctor/claude/public/casehub add <name>/ && git -C ... commit
```
With:
```
- [ ] git -C $CASEHUB_WORKSPACE/<name> push
```

### 5. Step 22 — de-personalise

Replace the hardcoded memory path with: "Update your Claude memory for the parent project
to reflect the new repos (path is developer-specific)."

### 6. Common Mistakes table — add row

| Workspace git not initialized | Sessions bleed into parent workspace; branches and artifacts from multiple projects entangle in parent git history | Run Step 14 git init steps; add `/<name>` to parent workspace `.gitignore` |

Replace remaining `/Users/mdproctor` and `mdproctor` references in the table with
`$CASEHUB_LOCAL`/`$CASEHUB_WORKSPACE`/`$GITHUB_USER`.
EOF
)"
```

- [ ] **Capture the issue number**

```bash
gh issue list --repo casehubio/parent --state open --limit 5
```

Note the issue number for reference.

---

## Part B — Initialize openclaw workspace
*(Execute in openclaw session, after Part A)*

### Task B1: Create GitHub workspace repo

- [ ] **Create the repo**

```bash
gh repo create mdproctor/wsp-casehub-openclaw --private --description "Workspace for casehub-openclaw — methodology artifacts"
```

Expected: repo created at `https://github.com/mdproctor/wsp-casehub-openclaw`

- [ ] **Verify**

```bash
gh repo view mdproctor/wsp-casehub-openclaw --json name,visibility --jq '"name: \(.name), visibility: \(.visibility)"'
```

Expected: `name: wsp-casehub-openclaw, visibility: PRIVATE`

---

### Task B2: Initialize git repo and configure

- [ ] **Init**

```bash
git init /Users/mdproctor/claude/public/casehub/openclaw
```

Expected: `Initialized empty Git repository in .../openclaw/.git/`

- [ ] **Add remote**

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw remote add origin https://github.com/mdproctor/wsp-casehub-openclaw.git
```

- [ ] **Verify remote**

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw remote -v
```

Expected:
```
origin  https://github.com/mdproctor/wsp-casehub-openclaw.git (fetch)
origin  https://github.com/mdproctor/wsp-casehub-openclaw.git (push)
```

---

### Task B3: Create .gitignore and INDEX.md stubs

- [ ] **Create `.gitignore`**

Write `/Users/mdproctor/claude/public/casehub/openclaw/.gitignore`:
```
proj
CLAUDE.md
.DS_Store
```

- [ ] **Create INDEX.md stubs** (so git tracks the empty artifact dirs)

```bash
printf "# Adr\n" > /Users/mdproctor/claude/public/casehub/openclaw/adr/INDEX.md
printf "# Blog\n" > /Users/mdproctor/claude/public/casehub/openclaw/blog/INDEX.md
printf "# Plans\n" > /Users/mdproctor/claude/public/casehub/openclaw/plans/INDEX.md
printf "# Snapshots\n" > /Users/mdproctor/claude/public/casehub/openclaw/snapshots/INDEX.md
printf "# Specs\n" > /Users/mdproctor/claude/public/casehub/openclaw/specs/INDEX.md
```

- [ ] **Verify stubs exist**

```bash
ls /Users/mdproctor/claude/public/casehub/openclaw/adr/INDEX.md && ls /Users/mdproctor/claude/public/casehub/openclaw/blog/INDEX.md && ls /Users/mdproctor/claude/public/casehub/openclaw/plans/INDEX.md && ls /Users/mdproctor/claude/public/casehub/openclaw/snapshots/INDEX.md && ls /Users/mdproctor/claude/public/casehub/openclaw/specs/INDEX.md && echo "all present"
```

Expected: `all present`

---

### Task B4: Initial commit and push

- [ ] **Stage all**

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw add .
```

- [ ] **Verify staged files — confirm no symlinks or unwanted files**

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw status --short
```

Expected staged (`A`): `.gitignore`, `HANDOFF.md`, `IDEAS.md`, `adr/INDEX.md`,
`blog/INDEX.md`, `plans/INDEX.md`, `snapshots/INDEX.md`,
`specs/2026-05-25-workspace-git-isolation-fix.md`, `specs/INDEX.md`.

NOT staged: `proj`, `CLAUDE.md` (excluded by `.gitignore`).

- [ ] **Commit**

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw commit -m "init: workspace scaffold for casehub-openclaw"
```

- [ ] **Push**

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw push -u origin main
```

---

### Task B5: Verify openclaw workspace

- [ ] **Git log shows init commit**

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw log --oneline
```

Expected: one commit — `init: workspace scaffold for casehub-openclaw`

- [ ] **Working tree is clean**

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw status --short
```

Expected: no output.

- [ ] **Symlinks are present on disk but not tracked**

```bash
ls -la /Users/mdproctor/claude/public/casehub/openclaw/proj
ls -la /Users/mdproctor/claude/public/casehub/openclaw/CLAUDE.md
git -C /Users/mdproctor/claude/public/casehub/openclaw ls-files proj CLAUDE.md
```

Expected: symlinks exist on disk; `git ls-files` returns no output (not tracked).

- [ ] **work-start can resolve paths** — confirm the skill's path resolution will work

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw rev-parse --show-toplevel
readlink -f /Users/mdproctor/claude/public/casehub/openclaw/proj
```

Expected:
```
/Users/mdproctor/claude/public/casehub/openclaw
/Users/mdproctor/claude/casehub/openclaw
```

✅ **Part B complete.**

---

## Part C — Initialize life workspace
*(Execute in life session, after Part A)*

### Task C1: Verify Part A completed

- [ ] **Confirm design/ is absent**

```bash
ls /Users/mdproctor/claude/public/casehub/life/design/ 2>/dev/null || echo "design/ absent — good"
```

Expected: `design/ absent — good`

- [ ] **Confirm files are present on disk**

```bash
ls /Users/mdproctor/claude/public/casehub/life/HANDOFF.md
ls /Users/mdproctor/claude/public/casehub/life/IDEAS.md
```

Expected: both present.

---

### Task C2: Create GitHub workspace repo

- [ ] **Create the repo**

```bash
gh repo create mdproctor/wsp-casehub-life --private --description "Workspace for casehub-life — methodology artifacts"
```

- [ ] **Verify**

```bash
gh repo view mdproctor/wsp-casehub-life --json name,visibility --jq '"name: \(.name), visibility: \(.visibility)"'
```

Expected: `name: wsp-casehub-life, visibility: PRIVATE`

---

### Task C3: Initialize git repo and configure

- [ ] **Init**

```bash
git init /Users/mdproctor/claude/public/casehub/life
```

- [ ] **Add remote**

```bash
git -C /Users/mdproctor/claude/public/casehub/life remote add origin https://github.com/mdproctor/wsp-casehub-life.git
```

- [ ] **Verify remote**

```bash
git -C /Users/mdproctor/claude/public/casehub/life remote -v
```

Expected:
```
origin  https://github.com/mdproctor/wsp-casehub-life.git (fetch)
origin  https://github.com/mdproctor/wsp-casehub-life.git (push)
```

---

### Task C4: Create .gitignore and INDEX.md stubs

- [ ] **Create `.gitignore`**

Write `/Users/mdproctor/claude/public/casehub/life/.gitignore`:
```
proj
CLAUDE.md
.DS_Store
```

- [ ] **Create INDEX.md stubs**

```bash
printf "# Adr\n" > /Users/mdproctor/claude/public/casehub/life/adr/INDEX.md
printf "# Blog\n" > /Users/mdproctor/claude/public/casehub/life/blog/INDEX.md
printf "# Plans\n" > /Users/mdproctor/claude/public/casehub/life/plans/INDEX.md
printf "# Snapshots\n" > /Users/mdproctor/claude/public/casehub/life/snapshots/INDEX.md
printf "# Specs\n" > /Users/mdproctor/claude/public/casehub/life/specs/INDEX.md
```

- [ ] **Verify all five present**

```bash
ls /Users/mdproctor/claude/public/casehub/life/adr/INDEX.md && ls /Users/mdproctor/claude/public/casehub/life/blog/INDEX.md && ls /Users/mdproctor/claude/public/casehub/life/plans/INDEX.md && ls /Users/mdproctor/claude/public/casehub/life/snapshots/INDEX.md && ls /Users/mdproctor/claude/public/casehub/life/specs/INDEX.md && echo "all present"
```

---

### Task C5: Initial commit and push

- [ ] **Stage all**

```bash
git -C /Users/mdproctor/claude/public/casehub/life add .
```

- [ ] **Verify staged files — confirm no symlinks or design/ artifacts**

```bash
git -C /Users/mdproctor/claude/public/casehub/life status --short
```

Expected staged (`A`): `.gitignore`, `HANDOFF.md`, `IDEAS.md`, `adr/INDEX.md`,
`blog/INDEX.md`, `plans/INDEX.md`, `snapshots/INDEX.md`, `specs/INDEX.md`.

NOT staged: `proj`, `CLAUDE.md`. No `design/` entries.

- [ ] **Commit**

```bash
git -C /Users/mdproctor/claude/public/casehub/life commit -m "init: workspace scaffold for casehub-life"
```

- [ ] **Push**

```bash
git -C /Users/mdproctor/claude/public/casehub/life push -u origin main
```

---

### Task C6: Verify life workspace

- [ ] **Git log**

```bash
git -C /Users/mdproctor/claude/public/casehub/life log --oneline
```

Expected: one commit — `init: workspace scaffold for casehub-life`

- [ ] **Clean status**

```bash
git -C /Users/mdproctor/claude/public/casehub/life status --short
```

Expected: no output.

- [ ] **work-start path resolution**

```bash
git -C /Users/mdproctor/claude/public/casehub/life rev-parse --show-toplevel
readlink -f /Users/mdproctor/claude/public/casehub/life/proj
```

Expected:
```
/Users/mdproctor/claude/public/casehub/life
/Users/mdproctor/claude/casehub/life
```

✅ **Part C complete.**

---

## Final Validation — run after all parts complete

Run from any terminal. No session dependency.

- [ ] **Parent workspace is clean and ignoring both dirs**

```bash
git -C /Users/mdproctor/claude/public/casehub status --short
git -C /Users/mdproctor/claude/public/casehub ls-files life/ openclaw/
git -C /Users/mdproctor/claude/public/casehub branch -a | grep issue-2-layer1 || echo "stale branch gone"
```

Expected: clean status (or only unrelated untracked files), no tracked life/openclaw files, stale branch gone.

- [ ] **Stale branch is gone from remote too**

```bash
gh api repos/mdproctor/$(git -C /Users/mdproctor/claude/public/casehub remote get-url origin | sed 's/.*\///;s/\.git//') \
  /branches 2>/dev/null | grep issue-2-layer1 || echo "remote branch confirmed gone"
```

- [ ] **openclaw workspace is isolated**

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw rev-parse --show-toplevel
```

Expected: `/Users/mdproctor/claude/public/casehub/openclaw` (not the parent path).

- [ ] **life workspace is isolated**

```bash
git -C /Users/mdproctor/claude/public/casehub/life rev-parse --show-toplevel
```

Expected: `/Users/mdproctor/claude/public/casehub/life`.

- [ ] **GitHub issue filed**

```bash
gh issue list --repo casehubio/parent --state open | grep "new-repo-checklist"
```

Expected: issue visible.

- [ ] **work-start will resolve correctly for openclaw** — the key test

```bash
git -C /Users/mdproctor/claude/public/casehub/openclaw rev-parse --show-toplevel && \
readlink -f /Users/mdproctor/claude/public/casehub/openclaw/proj && \
echo "work-start path resolution: OK"
```

Expected:
```
/Users/mdproctor/claude/public/casehub/openclaw
/Users/mdproctor/claude/casehub/openclaw
work-start path resolution: OK
```
