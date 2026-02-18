# gen-ps-doc — PS Knowledge Transfer Doc Generator

A Claude Code plugin that generates a formatted **Pre-Sales → PS Knowledge Transfer `.docx`** for any account by gathering live data from Glean (Salesforce + Gong), Granola, and Slack.

No cache files, no config, no TUI dependency. Works for any SC with the right MCP tools connected.

---

## What It Does

When you run `/gen-ps-doc "Account Name"`, Claude will:

1. **Fetch live data** in parallel — Salesforce opp (via Glean), account context (via Glean), Gong call transcripts (via Glean), recent meeting notes (via Granola), and your name (via Slack profile)
2. **Assemble** a structured data dictionary from everything found
3. **Generate** a formatted `.docx` using the official Ada PS Knowledge Transfer template structure
4. **Save** to `./ps-knowledge-transfer/` in your working directory
5. **Publish** a matching Notion page and return the URL

---

## Prerequisites

### 1. Python + python-docx

```bash
pip install python-docx
```

### 2. MCP Tools Connected in Claude Code

| Tool | Used For |
|---|---|
| **Glean** | Salesforce opp data, account context, and **Gong call transcripts** |
| **Granola** | Recent meeting notes |
| **Slack** | Your name/profile (SC field) |
| **Notion** | Publish a live page alongside the .docx |
| **Ada** *(optional)* | Bot metrics if account has a connected bot |

> **Note on Gong:** There is no standalone Gong MCP. Gong calls are accessed via Glean's Gong connector — make sure your Glean workspace has Gong indexed.

---

## Installation

### Option A — Claude plugin install (recommended)

```bash
claude plugin install https://github.com/ishaqunjhawala/gen-ps-doc-plugin
```

### Option B — Local install

```bash
git clone https://github.com/ishaqunjhawala/gen-ps-doc-plugin
claude plugin install ./gen-ps-doc-plugin
```

### Option C — Manual

Copy the contents of `commands/` to `~/.claude/commands/` and `scripts/` to `~/.claude/commands/`.

---

## Usage

```
/gen-ps-doc "Account Name"
/gen-ps-doc "Account Name" general email
/gen-ps-doc "Account Name" general email voice
```

### Arguments

| Argument | Required | Description |
|---|---|---|
| Account name | ✅ Yes | Name of the account, e.g. `"Grow Therapy"` |
| Sections | ❌ No | `general` (default), `email`, `voice` — space-separated |

### Examples

```
/gen-ps-doc "Acme Corp"
/gen-ps-doc "Bombas" general email
/gen-ps-doc "American Airlines" general email voice
```

---

## Output

- **`.docx` file**: `./ps-knowledge-transfer/PS_Knowledge_Transfer_<account>_<date>.docx`
- **Notion page**: Created automatically — URL returned at the end
- **Sections**: General Scoping (always), + Email and/or Voice if specified
- **File path** copied to clipboard automatically (macOS)

### What gets filled in automatically

| Field | Source |
|---|---|
| Client Overview, Business Drivers | Glean (Salesforce / company context) |
| Key Contacts & Roles | Glean |
| SFDC Opp URL, Close Date, ARR | Glean (Salesforce) |
| Tech Stack / Architecture | Glean + Gong transcripts |
| Primary & Secondary Use Cases | Glean + Gong transcripts |
| Risks & Objections | Glean + Gong transcripts |
| Volume / Scale Data | Glean + Gong transcripts |
| Sentiment & Buying Signals | Gong transcripts |
| Next Steps | Glean + Gong transcripts |
| Demo recap + Gong call URLs | Gong (via Glean) |
| Meeting Notes section | Granola (last 5 meetings) |
| SC Name | Slack profile |

Fields not found are marked **TBD** — never hallucinated.

---


## File Structure

```
gen-ps-doc-plugin/
├── .claude-plugin/
│   └── plugin.json        # Plugin manifest
├── commands/
│   └── gen-ps-doc.md      # Slash command definition
├── scripts/
│   └── ps_doc_skill.py    # Standalone .docx generator (no TUI dependency)
├── README.md
└── .gitignore
```

---

## Troubleshooting

**`python-docx` not found**
```bash
pip install python-docx
# or
pip3 install python-docx
```

**Glean returns no Salesforce data**
Make sure your Glean workspace has the Salesforce connector enabled and you have access.

**Granola returns no meetings**
Granola only has notes for meetings that were recorded in the Granola app.

**Gong returns no results**
Make sure your Glean workspace has the Gong connector enabled and calls are indexed. If not available, those fields will be marked TBD.

**Notion page not created**
Make sure the Notion MCP is connected in Claude Code. The page will be created at workspace level if no "PS Knowledge Transfer" parent page is found.

**SC name shows as "SC"**
Slack profile lookup failed — Claude will still generate the doc and you can edit the name in the `.docx`.
