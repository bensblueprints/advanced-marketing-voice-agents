# Advanced Marketing Voice Agents


> Recreation blueprint for a scraped-business-to-voice-demo system. Account-specific
> IDs and deployment values below are placeholders. No Calendly API key, ElevenLabs
> API key, webhook token, raw lead list, or prospect contact data is included.
>
> **Repository state:** this README is the architecture/recreation guide. The script
> paths describe the reference implementation; they are not a claim that those source
> files have all been uploaded to this repository. A clean implementation still needs
> the components listed below, secure credentials, approved business data, and testing.

## Recreation quick start

### Prerequisites

- Python 3.12+ and SQLite; a Linux server with systemd and Nginx for production.
- An ElevenLabs account with conversational agents, webhook tools, managed secrets,
  and an explicitly approved inbound phone number (or another supported test entry).
- A Calendly account/API credential permitted to query availability and create
  invitees; a configured active event type. API access depends on account permissions.
- A voice you are licensed/authorized to use. Choose a female voice for Massage/Nails.
- Authorized business listings; GHL is optional and separate from session storage.

### Suggested clean repository layout

```text
README.md
.env.example                      # placeholders only; never actual keys
.gitignore                        # .env*, runtime DBs, logs, exports, transcripts
scripts/
  export_requested_leadripper.py   # upstream-specific catalog adapter
  export_nails_massage.py
  build_business_voice_agents.py
  provision_priority_voice_agents.py
  supervise_priority_voice_agents.py
  reconcile_voice_agents.py
  build_industry_demos.py
  attach_demo_tools.py
  attach_business_phone_lookup.py
  demo_tools_server.py
  demo_session_store.py
  demo_calendly.py
  sync_demo_routing.py
  publish_demo_branch.py
  verify_published_demo_paths.py
  export_demo_directory.py
  update_demo_return_flow.py
  audit_real_calendar_agents.py
  update_real_calendar_flow.py
tests/                            # mocked API + live read-only smoke tests
deploy/
  business-demo-tools.service
  nginx.conf.example
runtime/                          # ignored; writable databases and ledgers
```

### Environment structure — NO real credentials

```dotenv
ELEVENLABS_API_KEY=REPLACE_SECURELY
DEMO_TOOLS_TOKEN=GENERATE_A_RANDOM_SECRET_AT_LEAST_32_CHARACTERS
DEMO_SESSION_DB=/opt/business-demo-tools/sessions.sqlite
DEMO_DIRECTORY_DB=/opt/business-demo-tools/demo-directory.sqlite
INDUSTRY_DIRECTORY_DB=/opt/business-demo-tools/industry-demo-directory.sqlite
INDUSTRY_RESOURCE_DB=/opt/business-demo-tools/industry-demo-resources.sqlite
CALENDLY_API_TOKEN=YOUR_CALENDLY_API_KEY_NOT_INCLUDED
CALENDLY_EVENT_TYPE=https://api.calendly.com/event_types/YOUR_EVENT_TYPE_UUID
```

Keep actual values in a server-only environment file with permissions `0600` or a
secret manager. Configure the master ID, reusable agent IDs, female voice ID, tool
base URL and event location in your implementation. The reference scripts contain
account-specific constants that must be replaced; they are not universal IDs.
Never put secrets in README examples, shell history, frontend JavaScript, or Git.

### Build in this order

1. Implement/import a catalog adapter. Existing LeadRipper export is one option;
   another authorized CSV source can produce the same normalized listing schema.
2. Normalize listings and create the authoritative queue/key and exception reports.
3. Implement the session store and authenticated backend endpoints below. Bind
   loopback only, add HTTPS reverse proxying, test unauthorized-request rejection.
4. Configure Calendly event type and location. Verify availability using **GET only**
   before attempting any real booking. Convert all outgoing timestamps to UTC `Z`.
5. Create the host and session/availability/booking webhook tools. Pass actual system
   conversation IDs, and store authentication using ElevenLabs secret references.
6. Export the full Massage/Nails catalog with phone indexes. Create one reusable
   agent per industry; attach name/city and phone lookup tools and native host returns.
7. Add host-to-industry transfers, preserving existing prompts and tools. Verify every
   agent and route with GET readbacks. Use an approved inbound demo entry point.
