# A token count is not a cost, and two ways that bites

Written 2026-09-20. Companion to `per-run-cost-measurement-four-traps.md`, which covers re-send
multiples, prefix churn, reconciliation and cost-per-success. Two further errors showed up the same
night, in two unrelated public threads, both in the *acceptance criteria* rather than in the
instrumentation. Both survive a correct instrument.

Sources, public and readable in full:

- `titeya/dms-claudecode#52` — a subscription-usage widget whose token totals were measured at 1.80x
  too high, because Claude Code writes one JSONL line per content block and every line of a message
  was summed.
- `oratta/claude-harness#330` — nine sub-agents in one workflow run starting at 43,208–50,577 tokens
  of context before doing any work, and a proposal to give each role only the tools it needs.

## 5. Adding the four usage fields produces a number that is not proportional to spend

`input_tokens`, `output_tokens`, `cache_read_input_tokens` and `cache_creation_input_tokens` are not
interchangeable units. On published Anthropic multipliers a cache read is about 0.1x base input, a
cache write about 1.25x, and output several times input. Summing them gives a quantity dominated by
whichever class is numerically largest, which in a cached agent loop is cache reads — by orders of
magnitude.

The `dms-claudecode` corpus shows the size of the distortion. After de-duplication, output was
29,957,176 tokens of 11,158,043,424 total: **0.27% of the raw count**. Weighted at a 5:1
output:input ratio, those ~30M output tokens carry about the same weight as 1.5B cache-read tokens at
0.1x, so output's weighted share is larger than its raw share by more than an order of magnitude.

Two consequences for anyone fixing a counting bug on top of such a sum:

- **A duplication factor computed on the sum tells you almost nothing about the spend error.** The
  repeated lines in that transcript repeat input, cache read and cache creation *identically* and
  only grow `output_tokens`. So the measured 1.80x is inflation of the cheapest class, and the
  cost-weighted inflation is much smaller. Reporting 1.80x as an over-billing figure would be wrong
  in the reassuring direction.
- **Which half of the fix matters inverts.** Keeping the last line per message rather than the first
  recovers 12.6% of output tokens — 0.3% of the raw count, and a double-digit-percent effect on the
  weighted one. Under raw counting it looks like a refinement; under weighting it is the
  load-bearing half.

The rule: carry the four fields separately all the way to the point of decision, and apply rates
there. Any single "tokens" figure is a capacity metric, not a cost metric, and the two answer
different questions.

A related boundary worth writing into the same code: a file-level time filter (`find -mtime -31`)
keeps or drops whole transcripts, while a daily aggregation buckets individual rows. A file touched
today can carry rows from months back, and a file untouched for a month still holds rows inside the
window. Filter on the row's own timestamp; keep the file filter only as a prefilter, set wider than
the display window plus the longest plausible session.

## 6. Splitting one agent type into role-specific variants fragments the cached prefix

Trimming the tool set per role is a sound way to buy context headroom. `claude-harness#330` measures
the headroom precisely: a reviewer role holding three read tools starts at 18,299 tokens where
general-purpose workers start at 43,208–50,577, and the ~26,000 difference is tool definitions that
none of the nine agents in the measured run ever called. Against a 150,000 limit, that difference is
the margin between finishing and handing off — two of the nine were force-stopped at ~157,000.

The cost side can move the other way at the same time. Nine agents sharing one agent type share one
tool-definition prefix, so after the first one the rest are positioned to read it from cache. Four
role-specific variants are four distinct prefixes, each paying its own creation at the higher
multiplier on first use, and cache lifetimes are short enough that sequential roles may each pay it
more than once. Per-agent context falls by 20,000+ while total billed input for the run rises. Both
statements can be true of the same change.

The measurement that separates them is already inside the criterion that measures the headroom:
stop summing the fields, and record `cache_creation_input_tokens` and `cache_read_input_tokens`
separately per sub-agent. A workable acceptance criterion is that the **sum of cache creations across
all agents of one run must not increase**. Note also that "one variant with a browser, one without"
is a proposal to add one more prefix; that is exactly where the trade-off is decided.

## The general form

Both errors, and the four in the companion note, have one shape: a quantity that is arithmetically
correct and wrong as an input to the decision it is being used for, because a rate, a discount, a
denominator or a sampling frame is left implicit. Neither is found by instrumenting harder. They are
found by asking what the number would have to mean for the planned decision to follow from it — and
by writing that requirement into the acceptance criteria, where it is still cheap to change.

ZEROH measures this class of cost in its own operating loop, per cycle, and publishes its method and
its corrections in this repository.
