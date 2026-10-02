# devloop

A driver that runs a spec-driven software project one step at a time, with **one fresh headless
Claude Code session per step**, and keeps the rules that judge the agent where the agent cannot
reach them.

**Status: design only.** The loop was planned and prototyped inside
[jobmatch-microservices](https://github.com/Yusuprozimemet/jobmatch-microservices), then taken out
before its first run (October 2026). The one-time setup was more than that migration needed. This
repository is where it can grow into a project of its own.

## Where it came from

jobmatch-microservices migrates a monolith to microservices one day spec at a time. Each step is
the same shape: an audit, one PR per track, a closing PR, then waiting for the maintainer to say
"merged". Two things made a loop attractive:

- **Context cost.** Every reply re-reads the whole conversation. By Day 17, 64% of the main
  sessions' context tokens were read after a session had passed 300k. A fresh session per step
  removes that.
- **The "say merged" round trip.** The maintainer was the clock. A driver can poll for the merge
  and start the next step on its own.

The prototype is still in that repository's history:

| Piece | Source | Removed by |
|---|---|---|
| Design doc | [`docs/agentic-loop.md`](https://github.com/Yusuprozimemet/jobmatch-microservices/blob/42b6009/docs/agentic-loop.md) | [#267](https://github.com/Yusuprozimemet/jobmatch-microservices/pull/267) |
| Driver | [`scripts/dev-loop.py`](https://github.com/Yusuprozimemet/jobmatch-microservices/blob/42b6009/scripts/dev-loop.py) | [#267](https://github.com/Yusuprozimemet/jobmatch-microservices/pull/267) |
| Agent guard | [`.github/workflows/agent-guard.yml`](https://github.com/Yusuprozimemet/jobmatch-microservices/blob/31568ad/.github/workflows/agent-guard.yml) | [#268](https://github.com/Yusuprozimemet/jobmatch-microservices/pull/268) |

## The idea

```
driver (outside the model)
  loop:
    next = the project's "next step" (from its spec tracker)
    a stopping point the plan names      -> run the auditor step, notify, exit 0
    an open PR                           -> wait for it
    budget or wall clock spent           -> notify, exit 3
    claude -p <step prompt>   fresh session, one step, one PR, --max-budget-usd, timeout
      last line: STATUS: PR <url> | STATUS: STOP <reason>
    STOP                                 -> notify, exit 2
    PR  -> wait for CI
           red: one fix session on the same branch; red again -> notify, exit 2
           green: attended   -> poll until a human merges (closed unmerged -> exit 2)
                  unattended -> merge gates -> gh pr merge --merge
    after the merge: archive the transcript, append to the run log, pull main
```

One step is exactly one unit of work: an audit and its spec-change PR, one track, or a closing
PR. The session does that step, opens its PR, and ends. **The driver, not the model**, decides
when to go on, merges (only in unattended mode), and writes the run log.

### Two modes, kept apart

| Mode | Who merges | What it tests |
|---|---|---|
| `attended` (default) | a human; the driver polls until the PR is merged | the same condition as a human-in-the-loop workflow, minus the round trip |
| `unattended` (`--auto-merge`) | the driver, only when every merge gate passes | the agent without review. Every merged PR is reviewed afterwards, and what would have been rejected is logged as data |

The loop is itself a new variable, so human review is not removed at the same time. Run attended
first, for long enough to compare with what came before; then unattended. Every switch of mode is
a row in the run log.

### Hard stops (the session ends with `STATUS: STOP`)

- Anything the project's rules say to stop and ask about.
- An audit finding that needs judgement rather than a factual correction, or any change to the
  plan itself.
- A step that would need a disabled or skipped test, a lowered threshold, or a new or upgraded
  dependency.
- A diff over the size limit, or more than one track's work in one PR.
- Any edit to the rules: the agent instructions, the plan, CI workflows, or the driver.

### Merge gates (unattended only; checked by the driver, not the session)

- Every required check is green, including the agent guard.
- The diff is under the size limit, and the PR is the only one open.
- The PR description records a break on purpose: the change was broken, the tests were seen to
  fail, the break was reverted.
- A daily cap on merged PRs; one more stops the run.

## Enforcement that lives where the agent cannot reach it

Local hooks and deny rules are a convenience: `sed -i` or `git push origin +HEAD` gets past them.
What holds:

- **An agent guard workflow** on `pull_request_target`. It runs the copy on `main`, never the
  PR's, and never checks out the PR's code; it reads the diff and the check runs through the API.
  Two gates:
  1. **Protected changes.** A PR not opened by the maintainer fails if it touches the rules or the
     evidence (agent instructions, plan, `.github/`, the driver, build files, contract tests,
     applied migrations), adds `@Disabled`/`@Ignore`, or removes more tests than it adds. A file
     whose diff the API does not return fails too. Override: the maintainer approves the PR's
     commit and re-runs the job.
  2. **Every other check is green.** Path-filtered workflows report nothing on PRs that miss their
     paths, so they cannot be required checks. The guard waits for all check runs on the PR's head
     and fails if any did not pass; it is the one required check that stands for them.
- **A separate GitHub identity for the agent**, so the guard can tell its PRs from the
  maintainer's.
- **Branch protection:** no force push, admins included; the guard required; and one required
  approving review once the agent has its own identity.
- **Two checkouts.** The driver runs from its own checkout of `main`; the sessions work in
  another. The run log and transcripts stay in the driver's checkout, which the sandbox never
  mounts.

```
driver checkout                 work checkout
  driver script                   the session edits, branches, commits, pushes (as the bot)
  run log                         mounted into the sandbox as /work
  transcripts
```

- **A sandbox:** a container or WSL2 distribution that mounts only the work checkout, entered
  through something like `--claude "docker exec -i -w /work jm-sandbox claude"`. Claude Code's own
  sandbox covers shell commands only and does not run on native Windows. Inside: the Docker
  socket, network to the package registry, GitHub and the issue tracker; no `.env` or other
  secrets.

## Setup, and what we learned before the first run

The setup is where the prototype stopped. In order:

1. **Make the agent guard a required check.** GitHub only offers a check once it has run on a PR.
   Pin it to GitHub Actions as its source, so a status posted from elsewhere cannot satisfy it.
2. **Create the bot identity and its token.** Give the bot **Write**, never Admin: an admin falls
   inside the review bypass below and could merge its own PRs.
   - A **fine-grained token cannot reach a repository owned by another personal account**, only
     the bot's own repositories or an organization's. For a repository on a personal account, use
     a classic token with `repo` scope (fine when the bot sees only that one repository), or a
     GitHub App installed on that repository alone.
3. **Require one approving review with a ruleset, not classic branch protection.** With
   "include administrators" on, a classic required review blocks the maintainer's own PRs too:
   GitHub does not let authors approve their own. A ruleset with **Repository admin** on its bypass
   list makes bot PRs wait for approval while the maintainer's still merge.
4. **Set up the sandbox and the two checkouts.** Check the identities: `gh auth status` shows the
   bot inside the sandbox and the maintainer in the driver checkout.
5. **Prove the guard can fail.** Open a throwaway bot PR that touches a protected path. The guard
   must fail it and the ruleset must block the merge. Close it unmerged.
6. **Tag the baseline** (`git tag loop-start && git push origin loop-start`), only after all of
   the above works. It is the rollback point.
7. **Pilot, attended:** two steps, then look at everything before going on.

## Budget

A per-step cost cap (`--max-budget-usd`), a wall-clock timeout per step, one CI-fix retry per PR,
and a ceiling for the whole run. Hitting a ceiling is a normal stop, logged as one.

## Records

- **Primary:** the session transcripts (copied per step into the driver's checkout) and the CI
  logs.
- **The run log**, written by the driver, not the model: per step the time, mode, PR, CI result,
  who merged it, cost, and the stop reason if any.
- **Secondary:** what the agent wrote about its own work (closing notes, end-of-phase reports),
  compared afterwards with the primary record.

Any stop opens an issue labelled `loop-stop` with the reason and the last lines of the log, so a
stall is seen when it happens.

## Undoing a run

Every merge is a merge commit, so one step comes out with `git revert -m 1 <merge>`, and a whole
run with those reverts in reverse order back to the tag. A database migration that already ran is
undone by a new migration, never a revert.

## Open questions for this repository

- **Generic or per-project?** The prototype read jobmatch's own "next step" tracker and rules.
  A standalone version needs an interface for "what is the next step" and "what are the rules".
- **Guard as a reusable workflow**, so a project calls it instead of copying it.
- **Sandbox image:** a published container with Claude Code, `gh` and Docker-in-Docker or the
  socket, so setup step 4 is one command.
- **Cheaper setup.** The setup is the reason the prototype never ran. Whatever can be scripted
  (protection, ruleset, tag, identity checks) should be.
