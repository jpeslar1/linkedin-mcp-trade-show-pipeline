# Connectors

Every MCP server this prompt uses, with setup links and the placeholders to replace in `prompt.md`. The prompt runs in 3 phases (pre-show, on-site, post-show) and Zevari is load-bearing across all 3.

## Zevari — LinkedIn MCP for Claude (REQUIRED, all 3 phases)

**What it does:** Live LinkedIn profile lookup (`linkedin_get_profile`), name+company live search (`linkedin_search_profiles`), email-to-LinkedIn resolution (`inbox_enrich_sender`), company intelligence (`agents_company_intelligence`), person research (`agents_research_person`), ICP scoring (`agents_icp_score`), hook personalization (`agents_generate_messages` + `library_context_get` `craft-outreach`), safe LinkedIn writes (`linkedin_send_message`, `linkedin_send_connection_request`), safety gating (`linkedin_get_safety_status`), post-event content signal scan (`linkedin_search_posts`), target save (`targets_save` + `targets_create_list`).

**Why it's the core of this workflow:** Every trade-show CRM (Cvent, Bizzabo, Goldcast, iCapture, Captello, Atlatl, Salesforce event lead capture, HubSpot event app) handles the data pipes — capture the badge, get it into CRM, fire a generic thank-you. None of them resolve attendees live to LinkedIn before the show (pre-show outreach goes out with stale registration titles), during the show (badge titles are stale-on-arrival), or after the show (post-show job changes happen the week of the conference). And none of them know whether an attendee actually posted about the show on LinkedIn afterward — the highest-intent post-show signal you can get. Zevari is what wires the LinkedIn signal layer into every phase.

The biggest unlocks specifically for trade shows:
- **Live title accuracy in Phase 1 + 2** — the `agents_personalize` hook reads "Director of Marketing" while LinkedIn shows "VP Marketing" — Zevari catches the diff and updates Pipedrive
- **Pre-show DM safety gating** — `linkedin_get_safety_status` keeps Phase 1 writes inside safe rate limits; you don't burn a LinkedIn account two weeks before show
- **Post-event signal** — `linkedin_search_posts` filtered to attendee handles in the 48-hour window reveals who actually cared. No event CRM I've seen surfaces this.
- **Post-show profile re-confirm** — attendees often change jobs the week of a show. Zevari resolves again 24h after close so the follow-up uses the new title.

