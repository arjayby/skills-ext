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

## Running it

The headless invocation is the whole point, so get it right — it needs `--yes`, or the run plans the batch, asks a question nobody is there to answer, and exits having shipped nothing:

```bash
claude -p "/ship-frontier --yes" --dangerously-skip-permissions \
  --output-format stream-json --verbose
```

Two flags are doing real work there:

- **`--yes`** satisfies the step-3 confirmation. Print mode has no interactive user, so a run without it always no-ops.
- **`--output-format stream-json --verbose`** is the only way to see progress. Plain `claude -p` prints *nothing* until the entire run finishes — a blank terminal for however many hours the batch takes, indistinguishable from a hang. Per-ticket detail lands in `.scratch/ship-frontier/<n>.log`, which `tail -f` follows.

**Headless is preferred because an interactive session will usually refuse to launch the nested run** — the permission classifier denies a nested `claude -p` that bypasses permissions, and the loop dies on its first ticket. To run it interactively anyway, allow the nested call in `.claude/settings.json` first:

```json
{ "permissions": { "allow": ["Bash(claude -p:*)"] } }
```

If a nested run is denied, stop and tell the user which of these to do. Never fall back to implementing the ticket yourself in the current context — that defeats the fresh-context guarantee and quietly turns one bounded run into an unbounded one.

## Arguments

- _(none)_ — ship every ticket that is or becomes unblocked, until the frontier is empty
- `<n> [<n>...]` — ship only these issues, in the order given, skipping the frontier query
- `--yes` — skip the step-3 confirmation and start shipping. **Required for any headless run.** Interactively it is optional; without it you get the plan and a prompt.
- `--no-merge` — stop after opening each PR. Every later ticket then branches off a `main` that lacks the earlier work, so this is only sound for a single ticket or for tickets with no edges between them. Say so if the user combines it with a dependent batch.
- `--dry-run` — print the plan and stop. Beats `--yes` if both are passed.

## Process

### 1. Preflight

Stop with a clear message if any of these fail. Do not try to repair them:

- `git status --porcelain` is empty — uncommitted work would be swept into the first ticket's branch and attributed to it
- `gh auth status` succeeds
- the tracker is GitHub

Then run `git fetch origin` before computing anything. A stale local default branch makes already-merged work look unbuilt — squash-merged branches keep their local commits, so `git log main..<branch>` reports work that is in fact already on the remote. Diagnose against `origin/<default>`, never a local branch you haven't just fetched.

Then record, once:

- the default branch — `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
- **whether the repo has CI** — does `.github/workflows/` hold anything triggering on `pull_request`? This decides whether step 4h waits.
- **the local gate** — the commands that must pass before you push.

Derive the local gate from the CI workflow when there is one: read it and mirror its sequence exactly. A local gate that differs from CI only discovers the same failure later and slower. With no CI, fall back to the repo's **typecheck** and **test** commands from `package.json` scripts, a `Makefile`, or `CLAUDE.md` / `AGENTS.md`.

**Mirror CI's order, not just its commands.** A build step often exists to generate files a later step depends on — gitignored codegen a typecheck cannot resolve without it. Reorder that and the gate fails on errors that have nothing to do with the ticket.

Find all of this now, not mid-loop. If the repo has no CI, the local gate is the *only* thing between generated code and the default branch — say so in the confirmation at step 3.

### 2. Compute the plan

A ticket belongs in the plan when it is open, labelled `ready-for-agent`, **and is actually a ticket**.

```bash
gh issue list --label ready-for-agent --state open --json number,title --jq '.[] | "\(.number)\t\(.title)"'
```

**Screen out parent specs before anything else.** `/to-spec` publishes specs and PRDs as issues in the same tracker, and a spec can easily carry `ready-for-agent` — usually because it was labelled before it was sliced. Handing one to `/implement` turns it loose on an entire feature in a single run: the opposite of a tracer bullet, and its children are usually already built. Two independent signals, either one disqualifying:

**It has no acceptance criteria.** `/to-tickets` always writes `## Acceptance criteria` with `- [ ]` checkboxes. A spec has none.