8. Test lookup ambiguity, unknown listings, business-name greeting, fictional intake,
   automatic return, real identity continuity, days-then-times, and consent refusal.
9. Only with explicit authorization, create one real test appointment, GET-verify it,
   then cancel that specific test event and GET-verify cancellation.
10. Optionally enable individual-agent provisioning and trie routing in bounded
    batches. This is **not required** for reusable Massage/Nails demonstrations.
11. Add CRM tagging only after verified agent creation and unambiguous contact match.

### Calendly API request structure

Availability:

```http
GET https://api.calendly.com/event_type_available_times?event_type=URL_ENCODED_EVENT_TYPE_URI&start_time=FUTURE_UTC_Z&end_time=FUTURE_UTC_Z
Authorization: Bearer YOUR_CALENDLY_API_KEY
```

Use a future range no longer than seven days. Convert returned UTC slots to the
prospect's IANA timezone, group by local date, offer days first and times second.
For an ambiguous “next Tuesday,” confirm the explicit date before submitting.

Real booking, only after explicit consent and confirmed real identity:

```http
POST https://api.calendly.com/invitees
Authorization: Bearer YOUR_CALENDLY_API_KEY
Content-Type: application/json
```

```json
{
  "event_type": "https://api.calendly.com/event_types/YOUR_EVENT_TYPE_UUID",
  "start_time": "FUTURE_TIMESTAMP_IN_UTC_ENDING_IN_Z",
  "invitee": {
    "name": "CONFIRMED_REAL_PROSPECT_NAME",
    "email": "CONFIRMED_REAL_PROSPECT_EMAIL",
    "timezone": "America/Chicago"
  },
  "location": {"kind": "zoom_conference"}
}
```

`zoom_conference` is specific to the reference event's configured location. Other
events require their supported location configuration; do not hardcode Zoom for
every event. Match Calendly's current API schema and account permissions.

Verify successful invitee status and returned event/invitee URIs. An uncertain POST
must be recorded for reconciliation, not blindly repeated. To cancel an explicitly
authorized test, POST a reason to `/scheduled_events/EVENT_UUID/cancellation`, then
GET the event and verify `status=canceled`. Never cancel a customer's event merely
because it was found during debugging.

### Minimal deployment shape

```text
Internet HTTPS -> Nginx -> 127.0.0.1:18336 Python tool service
                                     |-> SQLite sessions/booking ledger
                                     |-> read-only listing snapshots
                                     |-> Calendly API
ElevenLabs -> authenticated HTTPS webhook tools -> same backend
```

Use `Restart=always`, a restrictive umask, a dedicated least-privilege service user,
limited writable runtime directories, regular backups and a retention policy for
prospect data. The historical reference deployment and its limitations are described
below; do not mistake the reference server paths for an installation already present
in your environment.

---


Architecture, data lineage, operating procedures, and verification boundaries.

**Documented: November 23, 2026.** Counts below are point-in-time snapshots, not promises of full coverage. The owner reported the Massage demo worked perfectly after the latest corrections. Credentials, raw lead exports, and caller contact details are intentionally excluded.

## 1. What the system does

The system turns an existing scraped-business catalog into personalized AI receptionist demonstrations, captures real prospect details separately from fictional practice-customer information, returns callers to a sales host, and offers real team appointments through Calendly.

There are **two parallel demo architectures**:

1. **Business-specific agents:** an individual ElevenLabs agent per named listing, reached through verified directory routers.
2. **Reusable industry agents:** one Massage agent and one Nails agent look up a business during the call and adopt its confirmed name/location. These do **not** require an individual agent or a published business route.

The reusable architecture is the fast path for current Massage/Nails demos. The large individual-agent rollout remains a separate background project.

```mermaid
flowchart TD
    S[Existing scraper feeds / LeadRipper consolidation] --> C[PostgreSQL LeadRipper catalog]
    C --> E[Read-only category exports]
    E --> Z[Timezone split / phone normalization / import sheets]
    Z --> Q[Authoritative business SQLite queue]
    Q --> P[Bounded ElevenLabs provisioning]
    P --> B[Individual business agents]
    B --> R[Verified hexadecimal router tree]
    Q --> I[Full Massage / Nails lookup catalog]
    CALL[Inbound demo call] --> H[Master host]
    H -->|Massage / Nails| U[Reusable industry agent]
    U --> I
    H -->|Other published business| R
    R --> B
    U -->|Practice complete| H
    B -->|Practice complete| H
    H --> D[Durable prospect session storage]
    H --> A[Live Calendly availability]
    A --> CONSENT[Confirm real identity / date / timezone / consent]
    CONSENT --> BOOK[Calendly real booking]
```