**Setup:** [zevari.ai](https://zevari.ai) → connect your LinkedIn account → copy the MCP server URL into your Claude Code config (`~/.claude/mcp.json` or via `claude mcp add`).

**Placeholders to replace in `prompt.md`:**
- `[ZEVARI_MCP_ID]` — your Zevari MCP server ID (looks like `mcp__a1b2c3d4-...`)

**Endpoints used (across all 3 phases):**
- `linkedin_get_profile` — live current title + company (load-bearing across all phases)
- `linkedin_search_profiles` — name+company live search for rows without a LinkedIn URL
- `inbox_enrich_sender` — email → LinkedIn URL resolution
- `linkedin_send_message` — Phase 1 pre-show DMs (1st/2nd degree) — SAFE write
- `linkedin_send_connection_request` — Phase 1 3rd+ degree outreach with event-referencing note — SAFE write
- `linkedin_get_safety_status` — MUST be called before every LinkedIn write
- `agents_research_person` — Phase 1 personalized hook angle (recent post, promotion, mutual)
- `agents_company_intelligence` — Phase 1 + 2 company context for hooks + booth conversation bullets
- `linkedin_search_posts` — Phase 3 post-event content signal scan
- `profile_get` — load current ICP rules
- `agents_icp_score` — score attendees across all 3 phases
- `agents_generate_messages` + `library_context_get` (`craft-outreach`) — message generation per phase
- `targets_save` + `targets_create_list` — Phase 3 audit list `event-{slug}-post-show`
- `targets_archive` — archive DQs

## Pipedrive — Contacts + deals + event tagging (REQUIRED, all 3 phases)

**What it does:** Stores every contact across all 3 phases with multi-tag schema (`event:{slug}` + `phase:pre-show` / `phase:on-site` + `met_at_booth:true/false` + tier + post-show routing bucket) so the entire pipeline is queryable and the post-show ROI report can stitch booth-scan → meeting → deal → pipeline $.

**Why it's in this workflow:** Multi-phase event attribution. Without per-event + per-phase tagging, you can't tell whether the show worked, which phase generated pipeline, or which routing bucket converted best. The tag schema is the receipt.

**Setup:** Any Pipedrive MCP server. I use `mcp__pipedrive__` from the community list.

**Tag schema used:**
- `event:[EVENT_SLUG]` — applied in every phase
- `phase:pre-show` — Phase 1
- `phase:on-site` — Phase 2
- `tier:hot` / `tier:warm` / `tier:filler` — Phase 1 tiering
- `met_at_booth:true` (custom field) — Phase 2 booth scan
- `priority:hot_post_show` — Phase 2 AE 🔥 reaction
- `post_show:ae_1on1` / `post_show:archive` / `post_show:miss_with_signal` / `post_show:miss_no_signal` — Phase 3 routing
- `post_show_job_change:true` — Phase 3 detected change

**Endpoints used:**
- `search_contacts` / `search_organizations` — dedupe before creating
- `add_person` / `update_person` — contact create/update with LIVE Zevari title
- `add_organization` — create org if missing
- `add_note` — log phase + booth + live title diff per phase
- `add_tag` / `update_person_tags` — apply tag schema above
- `add_deal` — Phase 3 hot leads
- `search_persons` (by tag) — Phase 3 pulls all event-tagged contacts in one query

## Instantly — Pre-show + post-show campaigns (REQUIRED)

**What it does:** Enrolls attendees in 3 distinct campaigns across the trade show lifecycle:
1. `PRE_SHOW` — booth invite drip (D-21 personalized invite, D-7 reminder, D-1 "see you tomorrow at booth N")
2. `POST_SHOW_NO_MEET` — "sorry we missed you" sequence with signal/no-signal variants gated by custom var
3. `POST_SHOW_MET` — short courteous follow-up for MET + ICP_FAIL bucket

**Why it's in this workflow:** Sending logic and warmup pool already configured per campaign. The prompt enrolls and passes custom variables; it doesn't manage send-side logic.

**Setup:** [Instantly MCP docs](https://developer.instantly.ai/mcp) — install the MCP, paste your API key. Build the 3 campaigns once per recurring show series; the prompt reuses them across years.

**Placeholders:**
- `[INSTANTLY_MCP_ID]` — your Instantly MCP server ID
- 3 campaign UUIDs in the prompt: `PRE_SHOW`, `POST_SHOW_NO_MEET`, `POST_SHOW_MET`

**Endpoints used:**
- `list_leads` — dedupe check
- `add_leads_to_campaign_or_list_bulk` — enrollment with custom variables (`event_name`, `event_start_date`, `booth_number`, `tier`, `hook`, `signal`)

## Google Calendar MCP — On-site meeting scheduling (Phase 2 only)

**What it does:** When the AE reacts 📅 on a Slack booth card, the prompt creates a 30-minute meeting on the AE's calendar during the show window and invites the attendee's email.

**Why it's in this workflow:** AE is on the floor — they don't have time to switch tabs. Emoji-react in Slack → calendar invite fires. Saves 90 seconds per lead. Adds up fast.

**Setup:** Any Google Calendar MCP. Authenticate to the AE's calendar (with their explicit consent) so the prompt can write events.

**Placeholders:**
- `[GCAL_MCP_ID]` — your calendar MCP server ID

**Endpoints used:**
- `create_event` — booth follow-up meeting
- `list_events` — find next available slot in show window

## Claude in Chrome — Slack webhook (all 3 phases)

**What it does:** Posts to `hooks.slack.com` (the bash sandbox blocks this URL — Chrome is the reliable path). Used for the Phase 1 daily summary, the Phase 2 booth cards + 2-hourly pulse, the Phase 3 ROI report, and all error/approval alerts.

**Setup:** [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome).

**Tools used:**
- `mcp__Claude_in_Chrome__tabs_context_mcp` — open/find a tab
- `mcp__Claude_in_Chrome__javascript_tool` — fire the Slack webhook

**Placeholders:**
- `[SLACK_WEBHOOK_URL]` — your incoming webhook
- `[EVENT_CHANNEL]` — Slack channel for daily summaries + ROI report
- `[AE_DM_USER_ID]` — AE's Slack member ID (`U...`) — receives Phase 2 booth cards as DMs
- `[ALERT_CHANNEL]` — Slack channel where errors go
- `[YOUR_SLACK_USER_ID]` — your Slack member ID (`U...`) so the bot @-mentions you

## Source files — paths the prompt reads (REQUIRED)

These aren't MCP connectors but the prompt reads them directly from disk.

### Pre-show source (Phase 1)
- `[ATTENDEE_CSV_PATH]` — absolute path to a CSV of pre-show targets. Source can be the organizer's official attendee list, your scraped "likely-to-attend" list (past-year attendees + lookalikes), or pre-show contest entries from your booth's marketing page.

### On-site source (Phase 2)
- `[BOOTH_SCAN_FOLDER_PATH]` — absolute path to a folder where your booth scanner drops CSV exports every 2 hours during the show. Works with iCapture, Captello, Atlatl, Cvent OnArrival, or any scanner that exports CSV to a synced folder. The prompt watches this folder and processes new files.

## WebSearch

Built into Claude Code. Used for:
- Phase 1: pulling the event agenda (keynote, sessions, sponsors) for richer hooks
- Phase 1: looking up which vendors are sponsoring sessions an attendee might care about
- Phase 3: scanning event coverage post-close (industry press, recap blog posts) to enrich the AE 1:1 follow-up notes

No placeholder — just available.

## Event placeholders (set once per show)

These aren't connectors but they need to be set in `prompt.md`:

- `[EVENT_NAME]` — human-readable event name (e.g. `ContentTech 2026`)
- `[EVENT_SLUG]` — lowercase hyphen-separated (e.g. `contenttech-2026`). Used in Zevari list names, Pipedrive tags, Instantly custom vars.
- `[EVENT_START_DATE]` — `YYYY-MM-DD` first day of the show
- `[EVENT_END_DATE]` — `YYYY-MM-DD` last day of the show
- `[BOOTH_NUMBER]` — your booth number (e.g. `B42`) — referenced in every outreach message
