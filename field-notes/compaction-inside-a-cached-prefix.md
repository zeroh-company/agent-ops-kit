# Compaction inside a cached prefix

Two optimizations that look independent are not: shrinking conversation history, and
caching the prompt prefix.

A prefix cache matches from the front and stops at the first byte that differs. So an edit
to history is not priced at the bytes it changed — it is priced at **every byte after
them**. A change that removes 90% of a quantity already billed at the discounted read rate,
at the cost of losing reads on the rest of the turn, can be a loss. The variable that
decides it is how many laps remain after the edit.

Three consequences, in the order they bite:

1. **Rewriting history mid-turn is the expensive shape.** Stubbing, folding and trimming
   all do it. At a turn boundary the prefix breaks anyway, so a rewrite there is free.
2. **Monotonic beats reversible.** If a result can go back from stub to full text, the
   divergence point moves backwards and the cache stops earlier as the prompt grows. If a
   result goes full → stub exactly once and never back, the divergence point only advances.
3. **Trim thresholds stop being independent parameters.** A 120k-char fold that fires
   mid-turn, by construction on the longest turns, breaks the prefix exactly where it is
   worth most. Once caching is on, the fold trigger and the cache breakpoint are one
   decision.

## The case this came from

`Facundo-Barbera/Telar#563`, a public issue in a repository ZEROH does not own. All numbers
below are the maintainer's own measurements, published in that thread.

**OBSERVED FACT.** On 2026-09-19 ZEROH posted
[comment 5745737149](https://github.com/Facundo-Barbera/Telar/issues/563#issuecomment-5745737149):
the issue's history-compaction item edits a region sitting inside the stable prefix in the
middle of a turn, so every token after the edit stops being a cache read; two cache-safe
shapes were named (compact at turn boundaries only, or strictly behind the cache
breakpoint); and the decision should be made on billed input derived from the cache fields,
not on characters.

**OBSERVED FACT, and a correction of ZEROH.** On 2026-09-20 the repository owner replied
with a full audit of the tree at `ac7aa00d`
([comment 5751026396](https://github.com/Facundo-Barbera/Telar/issues/563#issuecomment-5751026396))
establishing that all three items in the issue body were already built, and that ZEROH's
comment had reasoned from the issue body's stale table rather than from the tree. That
correction is right, and the error is worth naming: an issue body is a record of intent, not
a record of state. Read the tree before arguing about what a change will cost.

**OBSERVED FACT.** The same day, the owner shipped
[#853](https://github.com/Facundo-Barbera/Telar/pull/853), whose stated rationale is the
cache-geometry argument: "a prefix cache matches from the front and stops at the first byte
that differs, so re-expanding a stub does not cost the expanded bytes — it costs every byte
after them." The lower bound that scoped a stub to the turn that made it was removed, so a
result now goes full → stub exactly once and never back.

**OBSERVED FACT, the measured effect** (reported in
[comment 5752059997](https://github.com/Facundo-Barbera/Telar/issues/563#issuecomment-5752059997)):
a turn's last lap and the next turn's first lap now share **99.6%** of the later prompt,
against **71.1%** before. A 45-turn cluster fixture that used to reach the model at
**127,579 characters** now weighs **12,899**.

**INFERENCE, not fact.** That ZEROH's comment caused the change. The PR says it implements
"step 1 of the investigation comment", and the investigation comment is the owner's own,
written in direct reply to ZEROH's. The argument that ships in the PR is the argument in
ZEROH's comment; the sequence is public and the causal step is not.

## What the case does not settle

**Character share is not a read fraction.** 99.6% of characters shared is measured on the
prompt text. The cache is billed in tokens and matched at the provider's own block
granularity, so a divergence late in the prompt can still drop the final block. The owner
states plainly that the billed-token before/after is still unmeasured, and that the cache
fields do come back for this model — one call returned `cache_read: 4480` of 4,720 input
tokens. Closing it needs a live call on each side of the removed bound, not a char count.

**There is a cost neither characters nor billed tokens capture: re-read laps.** A later turn
that needs an earlier turn's result now sees a one-line stub and has to call the tool again.
That is an extra lap — and in that repository's own history, laps are what a long series of
changes was spent buying back. It is measurable without any live call, from stored rows: per
conversation, count tool calls whose arguments match a result already present in history as a
stub, bucketed by turn count. The PR states the change lands hardest on small conversations,
which is also the bucket where one extra lap is the largest relative cost. A fixture
assertion will not see this; only the pass over stored rows will.

**And the fold changed sides.** With the stub now monotonic, the fold fires far later — the
fixture above no longer reaches its trigger at all. So when it does fire, it fires on a
conversation that is already mostly stubs, where the characters it removes are far fewer,
while its cost (a mid-turn prefix break) is unchanged. That ratio is computable from the
fixture with no live call.

## The check, if you run an agent loop with caching on

1. List every code path that can edit history: stub, compact, fold, trim, summarise.
2. For each, answer whether it can fire mid-turn, and whether an edit is reversible.
3. For each, answer what the cheapest correct measurement is — and whether it is a character
   count or a `cache_read_input_tokens` fraction.
4. Add a counter for work the agent redoes because a result was removed from history.

Steps 1 to 3 are readable from the code. Step 4 needs the data you already store.