## 2. Scraping, consolidation, and source exports

### Proven upstream boundary

This workspace consumes **existing scraped records** from the live LeadRipper catalog. The export code does not itself run a new scrape. It preserves `platform`, `scraped_at`, `updated_at`, `sources`, and `additional_fields` where available.

The precise upstream crawler schedules, scraper credentials, platform-specific collection logic, and refresh guarantees are **not established by this workspace**. Do not describe exports as newly scraped or independently verified.

### Catalog/export implementation

- Server SSH alias: `<YOUR_SERVER_SSH_ALIAS>`.
- Catalog container: `leadripper-catalog-web-1`.
- Python catalog connection: `consolidation.store.connect(readonly=True)`.
- Source view: `catalog.browse_leads`.
- Existing duplicate exclusions: `catalog.duplicates`, matched by `lead_id`.
- Export transaction: read-only `REPEATABLE READ`, using a streaming cursor with batches of 5,000.
- Classification: exact lowercased category labels first; industry is a fallback only when category is blank.
- Outputs: UTF-8/BOM CSVs, ZIP archives, and manifests with matching rules and counts.
- Formula-like CSV values are escaped for spreadsheet safety.

Relevant scripts:

| Script | Responsibility |
|---|---|
| `scripts/export_requested_leadripper.py` | Existing catalog discovery and eight trade-category exports |
| `scripts/export_nails_massage.py` | Nails/Massage categories, archives, and import sheets |
| `scripts/split_leadripper_timezones.py` | Timezone-separated source exports |
| `scripts/condense_leadripper_timezones.py` | Consolidated PST/MST/CST/EST files |
| `scripts/normalize_leadripper_phones.py` | Phone normalization and exception handling |
| `scripts/fix_import_address_header.py` | Address-header compatibility remediation |

The ten prepared industries are HVAC, Electricians, Tree-Trimmers, Barber-Shops, Pressure-Washing, Plumbers, Painters, Roofers, Nail-Salons, and Massage-Shops.

Massage matching includes massage therapists, spas, foot/sports/Thai massage and service/shop labels; unrelated schools/suppliers are not intended targets. This is category selection, not verification of every business's actual services.

Source folders:

```text
LeadRipper-Lists-2026-11-22/
LeadRipper-Nails-Massage-2026-11-22/
All-10-Industries-Import-Ready/
```

The confirmed Massage source is `LeadRipper-Nails-Massage-2026-11-22/Massage-Shops.csv`. Detailed originals and four-timezone variants are preserved separately from contact-only CRM sheets. Unknown/other timezones and invalid phone originals are held in exception/source files, not fabricated.

**Listing possession does not establish permission to call, text, or email.** No scraping/export/provisioning step authorizes outbound campaigns.

## 3. Authoritative queue and business identity

Local control database: `docs/elevenlabs-business-agents.sqlite`.

Prepared-source baseline: **812,387 rows across 40 files**, with **739,675 distinct named listings** queued and **33,712 unnamed rows held**. These are different measurements; their counts should not be summed as though all represent deduplicated listings.

`businesses` stores:

- `key`: SHA-256 of a JSON array of normalized identity fields.
- `industry`.
- `record`: original prepared listing JSON.
- `status`.
- `<YOUR_AGENT_ID>`, when known.

Business-key identity fields, in order:

```text
Business Name, Street Address, City, State, Postal Code, Country, Phone
```

Whitespace/control characters are cleaned and values case-folded before hashing. This deduplicates that identity representation; it is not a universal fuzzy entity-resolution system. A changed phone/address can produce a different key.

Typical lifecycle:

```text
pending -> creating -> created
                   -> needs_verification / held uncertainty
```

Reserve a row **before** a create POST. A timeout does not prove creation failed. Keep uncertain reservations and reconcile using GET-only inventory/readback; never blindly repeat a create POST.

