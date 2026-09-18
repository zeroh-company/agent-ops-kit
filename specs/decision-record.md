# Decision record

A decision record is a small JSON document written at the moment a decision is made, holding
the hypothesis behind it and the condition that would kill it.

Without one, an agent loop drifts: each run picks a plausible direction, none of them is ever
falsified, and nothing is ever abandoned. Sunk cost accumulates silently because there is no
written line saying what would count as failure.

## Fields

```json
{
  "decision_type": "hypothesis | strategy_change | capability_request | build",
  "primary_hypothesis": "who pays, for what, and why that is plausible",
  "target_customer": "narrow enough to be reachable",
  "problem": "the pain, stated as the customer would state it",
  "proposed_value": "what changes for them",
  "why_now": "what makes this timely rather than eventually true",
  "evidence": ["concrete observations, with URLs when they came from the web"],
  "smallest_test": "the cheapest action that could falsify the hypothesis",
  "expected_cost_usd": 0,
  "expected_time_to_signal": "hours or days, not someday",
  "success_signal": "an observable event, not a feeling",
  "failure_signal": "an observable event",
  "abandon_condition": "written before you are emotionally invested",
  "required_capabilities": ["what must exist for this to run at all"],
  "currently_blocked_by": "the one thing stopping execution, or null",
  "next_action": "the next concrete step",
  "confidence": 0.35
}
```

## The fields that actually do the work

**`abandon_condition`.** Write it before starting. An agent is unusually good at generating
reasons to continue, and a condition fixed in advance is the only cheap defense.

**`evidence`.** Force a distinction between what was observed and what was inferred. "Agents
are popular" is not evidence. "Two consecutive runs in our ledger re-tested the same tools,
at a paid run each" is.

**`currently_blocked_by`.** Exactly one item. If everything is blocking, nothing is
prioritized. Naming the single link that would unblock cash is what turns a wish list into a
request someone can act on.

**`confidence`.** Calibrate it and then check it later. A loop that records 0.9 and is wrong
repeatedly has a measurable bias worth correcting.

## Placement

One file per decision, `decisions/YYYY-MM-DD-slug.json`, never edited after writing. Append a
superseding record instead. The value is in the trail, and an edited trail has no value.
