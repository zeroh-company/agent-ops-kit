# Cycle cost policy

Every run of an autonomous loop costs money. When a run has nothing useful to do, it does not
stop by default; it invents work. Refactoring, restructuring, writing plans about plans. The
output looks like progress and the cash effect is negative.

A cycle cost policy is the written rule that permits the loop to stop.

## Minimal policy

```
1. Read the authoritative state. Not a cached copy, the source.
2. Check whether any capability changed since the last run.
3. If a capability changed, attack the one that unblocks revenue first.
4. If nothing changed and no test is pending, append one line to the log and end the run.
```

Step 4 is the whole point, and the hardest to implement, because ending early reads as
underperformance. It is not. A run that ends in ninety seconds having confirmed no change has
spent almost nothing and preserved the option to act when the situation changes.

## Where a run should refuse to build

Before producing an artifact, check that a path exists from that artifact to the outcome you
want. In our case the chain to the first dollar is: produce something deliverable, make it
visible to a buyer, get paid, record the revenue. Producing while the payment link is missing
manufactures inventory that cannot convert. The correct action is to name the missing link and
stop, not to build more inventory.

Apply the same test to your loop. If step N+1 is impossible, work on step N is waste, however
good it looks in the diff.

## What to log per run

Keep it to a handful of lines: what state you read, what changed, what you did, what you
chose not to do and why. The last item is the one that saves the next run a paid rediscovery.

## Distinguish waiting from stalling

Waiting is cheap and correct when the blocker is external and named. Stalling is a loop that
keeps producing motion around a blocker it has not named. The log makes the difference legible
after the fact, which is the only way anyone can audit whether the loop is behaving.
