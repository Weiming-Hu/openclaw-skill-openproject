---
name: openproject
description: "OpenProject CRUD skill — manage work packages, projects, groups, news, users, watchers, relations, notifications, time entries, comments, attachments, wiki pages, statuses, and more via OpenProject API v3 with API token auth. Supports cloud and self-hosted instances."
homepage: https://github.com/ALT-F1-OpenClaw/openclaw-skill-openproject
metadata:
  {"openclaw": {"emoji": "📊", "requires": {"env": ["OP_HOST", "OP_API_TOKEN"]}, "optional": {"env": ["OP_DEFAULT_PROJECT", "OP_MAX_RESULTS", "OP_MAX_FILE_SIZE"]}, "primaryEnv": "OP_HOST"}}
---

# OpenProject by @altf1be

Manage OpenProject work packages, projects, groups, news, users, watchers, relations, notifications, time entries, comments, attachments, wiki pages, and workflow transitions via the API v3. Works with both cloud and self-hosted instances.

## Setup

1. Log in to your OpenProject instance
2. Go to **My Account → Access Tokens → + Add**
3. Create an API token and copy it
4. Set environment variables (or create `.env` in `{baseDir}`):

```
OP_HOST=https://projects.xflowdata.com
OP_API_TOKEN=your-api-token
OP_DEFAULT_PROJECT=my-project
```

5. Install dependencies: `cd {baseDir} && npm install`

## Efficient Usage (read this first)

General rules for keeping calls cheap and output readable. Unfiltered queries on a
mature instance return hundreds of work packages, each with a full `description`
that may contain large blocks of embedded HTML (tables, mail quotes, figures).
That output routinely exceeds an agent's tool-output limit and gets truncated.

1. **Always narrow before you list.** Combine `--project`, `--open` / `--closed`,
   `--status`, and `--assignee` instead of listing everything and filtering later.
   ```bash
   node {baseDir}/scripts/openproject.mjs wp-list --project my-project --open
   ```
2. **Fetch a known ID directly.** If you already have the ID, use `wp-read --id N`
   rather than a list command. `wp-read` also prints the parent reference.
3. **Use `--json` for machine use, but reduce it before reading.** `--json` paginates
   over the whole result set and includes `description`. Write it to a file and project
   only the fields you need:
   ```bash
   node {baseDir}/scripts/openproject.mjs wp-list --open --json > /tmp/wp.json
   node -e 'const d=require("/tmp/wp.json");console.log(d.tasks.map(t=>[t.id,t.type,t.status,t.subject].join(" | ")).join("\n"))'
   ```
4. **Page size is configurable.** Non-JSON list commands honour `OP_MAX_RESULTS`
   (default 50). Raise it deliberately; do not assume a single call returned everything.
   The `--json` path pages through all results at 100 per request.
5. **There is no `--parent` flag.** To find the children of a parent work package
   (for example the tasks under a phase), query API v3 directly with a `parent` filter.
   The `filters` value is JSON and must be URL-encoded:
   ```bash
   curl -s -u "apikey:$OP_API_TOKEN" \
     "$OP_HOST/api/v3/work_packages?pageSize=100&filters=%5B%7B%22parent%22%3A%7B%22operator%22%3A%22%3D%22%2C%22values%22%3A%5B%2242%22%5D%7D%7D%5D"
   ```
   Decoded, that filter is `[{"parent":{"operator":"=","values":["42"]}}]`.
   The same pattern works for any API v3 filter the CLI does not expose.
6. **`--project` takes the project identifier**, not the display name. Run
   `project-list` once to map names to identifiers.
7. **Prefer one narrow query over a broad query plus local search.** Searching a full
   dump for a keyword wastes a round trip and risks truncation hiding the match.

## Commands

### Work Packages

```bash
# List work packages (with optional filters)
# --open / --closed filter by open vs closed category; --status takes an exact status name
node {baseDir}/scripts/openproject.mjs wp-list --project my-project --open --assignee me
node {baseDir}/scripts/openproject.mjs wp-list --project my-project --status "In progress"

# List work packages as JSON — paginated, includes start/due dates + description
node {baseDir}/scripts/openproject.mjs wp-list --status open --json

# Create a work package
node {baseDir}/scripts/openproject.mjs wp-create --project my-project --type Task --subject "Fix login bug" --description "Users can't log in"

# Read work package details
node {baseDir}/scripts/openproject.mjs wp-read --id 42

# Read work package details as JSON (incl. start/due dates + description)
node {baseDir}/scripts/openproject.mjs wp-read --id 42 --json

# Update a work package
node {baseDir}/scripts/openproject.mjs wp-update --id 42 --subject "New title" --priority High

# Delete a work package (requires --confirm)
node {baseDir}/scripts/openproject.mjs wp-delete --id 42 --confirm
```

