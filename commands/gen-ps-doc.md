---
description: Generate a Pre-Sales → PS Knowledge Transfer Google Doc for any account (fetches live data from Glean, Granola, Gong, Gmail & Slack)
argument-hint: "<account name>"
allowed-tools: Bash, Read, mcp__glean__chat, mcp__glean__search, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__query_granola_meetings, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__list_meetings, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__get_meetings, mcp__28d90b4b-0f1f-4d6d-96f3-e7a6107a9d3c__slack_search_public_and_private, mcp__28d90b4b-0f1f-4d6d-96f3-e7a6107a9d3c__slack_read_user_profile, mcp__ada__get_ada_metric, mcp__ada__get_ada_configuration
---

# Generate PS Knowledge Transfer Doc

Generate a Pre-Sales → PS Knowledge Transfer Google Doc by gathering live data from Glean (Salesforce + Gong + Gmail), Granola (meeting notes), and Slack. The output is a Google Doc saved to the "PS Hand Over Docs" folder in Google Drive. No local files, no Notion page.

## Arguments

The user invoked this command with: $ARGUMENTS

Parse the arguments:
- **Account name** (required) — first argument, e.g. `"Grow Therapy"`. If not provided, ask the user.

**ALL SCOPING SECTIONS (General, Chat, Email, Voice) ARE ALWAYS GENERATED — every KT doc includes all sections. For channels out of scope for Phase 1, include a brief blurb about what was discussed, mark the channel as "not included in this project", and preserve the table structure with TBD values for future use.**

---

## ⚡ EXECUTION MODEL — THREE PHASES

This skill runs in three stateless phases. Each phase must complete fully before the next begins.

| Phase | What happens |
|---|---|
| **Phase A** | Gather ALL raw data from every source. Count every item. Write ALL raw data verbatim to a Google Drive scratch file. |
| **Phase B** | Re-read the scratch file. Extract scoping answers. Determine channel scope from SFDC `product_channels`. Identify contradictions. |
| **Phase C** | Re-read the scratch file. Build the Google Doc. Run post-doc completeness check. Report back. |

**Why three phases?** Context compaction mid-session can cause data gathered early to be silently lost by the time synthesis runs. Phases A→B→C are protected against this because Phase B and C always read from the scratch file — never from conversation memory.

---

## Step 1 (Phase A): Gather Account Data + Write Scratch File

### Step 1 Overview
Run all sub-steps 1a–1f. After EACH sub-step, output a numbered list of every source found (title + date + URL). This is the **Source Count Gate** — it confirms data was captured before moving on.

### Source Count Gate (after every sub-step)
After completing each sub-step, output:
```
Sources found in [sub-step name]: [N]
1. [Title] — [Date] — [URL]
2. [Title] — [Date] — [URL]
...
Total: [N] sources
```
If N = 0, note it explicitly and do NOT skip — record "0 sources found" and proceed.

### 1a — Salesforce Opportunity (via Glean)
Query:
> "Find the open Salesforce opportunity for [account name]. Return: opportunity name, stage, ARR/amount, close date, **full Salesforce opportunity URL** (e.g. https://adasupport.lightning.force.com/lightning/r/Opportunity/[ID]/view), product channels (Chat/Email/Voice), AE name, SC name, next steps, forecast category, probability, employee count, region, segmentation."

**SC name**: extract from the Salesforce opportunity record. Use this as the `SC` field in the document header. If not found in Salesforce, default to `"SC"`.

⚠️ **LINK RULE**: The Salesforce URL must be the actual opportunity URL returned by Glean — never use `https://www.salesforce.com` or any generic placeholder. If no real URL is returned, mark it `N/A` and flag it in the Data Sources Audit.

### 1b — Account Context (via Glean)
Query:
> "What do we know about [account name] as an Ada prospect or customer? Return: company overview, HQ location, timezone, tech stack (CRM, ticketing, CCaaS tools), key contacts and their roles, primary use case, secondary use cases, business drivers, known risks or blockers, API/architecture requirements, key volumes (monthly chat/email/voice conversations, number of agents), telephony provider, IVR setup, email routing system, webform presence."

