# Trade Show Pipeline — 3-Phase Claude Code Prompt

A Claude Code prompt that runs the full trade show / conference booth pipeline for a B2B vendor exhibiting at `[EVENT_NAME]` (`[EVENT_START_DATE]` → `[EVENT_END_DATE]`, booth `[BOOTH_NUMBER]`, slug `[EVENT_SLUG]`).

This prompt has three distinct phases, run on different schedules. Read the phase header carefully and run only the phase that matches the current date relative to the show.

- **Phase 1 — Pre-show:** Run 21 days out, then again 7 days out
- **Phase 2 — On-site:** Run continuously during the show, processing the booth-scan folder every 2 hours
- **Phase 3 — Post-show:** Run once, 24 hours after `[EVENT_END_DATE]`

The reason this prompt exists: every "trade show CRM" (Cvent, Bizzabo, Goldcast, iCapture, Captello, Atlatl, Salesforce event lead capture, HubSpot event app) handles badge → CRM → generic thank-you. None of them know the badge title is stale, route post-show follow-up by post-event LinkedIn content signal, or re-confirm profiles after the show closes. Zevari does. This prompt wires it together.

---

## SAFETY + APPROVAL NOTIFICATIONS

If anything stops a phase or needs approval, send a notification to `[ALERT_CHANNEL]`. The bash sandbox blocks `hooks.slack.com` — use Chrome:

1. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true`
2. If the tab is on a `chrome://` URL, navigate to `https://google.com` first
3. Run via `mcp__Claude_in_Chrome__javascript_tool`:

```js
fetch('[SLACK_WEBHOOK_URL]', {
  method: 'POST',
  mode: 'no-cors',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    text: "<@[YOUR_SLACK_USER_ID]> ⚠️ *Trade Show Pipeline — Action Needed* — [EVENT_NAME]\n\n<error or approval ask>"
  })
}).then(r => ({status: r.status, type: r.type})).catch(e => ({error: e.message}))
```

`{status: 0, type: "opaque"}` = success.

Before any LinkedIn write (Phase 1 DM/connection request, etc.), call `linkedin_get_safety_status`. If the status is anything other than green/safe, pause writes and Slack-alert. Do NOT push through warnings.

---

## CONNECTORS

- **Zevari (LinkedIn MCP):** `mcp__[ZEVARI_MCP_ID]` — live profile resolution, name+company search, ICP scoring, hook personalization, safe DM + connection request writes, post-event content signal via `linkedin_search_posts`
- **Pipedrive:** `mcp__pipedrive__` — contacts + deals + event tagging across all 3 phases
- **Instantly:** `mcp__[INSTANTLY_MCP_ID]` — pre-show + post-show campaigns (3 campaign IDs: `PRE_SHOW`, `POST_SHOW_NO_MEET`, `POST_SHOW_MET`)
- **Google Calendar:** `mcp__[GCAL_MCP_ID]` — on-site meeting scheduling (Phase 2)
- **Chrome:** `mcp__Claude_in_Chrome__` — Slack webhook
- **WebSearch:** event context (keynote, sessions, sponsors) + post-event coverage scan

See `connectors.md`.

---

# PHASE 1 — PRE-SHOW

**Schedule:** Run twice — 21 days before `[EVENT_START_DATE]`, and again 7 days before.

**Goal:** Live-resolve every attendee on the pre-show list, ICP-score, pick the top 50-100 personalized targets, send "stop by booth [BOOTH_NUMBER]" outreach via email + LinkedIn, enroll in the pre-show drip, log to Pipedrive, daily Slack summary of meetings booked vs. target.

## Phase 1 — Step 1: load attendee list

Read `[ATTENDEE_CSV_PATH]`. This may be:

- Organizer's official attendee export (best case)
- Your scraped "likely-to-attend" target list (built from past-year attendees + their lookalikes)
- Pre-show contest entry list from your booth's marketing page

Normalize the same way you'd normalize any registration data:
- Trim whitespace on name, company, email
- Lowercase emails
- Strip company suffixes (`, Inc.`, ` Inc`, ` LLC`, etc.) for matching only — keep the original string
- Dedupe by lowercase-email, then by (first+last+normalized-company)

Log raw row count → normalized count. Big delta = bad source data; Slack-alert if delta > 15%.

