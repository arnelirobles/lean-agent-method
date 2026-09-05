# The lean agent method

How I run AI coding agents over a ticket backlog without burning a plan in a night. Written after doing exactly that, on my own project, in September 2026. The story is in the article; this is the method.

The numbers here are mine, at list prices, on one .NET codebase. Treat them as a shape, not a benchmark.

## The short version

A cheaper model drafts every change. Scripts, not instructions, run the mechanical checks. A cheap critic reviews every change against six fixed questions. The expensive model only sees what the critic cannot close. At most four changes in flight.

That took me from about 66 dollars of model use per change to about 25, with no drop in what got caught.

## 1. Triage inline, no agents

Read the open issues once, yourself. Sort each into a tier and do the cheap ones by hand.

| Tier | What |
| --- | --- |
| 0 | docs, tests only, config, CI files, README sweeps |
| 1 | a feature inside one module, additive |
| 2 | core, auth, tenancy, data shape, anything reachable without a login |

Close duplicates and anything already solved by other means while you are in there. Do not file follow-ups. A review finding is fixed in the change or dropped. A backlog that grows while you work is the failure this whole method exists to prevent.

## 2. The cascade

The cheap model drafts every tier. Before a change is opened, the preflight script runs the mutation check: the change description names the production line each new test depends on, the script reverts that line, and the named tests must fail. A test that passes either way is a gate failure, not a nit.

A second cheap agent, fresh context, reviews every change in a checked-out copy it can build and run. Six questions, each answered yes or no with the line that proves it:

1. Was the mutation reverted, and did the named tests fail?
2. Is every new route in every inventory, with the gate the issue asked for?
3. Does every session and raw query in the change name its tenant?
4. Does every new document, event or index have a migration or an upcaster?
5. Does anything from an exception, a request or a user string reach a log or a response unexamined?
6. Does any read-then-write on a shared row, or any outbound call to a URL that is not a constant, name its lock or its bound?

The expensive model takes the branch and the critic's list only when a no is not fixed in one round, or when a tier 2 answer is unsure. It does not start over.

Your continuous integration and whatever code review bot you use still run. They confirm. They do not decide.

## 3. The diff decides whether a critic runs, not the ticket

Tier from the ticket title is wrong, because the tickets that read as small are the ones that touch auth. `needs-review.sh` prints every rule a change fires; any output means a critic runs.

The rules in the script are specific to my codebase. The categories transfer: authentication and permissions, anything anonymous, raw SQL, schema and migrations, secrets and logging, concurrency and background work, deletion and retention, build and dependency files, and a test file that removes assertions. Docs, additive fields and assertion-only test additions fire nothing.

## 4. Scripts, not instructions

Three scripts do what I used to write into every agent brief. They live in the repository and are public:

- `scripts/preflight.sh` builds, runs the named test classes, checks the changelog and module versions, restores in locked mode before building so a stale lock file cannot pass, fails when a test filter matches zero tests, scans added lines including untracked files for house style, and parses any changed workflow file with a duplicate-key-rejecting parser.
- `scripts/sync-master.sh` merges the default branch, regenerates lock files when a project file changed, and reports conflicts.
- `scripts/needs-review.sh` prints the rules above.

They are in [BaryoDev/barakoCMS](https://github.com/BaryoDev/barakoCMS) under `scripts/`. Copy the shape, replace the checks with your own.

One trap worth inheriting along with the shape: `needs-review.sh` takes no arguments and diffs the working tree, folding untracked files in as new. That is deliberate, because a rule that only reads committed changes misses the file an agent has written and not yet added. It also means running it in a checkout full of other work walks all of that too. Run it inside a clean copy of the branch. I lost a few minutes to this before writing it down.

An instruction in a prompt is forgotten within a day. A script is not, and it costs no tokens to obey.

## 5. Send work back to the agent that did it

A review finding, a failed build or a conflict goes back to the agent that wrote the change, not to a new one. It still holds the context. A fresh agent re-reads the codebase to reach the same conclusion, and pays for the reading.

I measured this rather than assuming it. Resuming a drafter to make a two-line fix its reviewer had asked for cost about 4 thousand tokens. The same agent's first pass, which included reading the repository, had cost 154 thousand. The saving is not marginal.

There is a second reason, which matters more than the money. The agent that wrote the change knows what it already checked. A new one does not, so it either re-checks everything or, more often, assumes the first agent got the basics right and looks only at what you pointed it at.

## 6. Four changes in flight

The merge queue takes one change at a time and re-tests each against everything before it. Ten in flight means every later one gets brought up to date, re-tested, and sometimes repaired. That rework costs more than the parallelism saved.

## 7. Measure every batch

`workflow-cost.py` reads the per-agent transcripts a Claude Code workflow leaves behind, sums tokens by model, prices them, and prints cost per change against a baseline.

```
python3 workflow-cost.py <run-id> --prs 4 --baseline-per-pr 66
```

Point it at a run directory or a bare run id. Prices are list prices in the `PRICES` table at the top; edit them to whatever you actually pay. This is the only file here you can use unmodified.

After each batch, compare two things: cost per change, and what the critic caught versus what got past it. When the critic misses something, the fix is a new scripted check, not a more expensive critic. When a critic finds nothing on a tier for three batches, drop it from that tier.

## 8. Attack your own method

My first version of this escalated to the expensive model only when the automated checks failed. I tested that against the twelve real defects the expensive reviewers had caught that week, and every one of them was, by definition, something the checks had not noticed. The rule would have called in the strong model zero times for the things that mattered.

That is why the critic runs on every change now, and why the mutation check exists. Run your own version of that test before you trust any of this.

## What this does not fix

A backlog with no definition of finished refills faster than it drains. Automation widens the drain. It does not close the tap. Decide what done means first.

## Licence

MIT. Take it, change it, ship it.