### Projects

```bash
# List all projects
node {baseDir}/scripts/openproject.mjs project-list

# Read project details
node {baseDir}/scripts/openproject.mjs project-read --id my-project

# Create a project
node {baseDir}/scripts/openproject.mjs project-create --name "My Project" --identifier my-project
```

### Comments (Activities)

```bash
# List comments on a work package
node {baseDir}/scripts/openproject.mjs comment-list --wp-id 42

# Add a comment
node {baseDir}/scripts/openproject.mjs comment-add --wp-id 42 --body "Ready for review"
```

### Attachments

```bash
# List attachments on a work package
node {baseDir}/scripts/openproject.mjs attachment-list --wp-id 42

# Upload an attachment
node {baseDir}/scripts/openproject.mjs attachment-add --wp-id 42 --file ./screenshot.png

# Delete an attachment (requires --confirm)
node {baseDir}/scripts/openproject.mjs attachment-delete --id 10 --confirm
```

### Time Entries

```bash
# List time entries
node {baseDir}/scripts/openproject.mjs time-list --project my-project

# Log time (duration on a date)
node {baseDir}/scripts/openproject.mjs time-create --wp-id 42 --hours 2.5 --comment "Code review" --activity-id 1

# Log time with a specific start + end (finish time is auto-derived from the span)
node {baseDir}/scripts/openproject.mjs time-create --wp-id 42 --start 2026-09-08T13:00 --end 2026-09-08T14:30 --comment "Pairing"

# Update time entry
node {baseDir}/scripts/openproject.mjs time-update --id 5 --hours 3 --comment "Updated"

# Delete time entry (requires --confirm)
node {baseDir}/scripts/openproject.mjs time-delete --id 5 --confirm
```

### Statuses & Transitions

```bash
# List all statuses
node {baseDir}/scripts/openproject.mjs status-list

# Update work package status
node {baseDir}/scripts/openproject.mjs wp-update --id 42 --status "In progress"
```

### Users

```bash
# List all users (with optional filters)
node {baseDir}/scripts/openproject.mjs user-list
node {baseDir}/scripts/openproject.mjs user-list --status active --name "John"

# Read user details
node {baseDir}/scripts/openproject.mjs user-read --id 5

# Show current authenticated user
node {baseDir}/scripts/openproject.mjs user-me
```

### Notifications

```bash
# List notifications (all or unread only)
node {baseDir}/scripts/openproject.mjs notification-list
node {baseDir}/scripts/openproject.mjs notification-list --unread
node {baseDir}/scripts/openproject.mjs notification-list --reason mentioned --project 5

# Read notification details
node {baseDir}/scripts/openproject.mjs notification-read --id 100

# Mark as read (single or all)
node {baseDir}/scripts/openproject.mjs notification-mark-read --id 100
node {baseDir}/scripts/openproject.mjs notification-mark-read --all
node {baseDir}/scripts/openproject.mjs notification-mark-read --all --project 5

# Mark as unread
node {baseDir}/scripts/openproject.mjs notification-mark-unread --id 100
node {baseDir}/scripts/openproject.mjs notification-mark-unread --all
```

### Documents

```bash
node {baseDir}/scripts/openproject.mjs document-list
node {baseDir}/scripts/openproject.mjs document-read --id 5
node {baseDir}/scripts/openproject.mjs document-update --id 5 --title "Updated title"
```

### Revisions

```bash
node {baseDir}/scripts/openproject.mjs revision-read --id 10
node {baseDir}/scripts/openproject.mjs revision-list-by-wp --wp-id 42
```

### Capabilities & Actions

```bash
node {baseDir}/scripts/openproject.mjs capability-list --principal me
node {baseDir}/scripts/openproject.mjs capability-global
node {baseDir}/scripts/openproject.mjs action-list
node {baseDir}/scripts/openproject.mjs action-read --id work_packages/create
```

### My Preferences

```bash
node {baseDir}/scripts/openproject.mjs my-preferences-read
node {baseDir}/scripts/openproject.mjs my-preferences-update --time-zone "Europe/Brussels" --comment-order desc
```

### Render

```bash
node {baseDir}/scripts/openproject.mjs render-markdown --text "**bold** and _italic_"
node {baseDir}/scripts/openproject.mjs render-plain --text "plain text"
```

### Posts (Forum)

```bash
node {baseDir}/scripts/openproject.mjs post-read --id 5
node {baseDir}/scripts/openproject.mjs post-attachment-list --id 5
```