```bash
gh issue view <n> --json body -q .body | grep -cE '^- \[[ x]\]'
```

**Other issues name it as their parent.** `/to-tickets` opens each child with a `## Parent` section pointing back at the spec. Build the map once and check every candidate against it:

```bash
gh issue list --state all --limit 200 --json number,body \
  --jq '[ .[] | select(.body != null) | select(.body | test("## Parent"))
          | { child: .number,
              parent: (.body | capture("## Parent\\s+#(?<p>[0-9]+)") | .p | tonumber) } ]
        | group_by(.parent) | map({ parent: .[0].parent, children: [.[].child] | sort })'
```

Anything appearing as a `parent` is a spec. Exclude it, and name it in the step-3 confirmation along with the reason. **Never implement one, and never close or relabel one** — `/to-tickets` is explicit that parent issues are not yours to modify. If every child of a parent is closed, say so and let the user decide what to do about it.

Then, for each surviving ticket, read its open-blocker count. `blocked_by` counts only *open* blockers, which is exactly the live gate:

```bash
gh api repos/{owner}/{repo}/issues/<n> --jq '.issue_dependencies_summary.blocked_by'
```

Zero means shippable now. Non-zero means a blocker is still open — it still belongs in the plan, because this run will merge those blockers and clear it.

Order the plan **ascending by issue number**: `/to-tickets` publishes in dependency order, so the lowest open number is the earliest slice. Don't trust that ordering blindly — you re-verify each ticket's blocker count at (4a) before touching it.

If nothing survives the screen, report what you found and why, then stop.

### 3. Confirm once

Print the ordered plan — issue number, title, branch name, and for each blocked ticket the blockers this run will clear first. Then state plainly:

- anything the step-2 screen excluded, and why — a labelled spec is a mislabelling the user will want to know about
- each ticket will be implemented, PR'd, and **squash-merged into `<default>` with no further review**
- what the verification gate actually is, and whether CI exists to back it up
- the loop stops at the first failure and leaves that branch open for inspection

Then, depending on how you were invoked:

- **With `--yes`** — print the plan and start shipping immediately. Ask nothing. A question here has nobody to answer it, so the run would end having shipped nothing while reporting success.
- **Without `--yes`** — take one confirmation. After that, run to completion without prompting again.

On `--dry-run`, stop here regardless.

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
4. the local gate from (1) passes, run in its recorded order

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

**h. Wait for CI — if preflight found any.** Skip straight to (i) when the repo has no workflows; there is nothing to wait for and `gh pr checks` exits non-zero when no checks are reported.

```bash
gh pr checks <pr> --watch --fail-fast
```

**Do not delegate this wait to GitHub.** `gh pr merge --auto` only defers a merge when a *required* status check exists, and required checks come from branch protection — which is unavailable on private repos without GitHub Pro. Where protection is off, `--auto` merges immediately and CI becomes decorative: the code lands on the default branch and the workflow simply goes red behind it. Waiting here is what makes CI a gate rather than a notification.

A non-zero exit means a check failed. **Stop the loop**, exactly as at (e) — leave the branch and the open PR alone, and report the failing check.

**i. Merge.** `gh pr merge <pr> --squash --delete-branch`

**j. Confirm the close.** `gh issue view <n> --json state -q .state` must be `CLOSED`. If it isn't, the auto-close didn't fire — close it yourself with `gh issue close <n>`, or the next frontier query hands you this same ticket again. Add it to the shipped set either way.

Loop back to (a).

### 5. Report

One line per ticket: number, title, PR link, merged or failed. Then whatever remains on the frontier.

If the loop stopped early, **lead with the failure** — which ticket, which of the four checks broke, the branch name, and the log path — so the user can pick it up by hand. Then list what shipped before it and what never got attempted.
