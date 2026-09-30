---
name: review-pr
description: >
  Merge-readiness triage for a queue of open pull requests. Works through them one
  at a time, oldest first, and for each one asks the only question that matters
  before it lands: is this safe to merge *now*? The verdict weighs CI and test
  evidence, conflicts against the base branch as it exists after earlier merges,
  the repository's own documented rules, whether new behavior is actually tested,
  blast radius outside the diff, whether the PR does what its description claims,
  and whether it advances the product's stated design. Merges only on an explicit
  yes, one PR per yes. Use when you say "/review-pr", "review the PRs", "go through
  the PR queue", "can I merge these", or "is this PR safe to merge".
argument-hint: "[optional PR number — omit to walk the whole queue oldest-first]"
license: MIT
---

# /review-pr — Merge-Readiness Queue Triage

A pull request is not finished when its diff looks right. It is finished when it is
safe to land on the branch *as that branch stands right now* — with the earlier PRs in
the queue already merged, the CI green on the current head, and nothing outside the
diff quietly broken by it. This skill triages a queue on exactly that question, one PR
at a time, and merges only when you say so.

> **The verdict is the deliverable; the merge is a separate, authorized act.** Never
> fold the two together. A "safe" finding without your nod is a recommendation, and a
> nod for one PR is never a nod for the rest of the queue.

## Configure once for your setup

```yaml
REMOTE:            origin        # which remote identifies the repo; ask if there is more than one
QUEUE_ORDER:       oldest_first  # createdAt ascending — a PR that has waited longest gets decided first
MERGE_METHOD:      auto          # auto | squash | merge | rebase. "auto" copies what the repo already does
DELETE_BRANCH:     true          # delete the head branch after a successful merge
COMMENT_ON_BLOCKED: ask          # ask | never — whether to leave a PR comment on a NOT-READY verdict
DESIGN_INTENT:     null          # optional: path to a design doc / roadmap, if the repo keeps one
```

`MERGE_METHOD: auto` resolves to whatever the repository's recent merged PRs used, and
falls back to `squash` when there is no history to learn from. `DESIGN_INTENT` is
optional — without it, the design-fit signal below is evaluated against whatever intent
sources the repository already carries (a linked issue, a README vision statement, the
shape of the surrounding code), and reports itself as unevaluable when there are none.

## Scope

- One repository: the one the current checkout's `REMOTE` points at. If you are not in
  a checkout, ask which repository before doing anything.
- One PR at a time. Never form a verdict on PR *n+1* before PR *n* has been decided —
  later PRs are assessed against a base branch that earlier merges have already moved.
- Never leave the queue to go look at unrelated issues, or merge anything the queue
  does not contain.

## The loop

```pseudocode
review_pr_queue(repo):
    queue = list_open_prs(repo) sorted by QUEUE_ORDER
    if queue is empty:
        say "no open PRs on <repo>"; return

    show the queue up front:  # number, title, author, age, draft?, size
        "walking 4 PRs, oldest first"

    for pr in queue:
        refresh(queue)                  # a previous merge may have changed what exists
        if pr no longer open: note it; continue

        evidence = gather_live_evidence(pr)
        verdict = assess(pr, evidence)

        report(verdict)                 # chat is the primary channel — see "Reporting"
        offer_resolution(pr, verdict)   # required for anything short of MERGE-READY

        if verdict == MERGE_READY:
            if you_say_yes(pr.number):  # one PR, one approval — never batch
                merge(pr, evidence)     # re-verifies the head SHA first
            else:
                leave_it_open(pr)
```

## Gathering evidence

```bash
# which repo, and what it defaults to
gh repo view --json nameWithOwner,defaultBranchRef

# the queue, oldest first
gh pr list --state open --limit 100 \
  --json number,title,author,createdAt,isDraft,headRefName,baseRefName,url \
  --jq 'sort_by(.createdAt)'

# one PR, everything at once
gh pr view <N> --json number,title,body,author,createdAt,isDraft,mergeable,mergeStateStatus,\
reviewDecision,headRefOid,headRefName,baseRefName,additions,deletions,changedFiles,\
commits,files,statusCheckRollup,reviews,url

gh pr diff <N>
```

`mergeable` and `mergeStateStatus` read as `UNKNOWN` for a short window after a push —
that is a stale answer, not a clean one. Re-read them before trusting them, and never
report a check you have not actually read the status of.

