---
name: handoff
description: Write an execution spec to .handoff/plan.md, run it with dsh (DeepSeek Harness) with full access inside a throwaway microVM on a clone of the repo, verify and review the result, report, and apply the changes to the repo when the user approves. Use when the user says "hand off" or "hand it off to dsh", "send it to the executor", "run it in dsh", "use dsh", "use deepseek", "have deepseek/dsh do it", "run it in a vm", "dsh-vm", or otherwise asks to stop planning and let DeepSeek implement the work. (handoff-local runs dsh on the host without a VM.)
allowed-tools: Read, Glob, Grep, Write, Bash(dsh-vm:*), Bash(git rev-parse:*), Bash(git status:*), Bash(git diff:*), Bash(git -C:*)
---

Do not implement anything yourself. You plan, the executor implements, you verify.

EXECUTOR: dsh-vm run (dsh --profile headless --permission-mode danger-full-access, in a microVM)

The VM sees only a clone of the repo in a run dir, the DeepSeek key and the network. Inside
it dsh runs as root with no sandbox and no approvals: it may `dnf install` system packages,
install npm/pnpm/yarn packages, compile native code (gcc, make), run headless Chromium
(`chromium-browser`), and use the network. It has no SSH keys, no gh token and no git
remote, so it cannot push. Your real repo is untouched until step 6.

If `dsh-vm` is not installed or podman/krun is unavailable, say so and offer handoff-local.

## Cache discipline

The executor's provider caches the prompt prefix: the fixed prompt in step 3, then
`.handoff/plan.md` as the executor reads it, turn after turn. Cached input costs a small
fraction of uncached input. If one byte near the top changes, everything after it is
uncached for the rest of the run, so a small "improvement" to the plan can cost far more
than it saves. These rules keep the prefix byte-identical:

- **Never rewrite `.handoff/plan.md` once the executor has started.** That includes typo
  fixes, rewording, reordering, and fixing a wrong Acceptance command. Before the first run
  (step 2) you may edit it freely. After that, add an `## Amendments` section at the end of
  the file, or append to it if it is already there. Each amendment names the section and
  item it overrides and says what is now true. Earlier text stays exactly as it is.
- **Never put anything that changes between runs in the executor prompt:** no timestamp,
  date, run ID, branch name, or per-run note. Use the step 3 command word for word. Anything
  that differs per run goes in `## Amendments`.
- If the plan seems to need a rewrite partway through, don't rewrite it. Append an
  amendment, or stop and tell the user that the task needs a new plan.

## 1. Preflight

