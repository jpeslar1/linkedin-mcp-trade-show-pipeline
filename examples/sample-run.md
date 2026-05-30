# Sample run

A real-shape run of the 3-phase pipeline (sanitized — fictional vendor, fictional show, fictional people) from a recent trade show.

## Setup

- Vendor: fictional B2B SaaS (content ops platform for brand marketing teams)
- Event: **ContentTech 2026** (fictional industry conference)
- Show dates: 2026-04-22 → 2026-04-24 (3 days)
- Booth: B42
- Slug: `contenttech-2026`
- All connectors healthy

---

## PHASE 1 — Pre-show (run 21 days out, again 7 days out)

### Source data
- `[ATTENDEE_CSV_PATH]` → scraped target list of "people likely to attend ContentTech 2026" (past-year attendees + ICP-fit lookalikes from same companies)
- 1,200 rows in
- After whitespace trim + email lowercase + suffix-normalized dedupe: **1,156 unique targets**

### Step 2 — live resolution
- Path A (LinkedIn URL provided): 412 → resolved 408
- Path B (email only → `inbox_enrich_sender`): 521 → resolved 388
- Path C (name + company → `linkedin_search_profiles`): 223 → resolved 134
- **Net resolved: 930 / 1,156 (80%)**
- 226 unresolved, mostly free-email with common-name rows. Saved to `phase1-unresolved-pending-2026-04-01.csv`.
- Rate-limit pacing: spread across 3 days of Phase 1 activity.

### Step 3 — ICP score + tier

| Tier | Score range | Count | Action |
|---|---|---|---|
| HOT_TARGET | ≥ 9 | 87 | Personalized booth-invite outreach |
| WARM_TARGET | 7-8 | 198 | Personalized booth-invite outreach |
| FILLER | 5-6 | 412 | Generic "stop by booth B42" Instantly enrollment |
| DQ | < 5 | 233 | Archive |
| Unresolved | — | 226 | `phase1-unresolved-pending` for manual pass |

Combined HOT + WARM (285) was above the 100-cap, so demoted WARM rows with `agents_icp_score` < 7.5 to FILLER. Final personalized list: **87 HOT + 13 WARM = 100 targets** for one-by-one personalization.

### Step 4 — personalized outreach (100 personalized targets)

Connection-degree split via `linkedin_get_profile`:
- 1st degree: 12 → `linkedin_send_message` direct (event-referenced, calendar link)
- 2nd degree: 31 → `linkedin_send_message` direct
- 3rd+ degree: 57 → `linkedin_send_connection_request` with event note

All writes gated by `linkedin_get_safety_status` (green throughout). Spread across the 21-day-out and 7-day-out runs to stay inside connection-request weekly caps.

Sample personalized DM (1st degree):
```
Saw you'll be at contenttech 2026 next month — we'll be at booth b42 demoing
our content ops platform. happy to swap notes if you can swing by, or grab
15 min on my calendar: [link]
```

Sample connection-request note (3rd degree):
```
Coming to contenttech 2026 — happy to swap notes at booth b42 if you'll be there
```

### Step 5 — Instantly enrollment
- 100 personalized → `PRE_SHOW` campaign with `hook` custom var
- 412 FILLER → `PRE_SHOW` campaign with generic-invite custom var
- Total Phase 1 Instantly enrollments: **512**

### Step 6 — Pipedrive
- 599 contacts created/updated (100 personalized + 412 FILLER + 87 HOT extras enriched but not yet emailed)
- All tagged `event:contenttech-2026` + `phase:pre-show` + tier tag
- Live Zevari titles written to Pipedrive (NOT the registration form titles)

### Step 7 — daily Phase 1 Slack summary (last run, 7 days out)
```
🎟  Pre-show pipeline update — ContentTech 2026

Days to show: 7
Pre-show list size: 1,200
Live-resolved: 930
HOT_TARGET (personalized): 87
WARM_TARGET (personalized): 13
FILLER (generic invite): 412
DQ archived: 233

📧 Pre-show emails enrolled: 512
💬 LinkedIn DMs sent (1st/2nd): 43
🤝 Connection requests sent (3rd+): 57
📇 Pipedrive contacts: 599

📅 Meetings booked from pre-show outreach: 23 / target 30
```

