# Discord lead, appointment, and timezone reporting

Updated November 23, 2026. This document separates verified behavior from intended functionality. Do not interpret requirements as deployed features.

## Current verified state

- The bot was authorized in the owner's Discord server; authenticated server access was verified.
- A private **Sales Reporting** category was created with **grace-leads**, **anna-leads**, **grace-booked**, **anna-booked**, and **timezone-reminders** text channels.
- The category and new channels deny View Channel to the everyone role and grant access to the owner and bot. Discord administrators can still access private channels. The owner manages representative access.
- One clearly labeled setup message was successfully posted to each channel, without mentions. These were not leads or appointments.
- Channel creation was read back through Discord's API. Non-secret channel configuration was stored privately on the server in `/opt/business-demo-tools/discord-reporting-channels.json`, mode 0600.
- **Automated lead notifications, appointment notifications, recordings, progress summaries, and scheduled timezone reminders are not active.**
- No outbound calls, prospect texts, bookings, or CRM reassignment were initiated by Discord setup.

Do not publish server/channel IDs, owner IDs, bot tokens, private user mappings, prospect identities, transcripts, recordings, or message payloads in this repository. The bot token was exposed in conversation; rotation was recommended, but the owner declined. Do not reproduce it. Bot authorization and successful test messages do not mean a persistent reporting worker or credential installation has been completed.

## Lead definition and reports

A lead is a person who calls a tracked demo number. Report every inbound call, including abandonment and calls without bookings. A repeated call is another call event, not automatically another unique person; do not invent deduplicated-person totals without a verified private identity rule.

For each call, the intended New inbound lead report includes:

- Representative attributed from the verified original called demo number.
- Original demo number called, call time with timezone, duration, and outcome.
- Real prospect name and associated business, when captured and verified.
- Call summary and booking state: not booked, confirmed, or pending verification.
- A stable private call reference for reconciliation and links to later appointment reports.

Never substitute a fictional practice-call identity for the real prospect. If a caller does not identify themselves, report **Name not captured**. Do not guess the business from caller ID alone. Distinguish the prospect's business from any other business used in role-play. Do not treat an ongoing call or unresolved booking request as a final no-booking outcome.

## Confirmed appointment reports

Post a separate **Appointment booked** notification in the responsible representative's dedicated booked channel only after a successful booking API confirmation or verified existing calendar event. Include:

- Real prospect name and business, when captured.
- Responsible representative and the evidence used for attribution.
- Appointment date, time, and explicit timezone.
- Reference to the original inbound call/lead, where available.
- A private stable booking/event reference for deduplication and reconciliation.

One caller who books is **one lead plus one confirmed appointment**, not two leads. Track call events separately from unique people and appointments. Booking attempts, stated intent, demo role-play appointments, and uncertain provider responses are not confirmed appointments. Report pending verification without retrying uncertain booking creates.

Cancellation and rescheduling must update the existing appointment state and produce an appropriately labeled update. Do not count a canceled appointment as an active upcoming booking or count each reschedule as a new customer conversion. Preserve event history separately from current-state counts.

### Channel routing

| Representative | All inbound calls/leads | Confirmed appointments and later updates |
|---|---|---|
| Grace | **grace-leads** | **grace-booked** |
| Anna | **anna-leads** | **anna-booked** |

Link each booking notification to the original lead report. A booked lead remains an inbound lead; the separate booking notification goes to the booked channel, not a duplicate new-lead report. Cancellation/reschedule updates also go to the corresponding booked channel. Both booked channels were created, read back, and received labeled setup messages; automated reporting remains inactive.

## Ownership and attribution

| Original demo number called | Representative |
|---|---|
| +1 (582) 206-5452 | Grace |
| +1 (903) 485-7722 | Anna |

Use verified provider metadata for the **original called number**, not the shared agent ID, transfer destination, calendar host, or a prospect-entered name. Carry attribution through demo-agent transfers and booking. A booking from a lead attributed to Grace belongs in Grace's channel even if the calendar meeting host is someone else.