## Phase 1 — Step 2: live resolve every attendee

For each normalized row, pick the resolution path based on what data the row has. In priority order:

- **LinkedIn URL provided** → `linkedin_get_profile` directly
- **Email only** → `inbox_enrich_sender` → if it returns a profile, follow up with `linkedin_get_profile`
- **Name + company only** → `linkedin_search_profiles` with name + company-as-keyword filter; if exactly 1 confident match → `linkedin_get_profile`; if 2-3 candidates → pick the one whose current company string matches best, log the decision; if 0 or >3 ambiguous → log unresolved, skip

Respect LinkedIn's ~40 lookups/hour safe ceiling. For lists > 40, batch with 60-min pauses or split across days. If you hit the cap, save the remainder to `phase1-unresolved-pending-[YYYY-MM-DD].csv` and Slack alert — do NOT silently use the registration title.

## Phase 1 — Step 3: ICP score + tier

Call `profile_get` to load active ICP rules. Then `agents_icp_score` for every resolved attendee using company intelligence + live title.

If `agents_icp_score` returns `server_side_ai_unavailable`, apply ICP criteria manually from `profile_get`. Do not pause.

Bucket into tiers:

- **HOT_TARGET** (ICP score ≥ 9 OR perfect-fit named account) → personalized outreach (Step 4)
- **WARM_TARGET** (ICP score 7-8) → personalized outreach (Step 4)
- **FILLER** (ICP score 5-6) → generic "stop by booth [BOOTH_NUMBER]" template, enrolled in Instantly only
- **DQ** (< 5 or hard disqualifier) → archive, no outreach

If HOT + WARM combined exceeds ~100 attendees, cap it. Trying to personalize 800 attendees burns the week and degrades reply rate on the actual buyers. Pick the top 100 by ICP score, demote the rest to FILLER.

## Phase 1 — Step 4: personalized outreach (HOT + WARM only)

For each HOT_TARGET and WARM_TARGET:

1. Call `agents_research_person` for the hook angle (recent post, promotion, content, mutual connection)
2. Call `library_context_get` with `skill_slug='craft-outreach'` — read the formatting rules before writing
3. Draft a personalized "are you going to `[EVENT_NAME]`? Come by booth `[BOOTH_NUMBER]`" message

### Email draft rules
- First 7 words must reference the event by name: `"Saw you'll be at [event]"` or `"Caught your name on the [event] list"`
- Lowercase everything after the first letter (reads like a casual text)
- One-line value/booth pitch, then a calendar link
- No company-name pitch in the hook — that's what the booth conversation is for

### LinkedIn outreach path (check connection degree first via `linkedin_get_profile` → `connection_degree`)
- **1st degree** → direct `linkedin_send_message`, short note + calendar link, event-referenced first line
- **2nd degree** → direct `linkedin_send_message` (Zevari handles 2nd-degree DMs safely), same format as 1st-degree
- **3rd+ degree** → `linkedin_send_connection_request` with a note that fits the character cap; lead with `"Coming to [event] — happy to swap notes at booth [BOOTH_NUMBER] if you'll be there"`. No pitch, no calendar link in a connection note.

Every write must follow `linkedin_get_safety_status` check from the SAFETY section above.

## Phase 1 — Step 5: enroll in pre-show Instantly campaign

Call `add_leads_to_campaign_or_list_bulk` to enroll HOT + WARM + FILLER (all 3 tiers — different sequence variants are gated inside Instantly by custom variable, NOT here) in `PRE_SHOW` campaign.

Custom variables:
- `event_name`, `event_start_date`, `booth_number`, `tier`, `hook` (HOT/WARM only)

Set `skip_if_in_campaign: true`.

## Phase 1 — Step 6: Pipedrive contacts + event tag

For HOT + WARM + FILLER (not DQ):

1. Search Pipedrive (`search_contacts` by email, then by name)
2. Update existing contact with LIVE Zevari title + company OR create new
3. Add tags: `event:[EVENT_SLUG]` + `phase:pre-show` + tier tag (`tier:hot` / `tier:warm` / `tier:filler`)
4. Add note: `"Pre-show outreach for [EVENT_NAME] — booth [BOOTH_NUMBER]. Tier: [tier]. Live title: [live_title] @ [live_company]"`

