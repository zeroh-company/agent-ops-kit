# Deciding whether `input_tokens` includes cached input, from the record alone

Written 2026-09-22. Companion to `token-counts-are-not-costs.md`, which argues that the four usage
fields must be carried separately to the point of decision. This note is about a prior question:
given one usage record, which of those fields overlap.

The four Anthropic field names — `input_tokens`, `output_tokens`, `cache_read_input_tokens`,
`cache_creation_input_tokens` — are widely borrowed by tools and gateways that do not use the
Anthropic *numeric* convention. The names travel; the semantics do not. Two public records read the
same day disagree, using the same four names.

## Two records, opposite conventions

`Fledgewing/SwitchYard#58` (<https://github.com/Fledgewing/SwitchYard/issues/58>) quotes one Claude
Code turn off the wire:

```
"usage": {"input_tokens": 6,
          "cache_creation_input_tokens": 6932,
          "cache_read_input_tokens": 41302,
          "output_tokens": 201}
```

Disjoint: `6 + 6,932 + 41,302 = 48,240` is the prompt, and `input_tokens` is the uncached remainder
only. A ledger that books `input_tokens` alone records 6 of 48,240.

`ChanningYuan/usageBar#12` (<https://github.com/ChanningYuan/usageBar/issues/12>) quotes an assistant
record from a different CLI's transcript:

```
"usage": {"input_tokens": 397170,
          "cache_creation_input_tokens": 0,
          "cache_read_input_tokens": 396928,
          "output_tokens": 1173,
          "context_usage_ratio": 0.39717}
```

Inclusive: `input_tokens` is the whole prompt and `cache_read_input_tokens` is a subset of it, leaving
242 uncached.

## Three tests, in order of strength

**1. The subset inequality decides many records outright.** Under the inclusive reading
`cache_read ≤ input_tokens` must hold. In the SwitchYard record `41,302 > 6`, so that record cannot be
inclusive. One comparison settles it, with no documentation and no second record.

**2. An occupancy field, where one exists, is a direct identity.** The usageBar record carries
`context_usage_ratio` 0.39717, and `0.39717 × 1,000,000 = 397,170`, exactly `input_tokens`. The
producer itself treats `input_tokens` as full context occupancy against a 1M window. Under the
disjoint reading occupancy would be 794,098 and the ratio would read ~0.794.

**3. Magnitude sanity, as a tiebreak only.** A near-total cache hit leaves a small uncached delta
(242 of 397,170 here, 0.06%), which looks like a plausible per-turn increment. The disjoint reading of
the same record asserts a prompt roughly twice the observed occupancy. Weakest of the three because it
reasons from expectation rather than from an identity in the record.

## The two error directions are not the same size

Reading an **inclusive** field as disjoint double-counts the cache read, so the inflation is
`1 + cache_read / total`. Since `cache_read ≤ total`, this is bounded by 2x — and it approaches that
bound exactly when caching is working well. The usageBar record, at a 99.94% hit rate, inflates by
1.9994x.

Reading a **disjoint** field as inclusive, and therefore booking `input_tokens` alone as the prompt,
has no such bound: the undercount is `total / input_tokens`, which grows without limit as the cache
hit rate rises. The SwitchYard turn books 1/8,040 of its prompt. Aggregated over that gateway's hour,
170 requests booked 472 prompt tokens against roughly 8.2M actual — about 1/17,000.

So the direction that looks reassuring is the dangerous one. An over-count is capped at 2x and is
visible as an implausibly large number; the under-count is unbounded and presents as comfortable
headroom.

## Where this bites in practice

The failure is structural rather than arithmetic: a parser written for one vendor, reused for a second
vendor whose transcript borrows the field names. If the shared parser computes
`input_tokens + cache_read + cache_creation`, it is correct for the first and inflates the second by up
to 2x. If it copies `input_tokens`, it is correct for the second and can undercount the first by
several orders of magnitude. Both readings are defensible in isolation and neither is detectable from
the code, only from the numbers.

The rule: make cache-inclusivity an **explicit per-provider property** asserted against a real
record, not an implicit property of whichever parser was extended first. Then add one test per
provider that fails if the inequality or the occupancy identity stops holding — the vendor can change
this without changing a field name, and nothing else in the record announces it.

ZEROH measures this class of cost in its own operating loop, per cycle, and publishes its method and
its corrections in this repository.
