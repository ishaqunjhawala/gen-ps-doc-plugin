---
description: Generate a Pre-Sales → PS Knowledge Transfer .docx for any account (fetches live data from Glean, Granola, Gong & Slack)
argument-hint: "<account name>"
allowed-tools: Bash, Read, mcp__glean__chat, mcp__glean__search, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__query_granola_meetings, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__list_meetings, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__get_meetings, mcp__28d90b4b-0f1f-4d6d-96f3-e7a6107a9d3c__slack_search_public_and_private, mcp__28d90b4b-0f1f-4d6d-96f3-e7a6107a9d3c__slack_read_user_profile, mcp__ada__get_ada_metric, mcp__ada__get_ada_configuration, mcp__729d2fa9-4409-4a97-838a-8eb8d2b766cf__notion-create-pages, mcp__729d2fa9-4409-4a97-838a-8eb8d2b766cf__notion-search
---

# Generate PS Knowledge Transfer Doc

Generate a formatted Pre-Sales → PS Knowledge Transfer `.docx` by gathering live data from Glean (Salesforce + Gong), Granola (meeting notes), and Slack. No cache or config file needed.

## Arguments

The user invoked this command with: $ARGUMENTS

Parse the arguments:
- **Account name** (required) — first argument, e.g. `"Grow Therapy"`. If not provided, ask the user.

**ALL THREE SCOPING SECTIONS (General, Chat, Email, Voice) ARE ALWAYS GENERATED — do not make any section conditional or optional. Every KT doc must include all sections, fully populated from the data gathered.**

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

**CRITICAL: Before building the data dictionary, do a full synthesis pass over ALL gathered data (Salesforce, Glean context, Granola meetings, Gong calls) to extract every scoping answer possible. The goal is to populate as many fields as possible with REAL data — TBD is a last resort, not a default.**

For each scoping section, actively answer these questions from the data:

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

## Step 4: Assemble the Data Dictionary

From all gathered and synthesized data, build this Python dict. Use `"TBD"` ONLY when a field was genuinely not found anywhere in the data — never guess or hallucinate.