- If the request is exploratory or leaves the scope open (e.g. "look into", "what could we
  add", several possible features), stop here. Look into it, give the user the options with
  a recommendation, and wait for them to choose. Do not write the spec until the scope is
  agreed; the spec holds decisions only.
- Run `git rev-parse --show-toplevel`. If it fails, stop: the handoff needs git as its undo
  button. Tell the user to `git init` (or offer to) and stop.
- Work from that top-level directory for every step below.
- Run `git status --porcelain -- . ":(exclude).handoff"`. If anything is listed, stop and ask
  the user to commit or stash first, so the executor's diff is exactly its own work.
- If `.handoff/.gitignore` does not exist, write it with these two lines:
  `report.md` and `last-run.log`.
- Run `dsh-vm list`. If an earlier run of this repo is listed, ask the user whether to apply
  or clean it before starting a new one; don't leave old runs piling up.

## 2. Write the spec

Write `.handoff/plan.md` with exactly these sections:

    ## Goal
    ## Files to touch
    ## Steps
    ## Acceptance
    ## Parallel work
    ## Constraints
    ## Out of scope
    ## If blocked

- Decisions only — no reasoning, no alternatives, no hedges. Assume the executor has none
  of this conversation: name exact file paths, function names, commands, and versions.
- **Acceptance** is a list of runnable commands, each with its expected result
  (e.g. `npm test` exits 0; `node greet.js` prints `hello`). No prose-only criteria.
- **Parallel work** decides delegation for the executor:
  - Split into named workstreams when the task has two or more substantial pieces that
    share no files and need no result from each other (e.g. backend endpoint + frontend
    page + docs). For each workstream list: the files it owns, its steps, and its own
    acceptance command. Then add one final integration step for the lead that runs the
    full **Acceptance** after all workstreams finish.
  - Otherwise write `None — do it sequentially.` Small tasks, tightly coupled changes,
    or anything touching the same files stay sequential; delegation overhead is not free.
- **Constraints** always includes: do not commit, push, or switch branches; do not touch
  files outside **Files to touch** except `.handoff/report.md`. Installing dependencies
  (`npm ci`, `dnf install`) and temporary files outside the repo are fine: the VM is thrown
  away.
- The clone has no `node_modules` or other ignored files. If Acceptance needs them, make
  installing them (e.g. `npm ci`) the first step.
- Browser checks: the VM has `chromium-browser` (headless) and 2 GB of RAM. Ask for one
  browser at a time. If the project already uses Playwright or Puppeteer, use its own setup
  and scripts (`npx playwright install chromium` is fine; browser downloads are cached between
  runs). For a quick one-off check in a project without one, point it at the system Chromium:
  `executablePath: '/usr/lib64/chromium-browser/chromium-browser'`.

Then check the **Acceptance** commands, before the executor spends anything on them. The
code does not exist yet, so this tests the command, not the result:

- Check each command whose flags, paths, or arguments depend on the installed tool (test
  runners, linters, build tools). Skip commands that only run a file the plan creates
  (`node greet.js`) or standard git commands.
- Make a new temporary directory outside the repo with `mktemp -d`. Put the smallest
  example there that the command needs, in the layout the plan uses (e.g. one passing test
  file in `lib/`), and run the command from that directory. This example is the only code
  you write, and it never goes in the repo.
- If the command itself fails (unknown flag, tool missing, a path read the wrong way), fix
  it in `.handoff/plan.md` and check it again. If you cannot make it work, stop and tell the
  user.
- If a command cannot be tried on a small example (a full app build, anything needing a
  running service), leave it as written and name it in your report as unchecked.

## 3. Run the executor

From the top-level directory, run the EXECUTOR with the Bash tool,
`run_in_background: true` and `timeout: 6000000` (100 minutes: dsh-vm stops the VM itself at 90,
and the Bash tool's default background limit of 30 minutes would kill a long run early). dsh-vm clones the repo (with `.handoff/`) into a new run dir,
prints its path first, and writes dsh's output to `<run dir>/.handoff/last-run.log`:

    dsh-vm run "Implement .handoff/plan.md exactly as written. Before you finish, write .handoff/report.md listing files changed, commands run and their results, anything you could not do, and the exact next step. If a step is wrong or impossible, stop and report instead of improvising. Do not commit, push, or switch branches. If the plan's Parallel work section lists workstreams, delegate each one to a teammate or subagent with your agent-team or subagent tools, give each only its own files, wait for all of them, then run the integration step yourself; say in the report which workstreams were delegated."

Tell the user the executor is running in a microVM, and that they can watch it live in a
terminal with `dsh-vm log` (Ctrl+C stops watching, not the run). Do not poll; you will be
notified when it exits.

## 4. Verify and review (read-only)

When the run finishes, take the run dir from the first line of the command's output. Every
check below runs in the run dir (`git -C <run dir> ...`, or commands with that working
directory), not in the repo:

- Read `<run dir>/.handoff/report.md`. If it is missing, read the end of
  `<run dir>/.handoff/last-run.log`.
- If the output says `timed out after N minutes`, the VM was stopped mid-run: say so first in
  the report, and treat the run dir as partial work.
- Run `git status --porcelain` and `git diff --stat` to see what actually changed.
- Run each **Acceptance** command from the plan and compare against its expected result.
- Review the code: read the full `git diff` and every new untracked file (`git diff` does
  not show those). Check it against `.handoff/plan.md` for bugs, missing or half-done steps,
  changes outside **Files to touch**, and anything **Out of scope**. Passing Acceptance does
  not end the review.
- Do not edit any file during verification.

## 5. Report

Report: exit status, files changed (from git, not the report), each Acceptance check with
pass/fail, any Acceptance commands left unchecked before the run, your code review findings
(or that you found none), anything the executor said it could not do, and the exact next
step. Where the executor's report and your verification disagree, say so plainly.

Give the run dir path and ask whether to apply the changes to the repo.

Do not re-plan, patch, or fix failures yourself unless the user asks.

## 6. Apply or clean (only when the user says so)

- Apply: from the repo's top-level directory run `dsh-vm apply <run dir>`. It replays the
  run's changes as uncommitted changes (new files untracked) and copies `report.md` and
  `last-run.log` into `.handoff/`. Then run `dsh-vm clean <run dir>`.
- Discard: run `dsh-vm clean <run dir>`.

The user reviews the diff and commits, including `.handoff/plan.md`. `report.md` and
`last-run.log` stay ignored. Never leave a run dir behind once the user has decided.