Other ledgers include `metadata`, `demo_flow_updates_v3`, `demo_router_agents`, `demo_routing_verified`, `demo_tool_resources`, and `ghl_tag_outbox`.

## 4. Individual business-agent provisioning

Implementation: `build_business_voice_agents.py`, `provision_priority_voice_agents.py`, `supervise_priority_voice_agents.py`, and `reconcile_voice_agents.py`.

- Initial priority: HVAC/Roofers/Plumbers, balanced across PST/MST/CST/EST.
- Supervisor control: `docs/voice-provisioning-control.json`.
- Bounded defaults: two workers, batches of 120.
- `--industry Massage-Shops` supports a separate bounded industry batch.
- Conditional `pending -> creating` updates ensure a worker submits only rows it actually claims.
- GET requests retry transient failures; uncertain create POSTs are not automatically retried.
- Readback verifies configured greeting, prompt, tools, and return behavior before success is reported.
- Windows workers use OS-owned single-instance locks. Lock contention handling defers work and rolls back failed transactions.

Each business prompt uses its actual name and available listing details. The role-play may collect fictional service/time/customer preferences, but cannot invent verified prices, hours, modalities, credentials, staff, live availability, or real reservations.

Provisioning is **not** equivalent to published routing, a completed live call, a phone assignment, or applied CRM tagging.

## 5. Native routing for individual agents

Implementation: `sync_demo_routing.py`, `publish_demo_branch.py`, and `verify_published_demo_paths.py`.

The business key is hexadecimal, so routing uses a **hexadecimal prefix trie**. A directory agent chooses the next prefix until a leaf selects the exact business key. Leaves are capped at 100 business destinations; routers also have host fallback behavior.

Publication rules:

1. Verify/fix business return configuration.
2. Build/verify leaf destinations, refreshing stale leaves when newer creates were added.
3. Verify child routers before publishing parents.
4. Read back the master route while preserving its prompt, webhook tools, and unrelated routes.
5. Record complete paths in `demo_routing_verified`.
6. Run GET-only path audits.
7. Export and deploy only audited lookup snapshots.

Internal directory transfers suppress first messages. Entering an individual business demo enables its business greeting. Returning to the host suppresses the original intake greeting.

Incremental branch publication permits ready branches to become reachable before the entire tree is complete. Direct canary routes have also been used for selected test businesses.

Export script: `export_demo_directory.py`.
Runtime export: `/opt/business-demo-tools/demo-directory.sqlite`.
Deploy through `.next`, then atomic rename. `routing_connected` identifies exported complete-path coverage, not live-call success.

## 6. Reusable Massage and Nails demos

Implementation: `build_industry_demos.py`, `demo_tools_server.py`, and `attach_business_phone_lookup.py`.

| Role | ElevenLabs agent ID |
|---|---|
| Master host | `<YOUR_AGENT_ID>` |
| Reusable Massage | `<YOUR_AGENT_ID>` |
| Reusable Nails | `<YOUR_AGENT_ID>` |

Both reusable agents use the verified female voice **Lauren — Friendly, Comforting and Soft**, voice ID `<FEMALE_VOICE_ID>`. The host voice was not changed by that update.

The existing host number verified during setup is **<YOUR_DEMO_PHONE_NUMBER>**. It was not reassigned to create this shortcut: the host received new native transfer destinations instead.

### Per-call flow

1. Host collects actual prospect identity and separately identifies the business/industry.
2. Disclose practice scheduling once and obtain agreement.
3. Transfer to the industry agent, preserving history.
4. Industry lookup searches full prepared industry records, including those with no individual agent.
5. Confirm the exact business name and city/address. Multiple matches require caller selection.
6. Greet as the confirmed business and conduct a natural receptionist practice.
7. On completion announce return and immediately call `transfer_to_agent` back to the host.
8. Host welcomes the real prospect back, asks feedback/questions, offers customization and a real team appointment.

Catalog: `docs/industry-demo-directory.sqlite`, deployed as `/opt/business-demo-tools/industry-demo-directory.sqlite`.

Coverage: **38,382 Massage listings and 39,026 Nails listings**. Exported from the authoritative queue regardless of agent-creation status. Table `listings` contains business key, industry, name, city, address, original phone, and normalized phone. Indexes cover industry/city and phone.

