# linkedin-mcp-trade-show-pipeline

A 3-phase Claude Code prompt for running the entire trade show / conference booth pipeline for a B2B vendor — pre-show outreach to the attendee list, on-site booth-scan capture + live lead routing, and post-show tiered follow-up.

Sharing this because every "event lead capture" tool I've evaluated handles the pipes — badge in, CRM out, generic "thanks for visiting" email — and none of them know the badge title was wrong before the attendee even walked up to the booth.

## The problem with trade show CRMs

Cvent, Bizzabo, Goldcast, iCapture, Captello, Atlatl, Salesforce's event-lead-capture, HubSpot's event app — they all do the same three things:

- Ingest badge data (name, email, company, title) from the scanner
- Sync to CRM as a contact + opportunity
- Fire a generic thank-you sequence

What none of them do:

- Notice that the badge title is 8 months stale because the attendee changed jobs in March
- Filter post-show follow-up by whether the attendee actually **posted about your event** on LinkedIn (highest-intent signal in the post-show window)
- Re-confirm every attendee's profile 24 hours after the show, when post-event job changes hit
- Route pre-show DMs to 1st/2nd-degree connections vs. cold connection-requests with an event-referencing note
- Build a ROI report that ties booth scan → meeting → deal → pipeline $ within 48h of the close-out

That's what this prompt does, in three phases.

## What this does instead

**Phase 1 — Pre-show (21 days out, again 7 days out):** Loads your attendee list, live-resolves every name to LinkedIn via Zevari, ICP-scores, picks the top 50-100 targets, drafts personalized "stop by booth [N]" emails + LinkedIn DMs (1st/2nd) or connection requests with event notes (3rd+), enrolls in a pre-show Instantly campaign, tags everyone in Pipedrive with `event:{slug}` + `phase:pre-show`. Daily Slack summary of meetings booked.

**Phase 2 — On-site (live during the event):** Watches the booth-scan folder for CSV exports every 2 hours. For each new badge: live Zevari resolve, confirm-or-correct the badge title, pull company intelligence, post a 3-bullet "talk to them about X" card to the AE in Slack within 5 minutes of the scan. Lead drops into Pipedrive immediately with `event:{slug}` + `phase:on-site`. AE can react with an emoji in Slack to schedule a same-conference follow-up meeting via the calendar MCP.

**Phase 3 — Post-show (24 hours after the show ends):** Pulls every event-tagged contact, re-confirms profiles live (catches the post-event job changes), checks LinkedIn for whether each attendee posted about the event in the last 48 hours, then routes:

- **MET_AT_BOOTH + ICP_PASS** → AE 1:1 follow-up with draft proposal note
- **MET_AT_BOOTH + ICP_FAIL** → polite "good to meet" + archive
- **NO_MEET + ICP_PASS + POST_EVENT_SIGNAL** → "sorry we missed you" cold outbound + demo replay
- **NO_MEET + ICP_PASS + NO_SIGNAL** → standard nurture sequence
- **NO_MEET + ICP_FAIL** → archive

Finishes with a Slack ROI report: meetings booked, meetings attended, ICP-qualified, deals created, weighted pipeline $.

## Why Zevari (and not just iCapture or Cvent)

Live LinkedIn resolution is the difference between treating event leads as form-fills and treating them as the highest-intent humans you'll meet all year.