### Reminders

```bash
node {baseDir}/scripts/openproject.mjs reminder-list
node {baseDir}/scripts/openproject.mjs reminder-create --wp-id 42 --remind-at "2026-03-20T09:00:00Z" --note "Check status"
node {baseDir}/scripts/openproject.mjs reminder-update --id 3 --remind-at "2026-03-21T09:00:00Z"
node {baseDir}/scripts/openproject.mjs reminder-delete --id 3 --confirm
```

### Project Statuses

```bash
node {baseDir}/scripts/openproject.mjs project-status-read --id on_track
```

### Project Phases (Enterprise)

```bash
# List project phase definitions
node {baseDir}/scripts/openproject.mjs project-phase-definition-list

# Read a phase definition
node {baseDir}/scripts/openproject.mjs project-phase-definition-read --id 1

# Read a project phase
node {baseDir}/scripts/openproject.mjs project-phase-read --id 5
```

### Portfolios (Enterprise)

```bash
# List portfolios
node {baseDir}/scripts/openproject.mjs portfolio-list

# Read a portfolio
node {baseDir}/scripts/openproject.mjs portfolio-read --id 1

# Update a portfolio
node {baseDir}/scripts/openproject.mjs portfolio-update --id 1 --name "Q1 Portfolio"

# Delete a portfolio (requires --confirm)
node {baseDir}/scripts/openproject.mjs portfolio-delete --id 1 --confirm
```

### Programs (Enterprise)

```bash
# List programs
node {baseDir}/scripts/openproject.mjs program-list

# Read a program
node {baseDir}/scripts/openproject.mjs program-read --id 1

# Update a program
node {baseDir}/scripts/openproject.mjs program-update --id 1 --name "Platform Program"

# Delete a program (requires --confirm)
node {baseDir}/scripts/openproject.mjs program-delete --id 1 --confirm
```

### Placeholder Users (Enterprise)

```bash
# List placeholder users
node {baseDir}/scripts/openproject.mjs placeholder-user-list

# Read a placeholder user
node {baseDir}/scripts/openproject.mjs placeholder-user-read --id 10

# Create a placeholder user
node {baseDir}/scripts/openproject.mjs placeholder-user-create --name "Future Developer"

# Update a placeholder user
node {baseDir}/scripts/openproject.mjs placeholder-user-update --id 10 --name "Senior Developer"

# Delete a placeholder user (requires --confirm)
node {baseDir}/scripts/openproject.mjs placeholder-user-delete --id 10 --confirm
```

### Budgets (Enterprise)

```bash
# List project budgets
node {baseDir}/scripts/openproject.mjs budget-list --project my-project

# Read budget details
node {baseDir}/scripts/openproject.mjs budget-read --id 3
```

### Meetings (Enterprise)

```bash
# Read a meeting
node {baseDir}/scripts/openproject.mjs meeting-read --id 10

# List meeting attachments
node {baseDir}/scripts/openproject.mjs meeting-attachment-list --id 10

# Upload attachment to a meeting
node {baseDir}/scripts/openproject.mjs meeting-attachment-add --id 10 --file ./agenda.pdf
```

### Days (Working/Non-Working)

```bash
# Check if a date is a working day
node {baseDir}/scripts/openproject.mjs day-read --date 2026-03-18

# List days in a range
node {baseDir}/scripts/openproject.mjs days-list --from 2026-03-01 --to 2026-03-31

# List all non-working days (holidays)
node {baseDir}/scripts/openproject.mjs non-working-days-list

# View a non-working day
node {baseDir}/scripts/openproject.mjs non-working-day-read --date 2026-12-25

# Show week day schedule (which days are working)
node {baseDir}/scripts/openproject.mjs week-days-list

# View a specific week day
node {baseDir}/scripts/openproject.mjs week-day-read --day 6
```

### Configuration

```bash
# View instance configuration
node {baseDir}/scripts/openproject.mjs config-read

# View project-specific configuration
node {baseDir}/scripts/openproject.mjs project-config-read --project my-project
```

### OAuth

```bash
# Read an OAuth application
node {baseDir}/scripts/openproject.mjs oauth-app-read --id 1

# Read OAuth client credentials
node {baseDir}/scripts/openproject.mjs oauth-credentials-read --id 1
```

### Help Texts

```bash
# List all attribute help texts
node {baseDir}/scripts/openproject.mjs help-text-list

# Read a help text
node {baseDir}/scripts/openproject.mjs help-text-read --id 1
```

### Custom Fields & Options