The server's resource ledger is separate: `/opt/business-demo-tools/industry-demo-resources.sqlite`. It reserves uncertain tool/agent creates to prevent duplicates. Do not delete a reservation unless a definitive pre-network/schema rejection or GET-only reconciliation establishes what happened.

### Lookup behavior and limits

- Name normalization handles punctuation, accents and `&`/`and`.
- Approximate/prefix matching supports spoken variants; city matching supports close spellings.
- Phone lookup strips punctuation and normalizes a leading US `1` for ten-digit matches.
- Business phone is not caller callback consent or fictional customer identity.
- Shared numbers require choosing a business/location.
- Missing matches may use a caller-confirmed name/city, explicitly without a found address.
- Lookups use the local listing catalog, **not live Google/web research**.
- Full-catalog phone lookup currently covers Massage/Nails only.
- Results are bounded; missing desired city from the first results means ask city and search again, not conclude it does not exist.

Compatibility fix: the older `business/lookup` endpoint detects names containing `nail` or `massage` and searches the industry catalog, preventing the host from rejecting a business just because an individual agent is absent. Ambiguous industry names can still require explicit industry selection/transfer.

Regression example: Top Nails and Spa, Abbeville, 202 Broadmoore Blvd. Both “Abbeville” and “Abbyville” matched in live HTTPS tests. Its listed business phone is +1 (337) 898-2662.

## 7. Runtime backend and authenticated tools

Runtime directory: `/opt/business-demo-tools/` on `<YOUR_SERVER_SSH_ALIAS>`.

- Python HTTP service: `demo_tools_server.py`.
- systemd unit: `business-demo-tools`.
- Loopback listener: `127.0.0.1:18336`.
- Nginx HTTPS endpoint: `https://YOUR_TOOLS_DOMAIN/api/demo-tools/`.
- Authentication: Bearer service token; ElevenLabs tool headers reference a managed secret.
- Secret environment file: `service.env`, mode `0600`; provider credential environment file also restricted.
- Request body bounded to 8 KiB; request logs containing caller data/auth are suppressed; responses use `Cache-Control: no-store`.

| POST endpoint suffix | Purpose |
|---|---|
| `session/get` | Fetch real prospect session by actual conversation ID |
| `session/save` | Save confirmed master intake/follow-up fields |
| `business/lookup` | Name/city lookup with industry compatibility path |
| `industry/lookup` | Explicit full Massage/Nails catalog lookup |
| `business/phone` | Full business-phone lookup in industry catalog |
| `calendly/availability` | Real team-event availability |
| `calendly/book` | Consent-gated real booking |

ElevenLabs tool identity fields use `system__conversation_id`. Do not allow the LLM to fabricate a conversation ID.

## 8. Structured prospect identity and return continuity

Implementation: `demo_session_store.py`; runtime session database selected by `DEMO_SESSION_DB` (normally `/opt/business-demo-tools/sessions.sqlite`).

Tables: `prospect_sessions`, `prospect_session_audit`, and booking state in `real_bookings`.

Allowed confirmed prospect fields: `prospect_name`, `callback`, `email`, `timezone`, `business_key`, `stage`. Saves require `confirmed=true` and source `master_intake` or `master_followup`. Audit entries track source and changed field names.

The host is instructed to read the session at entry/return and reuse existing real identity. Fictional names/phones collected during practice must not overwrite it. This is **structured durable session storage**, distinct from the ElevenLabs transcript and distinct from a confirmed GHL contact update.

A reviewed live call returned to the real prospect's original name instead of the fictional practice name. Later live calls also exercised session-save tools. Those observations prove specific paths, not universal identity correctness across every agent.

## 9. Real Calendly appointments

Implementation: `demo_calendly.py`, `attach_demo_tools.py`, `fix_host_calendar_dates.py`.

Configured event:

- **Advanced Marketing AI Receptionist Discovery Demo Call**.
- 20 minutes; configured Zoom meeting location.
- Scheduling page: https://calendly.com/YOUR_ACCOUNT/YOUR_EVENT
- Runtime environment: `CALENDLY_API_TOKEN`, `CALENDLY_EVENT_TYPE`, `DEMO_SESSION_DB`.

