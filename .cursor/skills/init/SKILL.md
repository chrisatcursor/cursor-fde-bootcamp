---
name: init
description: Bootcamp environment doctor. Use when the user runs `/init`, asks to set up the bootcamp repo, says "set up my bootcamp environment", or you are a cohort participant starting work in this repo for the first time. Verifies prerequisites (gh, git, Node, yarn, optionally Docker), reports each check with ✓/✗ + a fix command on failure, then asks the user for permission to install anything missing. Reports completion when the local environment is ready for the workshops.
---

# /init — Bootcamp Environment Doctor

You are running this skill inside the **Cursor FDE Bootcamp** working repo. Your job is to confirm the participant's environment is ready for the workshops.

## Execution model

Run all checks **read-only first**. Print one line per check. After all checks complete, if anything in Tier 1 or Tier 2 is missing or installable, **stop and ask the user before installing anything**.

For each check, print one line:

```
✓ <Check name> — <one-line evidence>
✗ <Check name> — <one-line failure description>
   Fix: <exact shell command or action>
```

Use plain ASCII checkmarks (✓ ✗) — they render in any terminal.

**Do not run any installation commands during the checks themselves.** The checks are observation only. After all checks, summarize the missing dependencies and ask the user whether to install them.

## Tier 1 — Required (block if any fail)

These must pass before the participant can proceed. Surface failures at the end with proposed fix commands; don't auto-install.

### 1. GitHub CLI installed

Run:

```bash
gh --version
```

- ✓ if exit 0 and output starts with `gh version`
- ✗ if `command not found` → Note for end-of-run summary: `gh` is required. Suggested install: `brew install gh` (macOS) or see https://cli.github.com/

### 2. GitHub CLI authenticated

Run:

```bash
gh auth status
```

