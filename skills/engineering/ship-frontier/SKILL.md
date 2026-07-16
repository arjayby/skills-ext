---
name: ship-frontier
description: Ship the whole ticket frontier unattended — for each unblocked ready-for-agent ticket in dependency order, branch off latest main, run /implement in a fresh context, then verify, push, open a PR, and squash-merge. Stops at the first failure.
disable-model-invocation: true
---

# Ship Frontier

`/to-tickets` signs off by saying: *work the frontier one ticket at a time with `/implement`, clearing context between tickets.* This skill is that loop, plus the branch/PR/merge mechanics around each ticket.

It runs unattended and **merges to the default branch with no human review**. Confirm the batch once; then it runs to completion or stops dead at the first failure.

## Scope

`/implement` already applies `/tdd`, typechecks, runs the test suite, calls `/code-review`, and **commits to the current branch**. Never duplicate that work and never implement anything yourself inside this skill.

This skill owns only the ring around it:

> pick ticket → sync → branch → hand off to `/implement` → verify → push → PR → merge → repeat

## Requirements

- **The issue tracker must be GitHub.** Read `docs/agents/issue-tracker.md` to confirm. The loop needs a machine-readable frontier and a PR surface; a local-markdown tracker has neither. Any other tracker: say so plainly and stop.
- Blocking edges should be **GitHub native issue dependencies** — what `/to-tickets` writes. Only if the repo has no dependency edges at all, fall back to parsing a `Blocked by: #n` line from the issue body and checking each referenced issue is closed.

## Arguments

- _(none)_ — ship every ticket that is or becomes unblocked, until the frontier is empty
- `<n> [<n>...]` — ship only these issues, in the order given, skipping the frontier query
- `--no-merge` — stop after opening each PR. Every later ticket then branches off a `main` that lacks the earlier work, so this is only sound for a single ticket or for tickets with no edges between them. Say so if the user combines it with a dependent batch.
- `--dry-run` — print the plan and stop

## Process

### 1. Preflight

Stop with a clear message if any of these fail. Do not try to repair them:

- `git status --porcelain` is empty — uncommitted work would be swept into the first ticket's branch and attributed to it
- `gh auth status` succeeds
- the tracker is GitHub

Then record, once:

- the default branch — `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
- the repo's **typecheck** and **test** commands, from `package.json` scripts, a `Makefile`, or `CLAUDE.md` / `AGENTS.md`

Those two commands are your pre-push gate. Find them now, not mid-loop. If the repo has no CI workflows, they are the *only* thing between generated code and the default branch — say so in the confirmation at step 3.

### 2. Compute the plan

A ticket is on the **frontier** when it is open, labelled `ready-for-agent`, and has no open blockers.

```bash
gh issue list --label ready-for-agent --state open --json number,title --jq '.[] | "\(.number)\t\(.title)"'
```

For each, read its open-blocker count. `blocked_by` counts only *open* blockers, which is exactly the live gate:

```bash
gh api repos/{owner}/{repo}/issues/<n> --jq '.issue_dependencies_summary.blocked_by'
```

Zero means shippable now. Non-zero means a blocker is still open — but it will drop to zero as this run merges those blockers, so it belongs in the plan too.

Order the plan **ascending by issue number**: `/to-tickets` publishes in dependency order, so the lowest open number is the earliest slice. Don't trust that ordering blindly — you re-verify each ticket's blocker count at (4a) before touching it.

If nothing carries the label, report what's left and why, then stop.

### 3. Confirm once

Print the ordered plan — issue number, title, branch name, and for each blocked ticket the blockers this run will clear first. Then state plainly:

- each ticket will be implemented, PR'd, and **squash-merged into `<default>` with no further review**
- what the verification gate actually is, and whether CI exists to back it up
- the loop stops at the first failure and leaves that branch open for inspection

Take one confirmation. After that, run to completion without prompting again.

On `--dry-run`, stop here.

### 4. Ship each ticket

Keep a **shipped set** in memory for the run. Never pick an issue already in it. This is the backstop against the one failure mode that can loop forever: a ticket whose auto-close silently didn't fire, reappearing on the next frontier query and getting implemented twice.

Each iteration, recompute the frontier and take the **lowest-numbered unblocked ticket** not in the shipped set. Recomputing rather than walking the step-2 list is what lets a merge unblock its dependents mid-run.

**a. Sync.** `git checkout <default> && git pull --ff-only` — the previous ticket's merge lands here, and this is what unblocks its dependents.

**b. Branch.** `git checkout -b <type>/<slug>` — `<type>` from the repo's convention (`feat`, `fix`, `chore`), `<slug>` a kebab-case of the issue title.

**c. Record HEAD.** `git rev-parse HEAD`. You will compare against this to prove `/implement` actually committed.

**d. Hand off to `/implement` in a fresh context.** A nested headless run is the only way to give each ticket a clean context window *and* invoke `/implement` at all — it is `disable-model-invocation`, so no skill can reach it through the Skill tool.

```bash
mkdir -p .scratch/ship-frontier
claude -p "/implement GitHub issue #<n>. Fetch it with: gh issue view <n> --comments — implement exactly its acceptance criteria and nothing beyond them. You are already on branch <branch>: do NOT create a branch, do NOT push, do NOT open a PR, do NOT merge, do NOT touch any other issue. Commit to the current branch when you are done." \
  --dangerously-skip-permissions \
  > .scratch/ship-frontier/<n>.log 2>&1
rc=$?
```

Redirect; never pipe into `tee`. A pipe hands you `tee`'s exit code instead of Claude's, and the exit code is half your gate. Read the tail of the log when something fails.

**e. Verify.** This is the gate CI would otherwise be. All four must hold:

1. `rc` is `0`
2. `git rev-parse HEAD` differs from (c) — `/implement` committed something
3. `git status --porcelain` is empty — it left nothing behind
4. the typecheck and test commands from (1) both pass

Any failure → **stop the entire loop.** Leave the branch and its commits exactly as they are. Do not retry, do not patch it yourself, do not skip ahead to the next ticket. Report and hand back.

**f. Push.** `git push -u origin <branch>`

**g. Open the PR.** Conventional-commit title matching the repo's existing PR style:

```bash
gh pr create --title "<type>: <issue title>" --body "$(cat <<'EOF'
<one-paragraph summary of the behaviour this slice makes work>

Closes #<n>
EOF
)"
```

`Closes #<n>` is load-bearing. It closes the issue on merge, and closing is what drops the open-blocker count on every dependent. Omit it and the frontier never advances.

**h. Merge.** `gh pr merge <pr> --squash --delete-branch`

**i. Confirm the close.** `gh issue view <n> --json state -q .state` must be `CLOSED`. If it isn't, the auto-close didn't fire — close it yourself with `gh issue close <n>`, or the next frontier query hands you this same ticket again. Add it to the shipped set either way.

Loop back to (a).

### 5. Report

One line per ticket: number, title, PR link, merged or failed. Then whatever remains on the frontier.

If the loop stopped early, **lead with the failure** — which ticket, which of the four checks broke, the branch name, and the log path — so the user can pick it up by hand. Then list what shipped before it and what never got attempted.