```python
data = {
    "account": {
        "company": "<company name>",
        "hq": "<city, country>",
        "platform": "<industry / what they do>",
        "timezone": "<e.g. EST, PST>",
        "contacts": {
            "<Contact Name>": "<Role>",
        },
        "current_stack": "<CRM, ticketing, CCaaS — comma separated>",
        "primary_use_case": "<Phase 1 use case>",
        "secondary_use_cases": ["<Phase 2>", "<Phase 3>"],
        "business_drivers": ["<driver 1>", "<driver 2>"],
        "risks": ["<risk 1>", "<risk 2>"],
        "key_architecture": {
            "crm": "<e.g. Salesforce>",
            "ticketing": "<e.g. Zendesk>",
            "chat_platform": "<e.g. Intercom, Zendesk Chat>",
            "telephony": "<e.g. Genesys Cloud, Avaya, Five9, Twilio>",
            "ccaas": "<e.g. Genesys, Nice inContact>",
            "email_platform": "<e.g. Zendesk, Salesforce Service Cloud>",
        },
        "key_volumes": {
            "chat_monthly": "<number or range>",
            "email_monthly": "<number or range>",
            "call_monthly": "<number or range>",
            "agents": "<number>",
        },
        "next_steps": ["<step 1>", "<step 2>"],
        "salesforce_url": "<SFDC URL>",
        "demo_recap": {
            "date": "<YYYY-MM-DD>",
            "feedback": "<key feedback from demo>",
            "gong_call_url": "<Gong URL if found>",
        },
    },
    "opportunity": {
        "sf_url": "<SFDC URL>",
        "close_date": "<YYYY-MM-DD>",
        "product_channels": "<Chat / Chat; Email / Chat; Email; Voice>",
        "arr": "<number>",
        "stage": "<stage name>",
    },
    "chat_scoping": {
        "current_platform": "<what they use for chat today>",
        "monthly_volume": "<monthly chat conversations>",
        "handoff_setup": "<how chat hands off to human agents today>",
        "apis_needed": "<list of APIs/integrations needed for chat>",
        "segmentation": "<any segmentation rules>",
        "unique_use_cases": "<use cases unique to chat>",
        "kb_readiness": "<is KB formatted for AI ingestion or needs work>",
        "languages": "<languages required>",
        "authentication": "<any auth requirements for chat>",
    },
    "email_scoping": {
        "email_system": "<what email platform/system they use>",
        "customer_email_addresses": "<which addresses customers email>",
        "webform": "<do they have a webform? describe it>",
        "email_routing": "<how email is currently routed/triaged>",
        "ai_agent_email": "<what address the AI agent will respond as>",
        "gradual_rollout": "<do they need phased rollout? yes/no/TBD>",
        "unique_use_cases": "<use cases unique to email vs chat>",
        "ticketing_handoff": "<how email hands off to ticketing>",
        "routing_requirements": "<any routing rules>",
        "authentication": "<any auth requirements for email>",
        "cc_support": "<do they need CC on emails?>",
        "metadata": "<any metadata requirements>",
        "kb_additional": "<additional KB sources specific to email>",
    },
    "voice_scoping": {
        "telephony_provider": "<e.g. Genesys Cloud, Avaya, Five9, Twilio, Vonage>",
        "ccaas_platform": "<e.g. Genesys Cloud, Nice inContact, Talkdesk>",
        "sip_integration_type": "<SIP trunk type / integration method>",
        "current_ivr": "<describe current IVR setup and menu options>",
        "inbound_outbound": "<inbound / outbound / both>",
        "call_volume": "<monthly or daily call volume>",
        "agent_count": "<number of human agents on voice>",
        "missed_call_rate": "<% or description>",
        "primary_use_case": "<main reason customers call>",
        "call_categorization": "<how calls are currently categorized/triaged>",
        "secondary_use_cases": ["<use case 2>", "<use case 3>"],
        "apis_required": "<APIs needed for voice>",
        "dtmf_required": "<yes/no — do they need dial pad input>",
        "sms_capabilities": "<yes/no — SMS needed>",
        "cross_channel": "<any cross-channel interoperability needs>",
        "handoff_to_human": "<how voice hands off to human agents>",
        "routing_requirements": "<any call routing rules>",
        "voice_quality_feedback": "<feedback from voice demo>",
        "success_criteria": "<what does success look like for voice>",
        "voice_risks": "<voice-specific risks>",
    },
    "granola_notes": [
        {"title": "<meeting title>", "date": "<YYYY-MM-DD>", "summary": "<key points>"},
    ],
    "gong_calls": [
        {
            "date": "<YYYY-MM-DD>",
            "title": "<call title>",
            "url": "<Gong call URL>",
            "pain_points": ["<pain point 1>", "<pain point 2>"],
            "use_cases_mentioned": ["<use case 1>"],
            "objections": ["<objection 1>"],
            "tech_mentions": ["<tool or system mentioned>"],
            "volume_data": "<any volume/scale info mentioned>",
            "sentiment": "<positive / mixed / negative>",
            "next_steps": ["<agreed next step>"],
        }
    ],
    "sc_name": "<SC full name>",
}
```

---

## Step 5: Check python-docx and Run the Generator

### 5a — Ensure python-docx is installed
```bash
python3 -c "import docx" 2>/dev/null || pip install python-docx
```

### 5b — Write the data to a temp JSON file and run the script
```bash
python3 -c "
import json
data = <PASTE DATA DICT HERE AS PYTHON LITERAL>
with open('/tmp/ps_doc_data.json', 'w') as f:
    json.dump(data, f)
"

python3 "${CLAUDE_PLUGIN_ROOT}/scripts/ps_doc_skill.py" \
  --account "<account name>" \
  --sections general email voice \
  --sc-name "<SC name>" \
  --output-dir "./ps-knowledge-transfer" \
  --data-file "/tmp/ps_doc_data.json"
```

