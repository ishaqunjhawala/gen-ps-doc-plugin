---
description: Generate a Pre-Sales → PS Knowledge Transfer .docx for any account (fetches live data from Glean, Granola, Gong & Slack)
argument-hint: "<account name> [general] [email] [voice]"
allowed-tools: Bash, Read, mcp__glean__chat, mcp__glean__search, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__query_granola_meetings, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__list_meetings, mcp__b64aba26-624b-471d-a4c9-bc9c8ca47541__get_meetings, mcp__28d90b4b-0f1f-4d6d-96f3-e7a6107a9d3c__slack_search_public_and_private, mcp__28d90b4b-0f1f-4d6d-96f3-e7a6107a9d3c__slack_read_user_profile, mcp__ada__get_ada_metric, mcp__ada__get_ada_configuration
---

# Generate PS Knowledge Transfer Doc

Generate a formatted Pre-Sales → PS Knowledge Transfer `.docx` by gathering live data from Glean (Salesforce + Gong), Granola (meeting notes), and Slack. No cache or config file needed.

## Arguments

The user invoked this command with: $ARGUMENTS

Parse the arguments:
- **Account name** (required) — first argument, e.g. `"Grow Therapy"`. If not provided, ask the user.
- **Sections** (optional) — any combination of `general`, `email`, `voice`. Defaults to `general` only.

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
> "What do we know about [account name] as an Ada prospect or customer? Return: company overview, HQ location, timezone, tech stack (CRM, ticketing, CCaaS tools), key contacts and their roles, primary use case, secondary use cases, business drivers, known risks or blockers, API/architecture requirements, key volumes (monthly chat/email/voice conversations, number of agents)."

### 2c — Meeting Notes (via Granola)
- Call `query_granola_meetings` with query: `"[account name] meeting notes action items demo discovery"`
- Then call `get_meetings` for the 5 most recent account-related meetings to get full details

### 2d — Gong Call Transcripts (via Glean)
Glean indexes Gong calls — search for the most recent calls with this account:

Search 1 (keyword search for recent calls):
> Use `mcp__glean__search` with query: `"[account name]"`, app filter: `"gong"`, limit: 5, sort by recency

Search 2 (semantic search for key details):
> Use `mcp__glean__chat` with message: "Summarize the most recent Gong calls with [account name]. Extract: key pain points discussed, use cases mentioned, objections raised, technical requirements, pricing discussions, sentiment, next steps agreed, and any Gong call URLs."

From the Gong results extract:
- **Call summaries** — what was discussed in each call (pain points, use cases, objections)
- **Tech stack mentions** — any tools, systems, or integrations mentioned
- **Volume/scale data** — conversation volumes, agent counts, ticket volumes
- **Sentiment & buying signals** — champion strength, executive engagement, urgency
- **Gong call URLs** — direct links to individual call recordings
- **Most recent call date** — for the `demo_recap` field

### 2e — Ada Bot (skip if no bot mentioned)
- If Glean context mentions an Ada bot or demo instance for this account:
  - Call `get_ada_configuration` — playbooks, actions, custom instructions
  - Call `get_ada_metric` for `resolution_rate` and `csat_rate` (last 7 days)

---

## Step 3: Assemble the Data Dictionary

From all gathered data, build this Python dict. Use `"TBD"` for anything not found — never guess or hallucinate account details.

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
        },
        "key_volumes": {
            "chat_monthly": "<number>",
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
        "product_channels": "<Chat / Chat; Email / etc.>",
        "arr": <number>,
        "stage": "<stage name>",
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

## Step 4: Check python-docx and Run the Generator

### 4a — Ensure python-docx is installed
```bash
python3 -c "import docx" 2>/dev/null || pip install python-docx
```

### 4b — Write the data to a temp JSON file and run the script
```bash
# Write data dict to a temp file (avoids shell escaping issues with large JSON)
python3 -c "
import json, sys
data = <PASTE DATA DICT HERE AS PYTHON LITERAL>
with open('/tmp/ps_doc_data.json', 'w') as f:
    json.dump(data, f)
"

# Run the generator
python3 "${CLAUDE_PLUGIN_ROOT}/scripts/ps_doc_skill.py" \
  --account "<account name>" \
  --sections general [email] [voice] \
  --sc-name "<SC name>" \
  --output-dir "./ps-knowledge-transfer" \
  --data-file "/tmp/ps_doc_data.json"
```

The script prints `SUCCESS: <filepath>` on completion.

### 4c — Copy path to clipboard (macOS)
```bash
echo "<filepath>" | pbcopy
```

---

## Step 5: Report Back to the User

Tell the user:
- ✅ File name and full path
- Which sections were included (General / Email / Voice)
- Key data that was auto-filled vs left as TBD
- Offer: *"Say **'enhance ps doc for [account]'** and I'll query Granola transcripts and Gong calls to fill in the TBD fields with richer detail."*

---

## Important Notes

- **Always include `general`** — it's the base section for every Ada deal
- **Add `email`/`voice`** only if the user specified them, or if opp data shows those channels in scope
- **Never hallucinate** account details — TBD is always better than a wrong answer
- **Output dir** defaults to `./ps-knowledge-transfer/` relative to the user's current working directory
- **Script location** is `${CLAUDE_PLUGIN_ROOT}/scripts/ps_doc_skill.py` — this resolves correctly regardless of where the plugin is installed
- **Gong via Glean** — there is no standalone Gong MCP; use `mcp__glean__search` with `app: "gong"` and `mcp__glean__chat` to search indexed Gong transcripts. If Glean doesn't return Gong results, note it as TBD — don't skip the attempt
- **Gong data enriches**: pain points, tech stack, volumes, objections, and sentiment — prioritise these for the `business_drivers`, `risks`, `key_volumes`, and `key_architecture` fields
