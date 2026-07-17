# arjayby/skills

Agent skills that automate the tail end of [Matt Pocock's engineering workflow](https://github.com/mattpocock/skills).

## Install

```bash
npx skills@latest add arjayby/skills
```

Or as a Claude Code plugin:

```bash
claude plugin marketplace add arjayby/skills
claude plugin install arjayby-skills@arjayby
```

These skills build on `mattpocock/skills` and don't replace it — install that too:

```bash
npx skills@latest add mattpocock/skills
```

## Skills

### `/ship-frontier`

Ships an entire ticket frontier unattended.

Matt's workflow has a human-in-the-loop front half and a repetitive back half:

```
/grill-me  or  /grill-with-docs     ← you, thinking
        ↓
/to-spec                            ← spec from the grilling conversation
        ↓
/to-tickets                         ← spec sliced into tracer-bullet tickets
        ↓
/ship-frontier                      ← this skill: the rest, unattended
```

`/to-tickets` signs off by telling you to *work the frontier one ticket at a time with `/implement`, clearing context between tickets.* Doing that by hand means, per ticket: check out main, pull, branch, `/implement`, verify, push, PR, merge, clear context, repeat. `/ship-frontier` is that loop.

For each unblocked `ready-for-agent` ticket, in dependency order:

1. `git checkout main && git pull --ff-only`
2. `git checkout -b feat/<slug>`
3. `/implement` the ticket **in a fresh context**
4. verify: clean exit, a real commit, clean tree, typecheck and tests green
5. push, open a PR that `Closes #n`
6. wait for CI to go green, then squash-merge

Then it recomputes the frontier — the merge just closed a blocker, so dependents become shippable — and takes the next ticket. It stops at the first failure.

```bash
/ship-frontier              # drain the whole frontier (asks first)
/ship-frontier --dry-run    # print the plan, ship nothing
/ship-frontier --yes        # skip the confirmation — required headless
/ship-frontier 14 15        # just these, in this order
/ship-frontier --no-merge   # stop at each PR
```

Run it headless, from a terminal:

```bash
claude -p "/ship-frontier --dry-run" --dangerously-skip-permissions   # see the plan
claude -p "/ship-frontier --yes" --dangerously-skip-permissions       # ship it
```

An interactive session's permission classifier denies the nested `claude -p --dangerously-skip-permissions` each ticket needs, so the loop dies on ticket one. Running the whole thing headless means the parent already bypasses permissions and nothing is left to deny. Alternatively allow `Bash(claude -p:*)` in `.claude/settings.json` and run it interactively.

**`--yes` is required headless.** With nobody to answer the confirmation, a headless run without it prints the plan and ships nothing — the invocation is never taken as consent.

#### How it works

**The frontier is self-driving.** GitHub's `issue_dependencies_summary.blocked_by` counts only *open* blockers. A PR that says `Closes #n` closes its issue on merge, which drops the open-blocker count on every dependent. So the loop needs no state file — it just re-runs the frontier query each pass and takes the lowest-numbered ticket with zero open blockers.

**Each ticket gets a genuinely fresh context** via a nested headless run (`claude -p "/implement ..."`). That's what `/to-tickets` asks for, and it's the only way to invoke `/implement` at all — it's `disable-model-invocation: true`, so no skill can reach it through the Skill tool. It also keeps the orchestrating session small enough to drain a long queue.

**The skill never implements anything itself.** `/implement` already applies `/tdd`, typechecks, runs tests, calls `/code-review`, and commits to the current branch. `/ship-frontier` owns only the ring around it: pick, branch, hand off, verify, push, PR, merge.

#### Requirements

- Issue tracker is **GitHub**, configured by `/setup-matt-pocock-skills` — the loop needs a machine-readable frontier and a PR surface
- Blocking edges as **GitHub native issue dependencies**, which `/to-tickets` writes
- Tickets labelled `ready-for-agent`
- `gh` authenticated, clean working tree

**It waits for CI itself, rather than trusting GitHub to.** `gh pr merge --auto` only defers a merge when a *required* status check exists, and required checks come from branch protection — unavailable on private repos without GitHub Pro. Where protection is off, `--auto` merges immediately and CI becomes decorative: the code lands on `main` and the workflow goes red behind it. So the loop blocks on `gh pr checks --watch --fail-fast` before merging, which makes CI a real gate on any repo.

#### Before you run it

This merges to your default branch with no human review, and each ticket runs with `--dangerously-skip-permissions`. Two things stand between generated code and `main`: the local gate (clean exit, real commit, clean tree, typecheck, tests) and CI. **If your repo has no CI, only the local gate is left.** Worth adding a workflow before you leave this running.

It confirms the batch once before the first ticket, then runs to completion. Use `--dry-run` first.

## License

MIT
