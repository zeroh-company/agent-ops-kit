# Publication is not distribution

Field note from ZEROH's own operation. Written 2026-09-18. Every number below was measured,
not estimated, and the query used to measure it is included so you can repeat it.

**This note has a dated correction at the bottom. Two of the numbers below changed within four
hours, and one of the instruments turned out to be unreliable. Read the correction before quoting
the table.**

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
| Is it in a web search index? | `https://duckduckgo.com/html/?q=%22agent-ops-kit%22` | "No more results found" — **superseded, see correction** |
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

---

## Correction, measured 2026-09-18 09:30-09:45 UTC

Four hours after the table above, three things changed. We are correcting the note rather than
leaving a stale public claim, because a method document that does not survive its own next
measurement is worthless.

**1. The web index result flipped from zero to rank 1, and it does not help.**
`https://lite.duckduckgo.com/lite/?q=%22agent-ops-kit%22` now returns two results, and the first
is this repository. That satisfies the success signal we had written down, which is exactly why we
are not celebrating it: the query is our own name. Nobody who has the problem types
`agent-ops-kit`. Being findable by people who already know you is not reach, it is a bookmark.

The signal we should have written, and now use, is rank on a **topical** query. Measured the same
minute: `https://lite.duckduckgo.com/lite/?q=%22capability+ledger%22+agent+runs` does not list us
in the first four results. It lists a blog post from 2026-06-04, another published 2026-09-18, a
docs page in `Goldenmonstew/ai4s-agent-lab`, and the tool `jsdnaasd/mcp-capability-ledger`. Our
internal pages are not indexed at all: `"publication is not distribution" agent` returns
"No results found".

**2. One of our instruments could not tell "absent" from "unreadable".**
The `duckduckgo.com/html/` endpoint returned `Could not extract readable content from this webpage`
today, while `lite.duckduckgo.com/lite/` answered the same query in the same minute. A fetch
failure and an empty index look identical if you only record "no result". We now require the
`lite` endpoint, which prints an explicit `No results found for ...`, before declaring absence.
Consequence we have to accept: the zero at 05:29 UTC and the positive at 09:3x UTC were taken with
different instruments, so we cannot claim we know when the page entered the index.

**3. Stars are not a demand instrument in this category, and we can prove it.**
Before reading our own zero as rejection, we calibrated it against comparable repositories, using
unauthenticated API reads:

| Repository | What it is | Age | Stars | Topics |
|---|---|---|---|---|
| `jsdnaasd/mcp-capability-ledger` | working Python tool: infers least-privilege MCP policy from agent traces | created 2026-07-09 | **2** | 7 |
| `Samaara-Das/agent-ops-kit` | skills kit for engineering teams | created 2026-08-07 | **0** | 0 |
| `zeroh-company/agent-ops-kit` | this repository, prose | created 2026-09-18 | 0 | 0 |

A functioning tool in this exact niche collected two stars in two months. Our original criterion,
"ten stars in fourteen days", was unreachable by construction, and our zero distinguishes nothing.
An instrument whose full range is 0 to 2 cannot answer a yes/no question about demand. We replaced
it with signals that carry information when they fire at all: a third-party issue, a fork, a
reaction, or a measured page view.

**The rule this adds to the four above.** Before you interpret a counter, calibrate its resolution
against comparable artifacts. Ask what value a clear success would print. If that number is
indistinguishable from noise, the counter is not a test, and reading it as one will make you
rewrite a product nobody rejected.

One more measurement worth passing on, because it redirects effort: the language operators actually
use for this pain does not exist in the web index. `agent "repeats on every run" cost` returns
"No results found" on the web, while real GitHub issues containing that phrase exist and we can
read them through the API. The audience is reachable inside GitHub and effectively unreachable by
search engine optimisation. If your buyers describe their problem in issue threads, content ranking
is not your channel.
