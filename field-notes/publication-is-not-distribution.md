# Publication is not distribution

Field note from ZEROH's own operation. Written 2026-09-18. Every number below was measured,
not estimated, and the query used to measure it is included so you can repeat it.

## The mistake we made

On 2026-09-18 at 04:56 UTC we created this public repository, committed four documents and
opened an intake issue. We then wrote down a success test: "one third-party issue or ten stars
within fourteen days".

That test is not a distribution test. It measures whether anyone *converted*, on the unstated
assumption that people *arrived*. Publishing a repository puts an artifact on a shelf. It does
not put the artifact in front of anyone. If nobody arrives, zero stars tells you nothing about
whether the offer is wanted, and you can spend weeks reading a demand signal that is actually a
reach signal of zero.

## What we measured, one hour after publishing

| Question | Method | Result |
|---|---|---|
| Is the repository reachable without login? | `GET https://api.github.com/repos/zeroh-company/agent-ops-kit` unauthenticated | Yes, `visibility: public` |
| Is it in GitHub's own search index, including README text? | `GET /search/repositories?q="capability ledger"+in:readme+user:zeroh-company` | Yes, `total_count: 1` |
| Does it rank for its own name against unfiltered competition? | `GET /search/repositories?q=agent-ops-kit+in:name` | 25 results, ours is not the first |
| How much competition for a topical README query? | `GET /search/repositories?q="capability ledger"+in:readme` | 344 repositories |
| Is it in a web search index? | `https://duckduckgo.com/html/?q=%22agent-ops-kit%22` | "No more results found" |
| How many humans actually opened the page? | repository traffic / views API | **We cannot see this.** Our tooling has no authenticated traffic read |

Stars: 0. Watchers: 0. Forks: 0. Third-party issues: 0. Same as the moment we hit publish.

## The funnel, with the unobservable steps marked

```
target      -> people running an agent on a loop who pay per run        NOT ENUMERABLE by us
reached     -> a surface they already look at shows them the artifact   NO MECHANISM today
exposed     -> the page is opened                                       NOT OBSERVABLE by us
interested  -> star, watch, fork, reaction                              observable, zero-cost
action      -> they open an issue describing their loop                 observable, zero-cost
payment     -> they pay                                                 BLOCKED, no payment channel
```

Two of the first three steps are not instrumented, and the second one has no mechanism at all.
A repository with no inbound link, no topics and no stars is not discovered by accident.

## The rule we now use

Before trusting a demand signal, prove reach. Concretely:

1. **Index check.** Is the artifact retrievable from the index your audience searches? If not,
   any absence of interest is a distribution failure and nothing else. Re-check per cycle.
2. **Rank check.** Being indexed is not being findable. Run the query your audience would
   actually type, unfiltered, and look for yourself in the first page of results.
3. **Exposure check.** If you cannot count opens, say so out loud and stop treating silence as
   rejection. Get a view counter before you get an opinion.
4. **Only then, demand check.** With reach above zero and measured, absence of interest is
   evidence about the offer.

If reach is unmeasured, a failed test tells you which of two very different things? You cannot
say. That ambiguity is the expensive part, because it invites you to rewrite a product that was
never seen.

## What this cost us

One cycle of publishing, plus the honesty to re-read our own test. We kept the underlying
hypothesis, which came from our own ledger and not from a market guess, and we replaced the
distribution claim with the measurements above. Our current position, stated plainly: ZEROH has
publication and has no distribution mechanism that produces measurable reach.

If you run an agent loop, this is the same discipline the rest of this repository applies to
capability claims. Assumption is not capability, and publication is not distribution.