### Day-first booking flow

1. Establish the prospect's IANA timezone.
2. Resolve requested dates from authoritative current time, not model memory.
3. Query no more than seven days per availability window.
4. Offer up to three **available days** with weekday and month/day.
5. Once a day is selected, offer a few returned times on that day in the prospect's timezone.
6. If caller already selected “next Tuesday,” clarify the explicit intended date when ambiguous instead of restarting day selection.
7. Confirm real name/email/timezone and save them separately from practice identity.
8. Confirm full date/time/timezone and explicit consent to the real appointment/calendar notifications.
9. Recheck the selected slot; submit booking once.
10. Say “confirmed/booked” only after a successful Calendly response.

### Corrections made during live testing

- The model requested a date in 2025. Past ranges now return `past_date_range`, `server_now`, and `local_now`, explicitly identifying a date error rather than a disconnected calendar.
- Today's start is clamped into the future when the requested day began before now.
- Calendly rejected local-offset timestamps. Availability and booking now serialize UTC timestamps ending in `Z`.
- Availability responses group `available_days` and their returned times.
- Production booking now includes the configured Zoom location, matching the successful authorized API test.

### Durable booking states

`checking -> submitting -> booked`, or `needs_reconciliation` after uncertain/error outcomes. `conversation_id` is unique, preventing repeat submissions for the same call. Unavailable slots clear the pre-submission reservation. Pending results are not success and must not be blindly retried.

Current limitation: the booking adapter conservatively marks all POST exceptions uncertain; it does not yet retain full structured provider-error diagnosis or automatically reconcile every pending row. Check Calendly using GET/readback before any retry. Backend errors outside explicit validation can still surface as a generic 503.

### Verified evidence

A user-authorized direct API test created an active Calendly event, then canceled it and verified `canceled`. Event ID: `<REDACTED_TEST_EVENT_ID>`. That was a real controlled test, not a synthetic calendar response. The production payload was then aligned to the successful format.

The latest runtime audit also reports one `booked` conversation and one `needs_reconciliation` conversation. The owner subsequently reported a successful Massage demo. Do not infer from that statement that every calendar integration or every pending appointment is resolved.

## 10. Calendar behavior rollout across older agents

`audit_real_calendar_agents.py` enumerates calendar tools and their dependent agents instead of GET-scanning all business demos. It found **8,017 calendar-tool-connected agents**:

- 1 current host using the new demo Calendly backend.
- 7,993 fishing agents using fishing walkthrough tools.
- 2 restaurant agents using restaurant walkthrough tools.
- 21 older agents using direct Calendly-related tools.

`update_real_calendar_flow.py` applies additive day-then-time instructions with two workers, an OS-owned lock, prompt/tool-preservation readback, and a resumable server ledger `real-calendar-rollout.sqlite`.

These are different backend integrations. A prompt update does **not** prove the fishing/restaurant/direct integrations received the new Python backend's date serialization fix, nor that all calendars are the same owner/event type. Audit targets mean tool connectivity, not permission to turn fictional business appointments into real bookings.

Snapshot: **1,517 readback-verified prompt updates and 2 retry records**, with the remaining audited targets not yet recorded as verified at that check. This rollout was still in progress.

## 11. GHL imports and tagging

Prepared CSVs include business/contact fields, timezone and industry tags. Full import submissions were made, but import-error remediation remains incomplete.

`queue_demo_created_tags.py` creates/exports `ghl_tag_outbox` entries for created agents. Exact requested tag: **`demo created`**. It does **not** apply the tag itself.

Only tag after verified agent creation and careful GHL contact matching. Hold ambiguous matches. Outbox presence, exported rows and submitted imports are not proof of applied tags. Earlier evidence confirmed one specific applied tag; broad tag completion has not been established.

## 12. Rep dashboard: honest current boundary

`voice.YOUR_DOMAIN` resolves to the web server, has a valid HTTPS certificate, and redirects HTTP to HTTPS. At the last dashboard-specific deployment check it still served a holding page.

Local dashboard implementation files exist:

```text
scripts/voice_dashboard_server.py
scripts/voice_dashboard.html
scripts/deploy_voice_dashboard.py
```

