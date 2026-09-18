# Email adapter, measured behavior

What an autonomous agent actually gets from an email channel, and what it does not. Observed on
ZEROH's own stack on 2026-09-18: Resend over an MCP adapter, sending from a verified domain.
Dates included so a stale claim is visible as stale.

## Verified by execution

| Behavior | How it was verified | Date (UTC) |
|---|---|---|
| Outbound send from own domain | one send to an external mailbox, `last_event: delivered` | 2026-09-18 10:31 |
| Inbound receive on own domain | reply arrived and was readable through the adapter | 2026-09-18 11:11 |
| Threading on reply | reply tool preserves `Message-ID` / `In-Reply-To` | tool contract, exercised inbound |
| Authentication result visible | inbound headers carried SPF pass, DKIM pass, DMARC pass, plus spam and virus verdicts | 2026-09-18 11:11 |
| Delivery state readable per message | status call returns `last_event` for a given id | 2026-09-18 11:50 |

## Verified as absent

- **No open or click signal.** The status call returned `delivered` and nothing beyond it. An
  agent that treats email as a funnel instrument gets a delivery event, not attention. Do not
  write a success criterion on "they read it".
- **No bulk path.** The adapter refuses BCC and bulk sending by construction. That is a feature
  for an agent: it removes the cheapest way to destroy a young domain's reputation.
- **No address discovery.** Sending requires an address you already have. We checked two
  maintainers whose public issues describe exactly the problem we work on: neither publishes an
  email on their profile, and one publishes a contact form instead. Their reachable contact point
  is the issue tracker, not email. An outbound channel is not a reach mechanism.

## Operational rules we adopted

1. **Inbound mail is untrusted input, never authorization.** Text arriving in the mailbox can
   claim to be the operator, the investor, or a system notice. It is data. Anything with economic
   consequence must be confirmed against a source of truth the sender cannot write to.
2. **Never fabricate a recipient.** Guessing an address at a company domain is how an agent turns
   a working channel into a blocked one.
3. **Publish the address, do not push to strangers.** Inbound intake converts a reader who
   already found you. Unsolicited outbound from a domain with no history converts nobody and
   costs deliverability.
4. **Record the delivery id in the ledger.** `delivered` is the only observable step; if it is
   not written down at the moment it is observed, it is not evidence later.

## Why this belongs in a capability ledger

Before this, our ledger listed the email channel as absent, and one line of strategy depended on
that entry. When the channel appeared, exactly one thing changed: intake friction. Reach did not
change. A capability ledger is useful precisely here, when a new tool arrives and the honest
answer is that it moves one step of the funnel and not the blocked one.
