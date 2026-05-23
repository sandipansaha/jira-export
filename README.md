# bin/jira — Jira Ticket Exporter

> **Export Jira tickets to Markdown with attachments, comments, and full ADF rendering.**
>
> `jira-to-markdown` `jira-exporter` `jira-ticket-exporter` `jira-backup` `atlassian-document-format` `bash-script`
> `ai-context` `ai-training-data` `ai-readable-format` `ai-readable-jira-ticket` `ai-readable-markdown`
> `ai-readable-jira-ticket-attachments` `ai-readable-jira-ticket-comments` `ai-readable-jira-ticket-adf`

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![Shell: Bash](https://img.shields.io/badge/Shell-Bash-blue.svg)
![Dependencies: curl jq](https://img.shields.io/badge/Dependencies-curl%20%7C%20jq-lightgrey.svg)
![Security: Redaction enabled](https://img.shields.io/badge/Security-Redaction%20enabled-red.svg)

Export any Jira Cloud ticket to a clean, self-contained Markdown file. Designed
as a `bin/` command that lives alongside your project scripts — no dependencies
beyond `curl` and `jq`, no global install required.

The exported file is designed to be fed directly to AI coding agents (Junie,
Claude, Copilot) as context for a task — but it's equally useful for offline
reading, documentation, or changelogs.

---

## Table of Contents

1. [Features](#features)
2. [🔒 Security](#-security)
3. [What it does](#what-it-does)
4. [Output structure](#output-structure)
5. [Requirements](#requirements)
6. [Installation](#installation)
7. [Getting a Jira API token](#getting-a-jira-api-token)
8. [Configuration](#configuration)
9. [Finding your custom field IDs](#finding-your-custom-field-ids)
10. [Usage](#usage)
11. [What the Markdown file contains](#what-the-markdown-file-contains)
12. [🟢 Using this as a standalone tool — **no coding required**](#using-this-as-a-standalone-tool-no-coding-required)
13. [How it works internally](#how-it-works-internally)
14. [Security notes (detailed)](#security-notes)
15. [Troubleshooting](#troubleshooting)
16. [License](#license)

---

## Features

**Complete ticket export — everything in one file**
Fetches the full ticket: description, all custom fields, all comments
(paginated — nothing cut off), subtasks, linked issues, web resource links, and all
attachments. Every piece of information on the ticket ends up in a single
self-contained Markdown file.

**Fully configurable custom fields**
Define any number of custom field sections in `.env.jira` using a readable
JSON format. Each entry becomes its own `##` heading in the output,
in the order you specify. No script editing required to add or change fields.

**All attachment types downloaded — with risk classification**
Images are embedded inline in the Markdown. Every other safe file type is
downloaded into `attachments/` and linked locally. High-risk file types are
blocked; tabular files are flagged. Use `--no-attachments` to skip all downloads.

**Deep ADF rendering**
Atlassian's internal rich-text format (ADF) is converted to clean Markdown
entirely in `jq` — no Node.js, no Python, no external parser. Covers all
standard elements plus panels, task lists, decision items, expand blocks,
blockquotes, emoji, status badges, date nodes, nested tables, and inline links.

**Zero-dependency single script**
One Bash file. Requires only `curl` and `jq`. No build step, no package manager,
no global install — copy the script into your project's `bin/` and it works.

**Agent-directory output structure**
Exports into a configurable dot-directory at the project root (default `.jira/`),
with one subdirectory per ticket. Works like `.junie/` or `.claude/` — easy to
gitignore, easy to hand to an AI coding agent as task context.

---

## 🔒 Security

> **Sensitive content redaction is on by default.** Every exported Markdown file
> is filtered before it is written to disk using pattern-based rules. This applies
> to all users — developers, PMs, and anyone else running the script.
>
> **Important:** Redaction is best-effort and cannot guarantee detection of every
> possible secret or sensitive value format.

| What is protected | How |
|---|---|
| API keys, tokens, Bearer / Basic auth | Redacted when they match supported patterns → `[REDACTED — API key]` etc. |
| Passwords in `key=value` format | Redacted when they match supported patterns → `[REDACTED — password]` |
| Database URLs with credentials | Redacted when they match supported patterns → `proto://user:[REDACTED]@host` |
| AWS access key IDs (`AKIA...`) | Redacted when they match supported patterns → `[REDACTED — AWS access key]` |
| Private key / certificate headers | Redacted when they match supported patterns → `[REDACTED — private key or certificate]` |
| Email addresses | Redacted by default → `[REDACTED — email]` |
| Card numbers (13–16 digits) | Redacted by default → `[REDACTED — possible card number]` |
| High-risk attachments (`.pem`, `.key`, `.sql`, `.env`, `.bak`, ...) | **Not downloaded** — listed as ⛔ in output |
| Tabular attachments (`.csv`, `.xlsx`, `.tsv`, ...) | Downloaded but flagged ⚠️ for review |

**PII redaction is on by default.** Set `JIRA_REDACT_PII=false` in `.env.jira`
to disable email and card number redaction while keeping all credential redaction
active. Credential redaction cannot be disabled.

Every export prints the number of redacted values to the terminal so you always
know when something was stripped.

---

## What it does

Running `bin/jira JIRA-123` will:

- Fetch the full ticket from the Jira Cloud REST API
- Convert the description from Atlassian's internal format (ADF) to clean Markdown
- Extract and render: description, acceptance criteria, technical notes, QA notes
- Download **all** attachments — images are embedded inline in the Markdown; other files (PDFs, ZIPs, documents) are downloaded to `attachments/` and linked locally; high-risk file types are blocked and flagged
- Fetch all comments with author names and timestamps — paginated, so nothing is missed
- Collect all subtasks (with summary, status, and direct link)
- Collect all child work items for Epics (all issues associated with the Epic)
- Collect all linked issues (blocks, is blocked by, relates to, clones, etc.)
- Collect all web links / resource URLs added to the ticket
- Write everything into a single self-contained `.md` file

The whole thing runs in about 2–5 seconds for a typical ticket.

---

## Output structure

Each ticket gets its own directory, named by the ticket key, inside a
configurable agent directory at the project root:

```
.jira/                   ← configured by JIRA_DIR in .env.jira (default: .jira)
└── JIRA-123/             ← one folder per ticket, named by ticket key
    ├── JIRA-123.md       ← full ticket as Markdown
    └── attachments/     ← all downloaded attachments (images, PDFs, docs, etc.)
        ├── screenshot.png
        └── design-mockup.jpg
```

This follows the same convention as agent context directories like `.junie/` or
`.claude/` — one directory per working context, ignored by git.

---

## Requirements

| Tool   | How to check          | How to install          |
|--------|-----------------------|-------------------------|
| `bash` | `bash --version`      | Pre-installed on macOS  |
| `curl` | `curl --version`      | Pre-installed on macOS  |
| `jq`   | `jq --version`        | `brew install jq`       |

The script checks for `curl` and `jq` on startup and will tell you if either
is missing.

---

## Installation

### Step 1 — Copy the script into your project

```bash
cp bin/jira /path/to/your-project/bin/jira
```

### Step 2 — Make it executable

```bash
chmod +x /path/to/your-project/bin/jira
```

### Step 3 — Add the config file

Copy the sample config and fill in your credentials (see the next two sections
for how to get them):

```bash
cp .env.jira.sample /path/to/your-project/.env.jira
```

Open `.env.jira` in your editor and fill in at minimum:

```dotenv
JIRA_URL=https://yourcompany.atlassian.net
JIRA_EMAIL=you@company.com
JIRA_API_TOKEN=your_api_token_here
```

### Step 4 — Add to `.gitignore`

You never want to commit credentials or generated ticket files:

```bash
# Add to your project's .gitignore
echo ".env.jira" >> .gitignore
echo ".jira/"    >> .gitignore
```

> ⚠️ **Important:** `.env.jira` contains your API token. If you commit it to
> git and the repository is public (or gets leaked), anyone can read your Jira
> data. Always keep it in `.gitignore`.

### Step 5 — Test it

```bash
bin/jira --help
```

Then try a real ticket:

```bash
bin/jira JIRA-123
```

---

## Getting a Jira API token

You cannot use your Jira password for API access. Jira requires a dedicated API
token. Here is how to create one:

### Step 1 — Go to your Atlassian account security page

Open this URL in your browser:
[https://id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens)

You may be asked to log in.

### Step 2 — Create a new token

Click **"Create API token"**.

Give it a label that will help you remember what it is for, e.g.:
```
bin/jira — Jira export script
```

Click **"Create"**.

### Step 3 — Copy the token immediately

Atlassian **shows the token only once**. Copy it now before closing the dialog.
If you miss it, you will need to revoke the token and create a new one.

### Step 4 — Put it in your `.env.jira`

```dotenv
JIRA_API_TOKEN=ATATTxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> **What is this token used for?**
> The script uses it to make HTTP requests to Jira's REST API on your behalf.
> It is sent as a Basic Auth header:
> `Authorization: Basic base64(your_email:your_token)`
> This is the standard, documented Atlassian authentication method.

---

## Configuration

All configuration lives in `.env.jira` at the project root. The script also
reads from `.env` as a fallback, so if your project already has Jira vars in
`.env`, they will be picked up. Values in `.env.jira` take priority over `.env`.

### Required variables

| Variable          | Description                                          | Example                                  |
|-------------------|------------------------------------------------------|------------------------------------------|
| `JIRA_URL`        | Your Atlassian Cloud base URL, no trailing slash     | `https://mycompany.atlassian.net`        |
| `JIRA_EMAIL`      | The email address you use to log in to Jira          | `you@yourcompany.com`                    |
| `JIRA_API_TOKEN`  | The API token you generated above                    | `ATATTxxxxxxxxxxxxxxxxxxx`               |

### Optional variables

| Variable                 | Description                                                            | Default               |
|--------------------------|------------------------------------------------------------------------|-----------------------|
| `JIRA_DIR`               | Directory name (or absolute path) for exported tickets                 | `.jira`               |
| `JIRA_FETCH_SUBTASKS_RECURSIVELY` | Whether to recursively fetch details for all subtasks and child work items (for Epics). | `false` |
| `JIRA_CUSTOM_FIELDS`     | JSON array of `{"label": "...", "id": "..."}` objects. Each becomes its own `##` section in the exported Markdown, rendered in the order listed. | See below |
| `JIRA_MEDIUM_RISK_ATTACHMENTS` | Comma-separated list of file extensions to flag as ⚠️. | `xml, json, csv, xlsx, xls, tsv, ods` |

#### `JIRA_CUSTOM_FIELDS` format

Recommended JSON format (supports multi-line in `.env.jira`):

```json
JIRA_CUSTOM_FIELDS='[
  {"label": "Acceptance Criteria", "id": "customfield_10016"},
  {"label": "Technical Notes",     "id": "customfield_10040"},
  {"label": "QA Notes",            "id": "customfield_10041"}
]'
```

- Each entry is a JSON object with `label` and `id` keys
- The label becomes the `##` heading in the Markdown output
- Field IDs always look like `customfield_NNNNN`
- Add as many entries as your project needs — there is no limit
- If this variable is not set, no custom field sections are exported — only the Description is included
- **Tip:** Use `bin/jira --list-fields` to find the correct IDs for your project.

### Full `.env.jira` example

```dotenv
# Required
JIRA_URL=https://mycompany.atlassian.net
JIRA_EMAIL=you@yourcompany.com
JIRA_API_TOKEN=ATATTxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Output directory — works like .junie or .claude
JIRA_DIR=.jira

# Custom field sections — each becomes a ## heading in the Markdown output.
# Add or remove entries to match your Jira project's fields.
# Add more fields as needed.
JIRA_CUSTOM_FIELDS='[
  {"label": "Acceptance Criteria", "id": "customfield_10016"},
  {"label": "Technical Notes",     "id": "customfield_10040"},
  {"label": "QA Notes",            "id": "customfield_10041"}
]'

## Security & redaction
##
## JIRA_MEDIUM_RISK_ATTACHMENTS: Comma-separated list of file extensions that should be
## flagged as ⚠️ in the exported Markdown. Defaults to: xml, json, csv, xlsx, xls, tsv, ods
# JIRA_MEDIUM_RISK_ATTACHMENTS="xml, json, csv, xlsx, xls, tsv, ods"

## Credentials, tokens, API keys, and passwords are redacted when they match supported patterns.
## Redaction is best-effort and cannot guarantee detection of every sensitive format.
## PII redaction (emails, card numbers) is on by default. Set to false to disable.
JIRA_REDACT_PII=true

## Fetch each subtask and child work item (for Epics) recursively (default: false)
JIRA_FETCH_SUBTASKS_RECURSIVELY=false
```

---

## Finding your custom field IDs

> **Why do I need to do this?**
> Jira uses internal field IDs like `customfield_10016` for custom fields such
> as "Acceptance Criteria". These IDs are different on every Jira instance —
> what is `customfield_10016` at one company might be `customfield_10093` at
> another. The defaults in `.env.jira.sample` are common guesses but may not
> match your Jira.

### Option A — The easiest way (built-in command)

The script can fetch and list all custom fields available in your Jira instance. This is the recommended way to find the IDs you need.

```bash
bin/jira --list-fields
# or simply
bin/jira -l
```

You will see a table of IDs and Names:

```
ID                   | Name
-------------------- | ----------------------------------------
customfield_10016    | Acceptance Criteria
customfield_10040    | Technical Notes
customfield_10041    | QA Notes
...
```

Copy those IDs and add them to `JIRA_CUSTOM_FIELDS` in your `.env.jira` using the JSON format.

### Option B — Jira UI

1. Open any ticket in Jira
2. Click the three dots (⋯) menu → **"View issue fields"** or go to
   **Project Settings → Fields**
3. Find the field you want and look for its "Field ID" — it is shown in the URL
   when you click on the field name in settings

---

## Usage

All commands are run from the project root (where `bin/` lives):

### Export a ticket recursively

```bash
bin/jira JIRA-123 --recursive
```

Fetches the main ticket and then recursively exports every subtask and child work item (for Epics) listed on it. Each associated ticket gets its own folder and Markdown file.

### Export without downloading attachments

```bash
bin/jira JIRA-123 --no-attachments
```

Skips downloading all attachments. The Markdown file still lists them but links
to their remote Jira URLs instead of local files. Useful if you only need the
text content or are working offline.

### Export and open immediately

```bash
bin/jira JIRA-123 --open
```

Exports the ticket and then opens the `.md` file in your default application
(on macOS this is typically your Markdown viewer or text editor).

### Get help

```bash
bin/jira --help
```

### List available custom fields

```bash
bin/jira --list-fields
# or
bin/jira -l
```

Fetches all fields from your Jira instance and displays those marked as "custom". Use this to find the `fieldId` for your configuration.

### Where is the output?

After running `bin/jira JIRA-123`, look in:

```
.jira/JIRA-123/JIRA-123.md
```

Or if you customised `JIRA_DIR`:

```
.<your-dir>/JIRA-123/JIRA-123.md
```

---

## What the Markdown file contains

The exported `.md` file is structured into clearly labelled sections:

| Section                  | What it contains                                                      |
|--------------------------|-----------------------------------------------------------------------|
| **Title**                | Ticket key and summary (`# JIRA-123: My ticket title`)                 |
| **Metadata**             | Type, status, priority, reporter, assignee, dates, labels, components, sprint, epic, story points |
| **Description**          | Full ticket description, converted from Atlassian's rich text to Markdown |
| **Acceptance Criteria**  | The AC field — often a checklist of what "done" looks like            |
| **Technical Notes**      | Developer-facing notes: approach, constraints, architecture decisions  |
| **Subtasks**             | List of subtasks including their key, summary, and status              |
| **Child work items**      | List of all issues associated with an Epic                             |
| **Linked Issues**        | All related tickets: blocks, is blocked by, relates to, clones, duplicates |
| **Resources / Web Links**| External URLs added to the ticket (Figma designs, staging links, docs, etc.) |
| **Attachments**          | Non-image files (PDFs, documents) listed with download links          |
| **Comments**             | Every comment on the ticket, in chronological order, with author and date |

The description and all rich-text fields are converted from ADF (Atlassian
Document Format) to standard Markdown, including:

- Headings (`#`, `##`, `###`)
- Bold, italic, inline code, strikethrough
- Bullet and numbered lists
- Code blocks with language identifiers (e.g. ` ```php `)
- Tables
- Blockquotes and info/warning/note panels
- Task lists (checkboxes)
- Mentions (`@username`)
- Status badges and date fields
- Inline and block links
- Images (downloaded to `attachments/` and embedded with `![alt](attachments/file.png)`)

---

## 🟢 Using this as a standalone tool — no coding required

This section is for **Project Managers, Product Owners, and anyone who works
with Jira tickets** but doesn't write code day-to-day. You can use this script
entirely on your own — no developer help needed after the one-time setup.

### What you get

Running the script on any Jira ticket gives you a single `.md` (Markdown) file
containing everything on that ticket: the description, acceptance criteria,
all comments, linked tickets, attachments, and any custom fields your team uses.
That file can be uploaded or pasted directly into any AI tool.

### One-time setup (about 10 minutes)

You only do this once. After that, exporting any ticket takes a single command.

**1. Open Terminal**

On a Mac, press `Cmd + Space`, type `Terminal`, and press Enter.

**2. Check you have the required tools**

Paste this and press Enter:

```bash
curl --version && jq --version
```

If you see version numbers for both, you are ready. If `jq` is missing, install
it with [Homebrew](https://brew.sh):

```bash
brew install jq
```

If Homebrew itself is not installed, paste this first:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

**3. Download the script**

Create a folder for the tool and download the script into it:

```bash
mkdir -p ~/tools/jira-export/bin
curl -o ~/tools/jira-export/bin/jira https://raw.githubusercontent.com/your-repo/bin/jira
chmod +x ~/tools/jira-export/bin/jira
```

> Replace the URL with the actual raw file URL from your GitHub repository.

**4. Get your Jira API token**

Follow the steps in [Getting a Jira API token](#getting-a-jira-api-token) —
it takes about two minutes and only needs to be done once.

**5. Create your config file**

```bash
cp ~/tools/jira-export/.env.jira.sample ~/tools/jira-export/.env.jira
open ~/tools/jira-export/.env.jira
```

This opens the file in TextEdit. Fill in your three values and save:

```
JIRA_URL=https://yourcompany.atlassian.net
JIRA_EMAIL=you@yourcompany.com
JIRA_API_TOKEN=paste-your-token-here
```

### Exporting a ticket

Open Terminal and run:

```bash
cd ~/tools/jira-export
bin/jira PROJ-123
```

Replace `PROJ-123` with your actual ticket number (e.g. `WORK-456`, `DEV-789`).

The exported file will be at:

```
~/tools/jira-export/.jira/PROJ-123/PROJ-123.md
```

To open it immediately after export, add `--open`:

```bash
bin/jira PROJ-123 --open
```

This opens the `.md` file in your default text editor or Markdown viewer.

### Feeding it to an AI tool

Once you have the `.md` file, here is how to use it with the most common AI tools:

**ChatGPT / Claude / Gemini (web)**
1. Open the `.md` file in TextEdit or any text editor (`Cmd + O` to open)
2. Select all (`Cmd + A`) and copy (`Cmd + C`)
3. Paste it into the chat alongside your question

Or — even easier — most AI tools have a file upload button. Just upload the
`.md` file directly and ask your question.

**Notion AI**
1. Create a new page in Notion
2. Type `/` and choose **"Import"** → **"Text & Markdown"**
3. Upload the `.md` file — Notion converts it to a formatted page automatically
4. Use Notion AI on that page as normal

**Microsoft Copilot**
Upload the `.md` file as an attachment in your Copilot chat, or paste the
contents directly into the conversation.

**Claude Projects / ChatGPT Projects**
Upload the `.md` file to your project's knowledge base. The AI will have the
full ticket context for every conversation in that project — useful when working
on a single epic or feature over several days.

### Practical examples

> *"Here is my Jira ticket. Write a summary email to send to the client
> explaining what we are building and the expected timeline."*

> *"Based on this ticket, what are the main risks I should raise in our
> planning meeting?"*

> *"The acceptance criteria in this ticket are vague. Can you help me rewrite
> them as clear, testable statements?"*

> *"Read through the comments on this ticket and give me a summary of the
> decisions that were made and any unresolved questions."*

> *"Write a sprint review slide for this ticket — one sentence on what was
> built and why it matters."*

### Tips

- **Ticket too long to paste?** Use the file upload option instead — every
  major AI tool supports it.
- **Want just the text, no attachments?** Add `--no-attachments` to the
  command: `bin/jira PROJ-123 --no-attachments`
- **Export multiple tickets at once?** Use the `--recursive` flag to export
  a ticket and all its subtasks/child work items automatically.
- **Something went wrong?** Check the [Troubleshooting](#troubleshooting)
  section, or ask your developer to run it once and confirm the setup is
  working.

---

## How it works internally

This is useful context if you ever need to debug or extend the script.

```
bin/jira JIRA-123
     │
     ├─ 1. Load .env.jira (falls back to .env)
     ├─ 2. Validate config and issue key format
     │
     ├─ 3. GET /rest/api/3/issue/JIRA-123?fields=*all
     │      └─ All fields in one request, including custom fields
     │
     ├─ 4. GET /rest/api/3/issue/JIRA-123/comment (paginated, 100/page)
     │      └─ Fetches ALL comments, not just the first 50
     │
     ├─ 5. GET /rest/api/3/issue/JIRA-123/remotelink
     │      └─ Web links / external resources added to the ticket
     │
     ├─ 6. For each subtask and child work item (if --recursive):
     │      └─ Recursively call export_issue for each associated key
     │
     ├─ 7. For each attachment (all types — images, PDFs, ZIPs, docs):
     │      └─ curl -fL <attachment-url> → .jira/JIRA-123/attachments/<filename>
     │         For images: CDN redirect UUID is captured and mapped for ADF embedding
     │         For all other types: downloaded and listed with local relative links
     │
     └─ 8. Render everything to Markdown
            └─ ADF → MD conversion happens entirely in jq (no external parser)
               Includes: Description, Custom Fields, Subtasks, Linked Issues, Comments
               Written to .jira/JIRA-123/JIRA-123.md
```

### Why jq for ADF rendering?

Atlassian stores rich text in a JSON format called ADF (Atlassian Document
Format). It is a deeply nested JSON tree of `{type, content, attrs, marks}`
objects. The script converts this to Markdown using a recursive `jq` function
— no external libraries, no Node.js, no Python. This keeps the script a single
file with minimal dependencies.

The recursive-jq approach was inspired by
[thefiend/Jira-to-Markdown](https://github.com/thefiend/Jira-to-Markdown)
(MIT licence). The function in this script was written from scratch and covers
significantly more ADF node types, but the original idea came from that project.

---

<!--
SEO Keywords for Google and GitHub Search:
jira to markdown, export jira to markdown, jira ticket backup, atlassian document format to markdown,
jira adf to markdown, jira command line exporter, jira bash script, download jira attachments,
jira comments to markdown, redact pii jira, jira export tool, jira backup script,
ai context from jira, jira to claude context, jira to junie context
-->

---

## Security notes (detailed)

Full technical breakdown of every security practice built into the script:

**No credentials in shell history or logs**
The API token is read from `.env.jira` and never echoed or logged. The
Basic Auth value (base64 of `email:token`) is constructed in memory and passed
directly to `curl` via a variable, not as a command-line argument visible in
`ps`.

**No `eval` anywhere**
The `.env.jira` parser uses a `while read` loop with a `case` allowlist. Only
`JIRA_*` variables are ever exported — arbitrary shell code in `.env.jira`
cannot be executed.

**Path traversal protection on filenames**
Attachment filenames from Jira are sanitised before being used as file paths.
Characters like `/`, `..`, `\`, and control characters are stripped. A file
named `../../.bashrc` on Jira will not overwrite your shell config.

**Issue key validation**
The issue key argument (e.g. `JIRA-123`) is validated against the pattern
`^[A-Za-z][A-Za-z0-9_]+-[0-9]+$` before it is used as a directory name or
passed to the API. Arbitrary strings cannot be injected.

**`.env.jira` stays local**
The config file should always be in `.gitignore`. The script will still work
even if you only have `.env`, but `.env.jira` is the recommended location
specifically because it is purpose-scoped and easier to selectively ignore.

**Sensitive content redaction**
Before the Markdown file is written to disk, the entire output is piped through
a pattern-based redaction filter. The following supported patterns are replaced
with labelled `[REDACTED — ...]` placeholders:

- AWS Access Key IDs (`AKIA...`)
- Private key and certificate block headers (`-----BEGIN ... KEY-----`)
- Bearer tokens and Basic Auth credentials
- `api_key`, `secret_key`, `api_secret`, `access_token` in `key=value` format
- `password` and `pwd` in `key=value` format
- Database connection URLs with embedded credentials (`proto://user:pass@host`)

**PII redaction (on by default)**
When `JIRA_REDACT_PII=true` (the default), the filter also redacts:

- Email addresses
- Credit and debit card numbers (13–16 digit patterns)

Set `JIRA_REDACT_PII=false` in `.env.jira` to disable PII redaction while
keeping credential redaction active. The number of redacted values is printed
to the terminal after each export.

Redaction is best-effort and heuristic. Always review exported files before
sharing outside your trusted environment.

**Attachment risk classification**
Every attachment is classified before downloading:

| Risk level | File types                                                                                       | Action |
|---|--------------------------------------------------------------------------------------------------|---|
| **High** | `.env`, `.key`, `.pem`, `.p12`, `.pfx`, `.crt`, `.sql`, `.bak`, `.dump`, `.keystore`, and others | Not downloaded — listed as ⛔ in the Markdown |
| **Medium** | `.xml`, `.json`, `.csv`, `.xlsx`, `.xls`, `.tsv`, `.ods`                                         | Downloaded but flagged as ⚠️ with a review notice |
| **Safe** | Everything else                                                                                  | Downloaded normally |

---

## Troubleshooting

### `jq is required. Install with: brew install jq`

You are missing the `jq` tool. Run:

```bash
brew install jq
```

Then retry.

---

### `No .env.jira found at: /path/to/project`

The script could not find `.env.jira` (or `.env`) in your project root. Check:

1. You are running `bin/jira` from inside the project directory, not from
   somewhere else:
   ```bash
   cd /path/to/your-project
   bin/jira JIRA-123
   ```
2. The file exists:
   ```bash
   ls -la .env.jira
   ```
3. If it does not exist, create it from the sample:
   ```bash
   cp .env.jira.sample .env.jira
   ```

---

### `Authentication failed (HTTP 401). Check JIRA_EMAIL and JIRA_API_TOKEN.`

The credentials in `.env.jira` are wrong or the token has expired. Check:

1. `JIRA_EMAIL` matches the email on your Atlassian account exactly
2. `JIRA_API_TOKEN` is the API token, not your Jira password
3. The token has not been revoked — go to
   [https://id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens)
   to verify it still exists
4. There are no extra spaces or quotes in `.env.jira` around the token value

---

### `Not found (HTTP 404): /rest/api/3/issue/JIRA-123`

Either the ticket key is wrong, or your Atlassian account does not have access
to that project. Try opening the ticket directly in your browser:

```
https://yourcompany.atlassian.net/browse/JIRA-123
```

If the browser also shows "not found", the ticket does not exist or has been
deleted.

---

### A custom field section shows `_Not set._` but the field has content in Jira

Your Jira instance uses different custom field IDs. Follow the steps in
[Finding your custom field IDs](#finding-your-custom-field-ids), then update
the relevant entry in `JIRA_CUSTOM_FIELDS`:

```dotenv
# Corrected to your instance's actual IDs
JIRA_CUSTOM_FIELDS='[
  {"label": "Acceptance Criteria", "id": "customfield_10093"},
  {"label": "Technical Notes",     "id": "customfield_10055"}
]'
```

Only the field IDs change — the labels are whatever you want them to be.

---

### Images are missing from the markdown

This can happen when:

- You used `--no-attachments` — the flag intentionally skips all downloads
- The download failed silently — check the terminal output for any `Failed to download` warnings and retry

---

### The script is slow / times out

Tickets with many or large attachments can take longer. Use `--no-attachments`
to skip all downloads and get just the text content:

```bash
bin/jira JIRA-123 --no-attachments
```

If the API itself is slow, this is usually a Jira Cloud rate limit. Wait a
minute and retry.

---

### `Rate limited (HTTP 429). Wait a moment and retry.`

Jira Cloud limits how many API requests you can make per minute. This usually
only happens if you are exporting many tickets in quick succession. Wait 60
seconds and try again.

---

## License

MIT — see [LICENSE](LICENSE) for the full text.

The ADF→Markdown conversion approach was inspired by
[thefiend/Jira-to-Markdown](https://github.com/thefiend/Jira-to-Markdown),
also MIT licensed.
