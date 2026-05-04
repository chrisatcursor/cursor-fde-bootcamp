---
name: init
description: Bootcamp environment doctor. Use when the user runs `/init`, asks to set up the bootcamp repo, says "set up my bootcamp environment", or you are a cohort participant starting work in this repo for the first time. Verifies prerequisites (gh, git, Node, yarn, Docker, Postgres, model access), reports each check with ✓/✗ + a fix command on failure, and prints next steps. Does NOT install rules, sub-agents, or other scaffolding — those are built by the cohort during workshops.
---

# /init — Bootcamp Environment Doctor

You are running this skill inside the **Cursor FDE Bootcamp** working repo. Your job is to confirm the participant's environment is ready for the workshops, **not** to build their AI scaffolding for them. Cohort members create their own `.cursor/rules/`, sub-agents, and skills during the workshop sessions. Don't pre-create those.

## What this skill does (and does not do)

| Does | Does not do |
|---|---|
| Verify required tools (`gh`, `git`, `node`, `yarn`, optionally `docker`/`psql`) | Install rules into `.cursor/rules/` |
| Verify the participant is in the bootcamp repo | Install sub-agents into `.cursor/agents/` |
| Verify dependencies install (`yarn install`) | Install starter skills beyond this `/init` skill |
| Verify Cursor model access | Make architectural decisions for the cohort |
| Create `.bootcamp-marker` to discriminate cohort repo from arbitrary checkouts | Touch `apps/`, `packages/`, or anything else cal.diy ships |
| Print confirmation and next-steps message | Build, start, or seed the database |

This is intentional. The bootcamp is about cohort members **building** their scaffolding. /init only ensures they can do the building.

## Execution model

Run all checks **read-only first**, gather results, then take action only on `.bootcamp-marker` and on offering specific fix commands. Do not modify the repo otherwise.

For each check, print one line:

```
✓ <Check name> — <one-line evidence>
✗ <Check name> — <one-line failure description>
   Fix: <exact shell command or action>
```

Use plain ASCII checkmarks (✓ ✗) — they render in any terminal.

## Tier 1 — Required (block if any fail)

These must pass. If any fail, print the fix and tell the participant to re-run `/init` after fixing.

### 1. GitHub CLI installed

Run:

```bash
gh --version
```

- ✓ if exit 0 and output starts with `gh version`
- ✗ if `command not found` → Fix: `brew install gh` (macOS) or see https://cli.github.com/

### 2. GitHub CLI authenticated

Run:

```bash
gh auth status
```

