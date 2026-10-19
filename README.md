# Digital Wallet Transaction Notifications

How a **mobile wallet platform** tells customers that money moved: the SMS, push and in-app alerts, the templates behind them, what happens when a message does not arrive, and which alerts a customer should never be allowed to switch off. Written for operators and payment teams evaluating [EVD System](https://evdsystem.com/) by [MoboGage](https://evdsystem.com/about-mobogage/).

---

## What counts as a transaction notification

A **transaction notification** is the message a wallet sends after a ledger event: cash-in, cash-out, P2P sent or received, merchant payment, bill pay, reversal, or a failed attempt. It should carry just enough for the customer to recognise the event. That means amount, a masked counterparty, a reference, the time, and usually the new balance.

Transactional alerts are tied to one ledger entry; marketing messages are not. Run them on separate queues with separate sender IDs, or a fraud alert ends up stuck behind a promo blast.

## Why the alert carries more weight than it seems

In many wallet takeover cases, the first warning comes from the customer, not the risk engine. They get a "you sent 2,000 to 07••••381" SMS they did not expect and call the helpline within minutes. A wallet that notifies on every debit gets that early signal. One that only notifies on credits does not.

Alerts also do practical work at the counter. A customer doing cash-in at an agent usually waits for the SMS before walking away, so a few minutes of lag backs up the queue. A seller hands over goods only after the P2P credit alert lands. And in a dispute, the reference printed in the SMS is the fastest way for support to find the transaction.

## How the pipeline is usually wired

```
Ledger commits txn (txn_id)
        |
        v
  Outbox row written in the same DB transaction
        |
        v
  Notification worker
    - pick template by event_type + language
    - render (mask MSISDN, format amount, ledger balance)
    - choose channels from customer prefs + policy
        |
   +----+-------------+---------------+
   |                  |               |
 Push (FCM / APNs)   SMS gateway     In-app inbox
   |                  |               (always written)
 delivered?         final DLR?
   |                  |
 no -> SMS fallback  no -> retry on schedule, then mark undelivered
```

**Send after commit, never before.** Use an outbox table written inside the same database transaction as the ledger entry. If the transaction rolls back, the alert row disappears with it. Firing the SMS from application code before the commit is how a customer receives "you received 500" for money that never landed.

**Push first for cost, SMS for anything that matters.** Push is close to free but can be dropped silently when the token is stale. Many schemes send SMS anyway for security events and for debits above a threshold.

**One alert per event.** Build an idempotency key from `txn_id + event_type + recipient`. Retries reuse it. Without it, a gateway timeout turns into three "you sent" messages and a frightened customer.

**Templates are versioned and length-aware.** A GSM-7 SMS holds 160 characters per part. Arabic, Amharic or other non-Latin scripts fall back to UCS-2 at 70 characters per part, so a template that fits in one English part can cost three in Arabic. Keep a template per language and check the rendered length, not the raw template.

```
P2P_SENT_EN  You sent {amount} {ccy} to {cpty_masked}. Ref {ref}. Bal {balance}. Not you? Call {helpline}
CASHOUT_EN   Cash-out {amount} {ccy} at agent {agent_code} {time}. Ref {ref}. Bal {balance}
```

## Worked example (figures are illustrative)

Policy for this scheme (**illustrative**): push first, wait 30 seconds for a delivery receipt, then SMS. Any debit above 1,000.00 units also triggers SMS straight away. SMS retries at 1, 5 and 15 minutes, then the alert is marked undelivered.

A customer sends a P2P transfer of **1,200.00** at 20:14:00. Their push token is stale because they reinstalled the app on a new phone last month.

| Time | Event |
|------|-------|
| 20:14:00 | Ledger commits, outbox row created |
| 20:14:01 | Push sent; amount is above 1,000.00 so SMS also sent |
| 20:14:03 | SMS gateway returns `submitted` |
| 20:14:31 | No push receipt; SMS already in flight, so no second SMS |
| 20:14:40 | Final DLR `delivered` for the SMS |

The recipient's credit alert follows the same path. If the amount had been **300.00**, the SMS would have waited until 20:14:31 and gone out only after the push timed out.

Cost check, also **illustrative**: at 0.02 units per SMS part, the English alert fits in one part. The Arabic version renders to 104 characters, two UCS-2 parts, so 0.04 per send.

## Fields worth logging

| Field | Why |
|-------|-----|
| `notification_id`, `idempotency_key` | Stops duplicates, links retries |
| `txn_id`, `event_type` | Which ledger event triggered it |
| `recipient_msisdn` (masked in logs) | Who it went to |
| `channel`, `attempt_no` | Push, SMS or inbox, and which try |
| `template_id`, `template_version`, `language` | What text the customer actually saw |
| `provider_msg_id`, `dlr_status` | Gateway proof for disputes |
| `queued_at`, `sent_at`, `delivered_at` | Latency tracking |
| `pref_snapshot` | Opt-out state in force at send time |

## Opt-outs, and where they stop

Customers can reasonably mute marketing, choose push instead of SMS for small credits, or pick a language. Security alerts are different: PIN changes, new device bindings and password resets should always reach the customer, and many schemes refuse opt-out on large debits too. Store each preference change with a timestamp so a dispute can show what was in force.

## Things that go wrong

**Alert before commit.** A false credit message can cost real money if the recipient hands over goods. Outbox only.

**Stale balance in the text.** Rendering the balance from a cache while a concurrent debit posts produces a number that never existed. Render from the balance snapshot attached to the ledger event.

**Full numbers and names.** Printing the counterparty's full MSISDN or full name in an SMS leaks personal data to anyone holding the phone. Mask both.

**Unregistered sender IDs.** Some operators filter SMS from unregistered sender IDs. Alerts vanish and nobody notices until a fraud case.

**The 2 a.m. flush.** After a gateway outage, tens of thousands of queued alerts drain at once in the middle of the night. Give alerts an expiry; anything older than your cutoff goes to the in-app inbox only.

A delivery and incident runbook for ops is in [docs/notification-delivery-runbook.md](./docs/notification-delivery-runbook.md).

---

## FAQ

**What are digital wallet transaction notifications?**
They are the SMS, push or in-app messages a wallet sends after each ledger event, showing amount, masked counterparty, reference and balance so the customer can recognise or dispute the transaction.

**Should a wallet use SMS or push for transaction alerts?**
Both. Push is cheaper and fine for small credits. SMS should back it up for security events and larger debits, because push can be dropped without any error.

**How do you stop duplicate notifications?**
Use an idempotency key built from the transaction id, event type and recipient, and have every retry reuse it.

**Can customers turn off transaction notifications?**
They can usually mute marketing and choose channels for low-value alerts. Alerts for PIN changes, new devices and large debits should stay on.

**Does EVD System cover wallets as well as vouchers?**
MoboGage's broader stack includes digital wallet and mobile money alongside the EVD System voucher platform.

---

## Glossary

| Term | Meaning |
|------|---------|
| Outbox pattern | Writing the alert row in the same DB transaction as the ledger entry |
| DLR | Delivery receipt returned by an SMS gateway |
| Sender ID | The name or number an SMS appears to come from |
| UCS-2 | SMS encoding for non-Latin scripts, 70 characters per part |
| Idempotency key | Value that makes a retried send a no-op |
| Fallback | Switching channel when the first one does not confirm delivery |

## Further reading

- [Wallet cash-in, cash-out, transfers and reversals](https://evdsystem.com/evd-software/wallet-cash-in-out-transfers-and-reversals/) on evdsystem.com
- [Mobile money platform with merchant payment](https://evdsystem.com/mobile-money-platform-with-merchant-payment/) on evdsystem.com
- Related repo: [digital-wallet-p2p-transfer](https://github.com/iwtglobal/digital-wallet-p2p-transfer)

## License

MIT. See [LICENSE](./LICENSE). Educational material only, not legal or regulatory advice.