The script prints `SUCCESS: <filepath>` on completion.

### 5c — Copy path to clipboard (macOS)
```bash
echo "<filepath>" | pbcopy
```

---

## Step 6: Publish to Notion

**This step is MANDATORY — always run it, every time. Do NOT skip it or ask the user if they want it. A Notion page is always created alongside the .docx.**

### 6a — Parent page
The canonical Notion template lives at:
- **Template page ID**: `30b6162e53cd80f8b9d1c151f5eebf08`
- **Template URL**: https://www.notion.so/adasupport/MAKE-A-COPY-Pre-Sales-PS-Knowledge-Transfer-TEMPLATE-30b6162e53cd80f8b9d1c151f5eebf08

Search Notion for an existing parent/folder page to house the generated doc:
```
notion-search query: "PS Knowledge Transfer handoff"
```
- If a clear parent page is found (e.g. a "PS Handoffs" index page), use its page ID as parent.
- If not found, create the page at workspace level (no parent) — **do NOT use the template page itself as parent**.

### 6b — Build the Notion page content

**CRITICAL: The Notion page must use the EXACT same field names and section structure as the official Ada PS KT template (link above). Do NOT invent new section names or write in a narrative format. This is a structured scoping form. Copy field names character-for-character. ALL THREE SCOPING SECTIONS MUST BE PRESENT AND POPULATED.**

**CRITICAL: Every table cell must be filled with REAL data extracted from Gong, Granola, and Glean. Do NOT default to TBD — only use TBD for fields where no data was found anywhere. The chat, email, and voice scoping tables are the most important part of this document.**

Use Notion-flavored Markdown to construct the page:

