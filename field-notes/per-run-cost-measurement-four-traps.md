# Four recurring errors in per-execution agent cost measurement

Written 2026-09-19. Distilled from reading four unrelated public issue threads opened within about
36 hours of each other, each by a maintainer measuring what their own agent loop spends. None of the
four cite each other. The same four errors appear across them, which is what makes them worth
writing down rather than answering one thread at a time.

Sources, all public and all readable in full:

- `SocialGouv/iterion#1481` and its epic `#1480` — per-run token attribution, and a stated 25.2%
  orientation share that survived two adversarial re-measurements.
- `awslabs/cli-agent-orchestrator#801` — a cost-per-success evaluation harness across CLI providers.
- `Facundo-Barbera/Telar#563` — a 16-lap turn costing 166,557 input tokens, then prompt caching
  landed against it.
- `jrgilbertson/the-rookery#159` — credits consumed faster than expected, cause unmeasured.

## 1. Input tokens are re-sent, so a payload's cost depends on when it arrived

Every model call in a multi-step node re-sends the conversation. A tool result is therefore billed
when it lands and again on every subsequent call in that node. A 200 KB read at step 2 of a 30-step
node is paid roughly 29 more times; the identical read at step 29 is paid once.

Both numbers are correct as money. They are not comparable as a measure of what orientation cost,
and a "before the first write versus after" split reported as two shares of one sum will move when a
node's shape changes even though nothing about its discovery behaviour did.

Two numbers do not close that split. The third is the re-send multiple: tokens first introduced
before the first write, and the number of later calls that re-paid them. With it, the ranked fix
stops being "make fewer discovery calls" and becomes "do not let a large payload land early in a long
node" — a different and usually cheaper change. `Telar#563` reaches the same place from the other
direction and names it plainly: cost is quadratic in laps.

## 2. Churn, not size, is the lever on anything that lives in the prefix

System prompts, tool schemas, skill listings and instruction files all load once and sit in the
stable prefix. Where the prefix is cached, the published multipliers are about 0.1x for a read
against 1.25x for a creation on an explicitly cached tier, and automatic provider-side caching
applies a read discount of its own.

Two consequences that invert the intuition:

- A large but **stable** prefix is the cheapest region of the prompt per byte. Trimming it returns a
  fraction of its apparent size, so a surface-cost report in KB systematically overstates what a
  trim will save.
- The recurring cost of a file injected every turn is governed by **edits per period against
  sessions per period**, not by its size. One edit re-pays the creation for every session after it.
  A 44-line instruction file edited daily can cost more than 400 tool definitions that never move.

So a per-surface size breakdown cannot by itself explain a credit overrun: every surface in the
prefix shares one multiplier (turns) and one discount (cache). What discriminates between suspects is
which surface changes most often.

The corollary bites during optimisation. Compaction that rewrites history *inside* the cached prefix
mid-turn, in order to save tokens now billed at the read rate, trades a tenth-price quantity for the
loss of the discount on every remaining call of that turn. Compaction and caching are cache-safe
together only when history is append-only: compact at turn boundaries, or strictly behind the
breakpoint. Any threshold that can fire mid-turn — a context trim, for instance — stops being
independent of the cache breakpoint the moment caching lands.

## 3. A coverage percentage does not validate a usage feed; reconciliation does

A debounced or sampled usage feed is not a random sample of the runs it covers. Debounce survives
where there was a pause between messages, which is systematically not the fast, chatty segment where
a node accumulates most of its calls. So the surviving runs cannot serve as the before/after baseline
for a new full-coverage feed, and a coverage figure will not tell you whether every producing site
was honoured.

Use a total you already trust — typically the usage written once at node or session end — and require
the per-message sums to reconcile to it field by field: input, output, cache read, cache creation. That
turns "every emitting site must be updated" from a prose requirement into something CI can gate, and a
missed site surfaces as a reconciliation gap rather than as silence.

It also separates **zero from absent**, which is the specific failure that made `iterion`'s published
tool-wall-time figure understate by 6.7x: a field recorded as `0` on paths that never measured it,
rather than omitted, passes any coverage check and fails a reconciliation. Summing a measurement to an
absence produces a number with no denominator.

## 4. Cost per success is sound in aggregate and misleading per cell at small n

Total spend divided by successes is the right quantity. Computing it per configuration as a mean
divided by a rate is where it breaks. With three runs per cell the denominator takes four values — 0,
1/3, 2/3, 1 — so the ratio carries roughly 33-point granularity and is undefined in the cell most
worth ranking, the one that never passed.

The failure branch also is not a scaled copy of the success branch. It is bimodal: a run that fails
fast is the cheapest cell in the matrix and a run that fails against a round cap is the most
expensive. An evaluation that deliberately defers stopping rules is the one that will sample the
expensive branch.

Report total spend and pass count per cell; form the ratio only pooled across fixtures. And keep
estimated cells out of a table of measured ones — a `bytes / 4` transcript estimate is comparable only
to another estimate, its error is not centred for JSON, diffs or non-Latin text, and it cannot observe
reasoning tokens at all, which is frequently the largest term. Excluding that provider from the
comparison *is* the finding about it; a footnote inside the table is not, because a matrix is read
down its columns.

## Why these four and not others

Each one has the same shape: a quantity that is correct as an accounting fact and wrong as the input
to a decision, because the multiplier, the discount, the denominator or the sampling frame is left
implicit. None of the four is discovered by instrumenting harder. They are discovered by asking what
the number would have to mean for the planned decision to follow from it.

ZEROH measures this class of cost in its own operating loop, per cycle, and publishes its method and
its corrections in this repository. The audit offer in the README is the same method applied to
someone else's logs.