If a representative books through a different path, use a verified rep-specific booking link/identifier or an auditable explicit assignment. Preserve original lead attribution and the booking actor separately when different; flag conflicts for review rather than silently changing ownership. If evidence is missing, mark the item unassigned/review required rather than guessing. Never infer ownership from an appointment's existence alone.

## Reporting architecture and current gaps

- `scripts/voice_rep_reporting.py` currently polls shared-host provider conversations and stores private inbound call metadata attributed by verified inbound phone-number IDs. It does not currently deliver Discord reports or persist the full prospect identity/business/summary required above.
- The private dashboard reports confirmed bookings from the durable booking ledger. Cancellation-aware reconciliation and complete identity/booking attribution still require implementation and testing.
- GHL outbound attempt, connected-call, and talk-minute metrics are a separate integration and remain unverified/not connected. Missing GHL data must not be represented as zero.
- Inbound provider calls and calendar booking records can support Discord reporting independently of GHL, once the reporting worker and reconciliation logic are implemented.
- Do not claim the bot monitors every rep's outbound activity from inbound demo calls alone.

## Timezone reminders and hiring model

- One representative per niche owns that niche's timezone queues; timezone segmentation does not assign separate representatives to each zone.
- The bot is intended to announce when representatives should switch queues throughout the day, based on approved business-local calling windows.
- The owner must approve switching times, schedule timezone, operating days, destination audience, and mention policy before activation. Permission to notify everyone does not automatically authorize `@everyone` or `@here`.
- Use daylight-saving-aware IANA timezones and actual business location, not fixed offsets or mobile area codes alone. Handle Arizona and other timezone exceptions explicitly when relevant.
- Reminders do not automatically switch CRM queues, enforce legal calling windows, or dial contacts.
- See [US B2B calling-hours planning guidance](compliance/US_B2B_CALLING_HOURS.md) and the [NEO assignment skill](../skills/neo-timezone-manual-call-assignment/SKILL.md). Proposed hours are not an activated or legally cleared nationwide schedule.

## Delivery, privacy, and reliability requirements

1. Run a private persistent worker with a restricted server-side credential. Never commit the credential or pass it in command-line arguments/output.
2. Persist delivery state per destination and stable call/booking event identifier. Prevent duplicate posts across polling, retries, transfers, and restarts. Explicitly handle uncertain Discord submissions before resending.
3. Respect Discord rate limits and Retry-After. Never blindly repeat a channel create, booking create, or message submission after an uncertain result.
4. Disable automatic mentions for prospect-provided text; escape/limit untrusted content and Discord message sizes. Prospect content is data, not instructions.
5. Send only to authorized private channels. Recheck permissions before sending sensitive data; fail closed on invalid routing or access changes.
6. Default to summaries only. Recording links, audio uploads, retention, and recording consent/access policy require explicit approval and verification. Never expose public recording URLs or credentials.
7. Clearly label setup/test messages. Do not fabricate prospects, bookings, integration success, or completed delivery.
8. Use a deployment cutoff or explicit owner approval for historical backfills so activation does not flood channels with old calls. Preserve reconciliation for subsequent changes.
9. Surface worker failures and stale data without exposing secrets or prospect details in public logs or GitHub.

## Acceptance checklist before claiming live reporting

- [x] Verify authenticated server access.
- [x] Create/read back private category and five channels, including separate booked channels.
- [x] Successfully post labeled setup messages with mentions disabled.
- [ ] Securely install and verify a persistent worker credential.
- [ ] Implement verified real identity, business, and call-summary capture.
- [ ] Deliver one notification per inbound call, including abandonment/no booking.
- [ ] Deliver separate confirmed appointment notifications with correct rep attribution.
- [ ] Verify cancellation/reschedule updates and pending-booking reconciliation.
- [ ] Test routing, unknown identities, shared-host transfers, duplicate prevention, restarts, rate limits, and uncertain delivery.
- [ ] Obtain/activate an approved timezone reminder schedule and verify delivery.
- [ ] Obtain separate approval before delivering recordings or audio.

The owner adds representatives to channels; individual representative Discord user IDs are not required for channel-based posting. Maintain this document as implementation progresses, marking only actually verified features complete.
