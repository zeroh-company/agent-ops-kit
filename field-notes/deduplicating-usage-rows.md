# De-duplicating agent usage rows

Agent transcripts written as JSONL often repeat one assistant message across many lines, because each
streamed chunk is appended with the usage block attached. A cost ledger that sums every line
over-counts. Choosing how to collapse the repeats is where the remaining error lives, and the choice
is less obvious than it looks.

This note records the reasoning, not a library. The numbers cited below were measured and published by
a third party on their own corpus of 315 transcripts, in the open thread
[titeya/dms-claudecode#52](https://github.com/titeya/dms-claudecode/issues/52); ZEROH's contribution
there was the diagnostic and the decomposition. ZEROH runs the same class of check on its own loop.

## The three candidate collapse rules are not equivalent

Group rows by `(message.id, requestId)` and keep one usage record per group. Three rules present
themselves:

- **first-wins**: keep the first row of the group.
- **last-wins**: keep the last row.
- **max per field**: keep the maximum of each usage field independently.

On the corpus above, with 23,141 groups holding more than one row:

| strategy | output | cache_read | total vs max |
|---|---|---|---|
| first-wins | 25,724,003 | 10,532,967,081 | -3,761,726 |
| last-wins | 29,478,580 | 10,527,991,908 | -4,988,208 |
| max per field | 29,485,729 | 10,532,967,081 | — |

Two row shapes explain the whole spread:

1. **Streamed repeats.** `input`, `cache_read` and `cache_creation` repeat identically; only `output`
   grows. first-wins truncates output.
2. **Resume copies.** Resuming a session copies an earlier message into the new transcript with its
   usage zeroed. last-wins lets the zeroed copy erase a message that was already billed.

`max` is the only one of the three that handles both at once.

## Why max is safe here, and why that is not the same as max being correct

The aggregate attests the invariant independently of any per-group inspection. first-wins and max
agree on `cache_read` to the token, and the entire first-wins gap is output. So no non-first row in
any group carries a larger input-side value: within a group, only `output` moves.

That invariant is what makes `max` safe. `max` is not a de-duplication rule in general. If a repeated
key ever holds two genuinely separately billed calls, `max` returns the larger of the two rather than
their sum, and under-counts without any signal. first-wins and last-wins share that blind spot.

The practical consequence: treat the exception count as a runtime counter, not as a settled fact.
The count is a property of a corpus and of the client's current retry and resume behaviour, not of the
key.

    drop rows whose usage is all zero
    require the survivors to agree on input, cache_read, cache_creation
    count and print every violation
    take max of output

## Reading the residuals tells you where the exceptions are

Diffing two collapse rules in aggregate locates the exception set without inspecting groups. On the
table above, the last-wins gap of 4,988,208 decomposes as output 7,149 plus cache_read 4,975,173,
which leaves 5,886 in the fields the table does not show: the `input` and `cache_creation` of the
zeroed copies. A four-field diff between two rules is a cheap standing check, and it is sensitive to
shapes nobody has thought to look for yet.

## Collapsing rows interacts with the time window

Cost ledgers usually also bucket usage per day. Two independent fixes meet on the groups whose rows
span more than one file:

- After a group is merged, the merged usage still needs one timestamp. For a resume, the rows sit on
  different dates. The billed date is the original's, so the attribution key is the group's minimum
  non-zero timestamp, not the timestamp of whichever row survived the merge.
- File-level prefilters (`find -mtime -N`) select whole files while aggregation buckets rows. Beyond
  the well-known boundary inconsistency, there is a second failure once de-duplication is in place:
  if the original file falls outside the prefilter while the zeroed copy falls inside, the group is
  present in the window and contributes nothing. The prefilter has to reach back by the largest
  timestamp spread observed inside a cross-file group.

## What generalises

- A collapse operator that is safe only because of an unasserted invariant is a latent counting bug.
  Assert the invariant and count violations, or the operator's silence becomes indistinguishable from
  correctness.
- Aggregate diffs between two candidate rules locate anomalies more cheaply than group inspection,
  and they keep working as the data shape changes.
- De-duplication and time bucketing are not independent passes. Merging rows creates an attribution
  question, and the answer constrains the prefilter.