### 1c — Meeting Notes (via Granola)
- Call `query_granola_meetings` with query: `"[account name] meeting notes action items demo discovery scoping chat email voice"`
- Then call `get_meetings` for the 5 most recent account-related meetings to get full details
- **Extract from these meetings: any scoping answers discussed** — email setup, voice telephony, chat volumes, tech stack, use cases, handoff requirements, APIs, authentication, routing, IVR details, success criteria

### 1d — Gong Calls AND Email Exchanges (via Glean)
Glean indexes all Gong activity — both call transcripts and email exchanges captured by Gong Engage. **Every single result returned must be checked — no skipping, no sampling.**

**Search 1 — Raw keyword search (calls and emails):**
> Use `mcp__glean__search` with query: `"[account name]"`, app filter: `"gong"`, limit: 50, sort by recency
> This returns both call recordings and email threads.
> **EXHAUSTIVE RULE: Every result returned must be individually read and mined. If 10 calls are returned, all 10 are checked. If 30 are returned, all 30 are checked. Never stop at the first few results.**
> For each result, note its title, date, type (call transcript vs email exchange), and the **actual Gong URL for that call or email thread**. Then extract any scoping data it contains.

**Search 2 — Semantic extraction (calls and emails):**
> Use `mcp__glean__chat` with message: "From ALL Gong activity with [account name] — including every call transcript AND every email exchange — extract ALL of the following scoping details. Do not summarise across calls — give me specifics from each call and email separately: (1) Chat: monthly chat volume, current chat platform, chatbot/agent handoff setup, APIs needed, segmentation requirements; (2) Email: email system/platform, which email addresses customers contact, webform presence, email routing, AI agent email address, gradual rollout requirements, email use cases, ticketing/routing for email; (3) Voice: telephony provider, CCaaS platform, SIP integration type, current IVR setup, inbound vs outbound, call volume, agent count, missed call rate, voice use cases, DTMF requirements, SMS capabilities, handoff to human agents, routing requirements; (4) General: pain points, business drivers, tech stack, key contacts, objections, sentiment, next steps, commitments made, **Gong call/email URLs** (actual recording/thread links, not generic). List the source (call title + date + URL, or email subject + date + URL) for each fact."

**After both searches, reconcile all results:**
- Build a complete list of every Gong call and email thread found (title + date + type + actual URL)
- For each one, record what scoping data it contributed — even if it only confirmed something already known
- If a call or email thread yielded nothing useful, note it explicitly as "no new scoping data" — do NOT silently skip it
- Contradictions between calls (e.g. different volume numbers) must be flagged, not silently resolved — note both values and which call each came from

From the Gong results extract ALL of:
- **Chat scoping**: volumes, current platform, handoff setup, APIs, segmentation
- **Email scoping**: email system, routing, webform, AI agent email, use cases, rollout plan
- **Voice scoping**: telephony provider (Genesys? Avaya? Five9? Twilio?), CCaaS, IVR, call volumes, agent count, use cases, DTMF, SMS, routing, handoff
- **General**: pain points, tech stack, volumes, objections, sentiment, next steps, commitments made, actual Gong call/email URLs
- **Note**: Gong email exchanges often contain explicit scoping answers, pricing discussions, and commitments that don't appear in call transcripts — treat them as equally important

### 1e — Gmail (via Glean)
Glean indexes Gmail, so direct emails between the AE/SC team and the prospect are searchable. These often contain explicit commitments, scoping clarifications, and go-live timelines that didn't make it into Gong or Granola.

**EXHAUSTIVE RULE: Every email thread returned must be individually read — no skipping.**

**Search 1 — Incoming emails from the account domain:**
> Use `mcp__glean__search` with query: `"[account name]"`, app filter: `"gmail"`, limit: 50, sort by recency
> For each result, note the subject, sender, date, actual Gmail thread URL, and any scoping data it contains.