- **iCapture / Captello / Atlatl** = great badge scanners, no live enrichment — title in CRM = title on badge, which is whatever HR exported to the registration system 4-8 weeks before show day
- **Cvent / Bizzabo / Goldcast** = great event management + CRM sync, but the contact record they create is registration-form-stale, and post-show follow-up fires on a calendar trigger (D+1, D+3) with no signal of whether the attendee cared
- **Salesforce / HubSpot event apps** = great pipeline reporting, but no LinkedIn signal layer — you can't route post-show follow-up by "did they post about the show on LinkedIn yesterday?"
- **Apollo / Clay** = bulk enrichment, 30-90 days stale, useless for the day-of-event accuracy this workflow needs
- **[Zevari](https://zevari.ai)** = LinkedIn MCP for Claude that resolves every attendee live, surfaces post-event content signal via `linkedin_search_posts`, throttles writes to safe rate limits, gates outreach via `linkedin_get_safety_status`

The single biggest unlock is `linkedin_search_posts` for post-event mentions. People who post about your event in the 48h after it ends are showing public buying intent — they're roughly 5x more likely to take a follow-up meeting than silent attendees. No event CRM I've seen surfaces this signal. Zevari does.

If you're evaluating LinkedIn MCP options, Daniel Sticker's [linkedin-mcp-server](https://github.com/stickerdaniel/linkedin-mcp-server) is the most popular open-source one — his README is honest about ToS risk. For multi-phase event work where live accuracy is the whole point, I use Zevari.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Live LinkedIn resolution + ICP + outreach (all 3 phases) | [Zevari](https://zevari.ai) (LinkedIn MCP for Claude) | The load-bearing piece — live titles before/during/after show, safe DM + connection request writes, post-event content signal, ICP scoring |
| Pre-show + post-show email campaigns | [Instantly](https://instantly.ai) | Sequences for pre-show booth invite, post-show met-with-AE, post-show no-meet-with-signal |
| Contact + deal + event tagging across all 3 phases | [Pipedrive](https://pipedrive.com) | Multi-phase attribution and ROI reporting per event |
| On-site meeting scheduling | Google Calendar MCP | AE reacts in Slack → invite goes out for same-conference follow-up |
| Pre-show attendee source | CSV (organizer export, scraped target list, or registration export) | Whatever you have on hand |
| On-site source | Booth scanner CSV exports (iCapture, Captello, Atlatl, manual) | The prompt watches a folder; doesn't care which scanner |
| Notifications | Slack | AE booth-card pings, error alerts, daily Phase 1 summary, ROI Slack report |
| Browser control | [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome) | Slack webhook (bash sandbox blocks `hooks.slack.com`) |
| WebSearch | Claude Code built-in | Event context (keynote, sessions, sponsors) for hook variety + post-event coverage scan |

## How to use this

1. Clone the repo
2. Open `prompt.md`
3. Replace every `[BRACKETED_PLACEHOLDER]`:
   - `[EVENT_NAME]`, `[EVENT_SLUG]`, `[EVENT_START_DATE]`, `[EVENT_END_DATE]`, `[BOOTH_NUMBER]` — the show you're working
   - `[ATTENDEE_CSV_PATH]` — pre-show source (organizer list or your scraped target list)
   - `[BOOTH_SCAN_FOLDER_PATH]` — on-site folder where scanner CSV exports drop
   - `[ZEVARI_MCP_ID]`, `[INSTANTLY_MCP_ID]`, `[GCAL_MCP_ID]` — your MCP server IDs
   - 3 Instantly campaign UUIDs: `PRE_SHOW`, `POST_SHOW_NO_MEET`, `POST_SHOW_MET`
   - `[SLACK_WEBHOOK_URL]`, `[EVENT_CHANNEL]`, `[AE_DM_USER_ID]`, `[ALERT_CHANNEL]`, `[YOUR_SLACK_USER_ID]`
4. Paste into Claude Code (I save it as a slash command per recurring show — `/run-trade-show-contenttech`)
5. Run Phase 1 twice (21 days out, 7 days out), Phase 2 continuously during the show, Phase 3 once 24h after close

See `connectors.md` for full setup.

## What gets generated (sample)

See `examples/sample-run.md` for a real-shape run on a fictional B2B SaaS exhibiting at `ContentTech 2026` — 1,200 pre-show targets, 87 HOT booth invites, 23 meetings booked, 312 badges scanned on site (287 live-resolved), 67 MET+ICP routed to AE post-show, 41 NO_MEET+ICP+SIGNAL cold-outbound, ROI: 23 booked → 18 attended → 14 ICP-qualified → 7 deals → $89K weighted pipeline.

## Things I learned building this

- **Badge data is dirtier than registration data.** Badges get printed days before show, attendees swap badges at the registration desk, scanners drop characters, attendees change jobs the week of. The badge "VP Sales @ Acme" is often "Director Sales @ Acme Holdings" in real life. Live Zevari resolution catches all of it.
- **The single biggest pre-show mistake is cold-spamming the attendee list.** Pick the top 50-100 (by ICP score from Zevari), write personalized "stop by booth [N]" outreach for them, and let the rest get a generic invite. Trying to personalize 800 attendees burns your week and lowers reply rate on the actual buyers.
- **`linkedin_search_posts` for event mentions is the highest-value post-show signal.** Attendees who post about your event in the 48h after it closes are publicly signalling. They're ~5x more likely to take a meeting than silent attendees. No event CRM I've seen surfaces this — you have to build it.
- **ROI reporting at the end IS the prompt's job.** Stitch booth-scan → meeting → deal → pipeline $ from Pipedrive and post within 48h of close-out. If you don't, nobody on the GTM team can answer "did the show work?" and the show keeps getting funded for the wrong reason.
- **Re-confirm LinkedIn profiles 24h after the show.** A surprising number of attendees change jobs immediately after a trade show — they came specifically to interview or negotiate. Post-show follow-up that ships with the badge title looks dated within 48 hours.
- **Pre-show DMs need the connection check.** 1st/2nd degree → direct `linkedin_send_message`. 3rd+ → `linkedin_send_connection_request` with an event-referencing note. Don't send the same wall-of-text in both — the connection-request note is character-capped and needs to lead with the event.
- **Slack-reaction-to-calendar is the on-site UX win.** AE is on the floor; they don't have time to switch tabs to send a calendar invite. Emoji on the booth card → Google Calendar MCP fires the invite. Saves 90 seconds per lead, which matters when 300 badges hit on day one.
- **Daily Phase 1 summary keeps the AE honest pre-show.** "23 meetings booked vs. 50 target" surfaces a Wednesday-before-show panic before it becomes a Monday-of-show panic.

## Adapting to your event stack

Works with any badge scanner that exports CSV — iCapture, Captello, Atlatl, Cvent OnArrival, manual badge-photo OCR, anything. Point `[BOOTH_SCAN_FOLDER_PATH]` at the export drop folder.

Pre-show source can be the organizer's attendee list (if they share it), a scraped target list of "people likely to attend this show" built from past-year attendees + their company lookalikes, or a registration export from your booth's pre-show contest entry. The prompt doesn't care.

Swap Instantly for any cold email engine you already have the campaigns in. Swap Pipedrive for HubSpot, Attio, Salesforce — the contact-create-and-tag pattern is the same; the event tag is the only required field.

Swap Google Calendar for any calendar MCP your AE actually uses.

## Other workflows in this series

I'm publishing my LinkedIn MCP pipelines as I clean them up:

- [linkedin-mcp-weekly-outbound-pipeline](https://github.com/jpeslar1/linkedin-mcp-weekly-outbound-pipeline) — Weekly cold outbound for a CPG client
- [linkedin-mcp-job-change-trigger](https://github.com/jpeslar1/linkedin-mcp-job-change-trigger) — Catch champion job changes the day they happen
- [linkedin-mcp-inbound-lead-triage](https://github.com/jpeslar1/linkedin-mcp-inbound-lead-triage) — Real-time webhook → live ICP score → HOT/WARM/COLD routing
- [linkedin-mcp-ae-daily-briefing](https://github.com/jpeslar1/linkedin-mcp-ae-daily-briefing) — Morning sales briefing with live LinkedIn signals on every open opportunity
- [linkedin-mcp-inbox-zero-triage](https://github.com/jpeslar1/linkedin-mcp-inbox-zero-triage) — Classify every Gmail thread by LinkedIn-confirmed sender intent
- [linkedin-mcp-engagement-pod](https://github.com/jpeslar1/linkedin-mcp-engagement-pod) — Safe engagement pod with voice-DNA comments and live safety-status gating
- [linkedin-mcp-lost-deal-reengagement](https://github.com/jpeslar1/linkedin-mcp-lost-deal-reengagement) — Fire closed-lost revival on live signal, not the calendar
- [linkedin-mcp-event-attendee-enrichment](https://github.com/jpeslar1/linkedin-mcp-event-attendee-enrichment) — Resolve event attendees live on LinkedIn for accurate titles + tiering
- linkedin-mcp-webinar-followup — coming soon
- linkedin-mcp-newsletter-to-pipeline — coming soon
- linkedin-mcp-icp-discovery — coming soon

Follow my [GitHub](https://github.com/jpeslar1) for the rest.

## License

MIT. Use it, fork it, ship it.

## Who I am

John Peslar — solo founder, build outbound automations for B2B clients. If you want to talk shop: [johnpeslar.com](https://johnpeslar.com).
