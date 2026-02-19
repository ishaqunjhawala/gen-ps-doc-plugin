---
description: Generate a Pre-Sales → PS Knowledge Transfer Google Doc for any account (fetches live data from Glean, Granola, Gong & Slack)
argument-hint: "<account name>"
allowed-tools: Bash, Read, mcp__glean__chat, mcp__glean__search, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__query_granola_meetings, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__list_meetings, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__get_meetings, mcp__28d90b4b-0f1f-4d6d-96f3-e7a6107a9d3c__slack_search_public_and_private, mcp__28d90b4b-0f1f-4d6d-96f3-e7a6107a9d3c__slack_read_user_profile, mcp__ada__get_ada_metric, mcp__ada__get_ada_configuration
---

# Generate PS Knowledge Transfer Doc

Generate a Pre-Sales → PS Knowledge Transfer Google Doc by gathering live data from Glean (Salesforce + Gong), Granola (meeting notes), and Slack. The output is a Google Doc saved to the "PS Hand Over Docs" folder in Google Drive. No local files, no Notion page.

## Arguments

The user invoked this command with: $ARGUMENTS

Parse the arguments:
- **Account name** (required) — first argument, e.g. `"Grow Therapy"`. If not provided, ask the user.

**ALL SCOPING SECTIONS (General, Chat, Email, Voice) ARE ALWAYS GENERATED — every KT doc includes all sections. For channels out of scope for Phase 1, include a brief blurb about what was discussed, mark the channel as "not included in this project", and preserve the table structure with TBD values for future use.**

---

## Step 1: Identify the SC

Get the current user's name to fill the SC field in the doc:
- Call `slack_read_user_profile` with no user_id (returns current user)
- Extract `display_name` or `real_name`
- If unavailable, default to `"SC"` and continue — don't block on this

---

## Step 2: Gather Account Data (run all in parallel)

### 2a — Salesforce Opportunity (via Glean)
Query:
> "Find the open Salesforce opportunity for [account name]. Return: opportunity name, stage, ARR/amount, close date, Salesforce URL, product channels (Chat/Email/Voice), AE name, SC name, next steps, forecast category, probability, employee count, region, segmentation."

### 2b — Account Context (via Glean)
Query:
> "What do we know about [account name] as an Ada prospect or customer? Return: company overview, HQ location, timezone, tech stack (CRM, ticketing, CCaaS tools), key contacts and their roles, primary use case, secondary use cases, business drivers, known risks or blockers, API/architecture requirements, key volumes (monthly chat/email/voice conversations, number of agents), telephony provider, IVR setup, email routing system, webform presence."

### 2c — Meeting Notes (via Granola)
- Call `query_granola_meetings` with query: `"[account name] meeting notes action items demo discovery scoping chat email voice"`
- Then call `get_meetings` for the 5 most recent account-related meetings to get full details
- **Extract from these meetings: any scoping answers discussed** — email setup, voice telephony, chat volumes, tech stack, use cases, handoff requirements, APIs, authentication, routing, IVR details, success criteria

### 2d — Gong Call Transcripts (via Glean)
Glean indexes Gong calls — search for the most recent calls with this account:

Search 1 (keyword search for recent calls):
> Use `mcp__glean__search` with query: `"[account name]"`, app filter: `"gong"`, limit: 5, sort by recency

Search 2 (semantic search for scoping details):
> Use `mcp__glean__chat` with message: "From Gong calls with [account name], extract ALL of the following scoping details: (1) Chat: monthly chat volume, current chat platform, chatbot/agent handoff setup, APIs needed, segmentation requirements; (2) Email: email system/platform, which email addresses customers contact, webform presence, email routing, AI agent email address, gradual rollout requirements, email use cases, ticketing/routing for email; (3) Voice: telephony provider, CCaaS platform, SIP integration type, current IVR setup, inbound vs outbound, call volume, agent count, missed call rate, voice use cases, DTMF requirements, SMS capabilities, handoff to human agents, routing requirements; (4) General: pain points, business drivers, tech stack, key contacts, objections, sentiment, next steps, Gong call URLs."

From the Gong results extract ALL of:
- **Chat scoping**: volumes, current platform, handoff setup, APIs, segmentation
- **Email scoping**: email system, routing, webform, AI agent email, use cases, rollout plan
- **Voice scoping**: telephony provider (Genesys? Avaya? Five9? Twilio?), CCaaS, IVR, call volumes, agent count, use cases, DTMF, SMS, routing, handoff
- **General**: pain points, tech stack, volumes, objections, sentiment, next steps, Gong call URLs