```markdown
# Sales to Professional Services Handoff — [Account Name]

**Generated:** [Date] | **SC:** [SC Name]

---

## GENERAL SCOPING

| Field | SC Input |
|---|---|
| **Client Overview** *(Overview of Account + Business case with Ada)* | [company overview + HQ + industry + key business drivers from Gong/Granola] |
| **SFDC Opp** | [Salesforce URL] |
| **Solution Survey** | [link or TBD] |
| **Key client stakeholders & Roles** | [Name — Role, one per line, from Gong/Granola/Glean] |
| **Timezone** | [timezone from account data] |
| **Channels currently supported** | [chat: X/mo, email: X/mo, voice: X/mo — from volumes data] |
| **Agent Tech Stack** | [CRM + ticketing + CCaaS + chat platform — from tech stack data] |
| **KB Readiness** *(Formatted and ready for AI agent ingestion or updates required for AI ingestion)* | [assessment from Gong/Granola discussions or TBD] |
| **Project Scope** *(What will Phase 1 include? Phase 2? Phase 3?)* | Phase 1: [primary use case] / Phase 2: [secondary] / Phase 3: [tertiary] |
| **Expected Launch Date?** | [close date from SFDC] |
| **Success Criteria 30 days post launch** | [from Gong/Granola success criteria discussions or TBD] |
| **Channels** *(What channels will they plan to deploy on?)* | [Chat / Email / Voice — based on opp and discussions] |
| **Language Requirements** | [from Gong/Granola language discussions, default English if not mentioned] |
| **APIs / Personalization / Authentication Requirements** | [from Gong/Granola API/auth discussions] |
| **Segmentation Requirements** | [from Gong/Granola segmentation discussions or TBD] |
| **Product promises made to the client / FRs?** | [from Gong/Granola promises or TBD] |
| **Cluster** | [US / US2 / Maple / EU — from region data, default Maple] |
| **Number of Ai Agents** | [from Gong/Granola or TBD] |

### Miscellaneous

| Field | SC Input |
|---|---|
| **Enrolled in Ada Academy** | No (pre-signature) |
| **Security Requirements** | [from Gong/Granola security discussions or TBD] |
| **Link + invites to Demo/Sandbox instance** | [Gong call URL or TBD] |
| **Pilot / Opt out** | TBD |
| **Additional Notes / Risks** | [risks from data] |

## KEY NEXT STEPS

*From Platform Demo — [demo date from Gong/Granola]*

[List every next step found in Gong calls and Granola meeting notes]

1. [next step 1]
2. [next step 2]

## MEETING NOTES (from Granola)

[For each of the last 5 meetings found:]
**[Meeting Title]** ([Date])
[Summary of key points discussed]

---

## CHAT SCOPING

| Field | SC Input |
|---|---|
| **Current Chat Platform** *(What platform do they use for chat today?)* | [from tech stack / Gong — e.g. Intercom, Zendesk Chat, Salesforce Chat] |
| **Monthly Chat Volume** | [from volumes data / Gong] |
| **Current Handoff Setup** *(How does chat currently hand off to human agents?)* | [from Gong/Granola — describe current process] |
| **APIs / Integrations Required** | [from Gong/Granola API discussions] |
| **Segmentation Requirements** | [from Gong/Granola segmentation discussions or TBD] |
| **Chat-Specific Use Cases** *(Any use cases unique to chat?)* | [from use cases discussions] |
| **KB Readiness for Chat** | [assessment from discussions or TBD] |
| **Language Requirements** | [languages for chat — from discussions] |
| **Authentication Requirements** | [from Gong/Granola auth discussions or TBD] |

---

## EMAIL SCOPING

### Email Architecture

| Field | SC Input |
|---|---|
| **Tech Stack** *(Is the system your agents use to receive and respond to emails the same as your chat?)* | [from tech stack — is it same as chat platform or different?] |
| **Email landscape** *(Which email address(es) are your customers emailing?)* | [from Gong/Granola — list email addresses] |
| **Webform** *(Do you have a webform or contact form on your website?)* | [from Gong/Granola discussions — yes/no and describe] |
| **Custom / Filter Incoming Emails** | [from Gong/Granola — how they triage/filter incoming email] |
| **AI Agent / Human support** *(Which email address will the AI Agent respond as?)* | [from Gong/Granola scoping discussions or TBD] |
| **Launch plan** *(Do you require a gradual rollout?)* | [from Gong/Granola rollout discussions or TBD] |

### Email Configuration

| Field | SC Input |
|---|---|
| **Knowledge Base** *(Any additional sources specific to email?)* | [from Gong/Granola KB discussions or TBD] |
| **Use cases** *(Any use cases unique to email vs chat/voice?)* | [from Gong/Granola — list email-specific use cases] |
| **Workflow Mapping** | [from Gong/Granola workflow discussions or TBD] |
| **CC Support** | [from Gong/Granola or TBD] |
| **Metadata** | [from Gong/Granola metadata discussions or TBD] |

### Email Handoffs

| Field | SC Input |
|---|---|
| **Email / Ticketing** | [from Gong/Granola — how email tickets are created/routed] |
| **Routing** | [from Gong/Granola routing discussions or TBD] |

### Additional Requirements

| Field | SC Input |
|---|---|
| **Authentication** | [from Gong/Granola auth discussions or TBD] |
| **Conversation Start** | [from Gong/Granola — how email conversations are initiated] |

---

## VOICE SCOPING

### Voice Architecture

| Field | SC Input |
|---|---|
| **Telephony Provider** | [from Gong/Granola/Glean — e.g. Genesys Cloud, Avaya, Five9, Twilio, Vonage] |
| **CCaaS / Agent System** | [from tech stack / Gong — full CCaaS platform name] |
| **SIP Integration Type** | [from Gong/Granola technical discussions or TBD] |
| **Current IVR** | [from Gong/Granola — describe current IVR menu and flow] |
| **Inbound vs Outbound** | [from use cases — inbound / outbound / both] |
| **Call Volume** | [from volumes data / Gong — monthly or daily] |
| **Current Agent Count** | [from Gong/Granola/Glean volumes data] |
| **Missed Call Rate** | [from Gong/Granola pain points or TBD] |

### Voice Use Cases

| Field | SC Input |
|---|---|
| **Primary Voice Use Case** | [from Gong/Granola — main reason customers call] |
| **Call Categorization / Triage** | [from Gong/Granola — how calls are currently triaged] |
| **Secondary Voice Use Cases** | [from Gong/Granola — additional voice use cases] |

### Voice Technical Requirements

| Field | SC Input |
|---|---|
| **APIs Required for Voice** | [from Gong/Granola API discussions] |
| **DTMF / Dial Pad Input** | [from Gong/Granola technical discussions — yes/no + details] |
| **SMS Capabilities** | [from Gong/Granola — yes/no + details] |
| **Cross-Channel Interoperability** | [from Gong/Granola — any cross-channel needs] |

### Voice Handoffs

| Field | SC Input |
|---|---|
| **Handoff to Human Agents** | [from Gong/Granola — how voice calls hand off to humans] |
| **Routing Requirements** | [from Gong/Granola routing discussions or TBD] |

### Voice Quality & Success Criteria

| Field | SC Input |
|---|---|
| **Voice Quality Feedback from Demo** | [from Gong call discussing voice demo quality] |
| **Success Criteria for Voice** | [from Gong/Granola success criteria discussions or TBD] |
| **Voice-Specific Risks** | [from Gong/Granola risk discussions] |
| **Timeline** | [close date from SFDC] |

---

## GONG CALL HIGHLIGHTS

[For each Gong call found:]
### [Call Title] — [Date]
- **Pain Points:** [list]
- **Use Cases:** [list]
- **Objections:** [list]
- **Tech Mentions:** [list]
- **Sentiment:** [value]
- **Recording:** [URL]
```

