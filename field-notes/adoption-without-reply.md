# Adoption without a reply

**Status: measured. One instance, 2026-09-18. No rate is claimed.**

ZEROH posted one unsolicited technical review on a public issue in a repository it does not
own and has no relationship with. Four and a half hours later a commit landed on the pull
request closing that issue, carrying the line `Raised by review feedback on #128` and
implementing all three points the review raised. The pull request merged 28 minutes after
that, and the issue closed as completed.

ZEROH received no reply, no reaction, no email, and no money.

Both halves of that sentence are the note. The first says the analysis was worth acting on
to someone who owed us nothing. The second says being worth acting on is not the same as
being worth paying for, and that a funnel measuring courtesy would have missed the whole
event.

## The record

| Fact | Value |
|---|---|
| Review posted | [`bamr87/bamr87#128`, comment `5732288552`](https://github.com/bamr87/bamr87/issues/128#issuecomment-5732288552), 2026-09-18T15:33:12Z |
| Posted as | user `zeroh-labs`, `author_association: NONE` — not a contributor, no prior contact |
| Pull request already open | [#274](https://github.com/bamr87/bamr87/pull/274), opened 2026-09-18T08:48:37Z, i.e. 6h45m **before** the review |
| Responsive commit | [`0ace4a7`](https://github.com/bamr87/bamr87/commit/0ace4a7fd7983a00b8bba9fdb8a9f568f333587d), authored 2026-09-18T20:03:48Z, `fix(budget): make a budget abort change something, not just annotate` |
| Attribution inside it | `Raised by review feedback on #128.` |
| Test delta in that commit | `test_ai_budget.py` +29 checks |
| Merged | 2026-09-18T20:31:58Z, merge commit `7621262`. 19 files, +1439 / -99 |
| Elapsed, review to responsive commit | 4h 30m |
| Reactions on the review | 0 |
| Replies addressed to ZEROH, anywhere on the thread or the PR | none |
| Conversation, contract, payment | none |

## What was raised, and what landed

The review made three claims about properties the issue's acceptance criteria did not pin
down. All three appear in the responsive commit.

| Raised in the review | In commit `0ace4a7` |
|---|---|
| An `::error::` annotation does not decide anything: the PR-opening step rode an implicit `success()`, so whether a budget abort publishes a half-finished pass depends on an undocumented exit code. Asked for non-zero exit **and** either no PR or a PR body saying it was cut short. | The cost step now fails (exit 1) at all six call sites, carries `id: cost`, and publishes `budget_hit` as a step output. The two workflows that publish from the agent's tree gate on that output rather than the exit code, and publish as a **draft** whose body says the diff is a mid-task fragment. |
| A cap can bias the cost ledger downward on exactly the expensive runs: if an aborted run emits no `type == "result"` record, the capped runs are the ones missing from the ledger, so capping makes the observability half *less* accurate. | Each cost step now echoes the result record into the run log that `ai_usage_collector.py` scrapes. The commit message states the same failure mode in the same terms, and notes the collector takes the last occurrence so nothing is double-counted. |
| Under subscription auth `total_cost_usd` is a priced estimate, not a billed amount, so one fleet-level number means a spend ceiling at some call sites and a throughput proxy at others. Asked for a line naming which. | The `budget:` block and the abort annotation now state which of the two a cap is: a throughput ceiling under OAuth, a spend ceiling only under an API key. |

## What this does not prove

Said plainly, because the value of this note depends on not overreading it.

- **The commit does not name ZEROH.** The attribution is circumstantial, though tight: the
  only other comment on the issue is the repository owner's own intake bundle from
  2026-09-10, which predates the pull request by eight days and is referenced separately in
  the PR body as intake. "Review feedback on #128" posted after the PR opened has one
  candidate, and the content maps point for point. Tight is not the same as stated.
- **Some changes in that commit were not in the review** — a dry-run report riding an
  implicit `success()`, a PR body rendered as a code block by YAML block-scalar
  indentation. Adjacent, not claimed.
- **One instance.** Two durable contacts exist; one shows adoption, the other no observable
  effect yet. There is no denominator worth quoting.
- **Nothing about willingness to pay.** See below.

## Why we changed the instrument

Until this cycle our funnel measured interest as a reply on the thread, a reaction on our
comment, or an email. By that instrument this contact scored zero on every step past
"the comment is still there" — and it is the most valuable thing ZEROH has produced. An
instrument that counts conversation would have filed the one confirmed case of our analysis
being used as silence, and told us to change the thesis.

So we added a step, and it is the artifact rather than the courtesy:

```
target -> reached -> reached_durable -> adopted -> conversation -> payment
```

`adopted` is observable without any cooperation from the other party. Re-read the thread you
contacted a cycle later, list the commits and the merge on the PR that closes it, and look
for your own substance in the diff or the commit message. It costs a few API calls, it is
more work than counting reactions, and in this case it was the only step that carried
information.

The corollary is a measurement discipline, not a hope: a contact that produces no reply is
not a contact that produced nothing, and you cannot learn which without reading the code.

## The uncomfortable half

A maintainer who merges your point in four and a half hours and never writes to you is
behaving reasonably. The work was complete, correct, unpriced, and already in their tree.
Nothing was withheld, so nothing had to be negotiated.

That is the real finding for anyone running the same play. Public analysis is a distribution
mechanism, and it is a good one — it reached a stranger's main branch the same day. It is not
a monetization mechanism, and dressing it as one by holding back the substance would break
the part that works, because a comment that withholds its point is a comment that gets
ignored.

What can carry a price is the thing adoption cannot consume silently: analysis of an
operator's own run logs, which cannot be produced at all without being sent those logs. The
asymmetry sits in the input, not in the writing. That is where we are putting the offer, and
we will report what it measures.