### 2e — Ada Bot (skip if no bot mentioned)
- If Glean context mentions an Ada bot or demo instance for this account:
  - Call `get_ada_configuration` — playbooks, actions, custom instructions
  - Call `get_ada_metric` for `resolution_rate` and `csat_rate` (last 7 days)

---

## Step 3: Synthesize Scoping Answers

**CRITICAL: Before building the document, do a full synthesis pass over ALL gathered data (Salesforce, Glean context, Granola meetings, Gong calls) to extract every scoping answer possible. The goal is to populate as many fields as possible with REAL data — TBD is a last resort, not a default.**

### Out-of-Scope Channel Detection
Check the Salesforce `product_channels` field and Phase 1 scoping discussions:
- If **Email is not in Phase 1 scope**: note `email_out_of_scope = True` and capture a brief summary of what was discussed (timing, platform, future phase plans)
- If **Voice is not in Phase 1 scope**: note `voice_out_of_scope = True` and capture a brief summary of what was discussed (telephony details, timeline, future phase plans)
- If a channel IS in scope: fill every table cell from data

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

## Step 4: Build the Google Doc Content

**CRITICAL: This is a structured scoping form — NOT a narrative brief. Use the exact table structure below. Every table cell must be filled with REAL data extracted from Gong, Granola, and Glean. TBD is only for fields where no data was found anywhere. The Chat, Email, and Voice scoping tables are the most important part of this document.**

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

## GENERAL SCOPING

| Field | SC Input |
|---|---|
| **Client Overview** *(Overview of Account + Business case with Ada)* | [company overview + HQ + industry + key business drivers] |
| **SFDC Opp** | [Salesforce URL] |
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

### Miscellaneous

| Field | SC Input |
|---|---|
| **Enrolled in Ada Academy** | No (pre-signature) |
| **Security Requirements** | [from discussions or TBD] |
| **Link + invites to Demo/Sandbox instance** | [Gong call URL or TBD] |
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
**Recording:** [URL]
```

---

## Step 5: Create the Google Doc

**This step is MANDATORY — always run it, every time. Do NOT skip it or ask the user.**

### 5a — Find the "PS Hand Over Docs" folder
Use the Google Workspace MCP to find the folder ID for "PS Hand Over Docs" in Google Drive:
```
google-workspace: search Drive for folder named "PS Hand Over Docs"
```

### 5b — Create the Google Doc
Use `import_to_google_doc` to create the document with the full Markdown content from Step 4, placed in the "PS Hand Over Docs" folder:
```
import_to_google_doc({
  title: "PS Handoff — [Account Name] — [YYYY-MM-DD]",
  content: "[full markdown content from Step 4 with all real data substituted in]",
  folderId: "[PS Hand Over Docs folder ID from 5a]"
})
```

Save the returned Google Doc URL.

---

## Step 6: Report Back to the User

Tell the user:
- ✅ Google Doc URL (clickable link)
- Summary of what was auto-filled vs left as TBD — list which fields still need manual input

---

## Important Notes

- **No local files** — do not generate a .docx or save anything locally. The Google Doc is the only output.
- **No Notion page** — do not publish to Notion. Google Doc only.
- **ALL scoping sections are always included** — General, Chat, Email, and Voice every time. Never skip a section.
- **Out-of-scope channels**: When email or voice is not in Phase 1 scope, add the notes summary + ⚠️ callout above the table, and leave table rows as TBD.
- **Fill from data, not TBD** — For in-scope channels, every table cell should be populated from Gong, Granola, or Glean. TBD is only for genuinely unknown fields.
- **The scoping tables are the most important part of this document** — Chat, Email, and Voice scoping tables are what PS uses to build the implementation. In-scope tables must be filled.
- **Always create the Google Doc (Step 5)** — mandatory, no exceptions.
- **Never hallucinate** account details — TBD is always better than a wrong answer.
- **Gong via Glean** — there is no standalone Gong MCP; use `mcp__glean__search` with `app: "gong"` and `mcp__glean__chat` to search indexed Gong transcripts. If Glean doesn't return Gong results, note it as TBD — don't skip the attempt.
- **Gong data enriches scoping fields directly** — extract telephony provider, email system, chat platform, volumes, IVR setup, use cases, handoff process from Gong transcripts and put them into the correct scoping table cells.