**Do not mutate the checkout to do this.** Read the local base branch for repository
context; get everything PR-side from `gh`. If you genuinely need the full branch tree,
use a throwaway worktree (`git worktree add`) outside the repository and remove it after
— the working tree may hold uncommitted work that is none of the queue's business. Never
`git checkout` a PR branch over someone's dirty work.

## What the verdict weighs

Read the repository's own instruction file first (`AGENTS.md`, `CLAUDE.md`, or whatever
it uses) — its rules are inputs to the verdict, not background colour. Then weigh all
seven signals. A signal you could not evaluate is reported as unevaluated; it is never
quietly counted as a pass.

| # | Signal | The question | Hard blocker when |
|---|---|---|---|
| 1 | CI and test evidence | Are the required checks green **on the current head SHA**? | A required check is red, pending, missing, or skipped |
| 2 | Conflicts with what is landing | Does it still apply cleanly to the base branch *after the merges already made*? | Conflict, or `mergeStateStatus` is `DIRTY` / `BLOCKED` |
| 3 | The repository's own rules | Does the change respect the conventions and constraints the repo documents? | It contradicts a stated rule, or skips a step the repo requires |
| 4 | Test adequacy | Is the new behavior actually exercised by a test? | New or changed behavior ships with no test covering it, where the repo has a testing convention |
| 5 | Blast radius beyond the diff | What breaks outside the files it touches? | A caller, config, schema, migration, or documented interface is broken and the diff never mentions it |
| 6 | Description versus reality | Do the commits, the description, and the code tell the same story? | The change does materially more (or less) than it claims |
| 7 | Design and vision fit | Does this move the product where it is supposed to go? | It is **contradicted** by a cited intent source — see the honesty rule below |

Signals 1, 2, 5 and 6 are mechanically checkable; do those first and get them right.
Signals 3, 4 and 7 need judgment, and judgment is where a review quietly goes wrong.

### The honesty rule on signal 7

"Does this match the design and vision" is the one signal with no ground truth attached,
and it is the easiest one to fake. Judge it **only against a source of intent you can
name**: the `DESIGN_INTENT` doc, a linked issue or roadmap item, a README vision
statement, an architecture note, the established shape of the surrounding code.

> Cite the source in the verdict. If there is no source to cite, the correct finding is
> `NEEDS DECISION: no design intent available to check against` — not a confident
> opinion wearing a citation-shaped hat. A wrong vision call is more expensive than an
> absent one, because it survives review by sounding like a judgement.

## The verdict

| Verdict | Means | Requires |
|---|---|---|
| `MERGE-READY` | Safe to land now, as of the head SHA you checked | All seven signals cleared, each one actually read rather than assumed |
| `NOT READY` | At least one named blocker | The blocker named, evidence cited, at least one resolution prepared |
| `NEEDS DECISION` | No hard blocker, but a call that belongs to you | The decision stated plainly, with what each option costs |

`NEEDS DECISION` is a real verdict and using it is honest. Manufacturing a blocker to
look thorough is not, and neither is dressing a taste preference up as `NOT READY` —
that is how reviewers learn to ignore the ones that matter. Equally: a real blocker
softened to `NEEDS DECISION` is a failure, not diplomacy.

> **[rule] A clear bill of health has to be earned.** `MERGE-READY` on a large PR means
> every signal above was actually checked and found clean. If a signal was skipped for
> want of time or access, that is a `NEEDS DECISION` naming the gap, not a pass.

## Reporting

Chat is where the verdict lives. A PR comment is a courtesy to the author, secondary, and
never the place where a finding is *first* discovered.

```markdown
## PR #<N> — <title>  →  <MERGE-READY | NOT READY | NEEDS DECISION>

**Head:** `<short-sha>`  **Base:** `<base>` (+3 merged while walking the queue)  **Author:** @<handle>

| Signal | Result |
|---|---|
| CI and test evidence | 9/9 green on `a1b2c3d` |
| Conflicts with what is landing | clean — reapplied on top of #41, #42 |
| Repository's own rules | consistent with `AGENTS.md` §testing ladder |
| Test adequacy | new parser paths covered in `tests/test_parse.py` |
| Blast radius | touches `load_config()` — 4 call sites, all updated |
| Description vs reality | matches |
| Design and vision fit | matches issue #38 (cited) |

**Why:** <one or two sentences — the argument, not a restatement of the table>

**To merge:** `gh pr merge <N> --squash --delete-branch`
```