- ✓ if any account is "Logged in" (the active account is fine; we don't enforce which one)
- ✗ if output contains "You are not logged into" → Note: `gh auth login` (interactive — don't run automatically)

### 3. Git installed and configured

Run:

```bash
git --version && git config user.name && git config user.email
```

- ✓ if all three return non-empty values
- ✗ git missing → Note: `xcode-select --install` (macOS) or platform equivalent
- ✗ user.name or user.email blank → Note: `git config --global user.name "Your Name"` and `git config --global user.email "you@example.com"` (cohort participant should choose their own values; surface as something to do, not something to auto-run)

### 4. Node version meets cal.diy requirements

cal.diy README requires Node ≥18.x. Recommended: Node 20 LTS or 22 LTS. Run:

```bash
node --version
```

- ✓ if major version ≥ 20 (recommended)
- ⚠ warn but accept if major version is 18
- ✗ if < 18 or `command not found` → Note: install Node 22 via nvm (`nvm install 22 && nvm use 22`) or via [nodejs.org](https://nodejs.org/)

There is no `.nvmrc` in this repo.

### 5. Yarn 4.x available

Yarn 4.12.0 is pinned via corepack in `package.json` (`"packageManager": "yarn@4.12.0"`). Corepack ships with Node 16+ but may need to be enabled. Run:

```bash
yarn --version
```

- ✓ if output starts with `4.`
- If output is `1.x` (Yarn Classic): Note: `corepack enable` then re-run `yarn --version` from within this repo
- If `command not found`: Note: `corepack enable && corepack prepare yarn@4.12.0 --activate`

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

`yarn install` is **the one Tier 1 step that takes meaningful time**. Run it as part of the checks, not at end-of-run install consent — but only after confirming with the user that they're ready for a 15–20 minute wait. If `node_modules/` already exists, `yarn install` is fast (a few seconds) and can run without asking.

Check status:

- ✓ if exit code 0 (warnings OK; "Done with warnings" still counts as success)
- ✗ if exit code non-zero → Note: read the error; common causes are network failure, disk full, or wrong yarn version (re-check #5)

## Tier 2 — Recommended (warn, do not block)

These improve the experience but aren't required to start the workshops. Surface as warnings at the end of the run; don't auto-install.

### 7. Docker (recommended for previewing the app)

Cal.diy's `yarn dx` command spins up a Docker Postgres, runs migrations, and seeds test users in one command. Without Docker, previewing the running app is significantly more involved (manual Postgres install + configuration). Cohort participants who want to **build and click through the app locally** should install Docker.

```bash
docker --version && docker info
```

- ✓ if `docker --version` succeeds AND `docker info` succeeds (daemon running)
- ⚠ if Docker is installed but the daemon isn't running → Note: open Docker Desktop or start the docker daemon
- ⚠ if Docker is not installed → Note: install Docker Desktop or Rancher Desktop. Recommended path; without it, previewing the app requires manual Postgres setup. Don't block — surface in summary.

### 8. `.env` file exists

```bash
ls .env
```

- ✓ if `.env` exists at repo root
- ⚠ if not → Suggest: `cp .env.example .env`, then edit secrets per the README's "Setting up your first user" section. Don't auto-copy — the participant should make this decision themselves.

### 9. Free disk space ≥ 5 GB

`yarn install` needs ~1 GB; database, build artifacts, and Docker images push you past 5 GB total.

```bash
df -h .
```

- ✓ if available space ≥ 5 GB
- ⚠ if less → Recommend cleanup before continuing.

## Tier 3 — Informational (no pass/fail)

Print these for situational awareness; do not gate.

### 10. OS and known quirks

```bash
uname -s
```

- macOS on Apple Silicon (`uname -m` returns `arm64`): note that some Docker images need the `-arm` suffix per the cal.diy README.
- Windows: cohort participants on Windows should use WSL2 — note this.
- Linux: usually fine; surface the kernel version for context.

### 11. `.bootcamp-marker` file

If `.bootcamp-marker` exists at repo root:
- Print: `i  Bootcamp marker found — this is the cohort working repo.`

If it doesn't exist:
- Create it with these contents (this is the one file /init writes during checks):

```
Cursor FDE Bootcamp working repo.
This file is created by /init. Do not delete unless you understand what it does.
```

- Print: `i  Bootcamp marker created.`

Use this on subsequent /init runs to confirm the participant is in the right repo. The file is small, harmless, and idempotent — no consent needed.

## End-of-run install consent

After all checks have run, gather what failed or warned and present it to the user as a single summary:

```
Summary

Tier 1 (required):
  - <list each failed Tier 1 check, with the suggested fix command>

Tier 2 (recommended):
  - <list each warned Tier 2 check, with the suggested fix command>

Several of the items above can be installed automatically. Would you like
me to install them? Reply with the numbers from the list, "all", or "none".
```

**Then stop and wait for the user's response.** Do not install anything until they answer.

When the user responds:
- For each item the user approves, run the suggested install command.
- For items requiring interactive setup (e.g. `gh auth login`, `git config user.name`), explain what the user needs to do themselves rather than running them automatically.
- For `yarn install`, confirm separately (it takes 15–20 minutes); don't bundle it with quick installs.

After installs, re-run the affected checks once and report the final state.

## Idempotency

A second run of /init should:
1. Pass through every check the same way (no destructive actions).
2. Detect the existing `.bootcamp-marker` and skip recreating it.
3. Detect that `node_modules/` exists from the prior install and tell the participant `yarn install` will be fast this time.
4. Not modify any other file beyond `.bootcamp-marker`.

If everything passes, the second run should complete in well under a minute.

## Final message

After all checks complete (and any approved installs run):

**If all Tier 1 passed:**
```
✓ Setup complete. You're ready for the workshops.

Next steps:
- Open the warm-up exercise from the bootcamp pre-work doc when you're ready.
```

**If any Tier 1 failed (or were declined for install):**
```
✗ Setup incomplete. Resolve the issues above and re-run /init.

The blockers were:
- <list the failing Tier 1 checks>
```

## Edge cases and notes

- **Repo location check:** Before starting checks, verify the current directory contains a `package.json` with `"name": "calcom-monorepo"`. If not, print: `Not in the bootcamp repo. cd to the cursor-fde-bootcamp directory and re-run /init.`
- **Multiple gh accounts:** If the participant has multiple gh accounts (`gh auth status` shows several), do not switch them — verify *some* account is active and proceed.
- **Long-running install:** When invoking `yarn install`, stream progress to the user. Don't silently wait 20 minutes. If your runtime supports it, periodically print elapsed time.
- **Hooks and post-install:** `yarn install` triggers `husky install && turbo run post-install`. Both should succeed silently. If they fail, the failure surfaces in `yarn install`'s exit code — defer to that.
- **No upstream calls beyond the documented ones:** Don't try to clone other repos, talk to gh API for issue listings, or fetch external resources. /init is local-environment scope only.
- **Don't install without consent.** Even if a fix is "obvious," the user should decide. The single exception is creating `.bootcamp-marker`, which is small and trivially reversible.