**Search 2 — Semantic extraction from Gmail:**
> Use `mcp__glean__chat` with message: "From all Gmail emails related to [account name] — including emails from the prospect's domain and emails the Ada team sent to them — extract ALL of the following: (1) explicit scoping answers or commitments the prospect gave in writing; (2) go-live dates or launch timelines mentioned; (3) pricing or commercial discussions; (4) technical requirements (APIs, integrations, platforms); (5) any concerns, blockers, or risks mentioned; (6) email addresses of prospect contacts. List the source (email subject + date + sender + Gmail URL) for each fact."

**After both searches, reconcile all results:**
- Build a complete list of every Gmail thread found (subject + date + sender + actual URL)
- For each one, record what scoping data it contributed — even if it only confirmed something already known
- If a thread yielded nothing useful, note it explicitly as "no new scoping data" — do NOT silently skip it
- Contradictions with other sources must be flagged — note both values and their sources

**Note**: Gmail often contains the most direct and unambiguous scoping data — prospects frequently give written answers in email that are more precise than what's discussed verbally on calls.

### 1f — Ada Bot
- Search Glean for any Ada bot or demo instance associated with this account
- If found, call `get_ada_configuration` — playbooks, actions, custom instructions
- Call `get_ada_metric` for `resolution_rate` and `csat_rate` (last 7 days)
- **LINK RULE**: Capture the actual Ada dashboard URL for the bot handle (e.g. `https://app.ada.cx/[handle]`). This link must appear in the General Scoping section wherever the bot is referenced.

---

### ✅ Phase A Checkpoint — Write Scratch File to Google Drive

**This step is mandatory. Do NOT proceed to Phase B until the scratch file is written.**

After completing all sub-steps 1a–1f, create a Google Doc in the "PS Hand Over Docs" folder titled:
> `[Account Name] PS Doc Scratch — [YYYY-MM-DD]`

The scratch file must contain, verbatim and unedited:
1. **Source inventory** — every item found in every sub-step (title + date + URL + type)
2. **Raw SFDC data** — full Salesforce opportunity record as returned (all fields, verbatim)
3. **Raw Granola data** — full meeting notes from all meetings retrieved
4. **Raw Gong data** — every call transcript excerpt and email exchange found via Glean
5. **Raw Gmail data** — every thread subject, sender, date, URL, and extracted content
6. **Raw Ada data** — configuration, AR rate, CSAT rate

After writing the scratch file, output:
```
✅ Phase A complete.
Scratch file created: [Google Doc URL]
Sources captured:
  - Gong calls: [N]
  - Gong email threads: [N]
  - Gmail threads: [N]
  - Granola meetings: [N]
  - SFDC: [found / not found]
  - Ada bot: [found / not found]
Proceeding to Phase B.
```

---

## Step 2 (Phase B): Synthesize Scoping Answers

**PHASE B READS FROM THE SCRATCH FILE — not from conversation memory. Re-read the scratch file Google Doc before doing anything in this step.**

**CRITICAL: Before building the document, do a full synthesis pass over ALL gathered data (Salesforce, Glean context, Granola meetings, Gong calls) to extract every scoping answer possible. The goal is to populate as many fields as possible with REAL data — TBD is a last resort, not a default.**

### Channel Scope Determination — SFDC `product_channels` Drives Scope

**The SFDC `product_channels` field is the single source of truth for channel scope. Do NOT use Gong discussions, meeting notes, or any other source to determine scope.**

1. From the scratch file, read the verbatim value of `product_channels` from the Salesforce opportunity record
2. Paste that exact value into the `## ⚠️ CONTRADICTION FLAGS` section (see Step 3 template)
3. Map channels using this exact logic — no interpretation, no inference:
   - `product_channels` contains "Chat" or "Messaging" → **Chat is IN scope**
   - `product_channels` contains "Email" → **Email is IN scope**
   - `product_channels` contains "Voice" or "Phone" → **Voice is IN scope**
   - A channel NOT listed in `product_channels` → that channel is OUT of scope for Phase 1
4. If `product_channels` is blank or not found: default ALL channels to IN scope and flag it in `## ⚠️ CONTRADICTION FLAGS`
5. Record `email_in_scope`, `voice_in_scope`, `chat_in_scope` as True/False based purely on the above mapping

