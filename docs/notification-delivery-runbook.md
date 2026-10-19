# Notification Delivery Runbook

Ops steps for missing alerts, gateway incidents and template changes on a mobile wallet platform. Replace every figure with your own policy values.

## When a customer says "I never got the SMS"

1. Confirm wallet id, MSISDN, approximate time and the transaction reference if they have one.
2. Look up the notification by `txn_id`. Check every attempt: channel, `provider_msg_id`, `dlr_status`.
3. If the DLR says `delivered`, check for a full inbox, a blocked sender ID on the handset, or a recent SIM change.
4. If the DLR is missing or `failed`, check the gateway status for that time window and operator.
5. Resend once from the in-app inbox or helpline if policy allows. Do not trigger a fresh ledger event to "re-fire" the alert.

## Gateway outage

- [ ] Switch to the secondary SMS route if one is contracted
- [ ] Keep security alerts (PIN change, new device) at the head of the queue
- [ ] Set an expiry so alerts older than the cutoff (**illustrative**: 2 hours) go to the in-app inbox only
- [ ] Suppress marketing sends until the transactional backlog is clear
- [ ] After recovery, report undelivered counts by event type to risk

## Before releasing a template change

- [ ] Render every language version with real-length sample values
- [ ] Check SMS part count (GSM-7 160, UCS-2 70 per part)
- [ ] Confirm counterparty is masked and no full name appears
- [ ] Bump `template_version` so disputes show which text went out
- [ ] Test on at least one handset per major operator

## Weekly checks

- [ ] Median and p95 time from commit to delivery, by channel
- [ ] Undelivered rate by operator and sender ID
- [ ] Duplicate sends (same idempotency key, more than one delivered)
- [ ] Opt-out changes on security alert types (should be zero)