## Phase 1 — Step 7: daily Slack summary

After each Phase 1 run (the 21-day-out run, and the 7-day-out run), post to `[EVENT_CHANNEL]` via Chrome `javascript_tool`:

```
<@[YOUR_SLACK_USER_ID]> 🎟  *Pre-show pipeline update — [EVENT_NAME]*

Days to show: [N]
Pre-show list size: [N]
Live-resolved: [N]
HOT_TARGET (personalized): [N]
WARM_TARGET (personalized): [N]
FILLER (generic invite): [N]
DQ archived: [N]

📧 Pre-show emails enrolled: [N]
💬 LinkedIn DMs sent (1st/2nd): [N]
🤝 Connection requests sent (3rd+): [N]
📇 Pipedrive contacts: [N]

📅 Meetings booked from pre-show outreach: [N] / target [N]
```

If `meetings_booked / target < 50%` with 7 days to go, Slack-alert the AE.

---

# PHASE 2 — ON-SITE

**Schedule:** Run continuously during `[EVENT_START_DATE]` → `[EVENT_END_DATE]`. Process the booth-scan folder every 2 hours.

**Goal:** Every new booth scan gets live-resolved on LinkedIn, badge-title-vs-LinkedIn-title diff checked, a 3-bullet "talk to them about X" Slack card delivered to the AE within 5 minutes, and Pipedrive updated immediately with `phase:on-site` + booth conversation context. AE can react to schedule a same-conference follow-up via the calendar MCP.

## Phase 2 — Step 1: watch the booth-scan folder

Every 2 hours, list `[BOOTH_SCAN_FOLDER_PATH]` for new CSV exports. The scanner (iCapture, Captello, Atlatl, manual) drops CSVs with at least: name, email, company, badge_title, scanned_at, optional booth_notes / conversation_tags.

Track which files have already been processed via a local `processed-scans.json` index. Only process new files.

## Phase 2 — Step 2: ingest + dedupe within the show

For each new scan row:
- Normalize the same way as Phase 1
- Check Pipedrive for an existing contact with this email AND `event:[EVENT_SLUG]` tag — if found, this is a re-scan (same person scanned twice during the show), update the existing contact's note instead of creating a duplicate
- If not found, this is a new on-site lead

## Phase 2 — Step 3: live resolve via Zevari

Call `linkedin_get_profile` (if a LinkedIn URL or email resolves via `inbox_enrich_sender`), or `linkedin_search_profiles` for name + company.

**Capture the badge_title vs live_title diff explicitly.** Badge titles are notoriously stale — printed days before show, often from registration data that's 4-8 weeks old. If `badge_title != live_title`, flag it in the AE card as `title_drift: true` so the AE doesn't open the conversation with a wrong title.

Then call `agents_company_intelligence` on the live confirmed company for booth-conversation talking points (recent news, fundraise, hiring, product launches).

## Phase 2 — Step 4: AE booth card to Slack

Within 5 minutes of scan ingestion, DM the AE `[AE_DM_USER_ID]` via Chrome `javascript_tool`:

```
🎫 *Booth scan — [EVENT_NAME] booth [BOOTH_NUMBER]*

[Name] — [LIVE title] @ [LIVE company]
[if title_drift: "⚠️ Badge said '[badge_title]' — live LinkedIn title is '[live_title]'. Update CRM accordingly."]
LinkedIn: [URL]

Talk to them about:
• [bullet 1 from agents_company_intelligence — recent news / fundraise / launch]
• [bullet 2 — ICP fit angle, why they're at this show]
• [bullet 3 — mutual connection or recent post if `agents_research_person` returned one]

React 📅 to schedule a same-show follow-up meeting.
React 🔥 to mark as HOT (priority post-show).
React 🚫 to DQ.
```

## Phase 2 — Step 5: Pipedrive immediately

Create or update the contact with:
- LIVE Zevari title + company
- Tags: `event:[EVENT_SLUG]` + `phase:on-site` + `met_at_booth:true`
- Note: `"Scanned at booth [BOOTH_NUMBER] on [scanned_at]. Conversation: [booth_notes if present]. Live title: [live_title] @ [live_company]. Badge title: [badge_title]. Title drift: [yes/no]."`
- Custom field `met_at_booth = true` (Phase 3 uses this to route follow-up)