- ✓ if any account is "Logged in" (the active account is fine; we don't enforce which one)
- ✗ if output contains "You are not logged into" → Fix: `gh auth login`

### 3. Git installed and configured

Run:

```bash
git --version && git config user.name && git config user.email
```

- ✓ if all three return non-empty values
- ✗ git missing → Fix: `xcode-select --install` (macOS) or platform equivalent
- ✗ user.name or user.email blank → Fix: `git config --global user.name "Your Name" && git config --global user.email "you@example.com"`

### 4. Node version meets cal.diy requirements

cal.diy README requires Node ≥18.x. Recommended: Node 20 LTS or 22 LTS. Run:

```bash
node --version
```

- ✓ if major version ≥ 20 (recommended)
- ⚠ warn but continue if major version is 18 (works but unsupported by cal.diy upstream now)
- ✗ if < 18 or `command not found` → Fix: install Node 22 via nvm (`nvm install 22 && nvm use 22`) or [nodejs.org](https://nodejs.org/)

There is **no `.nvmrc`** in this repo. Don't try to read one.

### 5. Yarn 4.x available

Yarn 4.12.0 is pinned via corepack in `package.json` (`"packageManager": "yarn@4.12.0"`). Corepack ships with Node 16+ but may need to be enabled. Run:

```bash
yarn --version
```

- ✓ if output starts with `4.`
- If output is `1.x` (Yarn Classic): Fix: `corepack enable` then re-run `yarn --version` from within this repo
- If `command not found`: Fix: `corepack enable && corepack prepare yarn@4.12.0 --activate`

The first time corepack runs, it prints a banner about downloading yarn — this is expected and only happens once.

### 6. Repo dependencies install cleanly

Run from the repo root:

```bash
yarn install
```

**Set expectations before running:**
- This takes **~15–20 minutes** on a fresh checkout (3,500+ packages, ~800 MiB on disk).
- It is **normal** to see ~6 "doesn't provide X" peer-dependency warnings (`@prisma/client`, `react`, `next`, etc.). These are upstream issues in cal.diy and don't block the install.
- The script ends with `Done with warnings` — that's success.

Check status:

- ✓ if exit code 0 (warnings OK; "Done with warnings" still counts as success)
- ✗ if exit code non-zero → Fix: read the error; common causes are network failure, disk full, or wrong yarn version (re-check #5)

If the install has already been run successfully, `yarn install` is fast (a few seconds). Always safe to re-run.

### 7. Cursor model access works

Make a one-shot test. The participant's Cursor agent already has access to its own model — verify by attempting any tiny inference (e.g. ask the agent to repeat a one-word string). If the agent can read this file and execute /init, this check passes implicitly.

- ✓ if the agent is running this skill at all (you're proving it by reading this)
- ✗ implausible in this flow; if the participant reports model errors, direct them to Cursor settings → Model access

## Tier 2 — Recommended (warn, do not block)

These improve the experience but aren't required to start the workshops.

### 8. Postgres reachable

`yarn dx` spins up a Docker Postgres, so this check is only needed if the participant chooses the "manual setup" path from the README. Try:

```bash
psql --version
```

- ✓ if Postgres ≥ 13 found
- ⚠ warn if not found, but tell participant `yarn dx` covers them via Docker

### 9. Docker available (for `yarn dx`)

```bash
docker --version
```

- ✓ if Docker found (and `docker info` succeeds, indicating daemon is running)
- ⚠ warn if not found → Recommendation: install Docker Desktop or Rancher Desktop. If they prefer manual Postgres, they can skip this.

### 10. `.env` file exists

```bash
ls .env
```

- ✓ if `.env` exists at repo root
- ⚠ if not → Offer to copy: "Run `cp .env.example .env` to start with the example values, then edit secrets per the README's 'Setting up your first user' section."

Do **not** auto-copy. The participant should make this decision and edit the secrets themselves.

### 11. Free disk space ≥ 5 GB

`yarn install` needs ~1 GB; database, build artifacts, and Docker images push you past 5 GB total.

```bash
df -h .
```

- ✓ if available space ≥ 5 GB
- ⚠ if less → Recommend cleanup before continuing.

## Tier 3 — Informational (no pass/fail)

Print these for situational awareness; do not gate.

### 12. OS and known quirks

```bash
uname -s
```

If macOS on Apple Silicon (arm64): note that some Docker images need the `-arm` suffix (cal.diy README mentions this).
If Windows: cohort participants on Windows should use WSL2 — note this.
If Linux: usually fine.

### 13. `.bootcamp-marker` file

If `.bootcamp-marker` exists at repo root:
- Print: `i  Bootcamp marker found — this is the cohort working repo.`

If it doesn't exist:
- Create it with these contents:

```
Cursor FDE Bootcamp working repo.
This file is created by /init. Do not delete unless you understand what it does.
```

- Print: `i  Bootcamp marker created.`

Use it as a discriminator on subsequent /init runs to confirm we're in the right place. (Note the README at the top of the repo if you want context.)

## Idempotency

A second run of /init should:
1. Pass through every check the same way (no destructive actions).
2. Detect the existing `.bootcamp-marker` and skip recreating it.
3. Detect that `node_modules/` exists from the prior install and tell the participant the install will be fast this time.
4. Not modify any other file.

If everything passes, the second run should complete in well under a minute.

## Final message

After all checks complete:

**If all Tier 1 passed:**
```
✓ Setup complete. You're ready for the workshops.

Next steps:
- Skim the issues at https://github.com/chrisatcursor/cursor-fde-bootcamp/issues
- Fork this repo to your own GitHub account: gh repo fork --remote
- Open the warm-up exercise from the bootcamp pre-work doc when you're ready.
```

**If any Tier 1 failed:**
```
✗ Setup incomplete. Resolve the issues above and re-run /init.

The blockers were:
- <list the failing Tier 1 checks>

Apply the fixes printed inline above, then run /init again. The install step is safe to re-run.
```

## Edge cases and notes

- **Repo location check:** Before starting, verify the current directory contains a `package.json` with `"name": "calcom-monorepo"` and (if not the first run) a `.bootcamp-marker`. If neither, print: `Not in the bootcamp repo. cd to the cursor-fde-bootcamp directory and re-run /init.`
- **Active gh account:** If the participant has multiple gh accounts (`gh auth status` shows several), do not switch them — just verify *some* account is active and proceed.
- **Long-running install:** When invoking `yarn install`, stream progress to the user. Don't silently wait 20 minutes. If your runtime supports it, periodically print the elapsed time so the participant knows it hasn't hung.
- **Hooks and post-install:** `yarn install` triggers `husky install && turbo run post-install`. Both should succeed silently. If they fail, the failure surfaces in `yarn install`'s exit code — defer to that.
- **No upstream calls beyond the documented ones:** Don't try to clone other repos, talk to gh API for issue listings, or fetch external resources. /init is local-environment scope only.

## What this skill explicitly does NOT do

If a participant or another skill expects /init to install the bootcamp's scaffolding artifacts, that expectation is wrong. Refer them to the workshop curriculum:

- Workshop 1, Block 3: cohort writes their first three rules into `.cursor/rules/`
- Workshop 1, Block 4: cohort writes a code-reviewer sub-agent into `.cursor/agents/`
- Workshop 2: cohort iterates on the above plus adds a skill or two

If you, the agent running this skill, are tempted to be helpful and add a "starter" rule or sub-agent: don't. The whole bootcamp is the cohort building these artifacts themselves.
