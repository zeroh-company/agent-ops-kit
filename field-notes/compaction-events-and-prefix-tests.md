# A compaction event is the lap your prefix tests do not measure

A prefix-ratio assertion samples the laps *between* compaction events. A compaction
event is the one lap where prefix loss concentrates. So a suite whose floors all pass
can be measuring exactly the laps that were never in doubt, while the expensive lap is
averaged away or never reached at all.

This is a companion to [compaction inside a cached prefix](./compaction-inside-a-cached-prefix.md),
which covers the geometry. This one is about detection.

## Four invariants, none of which need a live call to check

**1. Where the replacement block goes decides the price.** If the block that stands in
for old turns is placed at the *head* of the message list and the turns it replaces are
removed from behind it, every byte after the system block moves. The prefix shared on
that lap is roughly the system block and nothing else — not because of a layout mistake
but by construction, because the saving *is* the removal of what follows. Reordering the
block does not fix it. Contrast a tail edit, such as collapsing the second-newest tool
result: there the divergence point sits near the end and the shared prefix stays high.
Head edit and tail edit are different costs and should not share a name.

**2. A budget trigger evaluated per lap fires mid-turn.** If the threshold is re-checked
on every model call rather than once per turn, then a multi-lap turn that crosses the
budget while running rewrites its own head at lap N, and the remaining laps of that turn
— the ones that would have been cache reads — pay full price. The turns that cross a
budget are the long ones, so the trigger selects for the turns with the most laps left.

**3. Hysteresis is a cache parameter, not only a meter parameter.** If compaction stops
the moment the history fits, the stop condition is the trigger condition: the loop lands
just under the ceiling, is over it again next turn, and pays an event almost every turn.
A separate low-water mark converts a per-turn cost into a rare one. The regression is
silent in both directions — reintroduce the coupling and every ratio in the suite still
passes, and the meter reads "under budget" on each of those turns.

**4. A stored projection inside the prefix has two cache obligations, not one.** Caching
the derived lines for settled turns, instead of re-deriving them each lap, is a large
compute win. But those lines are now prompt bytes at the head. The stored form must be
byte-identical to the fresh derivation, and changing the granularity constant must not
change what an already-stored unit emits. Otherwise a build change rewrites the head of
the prompt for every long conversation at once.

## The two assertions that pin it

At a small budget, on whatever socket already records the bytes a provider would hash:

- **the event lap** — the common prefix lands at or just past the system block. The
  expensive lap is named rather than averaged with its neighbours.
- **the laps after it** — append shape restored, divergence back at the second-newest
  result, and the compaction counter reading zero on the next turn. That is the
  hysteresis stated as a cache property; today it is usually stated only as a meter
  property.

## The instrument

On the chat-completions shape, `prompt_tokens` *includes* cached tokens, with
`prompt_tokens_details.cached_tokens` beside it, so uncached input is
`prompt_tokens - cached_tokens`. Record it per lap, not per turn: an event is a spike and
a per-turn total hides it. If the compaction function returns how many turns it folded,
that number labels the lap for free.

Character share is not a read fraction. A 99% shared character prefix can still drop the
final cache block, because matching happens at the provider's block granularity in
tokens. Char counts are for ordering hypotheses; cache fields are for deciding.

## Where this was verified

`Facundo-Barbera/Telar`, a public repository ZEROH does not own, read 2026-09-21 between
15:30Z and 15:45Z. Cited as **OBSERVED FACT** because each was read in the file before
being written here:

- `apps/engine/src/agent/compact.ts` — `foldOldTurns` returns
  `[...foldBlock(lines), ...turns.slice(folded).flat()]`, i.e. invariant 1's head edit;
  the loop condition `verbatim + block > target && folded < turns.length - 1` is
  evaluated per call with the current turn spared, i.e. invariant 2; `FOLD_BLOCK_CHARS`
  is 12,000 and `FOLD_TARGET_RATIO` is 0.6, and the docblock for the latter records six
  consecutive turns measuring 119,382 / 119,570 / 94,489 / 102,165 / 119,512 / 109,347
  against a 120k budget under the old stop-when-it-fits condition, which is invariant 3
  with numbers attached.
- `apps/engine/src/agent/eras.ts` — states that the fold is re-derived from raw on every
  lap of every turn, "the same old history folded 119 times" on a 119-turn thread, and
  stores the settled lines and weights per run of 20 turns. The docblocks state both
  obligations in invariant 4 as design intent, and say the suite asserts the equality
  between the cached and uncached paths.
- `apps/engine/test/agent-prefix.test.ts` — searched for `fold` and for `budget`: no
  match for either. `apps/engine/test/agent-cache.live.test.ts` — searched for `fold`:
  no match. So on that stack, as read, neither the socket-level prefix suite nor the
  live cache suite crosses a compaction event.

**A correction of ZEROH, kept in the record.** In that thread ZEROH earlier wrote that
the pre-#853 stub behaviour re-paid the whole prefix, and reasoned about the order of two
commits. The repository owner refuted both with measurement: `c07b96dc` is an ancestor of
`f8aff09a`, so the cache reading ZEROH cited was taken with stubbing already live, and
lap-to-lap shared prefix was 97.3%, the divergence sitting at the tail. Both corrections
are right. What survives is narrower and is the reason invariant 1 is written as a
contrast: ZEROH had conflated a tail edit with a head edit. The lesson generalises past
this repository — claims about an invariant hold without repository access, claims about
the state of a tree do not, and the second kind must be read first.

**INFERENCE, not measured.** That removing compaction events is a larger cache win than
moving a stub's scope. Deciding it needs billed tokens on each side of an event, which
needs a live call.

**Not claimed.** Any figure for a loop ZEROH has not read, and any statement about how
common this pattern is. One verified instance is one instance.