### Chat Scoping Synthesis
- What platform do they currently use for chat? (from tech stack / Gong)
- What is their monthly chat volume? (from volumes / Gong)
- How do they currently hand off to human agents? (from Gong / Granola)
- Do they need APIs or integrations for chat? (from requirements / Gong)
- Any segmentation rules? (from Gong / Salesforce)
- What use cases are unique to chat? (from use cases discussion)

### Email Scoping Synthesis
- What email platform/system do they use? (CRM / ticketing / Gong)
- Which email address(es) do customers contact? (from Gong / Granola)
- Do they have a webform or contact form? (from Gong / website context)
- How is email currently routed? (from Gong / tech stack)
- What email address will the AI agent respond as? (from scoping discussions)
- Do they need a gradual rollout? (from Gong / risk discussions)
- What use cases are unique to email vs chat? (from use cases)
- How does email handoff to ticketing work? (from Gong / Granola)
- Any authentication requirements for email? (from API/auth discussions)

### Voice Scoping Synthesis
- Who is their telephony provider? (Genesys, Avaya, Five9, Twilio, Vonage, etc.)
- What CCaaS platform do they use? (from tech stack / Gong)
- What SIP integration type? (from Gong / technical discussions)
- What does their current IVR look like? (from Gong / Granola)
- Inbound or outbound or both? (from use cases)
- What is their call volume? (from volumes data)
- How many human agents do they have? (from Gong / Granola)
- What is their missed call rate? (from pain points / Gong)
- What is the primary voice use case? (from use cases discussion)
- Do they need DTMF / dial pad input? (from Gong / technical)
- SMS capabilities needed? (from Gong / requirements)
- How do calls hand off to human agents? (from Gong / current process)
- Any routing requirements? (from Gong / IVR discussion)

---

## Step 3 (Phase C): Build the Google Doc Content

**PHASE C READS FROM THE SCRATCH FILE — not from conversation memory. Re-read the scratch file Google Doc before doing anything in this step.**

**CRITICAL: This is a structured scoping form — NOT a narrative brief. Use the exact table structure below. Every table cell must be filled with REAL data extracted from Gong, Granola, and Glean. TBD is only for fields where no data was found anywhere. The Chat, Email, and Voice scoping tables are the most important part of this document.**

**LINK RULES (apply everywhere in the document):**
- **SFDC Opp**: must be the actual Salesforce opportunity URL (e.g. `https://adasupport.lightning.force.com/lightning/r/Opportunity/[ID]/view`) — never `https://www.salesforce.com`
- **Gong calls/emails**: must be the actual Gong recording or email thread URL returned from Glean — never a generic Gong link
- **Gmail threads**: must be the actual Gmail thread URL — never a generic Gmail link
- **Granola meetings**: must be the actual Granola meeting link if available
- **Ada bot handle**: must be the actual `https://app.ada.cx/[handle]` URL
- If a real URL was not returned by any data source, write `N/A` — never substitute a homepage or generic URL

**FORMATTING RULES — always apply these when writing the doc content:**
- **Multi-item table cells** (contacts, use cases, APIs, risks, tech stack items, languages): each item must be on its own line using `<br>` to separate items within a table cell. Never write them as a comma-separated inline list when there are 2+ items.
- **Numbered lists** (KEY NEXT STEPS, any ordered sequence): always use `1.`, `2.`, `3.` on separate lines with a blank line before the list starts. Never inline numbered items.
- **Bullet lists** (Gong highlights — pain points, use cases, objections, tech mentions): always use `-` on separate lines with a blank line before the list starts. Never inline bullet items.
- **Meeting notes** (MEETING NOTES section): each meeting gets its own block — bold title + date on one line, then the summary on the next line(s). Separate meetings with a blank line.
- **Project Scope field**: Phase 1, Phase 2, Phase 3 each on a new line using `<br>` — e.g. `Phase 1: [use case]<br>Phase 2: [use case]<br>Phase 3: [use case]`
- **Key client stakeholders field**: one contact per line using `<br>` — e.g. `John Smith — VP Engineering<br>Jane Doe — Head of CX`