Report one PR, get the answer, move on. Do not dump the whole queue's verdicts at once
and ask for blanket approval — the verdicts are only valid against the base branch as it
stood when each was made, and the second and third are guesses once the first lands.

## Merging

Merging is the consequential act, and it happens on your nod and nothing else.

```pseudocode
merge(pr, evidence):
    current = fetch_live_head_oid(pr)
    if current != evidence.head_sha_that_you_reviewed:
        report "head moved since review — re-assessing"     # someone pushed after you looked
        return reassess(pr)

    method = resolve_merge_method(pr)                       # MERGE_METHOD, auto -> repo's own habit
    run: gh pr merge <N> --<method> [--delete-branch]
    confirm: PR state is MERGED, not merely "merge queued"
    note what landed, so the next PR is assessed against the new base
```

- **One approval, one PR.** "Yes, merge them all" is not an approval for PRs you have
  not assessed yet.
- **Never `--admin`, never `--auto`, never force.** `--admin` bypasses the protections
  you are here to respect; `--auto` merges on CI you never read.
- **The base moves.** After each merge, the remaining PRs' conflict status and CI are
  potentially stale. Re-read them; do not carry a verdict across a merge.
- If the branch is protected and the repo requires a merge queue, `gh pr merge` enqueues
  rather than merges. Say which happened — "enqueued" reported as "merged" is the worst
  possible lie this skill could tell.

## When a PR is NOT READY

Report the verdict, then **prepare at least one concrete way out**. "Needs changes" with
no route forward is a dead end for the author. Name the remedy, be ready to execute it,
and let them pick:

| Blocker | Candidate remedies |
|---|---|
| Conflicts with the base | rebase onto the updated base; merge the base into the head; re-land the change as a fresh PR |
| Red CI from staleness | rebase and let CI re-run; fix the underlying failure |
| Red CI from a real defect | fix forward on the branch; revert the offending commit and re-land in pieces |
| Untested new behavior | add the test; or land behind a flag with the test following |
| Blast radius outside the diff | update the affected callers here; or narrow the PR to the file that is actually ready |
| Contradicts a documented repo rule | conform to the rule; or change the rule deliberately, in its own PR, with the reasoning |
| Off-vision or out of scope | close it and open a focused PR for the part that is on-vision; or re-scope it |
| Superseded by an earlier merge in this queue | close it |

Offer a PR comment when the author would otherwise be left guessing — and when you do,
**draft it and show it before posting**. A comment is a public statement attributed to
you: accurate, specific, and free of internal shorthand. Never a comment you have not
read yourself.

## Prohibitions

```pseudocode
NEVER merge without an explicit yes for that specific PR.
NEVER carry a verdict across a merge — re-read conflict status and CI.
NEVER report a check's status you have not read.
NEVER claim a design finding without citing a source for the intent.
NEVER edit the PR's code to make it pass review — that is a different, authorized task.
NEVER force-push, --admin, or --auto.
NEVER delete a branch that is not merged.
NEVER leave a PR comment the maintainer has not seen first.
NEVER touch uncommitted work in the local checkout to service the queue.
```

## What this isn't

- **Not a code review.** It does not adjudicate style, naming, or architecture taste, and
  it does not go line by line. Signals 1–2 and 5–6 are a merge gate, not a critique. For a
  deep security or correctness audit of a branch, use a dedicated audit skill if your setup
  has one; this skill assumes one has already run, or does not exist.
- **Not a triage of open issues** — only pull requests.
- **Not unattended.** It stops for a verdict and stops again for a nod. If you want a
  queue merged without a human in the loop, that is a different tool and a different risk
  decision.

## Adopting this

Copy `SKILL.md` into your agent's skills directory (Claude Code: `~/.claude/skills/review-
pr/`; project-scoped: `.claude/skills/review-pr/`), then fill in the **Configure once for
your setup** block. Requires the GitHub CLI (`gh`) on `PATH`, authenticated, with read
access to the repository; merging additionally needs write access and the
`repo`/`pull_request` scope.

For agents that read a plain instructions file instead of a skills directory (Warp,
Copilot via `AGENTS.md`, etc.), paste the **The loop**, **What the verdict weighs**, and
**Prohibitions** sections directly into that file — none of the mechanism here is
specific to one agent.