## Phase 2 — Step 6: Slack-reaction → calendar invite

Watch `[AE_DM_USER_ID]`'s reactions on each booth card. If 📅 reaction:

- Call the Google Calendar MCP to create an event for the next available slot during the show OR end-of-day on a show day
- 30-minute meeting at the booth (or hotel coffee shop if booth is closed)
- Invite the attendee's email + the AE
- Subject: `"[live_company] x us — [EVENT_NAME] follow-up at booth [BOOTH_NUMBER]"`
- Reply in the Slack DM thread with `"📅 invite sent for [time]"`

If 🔥 reaction → update Pipedrive with `priority:hot_post_show` tag for Phase 3 to use.
If 🚫 reaction → update Pipedrive contact, archive in Zevari, exclude from Phase 3.

## Phase 2 — Step 7: 2-hourly Slack pulse

Every 2 hours during show, summary to `[EVENT_CHANNEL]`:

```
🎟  *Booth pulse — [EVENT_NAME] day [N]*

Badges scanned (last 2h): [N]
Total badges (event so far): [N]
Live-resolved on LinkedIn: [N] ([%])
Title drift detected: [N] ([%])
📅 Meetings booked via Slack reaction: [N]
🔥 HOT flagged for post-show: [N]
🚫 DQ'd at booth: [N]
```

---

# PHASE 3 — POST-SHOW

**Schedule:** Run once, exactly 24 hours after `[EVENT_END_DATE]`.

**Goal:** Re-confirm every event-tagged contact's LinkedIn profile, check for post-event content signal, route into 5 tier buckets, enroll the right ones in the right post-show Instantly campaigns, save audit lists, generate the post-event ROI Slack report.

## Phase 3 — Step 1: pull all event-tagged Pipedrive contacts

Search Pipedrive for all contacts tagged `event:[EVENT_SLUG]` (this catches both Phase 1 pre-show and Phase 2 on-site). Pull:
- Contact ID, email, name, current Pipedrive title/company
- Tags (so you know `phase:pre-show` vs. `phase:on-site` vs. both)
- Custom field `met_at_booth` (true/false)
- Existing Pipedrive notes

## Phase 3 — Step 2: re-confirm LinkedIn profiles

For each contact, call `linkedin_get_profile` again. Titles may have shifted in the 48-72h post-show window — surprising number of attendees change jobs right after a show (they came specifically to interview or negotiate, and the move closes that week).

If title or company changed since Phase 1/Phase 2 capture, update Pipedrive with the new live data and tag `post_show_job_change:true`. Slack-alert these — they're often the highest-intent signal of all (they just changed jobs at a show YOU exhibited at).

Respect the 40/hour rate cap. For >40 contacts, batch across an evening.

## Phase 3 — Step 3: post-event content signal scan

For each contact, call `linkedin_search_posts` filtered to the attendee's profile, posted in the last 48 hours.

If they posted anything that mentions `[EVENT_NAME]`, the slug, the keynote, a sponsor, a session, or even a generic "great show" reference → flag `post_event_signal:true` on that contact.

People who post about your event publicly are the highest-intent post-show signal you'll get. Don't skip this step even if it adds 30 minutes to Phase 3 — it's the difference between treating post-show as a calendar trigger and treating it as an intent signal.

## Phase 3 — Step 4: ICP re-score (with fresh data)

Call `agents_icp_score` again using the re-confirmed title + company + any new content signal. Some attendees who scored TIER_2 on registration data score TIER_1 post-show because their live current company is bigger, better-funded, or in a hotter ICP segment than the registration form showed.

Mark each contact with current `icp_pass` (true/false) based on profile_get rules.

## Phase 3 — Step 5: route into 5 buckets

Combine `met_at_booth` + `icp_pass` + `post_event_signal` into 5 buckets:

- **MET_AT_BOOTH + ICP_PASS** → **AE 1:1 follow-up**. Draft a personalized proposal note via `agents_generate_messages`. Tag in Pipedrive `post_show:ae_1on1`. Slack-DM the AE with the contact + draft note. Hot lead.
- **MET_AT_BOOTH + ICP_FAIL** → **polite "good to meet" + archive**. Enroll in `POST_SHOW_MET` (the short courteous variant). Tag `post_show:archive`. Archive in Zevari.
- **NO_MEET + ICP_PASS + POST_EVENT_SIGNAL** → **"sorry we missed you" cold outbound + demo replay**. Personalized hook quoting their post-event LinkedIn content. Enroll in `POST_SHOW_NO_MEET` with `signal:true` custom var. Tag `post_show:miss_with_signal`. Highest-leverage bucket of the 5.
- **NO_MEET + ICP_PASS + NO_SIGNAL** → **standard event nurture**. Enroll in `POST_SHOW_NO_MEET` with `signal:false` custom var. Tag `post_show:miss_no_signal`.
- **NO_MEET + ICP_FAIL** → **archive**. Archive in Zevari, no outreach.

## Phase 3 — Step 6: Zevari audit list

Save all 5 buckets (except final archives) to a Zevari list `event-[EVENT_SLUG]-post-show` via `targets_save` + `targets_create_list`. Tag each target with the bucket so the post-show data is queryable for 6-month attribution.

## Phase 3 — Step 7: post-event ROI Slack report

Stitch numbers from Pipedrive + Phase 1 daily summaries + Phase 2 pulses + Phase 3 routing. Post to `[EVENT_CHANNEL]` via Chrome:

```
<@[YOUR_SLACK_USER_ID]> 📊 *Post-show ROI — [EVENT_NAME]*

**Pre-show (Phase 1):**
Pre-show list size: [N]
Live-resolved: [N]
HOT + WARM personalized outreach: [N]
Meetings booked from pre-show: [N]

**On-site (Phase 2):**
Total badges scanned: [N]
Live-resolved on LinkedIn: [N] ([%])
Title drift detected on badges: [N] ([%])
Meetings booked via Slack reaction: [N]
HOT flagged at booth: [N]

**Pipeline (Phase 3):**
Total event-tagged contacts: [N]
Post-show job changes detected: [N]
Post-event LinkedIn signal detected: [N]

Routing buckets:
🎯 MET + ICP_PASS → AE 1:1: [N]
🤝 MET + ICP_FAIL → courtesy archive: [N]
🔥 NO_MEET + ICP_PASS + SIGNAL → cold outbound with hook: [N]
📥 NO_MEET + ICP_PASS + NO_SIGNAL → standard nurture: [N]
🚫 NO_MEET + ICP_FAIL → archive: [N]

**Funnel:**
Meetings booked (pre + on-site): [N]
Meetings attended: [N]
ICP-qualified post-meeting: [N]
Deals created in Pipedrive: [N]
Weighted pipeline $: $[N]

Zevari audit list: event-[EVENT_SLUG]-post-show
```

---

## REFLECTION (after Phase 3)

Briefly note:

- **Badge title drift %** — the receipt for why this workflow exists. If > 15%, surface to marketing.
- **Pre-show resolution rate** — if < 80%, source data was too thin; ask organizer for a richer export next year or invest in pre-show scraping.
- **Post-event signal rate** — what % of NO_MEET ICP_PASS attendees posted about the event? If high, the show has strong content energy and is worth re-exhibiting. If near 0, audience isn't LinkedIn-active.
- **Booked → attended drop-off** — if > 40%, AE follow-up the morning of meetings is weak. Recommend day-of confirmation DMs next show.
- **Pipeline $ vs. show cost** — Pipedrive weighted pipeline / show total cost (booth + travel + swag). The number that gets the next budget approved.
- **Phase 2 cycle time** — was the 2-hour cadence fast enough? If AE complained leads were stale, tighten to 1h next show.

---

## Schedule recap

| Phase | When to run | Cadence |
|---|---|---|
| 1 — Pre-show | 21 days out, then 7 days out from `[EVENT_START_DATE]` | 2 runs total |
| 2 — On-site | `[EVENT_START_DATE]` → `[EVENT_END_DATE]` | Every 2 hours during show floor hours |
| 3 — Post-show | 24 hours after `[EVENT_END_DATE]` | 1 run |

Save the whole 3-phase prompt as a Claude Code slash command per recurring show series, e.g. `/run-trade-show-contenttech`, with the placeholders pre-filled for that show. Phase header tells Claude which section to run based on the current date relative to `[EVENT_START_DATE]` / `[EVENT_END_DATE]`.
