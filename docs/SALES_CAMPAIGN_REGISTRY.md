# Sales campaign and rep registry

Updated November 23, 2026. Owner-confirmed assignments; no prospect data or credentials belong in this public document.

## Current assignments

| Rep | Campaigns | Inbound demo line | Current implementation |
|---|---|---|---|
| Grace | Heating & Air Conditioning (HVAC), Plumbers, Roofers | +1 582-206-5452 | Contractor entry line, shared demo host |
| Anna | Nail Salons, Massage Parlors | +1 903-485-7722 | Industry entry line, shared demo host and reusable industry demos |

These are two rep lines, not three separate contractor campaign lines. Grace currently covers all three contractor campaigns; individual campaign attribution must come from the confirmed business industry or a future campaign-specific entry point. Do not guess an industry when no business was identified.

The two numbers currently share the master sales/booking agent. They are not dedicated rep-specific cloned agents yet. Attribute calls from the called number captured in verified provider metadata, not agent ID alone. Preserve attribution when a call transfers to a reusable demo agent or directory agent.

## Adding reps

1. Obtain owner-approved rep name, campaign(s), notification destination, and phone assignment.
2. Inspect existing numbers; obtain approval before purchase or reassignment.
3. Clone/configure a dedicated sales host when isolation is needed. Preserve calendar, session, lookup, consent, human-transfer and return behavior.
4. Explicitly solve return routing: current business/reusable agents return to the existing host. A cloned host alone does not make return destinations rep-specific. Preserve original entry-number/rep attribution throughout the call.
5. Assign and GET-verify the approved number; test inbound demo, return, booking, human handoff, and unanswered handoff behavior.
6. Add the approved mapping here and in a private runtime registry. Never commit a bot token, personal transfer number, lead identity, transcript or recording URL here.

## Call reporting design — not deployed

Every inbound call must be durably logged, including calls that end before intake, practice-only calls, abandoned bookings, failed bookings, and successful bookings.

Recommended event model:

- `call.started`: when a verified provider event or reconciled call inventory establishes the call exists.
- `call.completed`: final call state, duration, disposition, rep, called line and campaign if known.
- `booking.confirmed`: only after successful Calendly creation, including event identity and explicit booked time/timezone.
- `booking.pending` / `booking.failed`: distinct from confirmed; no assumed appointment.
- `transfer.attempted` / `transfer.failed` / `transfer.connected`: only according to actual provider evidence.

Private record fields:

```text
provider_conversation_id / call_id
original_called_number, rep_id, campaign_id (nullable), attribution_source
call_started_at_utc, call_ended_at_utc, duration_seconds, disposition
confirmed_prospect_name, confirmed_callback, confirmed_email (nullable)
business_key, business_name, business_city, industry (nullable)
demo_agent_id, demo_completed, return_to_host_observed
booking_state, calendly_event_uri, appointment_start_utc, appointment_timezone
human_transfer_state, caller_stated_objections
recording_reference / transcript_reference (private, retention-controlled)
notification_state, provider_event_id, retry_count
```

Unknown values remain unknown. Fictional practice contact details must not be used as lead identity. Call arrival time and appointment time are separate fields. A final transcript is not available at call start.

## Discord integration requirements

No Discord credentials or active Discord notification integration were located in the demo service's environment or the reviewed workspace at this check. This does not prove an existing bot does not exist in another service/account.

Before creating a bot, inspect the owner's existing Discord application/server configuration. A bot/application ID alone does not authorize posting: use a securely installed bot token or approved channel webhook, the destination server/channel IDs, and narrowly scoped permissions.

Implement signed/provider-authenticated inbound events, a durable event/outbox store, idempotent processing, retry/backoff, and periodic read-only conversation reconciliation. Do not assume the provider emits a real-time call-start event until its event support is verified; post-call webhooks alone cannot prove real-time arrival reporting.

Discord messages should identify rep/campaign, business demoed, confirmed lead identity, call timestamp/duration/disposition, and confirmed appointment date/time/timezone when booked. Use restricted channels and minimize personal data. Do not post recordings or full transcripts publicly. Receiving audio/recordings requires separately verifying provider recording settings, access controls, retention and applicable caller notice/consent requirements; an activity notification is not the same as recording audio.

## Dashboard status

`voice.advancedmarketing.co` currently points at a static HTTPS holding page. Server inspection confirmed dashboard application files/database absent and `voice-dashboard` inactive. Local prototype files are not a deployed dashboard.

Deployment still needs security review/testing, secure account onboarding, rep mappings, service installation and reverse-proxy cutover. Do not publish or share default passwords. No dashboard completion or Discord delivery is claimed by this registry.