Meetings booked from pre-show personalized outreach: **23** (target was 30 — soft alert to AE, but not below the 50% threshold).

---

## PHASE 2 — On-site (during the 3-day show)

### Source data
- `[BOOTH_SCAN_FOLDER_PATH]` → iCapture exports dropped every 2 hours
- 3-day total: **312 badges scanned**

### Step 3 — live resolution per scan
- 287 / 312 live-resolved on LinkedIn = **92%**
- 25 unresolved (mostly very common names + free emails captured at the badge desk by attendees who forgot real cards)

### Title drift detected on badges
- 58 / 287 resolved had `badge_title != live_title` = **20% drift rate**
- Examples:
  - Badge: "Director, Content" — live: "VP, Content" (promoted 2 months before show)
  - Badge: "Marketing Manager" — live: "Head of Brand" (promoted 6 weeks before)
  - Badge: "Co-Founder" — live: "Co-Founder & CEO" (title formalized post-fundraise)

### Step 4 — AE booth cards
Each scan got a Slack DM to the AE within 5 minutes. Sample card:
```
🎫 Booth scan — ContentTech 2026 booth B42

Priya Nair — VP Content @ Crunch Co
⚠️ Badge said "Director, Content" — live LinkedIn title is "VP Content". Update CRM accordingly.
LinkedIn: linkedin.com/in/priya-nair-[id]

Talk to them about:
• Just announced Series B ($24M) two weeks ago — content team is doubling
• Currently using a manual workflow that doesn't scale — perfect ICP for our enterprise tier
• Posted last month about wanting "a single source of truth for brand content" — exactly our positioning

React 📅 to schedule a same-show follow-up meeting.
React 🔥 to mark as HOT (priority post-show).
React 🚫 to DQ.
```

### Step 5 — Pipedrive
- 312 contacts created/updated with `phase:on-site` + `met_at_booth:true`
- LIVE Zevari titles written (not badge titles)
- Booth note + conversation tags appended per contact

### Step 6 — Slack-reaction → calendar invite
- AE 📅 reactions across 3 days: **47** → 47 calendar invites fired via Google Calendar MCP for during-show or same-day follow-up
- AE 🔥 reactions: **31** → tagged `priority:hot_post_show` for Phase 3
- AE 🚫 reactions: **18** → archived at booth, excluded from Phase 3

### Step 7 — 2-hourly Slack pulses (day 2 sample)
```
🎟  Booth pulse — ContentTech 2026 day 2

Badges scanned (last 2h): 38
Total badges (event so far): 187
Live-resolved on LinkedIn: 172 (92%)
Title drift detected: 36 (19%)
📅 Meetings booked via Slack reaction: 28
🔥 HOT flagged for post-show: 19
🚫 DQ'd at booth: 11
```

---

## PHASE 3 — Post-show (24 hours after show close)

### Step 1 — pull event-tagged contacts
- Total contacts tagged `event:contenttech-2026`: **911** (599 from Phase 1 + 312 from Phase 2, deduped where the same person appeared in both — 0 overlap this run because Phase 2 dedupe caught it during the show)

### Step 2 — re-confirm profiles via `linkedin_get_profile`
- 911 / 911 re-confirmed
- **Post-show job changes detected: 2** (both attended the show — one took a new VP role at a competitor, one moved agency-side). Tagged `post_show_job_change:true`. AE notified — these became immediate outreach priorities.

### Step 3 — post-event content signal scan via `linkedin_search_posts`
- Filtered to each attendee's profile, last 48 hours, mentioning ContentTech / the slug / sponsors / keynote
- **84 attendees posted about the event** = 9% of the 911 → flagged `post_event_signal:true`
- 41 of those 84 were in the NO_MEET + ICP_PASS bucket (the high-leverage Phase 3 routing bucket)

