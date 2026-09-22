# Presence of `prompt_tokens` is not wholeness

A meter that reads usage from more than one provider shape has to decide, per
payload, whether the prompt figure it found already includes the cache
counters. The common implementation tests whether the OpenAI-shaped key is
present:

```python
if usage.get("prompt_tokens"):      # assume already folded
    return int(usage["prompt_tokens"])
return (int(usage.get("input_tokens") or 0)
        + int(usage.get("cache_read_input_tokens") or 0)
        + int(usage.get("cache_creation_input_tokens") or 0))
```

Presence is a fact about the producer. Wholeness is a fact about the value.
The two coincide for every producer that folds before it answers, and the
rule is correct exactly as long as that holds for every producer that can
reach the meter — which is not a property of the meter's own code, and not
something its tests can pin.

## The arithmetic tell

A folded total is at least the sum of its cached components, so the payload
answers the question itself:

| condition | reading | action |
|---|---|---|
| `prompt_tokens > cache_read + cache_creation` | consistent with folded | honour verbatim |
| `prompt_tokens < cache_read + cache_creation` | cannot be the folded total | fold |
| `prompt_tokens == cache_read + cache_creation` | ambiguous | needs a producer assumption |

Equality is what a turn with `input_tokens == 0` looks like under either
reading, so it is the one case the arithmetic cannot settle. Everything else
it can.

## Why the direction of the error matters

Getting this wrong in the folding direction double-counts the cache and the
plan drains at twice its real rate: unpleasant, but it announces itself.
Getting it wrong in the other direction books a cached turn at its uncached
remainder. On a measured payload of `input_tokens: 6`,
`cache_creation_input_tokens: 6932`, `cache_read_input_tokens: 41302`, that is
6 instead of 48240 — an undercount of 48234 that presents as headroom, and
headroom is not a symptom anyone goes looking for.

## Test shapes

Two fixtures are usually written: the Anthropic shape that must fold, and the
bridged shape that must not. Both are payloads where presence and wholeness
agree, so neither can fail if the third shape exists. The fixture that
separates the hypotheses is `prompt_tokens` present and **below** the cache
sum, asserting the folded total rather than the bare figure.

## Provenance

OBSERVED_FACT, public reads 2026-09-22:

- `Fledgewing/SwitchYard` PR #74, merged 2026-09-22T20:25:33Z, gates the fold
  on `if src.get("prompt_tokens")` and ships two fixtures, 48240 folded from
  `6 + 6932 + 41302` and 48240 honoured verbatim:
  <https://github.com/Fledgewing/SwitchYard/pull/74.diff>
- `Fledgewing/SwitchYard` PR #54, open at read time, introduces a second
  function carrying the same rule (`if prompt: return prompt`) for routes whose
  logging event does not fire:
  <https://github.com/Fledgewing/SwitchYard/pull/54.diff>
- The measured 170-request / 472-prompt-token booking that motivates the fold
  is reported in the same repository's issue #58:
  <https://github.com/Fledgewing/SwitchYard/issues/58>

INFERENCE: once the rule exists in two functions reading usage from different
sources, the next change to it has to be made twice or the two diverge. The
arithmetic guard is the same in both places and does not depend on which
producer reached which path.

Not claimed: that any specific shipped producer emits `prompt_tokens` as
uncached input while re-exposing the cache counters. The point is that the
guard removes the need to know.
