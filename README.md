# gen-ps-doc — PS Knowledge Transfer Doc Generator

A Claude Code plugin that generates a **Pre-Sales → PS Knowledge Transfer Google Doc** for any account by gathering live data from Glean (Salesforce + Gong + Gmail), Granola, and Slack.

No local files, no config, no dependencies to install. Output goes straight to Google Drive.

---

## What It Does

When you run `/gen-ps-doc "Account Name"`, Claude will:

1. **Fetch live data** in parallel — Salesforce opp (via Glean), account context (via Glean), Gong call transcripts + email exchanges (via Glean), Gmail threads (via Glean), recent meeting notes (via Granola), and your name (via Slack profile)
2. **Synthesize scoping answers** — mines all gathered data to populate every scoping field
3. **Build the doc** — structured scoping form with all four sections: General, Chat, Email, and Voice
4. **Create a Google Doc** in your "PS Hand Over Docs" Google Drive folder and return the URL

---

## Prerequisites

### MCP Tools Connected in Claude Code

| Tool | Used For |
|---|---|
| **Glean** | Salesforce opp data, account context, Gong call transcripts, Gong email exchanges, and Gmail threads |
| **Granola** | Recent meeting notes |
| **Slack** | Your name/profile (SC field) |
| **Google Workspace** | Create the Google Doc in Drive |
| **Ada** *(optional)* | Bot metrics if account has a connected bot |

> **Note on Gong:** There is no standalone Gong MCP. Gong calls and email exchanges are accessed via Glean's Gong connector — make sure your Glean workspace has Gong indexed.

> **Note on Gmail:** Gmail threads are accessed via Glean's Gmail connector (`app: "gmail"`). These often contain the most explicit written scoping answers from prospects — make sure Gmail is indexed in your Glean workspace.

---

## Installation

### Option A — From GitHub (run outside a Claude session)

```bash
claude plugin install https://github.com/ishaqunjhawala/gen-ps-doc-plugin
```

### Option B — Manual (always works)

```bash
git clone https://github.com/ishaqunjhawala/gen-ps-doc-plugin
cp gen-ps-doc-plugin/commands/gen-ps-doc.md ~/.claude/commands/gen-ps-doc.md
```

### Post-install: Add your Granola & Slack tool IDs (optional but recommended)

Granola and Slack MCP tools use instance-specific IDs that are unique to each Claude setup. The command works without them — Claude will just ask for permission on first use each session. If you want to skip that prompt, add your IDs to the `allowed-tools` line in `~/.claude/commands/gen-ps-doc.md`:

1. Run `/mcp` in Claude Code to see all connected tools
2. Find your Granola tools: `query_granola_meetings`, `list_meetings`, `get_meetings`
3. Find your Slack tools: `slack_search_public_and_private`, `slack_read_user_profile`
4. Note the full tool name including the UUID prefix (e.g. `mcp__b64aba26-xxxx__query_granola_meetings`)
5. Add them to the `allowed-tools` line in your local copy of the command

---

## Usage

```
/gen-ps-doc "Account Name"
```

### Examples

```
/gen-ps-doc "Acme Corp"
/gen-ps-doc "Grow Therapy"
/gen-ps-doc "American Airlines"
```

---

## Output

- **Google Doc** created in your "PS Hand Over Docs" Google Drive folder
- **Title**: `PS Handoff — [Account Name] — [YYYY-MM-DD]`
- **Sections**: General Scoping, Chat Scoping, Email Scoping, Voice Scoping — always all four
- For channels not in Phase 1 scope, a ⚠️ callout is shown above the table and rows are left as TBD for future use

### What gets auto-filled

| Field | Source |
|---|---|
| Client Overview, Business Drivers | Glean (Salesforce / company context) |
| Key Contacts & Roles | Glean + Gong |
| SFDC Opp URL, Close Date, ARR | Glean (Salesforce) |
| Agent Tech Stack | Glean + Gong |
| Chat platform, volumes, handoff setup | Gong transcripts + Gmail + Granola |
| Email system, routing, webform | Gong transcripts + Gmail + Granola |
| Telephony provider, CCaaS, IVR | Gong transcripts + Gmail + Granola |
| Primary & Secondary Use Cases | Glean + Gong + Gmail |
| Risks & Objections | Gong transcripts + Gmail |
| Next Steps | Gong + Gmail + Granola |
| Written commitments & go-live dates | Gmail threads |
| Meeting Notes | Granola (last 5 meetings) |
| SC Name | Slack profile |

Fields not found anywhere are marked **TBD** — never hallucinated.

---

## File Structure

```
gen-ps-doc-plugin/
├── .claude-plugin/
│   └── plugin.json        # Plugin manifest
├── commands/
│   └── gen-ps-doc.md      # Slash command definition
├── README.md
└── .gitignore
```

---

## Troubleshooting

**Glean returns no Salesforce data**
Make sure your Glean workspace has the Salesforce connector enabled and you have access.

**Granola returns no meetings**
Granola only has notes for meetings recorded in the Granola app.

**Gong returns no results**
Make sure your Glean workspace has the Gong connector enabled and calls are indexed. If not, those fields will be marked TBD.

**Gmail returns no results**
Make sure your Glean workspace has the Gmail connector enabled. If not, those fields will be marked TBD.

**Google Doc not created**
Make sure the Google Workspace MCP is connected in Claude Code and you have write access to Google Drive.

**SC name shows as "SC"**
Slack profile lookup failed — edit the SC name directly in the Google Doc.