Construct the full document content as Markdown. Substitute ALL placeholders with real data before creating the doc.

```markdown
# Sales to Professional Services Handoff — [Account Name]

**Generated:** [Today's date] | **SC:** [SC Name]

---

## ⚠️ CONTRADICTION FLAGS

> This section is **mandatory** and must appear in every document. It cannot be omitted. If no contradictions were detected, write "None detected." — do not leave this section blank or skip it.

**SFDC `product_channels` (verbatim):** `[paste exact value here — e.g. "Messaging + Email"]`
**Channels in scope (derived from above):** [Chat / Email / Voice — list only what appears in product_channels]
**Channels out of scope for Phase 1:** [list channels not in product_channels, or "None"]

**Data contradictions found across sources:**
[List every field where two or more sources gave different values. Format:
- **[Field name]**: [Source A] says [value A] | [Source B] says [value B] — PS team to verify with client
Or if none: "None detected."]

---

## GENERAL SCOPING

| Field | SC Input |
|---|---|
| **Client Overview** *(Overview of Account + Business case with Ada)* | [company overview + HQ + industry + key business drivers] |
| **SFDC Opp** | [actual Salesforce opportunity URL — e.g. https://adasupport.lightning.force.com/lightning/r/Opportunity/[ID]/view] |
| **Solution Survey** | [link or TBD] |
| **Key client stakeholders & Roles** | [Name — Role<br>Name — Role<br>(one per line using \<br\>)] |
| **Timezone** | [timezone] |
| **Channels currently supported** | [chat: X/mo, email: X/mo, voice: X calls/mo] |
| **Agent Tech Stack** | [CRM, ticketing, CCaaS, chat platform] |
| **KB Readiness** *(Formatted and ready for AI agent ingestion or updates required for AI ingestion)* | [assessment from discussions or TBD] |
| **Project Scope** *(What will Phase 1 include? Phase 2? Phase 3?)* | Phase 1: [primary use case]<br>Phase 2: [secondary]<br>Phase 3: [tertiary] |
| **Expected Launch Date?** | [close date from SFDC] |
| **Success Criteria 30 days post launch** | [from discussions or TBD] |
| **Channels** *(What channels will they plan to deploy on?)* | [Chat / Email / Voice] |
| **Language Requirements** | [languages, default English if not mentioned] |
| **APIs / Personalization / Authentication Requirements** | [from API/auth discussions] |
| **Segmentation Requirements** | [from discussions or TBD] |
| **Product promises made to the client / FRs?** | [from discussions or TBD] |
| **Cluster** | [US / US2 / Maple / EU — default Maple] |
| **Number of Ai Agents** | [from discussions or TBD] |
| **Ada Bot Handle** | [actual https://app.ada.cx/[handle] URL, or N/A if no bot found] |

### Miscellaneous

| Field | SC Input |
|---|---|
| **Enrolled in Ada Academy** | No (pre-signature) |
| **Security Requirements** | [from discussions or TBD] |
| **Link + invites to Demo/Sandbox instance** | [actual Gong call URL or actual Granola meeting link — N/A if not found] |
| **Pilot / Opt out** | TBD |
| **Additional Notes / Risks** | [risks from data] |

## KEY NEXT STEPS

*From Platform Demo — [demo date]*

1. [next step]
2. [next step]
3. [next step]

## MEETING NOTES (from Granola)

**[Meeting Title]** ([Date])
[Summary of key points discussed]

**[Meeting Title]** ([Date])
[Summary of key points discussed]

---

## CHAT SCOPING

| Field | SC Input |
|---|---|
| **Current Chat Platform** *(What platform do they use for chat today?)* | [e.g. Intercom, Zendesk Chat, Salesforce Chat — from tech stack / Gong] |
| **Monthly Chat Volume** | [from volumes data / Gong] |
| **Current Handoff Setup** *(How does chat currently hand off to human agents?)* | [describe current process from Gong/Granola] |
| **APIs / Integrations Required** | [from Gong/Granola API discussions] |
| **Segmentation Requirements** | [from discussions or TBD] |
| **Chat-Specific Use Cases** *(Any use cases unique to chat?)* | [from use cases discussions] |
| **KB Readiness for Chat** | [assessment from discussions or TBD] |
| **Language Requirements** | [languages for chat] |
| **Authentication Requirements** | [from auth discussions or TBD] |

---

## EMAIL SCOPING

[IF email is OUT OF SCOPE: insert this block]
[email notes summary — what was discussed about email during sales conversations]

> ⚠️ **Email channel is not included in the scope of this project.** The table below is preserved for future reference — fields are left as TBD.

[IF email IS IN SCOPE: fill all table cells from Gong/Granola/Glean data]

### Email Architecture

| Field | SC Input |
|---|---|
| **Tech Stack** *(Is the system your agents use to receive and respond to emails the same as your chat?)* | [same as chat or different — from tech stack] |
| **Email landscape** *(Which email address(es) are your customers emailing?)* | [list email addresses from Gong/Granola] |
| **Webform** *(Do you have a webform or contact form on your website?)* | [yes/no + description from Gong/Granola] |
| **Custom / Filter Incoming Emails** | [how they triage/filter incoming email] |
| **AI Agent / Human support** *(Which email address will the AI Agent respond as?)* | [from scoping discussions or TBD] |
| **Launch plan** *(Do you require a gradual rollout?)* | [from rollout discussions or TBD] |

### Email Configuration

| Field | SC Input |
|---|---|
| **Knowledge Base** *(Any additional sources specific to email?)* | [from KB discussions or TBD] |
| **Use cases** *(Any use cases unique to email vs chat/voice?)* | [email-specific use cases from Gong/Granola] |
| **Workflow Mapping** | [from workflow discussions or TBD] |
| **CC Support** | [from discussions or TBD] |
| **Metadata** | [from discussions or TBD] |

### Email Handoffs

| Field | SC Input |
|---|---|
| **Email / Ticketing** | [how email tickets are created/routed from Gong/Granola] |
| **Routing** | [routing rules from discussions or TBD] |

### Additional Requirements

| Field | SC Input |
|---|---|
| **Authentication** | [from auth discussions or TBD] |
| **Conversation Start** | [how email conversations are initiated or TBD] |

---

## VOICE SCOPING

[IF voice is OUT OF SCOPE: insert this block]
[voice notes summary — what was discussed about voice during sales conversations]

> ⚠️ **Voice channel is not included in the scope of this project.** The table below is preserved for future reference — fields are left as TBD.

[IF voice IS IN SCOPE: fill all table cells from Gong/Granola/Glean data]

### Voice Architecture

| Field | SC Input |
|---|---|
| **Telephony Provider** | [e.g. Genesys Cloud, Avaya, Five9, Twilio, Vonage — from Gong/Granola/Glean] |
| **CCaaS / Agent System** | [full CCaaS platform name from tech stack / Gong] |
| **SIP Integration Type** | [from technical discussions or TBD] |
| **Current IVR** | [describe current IVR menu and flow from Gong/Granola] |
| **Inbound vs Outbound** | [inbound / outbound / both — from use cases] |
| **Call Volume** | [monthly or daily call volume from data] |
| **Current Agent Count** | [from volumes data / Gong] |
| **Missed Call Rate** | [from pain points / Gong or TBD] |

### Voice Use Cases

| Field | SC Input |
|---|---|
| **Primary Voice Use Case** | [main reason customers call — from Gong/Granola] |
| **Call Categorization / Triage** | [how calls are currently triaged from Gong/Granola] |
| **Secondary Voice Use Cases** | [additional voice use cases from Gong/Granola] |

### Voice Technical Requirements

| Field | SC Input |
|---|---|
| **APIs Required for Voice** | [from API discussions or TBD] |
| **DTMF / Dial Pad Input** | [yes/no + details from technical discussions] |
| **SMS Capabilities** | [yes/no + details from discussions] |
| **Cross-Channel Interoperability** | [cross-channel needs from discussions or TBD] |

### Voice Handoffs

| Field | SC Input |
|---|---|
| **Handoff to Human Agents** | [how voice calls hand off to humans from Gong/Granola] |
| **Routing Requirements** | [routing rules from discussions or TBD] |

### Voice Quality & Success Criteria

| Field | SC Input |
|---|---|
| **Voice Quality Feedback from Demo** | [from Gong call discussing voice demo] |
| **Success Criteria for Voice** | [from success criteria discussions or TBD] |
| **Voice-Specific Risks** | [from risk discussions or TBD] |
| **Timeline** | [close date from SFDC or planned future phase timeline] |

---

## GONG CALL HIGHLIGHTS

### [Call Title] — [Date]

**Pain Points:**
- [item]
- [item]

**Use Cases:**
- [item]
- [item]

**Objections:**
- [item]

**Tech Mentions:**
- [item]

**Sentiment:** [value]
**Recording:** [actual Gong recording URL — not a generic link]

---

## 📋 DATA SOURCES AUDIT

*Auto-generated — every source checked on [Today's date]*

> ⚠️ **Contradiction flags are mandatory.** After all data is gathered, actively cross-check every field across all sources. If any source gives a different value for the same field (e.g. Gong says chat volume = 10k/mo, Gmail says 15k/mo), both values must be listed here with their source. Do not silently resolve contradictions — flag them explicitly so the PS team can verify with the client.

### Salesforce (via Glean)
- SFDC Opp URL: [actual URL found, or "not returned — marked N/A in doc"]
- ARR: [found ($X) / not found]
- Close Date: [found / not found]
- AE Name: [found / not found]
- Channels: [found (X) / not found]
- Stage: [found / not found]

### Account Context (via Glean)
- Company Overview: [found / not found]
- HQ: [found / not found]
- Timezone: [found / not found]
- Tech Stack: [found (X, Y, Z) / not found]
- Key Contacts: [found (N contacts) / not found]
- Business Drivers: [found / not found]
- Key Volumes: [found / not found]

### Gong Call Transcripts (via Glean)
- Calls found: [N] — ALL [N] checked
- [Call Title] ([Date]) ([actual Gong URL]) — [what it contributed, or "no new scoping data"]
- *(repeat for every call)*
- ❌ Fields not found in any call: [list or "none"]
- ⚠️ Contradictions: [e.g. "Chat volume: 10k/mo stated in [Call A, Date, URL] vs 15k/mo stated in [Call B, Date, URL]" — or "none detected"]

### Gong Email Exchanges (via Glean)
- Email threads found: [N] — ALL [N] checked
- [Subject] ([Date]) ([actual Gong email URL]) — [what it contributed, or "no new scoping data"]
- *(repeat for every thread)*
- ❌ Fields not found: [list / "No email exchanges indexed" if none returned]
- ⚠️ Contradictions: [specific contradictions with source + URL, or "none detected"]

### Gmail (via Glean)
- Threads found: [N] — ALL [N] checked
- [Subject] ([Date], from [Sender]) ([actual Gmail thread URL]) — [what it contributed, or "no new scoping data"]
- *(repeat for every thread)*
- ❌ Fields not found: [list / "No Gmail threads indexed" if none returned]
- ⚠️ Contradictions: [specific contradictions with source + URL, or "none detected"]

### Granola Meeting Notes
- Meetings found: [N] — ALL [N] checked
- [Meeting Title] ([Date]) ([Granola meeting link if available]) — [what it contributed, or "no new scoping data"]
- *(repeat for every meeting)*
- ❌ Fields not found in any meeting: [list or "none"]
- ⚠️ Contradictions: [specific contradictions with source, or "none detected"]

### Ada Bot
- Bot handle found: [yes — https://app.ada.cx/[handle] / no bot found for this account]
- AR Rate (last 7d): [X% / not retrieved]
- CSAT Rate (last 7d): [X% / not retrieved]

### ⚠️ Fields Still TBD — Needs Manual Input
- **General**: [field names, or "none"]
- **Chat**: [field names, or "none"]
- **Email**: [field names, or "none"]
- **Voice**: [field names, or "none"]
```