They contain proposed sign-in, business search, admin number registration, timed activation, history, and expiry restoration. **They were not deployed/validated as a finished dashboard.** Rep accounts and rep-number mappings were not created by the confirmed work. Do not run installation blindly or claim the dashboard is live.

Proposed phone activation uses ElevenLabs phone-number PATCH and GET verification, a 15-minute interval, and restoration of the original assignment. It requires approved rep-number mapping and explicit activation. Existing production/receptionist numbers must not be reassigned implicitly. A reusable agent route added to the host is not a phone-number reassignment.

## 13. Fresh coverage snapshot

November 23, 2026 audits, taken while workers were running:

| Measurement | Count |
|---|---:|
| Individual business agents created | 29,971 |
| Return fixes recorded verified | 12,014 |
| Return fixes remaining among created | 17,957 |
| Complete routing paths recorded verified | 27,330 |
| Routing paths remaining among created | 2,641 |
| Verified directory routers | 3,406 |
| Deployed directory destinations | 29,490 |
| Deployed connected paths | 27,330 |
| Structured live-call prospect sessions | 2 |
| Synthetic prospect sessions | 1 |
| Booking ledger: booked | 1 |
| Booking ledger: needs reconciliation | 1 |

Local creation and remote export can differ because deployments are snapshots. Route verification ledgers, return-update ledgers, deployed files and live-call observations measure different things; do not collapse them into one “complete” number.

## 14. Safe operating procedures

Read-only checks:

```powershell
python scripts/audit_demo_integrations.py
python -m unittest discover -s tests -p test_demo_*.py -q
ssh -o BatchMode=yes <YOUR_SERVER_SSH_ALIAS> "python3 /opt/business-demo-tools/audit_demo_runtime_remote.py"
```

The latest local suite passed **19 tests**, covering session separation, consent/idempotency, uncertain bookings, UTC dates, phone matching, routing/path checks, directory exports, and return queue lock recovery. Unit tests do not substitute for provider readback or a live call.

Controlled operations (inspect current locks/processes first):

```powershell
# Bounded reuse of pending authoritative rows; provider key supplied securely.
python scripts/provision_priority_voice_agents.py --industry Massage-Shops --limit 40 --workers 2

# Full reusable catalog export, without creating individual agents.
python scripts/build_industry_demos.py --export docs/industry-demo-directory.sqlite

# Publish an explicitly selected ready branch, then run audit/export/deploy.
python scripts/publish_demo_branch.py --prefix <ready-hex-prefix>
```

Do not blindly rerun the original reusable provisioning script after later phone-tool/voice customizations: its exact initial-template assertions may no longer match the intentionally customized agents. Read existing resources first and patch only intended fields.

Operational safeguards:

- Keep provider secrets server-side or in ElevenLabs secret references; never commit environment files.
- Never commit raw lead databases, session databases, caller transcripts/contact details, or API responses containing auth headers.
- Use reservations/readbacks to reconcile uncertain creates; never recreate blindly.
- Preserve master prompt, routes, and unrelated tools when publishing branches.
- Read back before changing a live number; only use explicitly approved demo lines.
- Do not restart duplicate supervisors/return/routing/calendar-rollout workers.
- Do not claim CRM delivery, outbound follow-up, booking or cancellation without successful tool evidence.
- A real booking test requires explicit authorization; cancel only the authorized test event when requested.
- Do not interpret conversation-history continuity as proof that structured storage ran; check actual session tools/rows.

## 15. Known unfinished work

1. Complete individual-agent creation across all named listings.
2. Clear remaining return-fix and routing coverage backlogs, with current readbacks and exports.
3. Finish the 8,017-agent calendar presentation rollout and separately verify legacy calendar backends.
4. Reconcile the earlier pending real booking without duplicate submission.
5. Finish CRM import remediation and verified per-contact tagging.
6. Deploy and validate the actual rep dashboard; create accounts and approve number mappings.
7. Improve structured provider-error logging, clock grounding before first availability request, and pending-booking reconciliation.
8. Keep name/location/phone disambiguation and live-call regression tests as catalogs expand.

**Current proven fast path:** inbound host -> reusable Massage/Nails agent -> confirmed business lookup -> personalized practice -> automatic return -> real prospect follow-up -> consent-gated Calendly setup scheduling. This path is independent of completing hundreds of thousands of individual agents.