### 6c — Create the Notion page
```
notion-create-pages({
  pages: [{
    properties: { title: "PS Handoff — [Account Name] — [YYYY-MM-DD]" },
    content: "[full markdown from 6b above, with all real data substituted in]"
  }]
})
```

Save the returned Notion page URL.

---

## Step 7: Report Back to the User

Tell the user:
- ✅ `.docx` file name and full path (copied to clipboard)
- ✅ Notion page URL
- Summary of what was auto-filled vs left as TBD — list which fields still need manual input

---

## Important Notes

- **ALL scoping sections are always included** — General, Chat, Email, and Voice are generated every time, regardless of what channels the opp shows. Never skip a section.
- **Fill from data, not with TBD** — Every table cell should be populated from Gong transcripts, Granola meetings, or Glean context. TBD is only for genuinely unknown fields with zero data found.
- **The scoping tables are the most important part of this document** — the Chat, Email, and Voice scoping tables are what PS uses to build the implementation. They must be filled out.
- **Always publish to Notion (Step 6)** — this is not optional. Every run produces both a `.docx` AND a Notion page. Never skip or ask the user about it.
- **Never hallucinate** account details — TBD is always better than a wrong answer
- **Output dir** defaults to `./ps-knowledge-transfer/` relative to the user's current working directory
- **Script location** is `${CLAUDE_PLUGIN_ROOT}/scripts/ps_doc_skill.py` — this resolves correctly regardless of where the plugin is installed
- **Gong via Glean** — there is no standalone Gong MCP; use `mcp__glean__search` with `app: "gong"` and `mcp__glean__chat` to search indexed Gong transcripts. If Glean doesn't return Gong results, note it as TBD — don't skip the attempt
- **Gong data enriches scoping fields directly** — extract telephony provider, email system, chat platform, volumes, IVR setup, use cases, handoff process from Gong transcripts and put them into the correct scoping table cells