---

## Step 4: Create the Google Doc

**This step is MANDATORY — always run it, every time. Do NOT skip it or ask the user.**

### 4a — Find the "PS Hand Over Docs" folder
Use the Google Workspace MCP to find the folder ID for "PS Hand Over Docs" in Google Drive:
```
google-workspace: search Drive for folder named "PS Hand Over Docs"
```

### 4b — Create the Google Doc
Use `import_to_google_doc` to create the document with the full Markdown content from Step 4, placed in the "PS Hand Over Docs" folder:
```
import_to_google_doc({
  title: "PS Handoff — [Account Name] — [YYYY-MM-DD]",
  content: "[full markdown content from Step 3 with all real data substituted in]",
  folderId: "[PS Hand Over Docs folder ID from 4a]"
})
```

Save the returned Google Doc URL.

---

## Step 5: Post-Doc Completeness Check + Report Back

### Step 5a — Completeness Check (mandatory before reporting)
After the Google Doc is created, perform a completeness check by re-reading the scratch file:
1. Scan every table cell in the Google Doc for: TBD, Unknown, blank values
2. For each TBD: cross-reference the scratch file — if the data exists in the scratch file, fill it in the doc now; if not, leave it TBD and add it to the report
3. Verify the `## ⚠️ CONTRADICTION FLAGS` section exists and contains the verbatim SFDC `product_channels` value
4. Verify all URLs in the doc are real (non-placeholder) — if any placeholder URL was written, replace it with `N/A` and flag it

