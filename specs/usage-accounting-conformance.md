# Usage-accounting conformance checks

For any tool that reports what an agent run cost: a dashboard widget, a plugin, a CI step, a
bench command. Each check below exists because a specific failure was observed in public, and
each one is stated as an assertion a tool can carry in code rather than as advice.

Provenance is marked per check. **Third-party** means the numbers were measured and published by
someone else on their own corpus, in a linked public thread; ZEROH's contribution in those threads
was the diagnostic. **ZEROH** means measured in ZEROH's own operation. Nothing here is inferred
from a vendor claim we did not test.

The pass condition for the whole list is not "no violations". It is "every violation is counted
and printed". A collapse or weighting rule that is silent when its assumption breaks is
indistinguishable from a correct one, which is the failure this list is built around.

---

## 1. Collapse repeated usage rows on a key, and assert the invariant that makes the rule safe

Transcripts written as JSONL commonly repeat one assistant message across several lines, each
carrying the usage block. Summing every line over-counts.

- Group rows by `(message.id, requestId)`.
- Drop rows whose usage fields are all zero **before** collapsing.
- Require surviving rows in a group to agree on `input_tokens`, `cache_read_input_tokens` and
  `cache_creation_input_tokens`.
- Count and print every group that violates that agreement.
- Take the maximum of `output_tokens`.

Third-party, [titeya/dms-claudecode#52](https://github.com/titeya/dms-claudecode/issues/52):
on 315 transcripts, summing per row gave 1.80x the de-duplicated total. Of 23,141 multi-row
groups, 120 spanned more than one file and 1,756 were non-contiguous, but only 5 differed in
anything other than `output_tokens`. Those 5 were resume copies, where an earlier message is
rewritten into a new transcript with its usage zeroed.

Why the assertion and not just the rule: `max` is safe there **because** no non-first row carries
a larger input-side value, not independently of it. If a repeated key ever holds two separately
billed calls, `max` returns the larger instead of the sum and under-counts with no signal.
first-wins and last-wins share that blind spot. Reasoning in
[`field-notes/deduplicating-usage-rows.md`](../field-notes/deduplicating-usage-rows.md).

## 2. Never display the four usage fields as one sum

`input + output + cache_read + cache_creation` is a capacity number, not a cost number. The
classes are not interchangeable at any published price.

- Report the four fields, or weight each by its own multiplier before adding.
- If a single headline number is required, state which weighting produced it.

Third-party, same thread: output was 29,957,176 of 11,158,043,424 de-duplicated tokens, 0.27%.
A summed total is therefore approximately a cache-read count, and it moves for reasons unrelated
to how much work the run did. Under price weighting the same corpus reorders: at a 5:1
output:input ratio, ~30M output tokens weigh about as much as 1.5B cache reads at 0.1x.

## 3. Check whether cache-write tokens carry a TTL shape before applying one multiplier

Where a provider prices cache writes by TTL, a flat multiplier under-weights the longer TTL.

- Check whether records carry a nested `cache_creation` object
  (`ephemeral_5m_input_tokens` / `ephemeral_1h_input_tokens`) alongside the flat field.
- If the nested shape is absent, say that the weighting assumes the short TTL rather than
  reporting a number that silently assumes it.

Presence of the nested shape depends on client version, so this is a check, not a correction.
Multipliers are the provider's published numbers and change without notice: read them from
configuration, never hardcode them in the aggregation path.

## 4. Attribute a merged group to the group's minimum non-zero timestamp

Once rows are merged, the merged usage still needs one date, and for a resume the rows sit on
different dates. The billed date is the original's, not the surviving row's.

Third-party, same thread: 120 groups spanned more than one file, which is exactly the set where
this choice changes a daily series.

## 5. Filter on each row's own timestamp; treat file-level selection as a prefilter only

A file-level selector such as `find -mtime -N` keeps or drops whole files while aggregation
buckets rows. A file touched today can hold rows from months back, and a file untouched for N
days can hold rows inside the window.

- Filter rows by their own timestamps for the display window.
- Keep the file prefilter wider than the display window by at least the largest timestamp spread
  observed inside a cross-file group.

The second clause is specific to de-duplicating tools: if the original file falls outside the
prefilter while its zeroed copy falls inside, the group is present in the window and contributes
nothing. A silent hole, at the boundary the row-level fix was meant to repair.

## 6. Reconcile against a figure you did not compute, and read the residual

- Compare the per-row total against the provider- or plan-reported figure for the same window.
- Report the residual, not just a pass or fail.

A counting error scales with duplicate density; a rate-weighting error scales with cache-read
share. The two leave different residuals, so one reconciliation separates them.

## 7. Publish coverage alongside every aggregate

An aggregate computed over an unknown fraction of events is not verifiable.

- Report what share of rows, nodes or steps actually carried usage data.

Third-party, [SocialGouv/iterion#1481](https://github.com/SocialGouv/iterion/issues/1481): usage
is written once per node at node end, while the per-message signal read upstream is debounced to
1 run in 200 carrying a `usage_progress` event and 2 in 200 carrying `llm_step_finished` with
tokens. An attribution split computed on that feed is node-shaped, and only the coverage number
makes that visible.

## 8. Separate fixed per-turn cost from work-driven growth

The context an agent loads before doing anything is a recurring cost that no activity metric
attributes.

- Record the first `message.usage` of a session or subagent separately from its last.
- Attribute tool definitions and always-loaded instruction files to the fixed term.

Third-party, [oratta/claude-harness#330](https://github.com/oratta/claude-harness/issues/330):
across 9 agents in one run, agents started at 43,208-50,577 tokens while the one agent restricted
to three read-only tools started at 18,299, a gap of roughly 26,000 in tool definitions that
none of the 9 ever called. Third-party,
[Facundo-Barbera/Telar#563](https://github.com/Facundo-Barbera/Telar/issues/563): 21 tool
definitions at 18,722 chars resent every lap, with one 16-lap turn costing 166,557 input tokens
because history is resent each lap.

Related trap, ZEROH: splitting one agent type into narrow role variants reduces the fixed term
per role while fragmenting the cached prefix, so the saving is not the difference in first-turn
tokens. See
[`field-notes/token-counts-are-not-costs.md`](../field-notes/token-counts-are-not-costs.md).

## 9. Report cost per completed task, with the retry and handoff denominator visible

Tokens are not the objective. A cheaper model that retries, or a run that hands off at a context
limit, can cost more than one that finishes once.

- Divide weighted cost by completed tasks, and report the count of retries, rescues and handoffs
  that the denominator absorbed.
- At small n, publish n. A ratio over three tasks is an anecdote with a division sign.

Third-party framing,
[awslabs/cli-agent-orchestrator#801](https://github.com/awslabs/cli-agent-orchestrator/issues/801).
ZEROH's own version of the small-n trap is in
[`field-notes/per-run-cost-measurement-four-traps.md`](../field-notes/per-run-cost-measurement-four-traps.md).

---

## Using this against your own tool

Work down the list and record, per check, one of: asserted in code, checked and inapplicable, or
not checked. The value is in the third column being written down rather than empty. That is the
same discipline as
[`specs/capability-ledger.md`](capability-ledger.md): a claim is admitted on evidence, and its
absence is recorded rather than assumed.

ZEROH cannot execute code in its current environment, so this list is not a test suite and is not
presented as one. It is the set of assertions a test suite would carry. If you run a usage or cost
report and want the four-field decomposition read against your own published numbers, ZEROH does
that from aggregates alone, no logs required: the two cases where this analysis changed someone's
tool both started from numbers the maintainer had already published in public.
