# Capability ledger

A capability ledger is the list of things your agent can actually do, where an entry is
admitted only after a real tool call returned a real success. Everything else is a guess.

## Why it pays for itself

An agent that starts each run without this list has two bad options: re-test its tools, which
costs a paid run and produces no output, or assume, which produces confident work on a path
that cannot complete. We hit the second failure directly: an earlier cycle planned around
sending email because the cost ledger listed an email provider at zero cost. The provider was
paid for. The tool was not exposed to the agent. Provisioned is not the same as reachable, and
only a successful call distinguishes them.

## Format

Two tables. Available and unavailable. The second is worth more than the first, because it is
what stops a future run from re-testing.

```markdown
## Verified available

| Capability | How it was verified | Cost |
|---|---|---|
| Commit a text file | commit d52799a, 2026-09-18 03:21 | zero |

## Verified unavailable

| Missing capability | Evidence |
|---|---|
| Send email | provider paid for, tool not exposed to the agent |
| Execute code | no shell in environment |
```

## Rules that make it trustworthy

1. **Evidence or nothing.** An entry cites a commit sha, an error string, or a timestamp. An
   entry that cites a belief is a liability, because the next run will trust it.
2. **One transient failure is not an absence.** Upstream errors happen. We logged two failed
   list calls against an API that worked minutes later. Retry before you write it down as
   unavailable, or you will permanently disable a capability you own.
3. **Date the ledger.** Environments change between runs. A stale ledger read as current is
   how an agent misses the capability that just unblocked it.
4. **Separate the tool from the permission.** "Tool exposed" and "tool authorized for this
   target" are different entries. An adapter scoped to one organization will happily accept a
   call and fail on a target outside it.
5. **Record adapter quirks next to the capability**, not in prose somewhere. See
   [`../catalog/github-mcp-adapter.md`](../catalog/github-mcp-adapter.md) for the shape.

## First run

Spend one run enumerating tools and calling the cheap read-only ones. Write the result down.
That run pays for itself the second time the loop starts.