### Step 4 — ICP re-score
- Some Phase 1 TIER_2 contacts re-scored TIER_1 on fresh data (4 contacts had Crunchbase fundraise updates that pushed them into ICP). Promoted to MET / NO_MEET ICP_PASS buckets.

### Step 5 — routing into 5 buckets

| Bucket | Count | Action |
|---|---|---|
| 🎯 MET + ICP_PASS | 67 | AE 1:1 follow-up with draft proposal note |
| 🤝 MET + ICP_FAIL | 23 | `POST_SHOW_MET` courtesy sequence, archive |
| 🔥 NO_MEET + ICP_PASS + SIGNAL | 41 | `POST_SHOW_NO_MEET` cold outbound, signal-quoted hook |
| 📥 NO_MEET + ICP_PASS + NO_SIGNAL | 218 | `POST_SHOW_NO_MEET` standard nurture |
| 🚫 NO_MEET + ICP_FAIL | 562 | Archive |

(67 + 23 + 41 + 218 + 562 = 911 ✓)

### Step 6 — Zevari audit list
- Saved to `event-contenttech-2026-post-show` via `targets_save` + `targets_create_list`
- All 349 non-archived contacts tagged with their routing bucket

### Step 7 — post-event ROI Slack report
```
📊 Post-show ROI — ContentTech 2026

Pre-show (Phase 1):
Pre-show list size: 1,200
Live-resolved: 930
HOT + WARM personalized outreach: 100
Meetings booked from pre-show: 23

On-site (Phase 2):
Total badges scanned: 312
Live-resolved on LinkedIn: 287 (92%)
Title drift detected on badges: 58 (20%)
Meetings booked via Slack reaction: 47
HOT flagged at booth: 31

Pipeline (Phase 3):
Total event-tagged contacts: 911
Post-show job changes detected: 2
Post-event LinkedIn signal detected: 84

Routing buckets:
🎯 MET + ICP_PASS → AE 1:1: 67
🤝 MET + ICP_FAIL → courtesy archive: 23
🔥 NO_MEET + ICP_PASS + SIGNAL → cold outbound with hook: 41
📥 NO_MEET + ICP_PASS + NO_SIGNAL → standard nurture: 218
🚫 NO_MEET + ICP_FAIL → archive: 562

Funnel:
Meetings booked (pre + on-site): 23
Meetings attended: 18
ICP-qualified post-meeting: 14
Deals created in Pipedrive: 7
Weighted pipeline $: $89,000

Zevari audit list: event-contenttech-2026-post-show
```

---

## Reflection notes

- **Badge title drift 20%** — the headline metric. 58 of 287 booth scans had a stale badge title that Zevari corrected before the AE walked over. Without this layer, 58 conversations open with "great to meet you, Director" when they're actually a VP for 2 months. Show that number to anyone questioning the Zevari spend.
- **Post-event content signal 9%** — 84 attendees posted publicly about the event in 48h. 41 of them were in NO_MEET + ICP_PASS, which became the highest-leverage post-show outbound bucket. Cvent / Bizzabo / etc. don't have access to this signal layer.
- **Pre-show resolution 80%** — strong, because the target list was scraped from past attendees who tend to have richer LinkedIn presence. For organizer-supplied lists with more free-email registrants, expect 65-75%.
- **Booked → attended drop-off 22%** (23 → 18) — within normal range. The 47 on-site Slack-reaction meetings had a higher attendance rate (since they were scheduled during the show, not 7 days out).
- **Pipeline $ vs. show cost** — $89K weighted pipeline against ~$32K show cost (booth + travel + swag for 4 people). 2.8x ratio at the pipeline stage. Conversion to closed-won will determine final ROI but the metric is in range to re-exhibit next year.
- **Phase 2 cycle time** — 2-hour cadence was fine on day 1, felt slow on day 2 afternoon when scans surged. Next show: tighten to 1h on the busiest day.
- **Total Claude active time across 3 phases:** ~3.5 hours over the 4-week window (vs. ~16 hours of AE time saved on triage, manual enrichment, and post-show list-building).