```bash
# List items for a hierarchical custom field
node {baseDir}/scripts/openproject.mjs custom-field-items --id 10

# Read a custom field item
node {baseDir}/scripts/openproject.mjs custom-field-item-read --id 25

# Get a custom field item's branch (ancestors)
node {baseDir}/scripts/openproject.mjs custom-field-item-branch --id 25

# Read a custom option value
node {baseDir}/scripts/openproject.mjs custom-option-read --id 3
```

### Custom Actions

```bash
# Read a custom action
node {baseDir}/scripts/openproject.mjs custom-action-read --id 1

# Execute a custom action on a work package
node {baseDir}/scripts/openproject.mjs custom-action-execute --id 1 --wp-id 42
```

### Groups

```bash
# List all groups
node {baseDir}/scripts/openproject.mjs group-list

# Read group details with members
node {baseDir}/scripts/openproject.mjs group-read --id 3

# Create a group with members
node {baseDir}/scripts/openproject.mjs group-create --name "Dev Team" --members 5,6,7

# Update a group (replaces member list)
node {baseDir}/scripts/openproject.mjs group-update --id 3 --name "Engineering" --members 5,6,7,8

# Delete a group (requires --confirm)
node {baseDir}/scripts/openproject.mjs group-delete --id 3 --confirm
```

### News

```bash
# List all news
node {baseDir}/scripts/openproject.mjs news-list

# Read news details
node {baseDir}/scripts/openproject.mjs news-read --id 5

# Create a news item
node {baseDir}/scripts/openproject.mjs news-create --project my-project --title "Sprint 12 Released" --summary "Bug fixes and performance" --description "Full details here..."

# Update a news item
node {baseDir}/scripts/openproject.mjs news-update --id 5 --title "Updated headline"

# Delete a news item (requires --confirm)
node {baseDir}/scripts/openproject.mjs news-delete --id 5 --confirm
```

### Watchers

```bash
# List watchers on a work package
node {baseDir}/scripts/openproject.mjs watcher-list --wp-id 42

# Add a watcher
node {baseDir}/scripts/openproject.mjs watcher-add --wp-id 42 --user-id 5

# Remove a watcher
node {baseDir}/scripts/openproject.mjs watcher-remove --wp-id 42 --user-id 5

# List users available as watchers
node {baseDir}/scripts/openproject.mjs watcher-available --wp-id 42
```

### Relations

```bash
# List all relations (optionally filter by work package or type)
node {baseDir}/scripts/openproject.mjs relation-list --wp-id 42
node {baseDir}/scripts/openproject.mjs relation-list --type blocks

# Read relation details
node {baseDir}/scripts/openproject.mjs relation-read --id 5

# Create a relation (types: relates, duplicates, blocks, precedes, follows, includes, partof, requires)
node {baseDir}/scripts/openproject.mjs relation-create --wp-id 42 --to-wp-id 43 --type blocks
node {baseDir}/scripts/openproject.mjs relation-create --wp-id 42 --to-wp-id 43 --type precedes --lag 2

# Update a relation
node {baseDir}/scripts/openproject.mjs relation-update --id 5 --type follows --lag 1

# Delete a relation (requires --confirm)
node {baseDir}/scripts/openproject.mjs relation-delete --id 5 --confirm
```

### Wiki Pages

```bash
# Read a wiki page
node {baseDir}/scripts/openproject.mjs wiki-read --id 72

# List attachments on a wiki page
node {baseDir}/scripts/openproject.mjs wiki-attachment-list --id 72

# Upload an attachment to a wiki page
node {baseDir}/scripts/openproject.mjs wiki-attachment-add --id 72 --file ./diagram.png
```

### Reference Data

```bash
# List work package types
node {baseDir}/scripts/openproject.mjs type-list

# List priorities
node {baseDir}/scripts/openproject.mjs priority-list

# List project members
node {baseDir}/scripts/openproject.mjs member-list --project my-project

# List versions/milestones
node {baseDir}/scripts/openproject.mjs version-list --project my-project

# List categories
node {baseDir}/scripts/openproject.mjs category-list --project my-project
```

## Security

- API token auth (Basic auth with `apikey` as username)
- No secrets or tokens printed to stdout
- All delete operations require explicit `--confirm` flag
- Path traversal prevention for file uploads
- Built-in rate limiting with exponential backoff retry
- Lazy config validation (only checked when a command runs)

## Dependencies

- `commander` — CLI framework
- `dotenv` — environment variable loading
- Node.js built-in `fetch` (requires Node >= 18)

## Author

Abdelkrim BOUJRAF — [ALT-F1 SRL](https://www.alt-f1.be), Brussels 🇧🇪 🇲🇦
X: [@altf1be](https://x.com/altf1be)
