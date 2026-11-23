---
name: neo-timezone-manual-call-assignment
description: Create timezone-specific manual-call workflows for one niche representative and enroll matching smart lists once, with duplicate checks.
tools: browseros-neo
---

# Timezone manual-call assignment

## Operating model

- Hire one representative per niche, not one representative per timezone.
- All timezone queues for a niche belong to that niche's designated representative. Verify the private assignment registry before writes; do not infer reassignment from a hiring plan.
- Representatives switch between Eastern, Central, Mountain, and Pacific queues throughout the day to reach businesses during approved local calling windows.
- Discord is intended to notify all relevant representatives when to switch queues. Queue segmentation alone does not schedule notifications, automatically switch queues, or enforce calling hours.
- Scheduled Discord notifications are a requirement, not a claimed deployed feature. Obtain approved calling windows, schedule timezone, operating days, destination server/channel, mention policy, and securely installed credentials before activation. Do not assume permission to use @everyone.
- Use IANA timezone identifiers and daylight-saving-aware calculations, not fixed UTC offsets or EST/CST/MST/PST abbreviations alone. Preserve each lead's actual timezone where verified; flag unknown or ambiguous timezone data.
- Notifications should say which queue to work, the effective time with timezone, and the next transition. Persist delivery state and deduplicate by schedule version, transition, date, and destination. Do not send stale catch-up reminders after restarts.
- Do not invent optimum calling hours. Use owner-approved windows initially and evaluate connected calls and bookings by niche and local hour.

## Browser procedure

1. Load the BrowserOS neo skill and call `mark_skill_run` with this skill name. List tabs first and work in an owned tab. Use `browser.pages.list()`, `browser.open(url)`, or `browser.page(id)` inside `run`. Inspect Contacts > Bulk Actions with `page.read()` before any write. Skip existing processing or completed batches; reconcile uncertain submissions instead of resubmitting.
2. Use `page.goto(url)`, `page.snapshot({mode: "interactive"})`, and `page.click(ref)` to inspect an approved source workflow. Inspect Contact Tag, Manual call > Advanced Settings, and Wait. Use `page.screenshot()` where custom iframe controls omit labels. Verify explicit representative assignment, trigger tag, wait duration/unit, re-entry behavior, and absence of automated calls, SMS, or email.
3. From Workflows list, choose the source row's Workflow list actions > Duplicate workflow. Fill the new niche/timezone/rep name, read it back, and click Create once. Verify one Draft with zero enrollment. If a workflow-name link opens a new tab, discover its URL through `browser.pages.list()` and navigate the owned tab to it. Do not repeat an uncertain create.
4. Open the copied Contact Tag trigger and replace its source tag with the exact unique niche/timezone/rep tag. Search, select or create the tag as appropriate, and visually verify that the old tag is not retained. Save trigger and workflow, then use `page.check(ref)` on the publish switch. Dismiss any covering alert first. Verify Published in the workflow list. Preserve and verify explicit assignment to the same niche representative.
5. Open Contacts and select the exact niche/timezone smart list. Search within More if necessary. If filling unchanged search text fails to filter, use `page.focus(ref)`, `page.press("End")`, `page.press("Backspace")`, and `page.type(lastCharacter)`. Verify list name, filters, contact count, and sample industry/timezone tags. Select the header checkbox and then Select all contacts, not merely the current page.
6. Click Trigger automation. Fill the whole form in one `act` call using `fields[]` for the unique tracking label and workflow search. Select the exact published workflow. Verify the selected workflow, representative, timezone, mode, and full contact count. Click Add to automation exactly once. Enrollment creates manual calling actions; do not start calls or add automated messages.
7. Use Check progress or the Bulk Actions page to verify each accepted submission. Report Processing versus Complete accurately. Recheck the published workflows' enrollment and assignment settings without reenrolling contacts. Do not claim every task was created merely because submission succeeded.

## Verification and safety

- This procedure was exercised through the live browser UI. It is a guided skill, not a verified unattended fast flow.
- Derive element references from fresh snapshots; never copy historical `eN` references.
- Check `matched` whenever using `page.waitForText()` or `page.waitForSelector()`. Accessibility names are not necessarily visible text; do not wait for an accessibility-only label as page text. Use a fresh snapshot or appropriate selector instead.
- The workflow iframe may omit selected tag and representative labels from its accessibility output; verify screenshots rather than guessing.
- Tag triggers apply to future tag additions. Existing tagged contacts require deliberate direct enrollment; never both re-tag and directly enroll the same cohort without checking duplicate behavior.
- Keep existing active batches, call routes, unrelated workflows, and contact ownership unchanged unless authorized.
- Keep credentials, private user/location IDs, prospect records, recordings, and browser session logs out of GitHub. Public examples must use placeholders.
- Discord scheduling must be independently implemented and delivery-tested. Saving this skill does not activate the bot.
