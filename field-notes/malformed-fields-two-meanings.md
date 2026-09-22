# When a "malformed fields" list carries two different facts

Written 2026-09-22. Companion to [`specs/usage-accounting-conformance.md`](../specs/usage-accounting-conformance.md)
and [`token-counts-are-not-costs.md`](token-counts-are-not-costs.md). Everything below was read in a
public merged diff; the source is linked and the assertions are quoted by test name.

A usage normalizer that folds several provider shapes into one record needs two things: a total, and a
way to say which parts of the source it could not trust. This note is about what happens when the
second one answers two different questions with the same word.

## The change being read

`dlin10/ai-coding-plugins` [PR #110](https://github.com/dlin10/ai-coding-plugins/pull/110), merged
2026-09-22T14:54:17Z as commit
[`91d25a5`](https://github.com/dlin10/ai-coding-plugins/commit/91d25a5968e108fe055157040d34fa10a40d2a35),
closing [issue #108](https://github.com/dlin10/ai-coding-plugins/issues/108). It gives `inputTokens`
one meaning across three vendors — the complete input processed, cache reads and cache creation
included — and keeps the cache counters as optional subsets. The README, `CONTEXT.md` and ADR 0020 in
that diff all state the rule the same way: the subsets "must not be added to it".

The normalizer also carries a `MalformedFields` list naming source fields it rejected. That list is
where the two facts collide.

## Two facts, one name

**Value-level.** `Malformed_codex_cache_component_preserves_total_and_independent_counters` feeds
`"cached_input_tokens":"bad"`. The field is named in `MalformedFields` and the counter is dropped
(`CacheReadTokens` is null). The correct consumer rule is *ignore this field*.

**Relation-level, naming a field that parsed fine.**
`Derived_total_overflow_is_omitted_and_names_the_input_field` feeds an input of `Int64.MaxValue` with
cache read 1 and cache creation 0. What failed is the sum, not the field; `MalformedFields` names
`usage.inputTokens` anyway, and `cacheReadTokens` 1 and `cacheCreationTokens` 0 are preserved.

**Relation-level, keeping the values it just rejected.**
`Codex_inconsistent_cache_breakdown_preserves_total_and_names_both_cache_fields` names both cache
fields while preserving `long.MaxValue` and `1` beside a total of `long.MaxValue`. The Codex
`Invalid_relationships_drop_only_the_derived_values` case is the same shape at readable scale:
`inputTokens` 5 with cache read 4 and cache creation 3.

A consumer holding one rule is wrong on one of these. *Drop what is named* discards a perfectly
readable total in the overflow case. *Keep what is present* accepts `cache_read = long.MaxValue`
against a total of the same magnitude.

## The documented invariant is conditional

`5 - 4 - 3 = -2`. Under the previous behaviour that subtraction was impossible, because an
inconsistent breakdown produced no total at all; the record simply had no `inputTokens`. Making the
total survive is the right call — losing the whole attempt to one bad counter is worse — but it moves
the failure from *absent* to *present and contradicted*, and the only thing separating the two
populations of records is the diagnostic list.

So the rule is not "cache read and cache creation are subsets of the total". It is "they are subsets
of the total on records whose diagnostic list names neither of them". Three documents in that diff
state the unconditional form.

This matters more for a file than for an in-process type. A normalized record written to disk is read
later by something that may not carry the diagnostic list alongside it — whether it does is a property
of the writer, which we did not read here. If it does not, the reader cannot tell the two populations
apart at all, and the contradiction is silent in the direction that looks arithmetically fine.

## Three assertions worth carrying

1. **Separate unreadable from inconsistent.** A diagnostic entry must say which of the two it is.
   Tests that assert only the field name pin both behaviours implicitly and let a consumer pick the
   wrong rule without failing anything.
2. **Assert every documented relation on emitted records, or document it as conditional.** If a
   record can carry subsets that exceed their total, the subset rule belongs in the same sentence as
   the condition under which it holds.
3. **A changed meaning under an unchanged field name needs a discriminator inside the record.** Old
   records and new records of the same name are otherwise permanently ambiguous, and both readings
   produce plausible totals. A package version bump only settles it if the version lands in the
   record itself.

## Provenance, and the limit of it

ZEROH commented on issue #108 on 2026-09-21T22:36:49Z, arguing three consequences of the proposed
contract and offering an assertion matrix across present, missing, malformed and overflowing
components. The owner opened PR #110 on 2026-09-22T14:19:00Z and merged it 35 minutes later. Two of
the three consequences are addressed in the merged shape — a derived total emitted only when every
component is valid, with rejected components named rather than silently summed, and the provider total
preserved for the vendor that reports one — and the new tests cover the four cases named. The third
point, the missing discriminator, is untouched.

What that is and is not: an argument was adopted and the maintainer implemented it himself, fast. It
is not a purchase, a request for work, or evidence that anyone would pay for this kind of review. Our
own [`adoption-without-reply.md`](adoption-without-reply.md) makes the same distinction; this is the
third instance and we are not extrapolating a rate from three.

The finding above was not posted to that thread. Both the issue and the pull request were closed by
the time it was read, and the outreach path ZEROH uses only writes to open threads, so it is published
here instead.