### Step 5b — Report Back
Tell the user:
1. ✅ Google Doc URL (clickable link)
2. ✅ Scratch file URL (clickable link)
3. One-line source summary (e.g. "Checked 6 Gong calls, 3 Gong email threads, 5 Gmail threads, 4 Granola meetings")
4. Fields still TBD needing manual input — so the SC knows exactly what to fill in before handing the doc to PS
5. Any ⚠️ contradiction flags raised — call these out explicitly so the user knows what to verify

The full Data Sources Audit is at the end of the Google Doc itself.

---

## Important Notes

- **No local files** — do not generate a .docx or save anything locally. The Google Doc is the only output.
- **No Notion page** — do not publish to Notion. Google Doc only.
- **ALL scoping sections are always included** — General, Chat, Email, and Voice every time. Never skip a section.
- **Out-of-scope channels**: When email or voice is not in Phase 1 scope, add the notes summary + ⚠️ callout above the table, and leave table rows as TBD.
- **Fill from data, not TBD** — For in-scope channels, every table cell should be populated from Gong, Granola, or Glean. TBD is only for genuinely unknown fields.
- **The scoping tables are the most important part of this document** — Chat, Email, and Voice scoping tables are what PS uses to build the implementation. In-scope tables must be filled.
- **Always create the Google Doc (Step 4)** — mandatory, no exceptions.
- **Never hallucinate** account details — TBD is always better than a wrong answer.
- **Gong via Glean** — there is no standalone Gong MCP; use `mcp__glean__search` with `app: "gong"` and `mcp__glean__chat` to search indexed Gong transcripts. If Glean doesn't return Gong results, note it as TBD — don't skip the attempt.
- **Gong data enriches scoping fields directly** — extract telephony provider, email system, chat platform, volumes, IVR setup, use cases, handoff process from Gong transcripts and put them into the correct scoping table cells.
- **Gmail via Glean** — use `mcp__glean__search` with `app: "gmail"` to search indexed Gmail threads. Gmail often contains the most explicit written scoping answers — treat it as equally important as Gong. If Glean doesn't return Gmail results, note it as TBD — don't skip the attempt.
- **Real links only** — every URL in the document must be an actual URL returned from a data source. Never substitute a homepage, a generic domain, or a placeholder. If no real URL was found, write `N/A`.
- **Data Sources Audit is always last** — it is the final section of every generated document, written after all content sections are complete.
- **Contradictions must be flagged** — never silently resolve conflicting data. Always surface both values with their sources in the Data Sources Audit, and call them out in Step 5.
