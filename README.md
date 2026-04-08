# notion2cal

Sync a Notion database to Apple Calendar (or any other iCal-compatible calendar). A GitHub Actions workflow periodically queries the Notion API and generates an `.ics` file that can be subscribed to via its raw URL.

## Prerequisites

- A [Notion](https://www.notion.so) account with a database that contains a **Date** property
- A [GitHub](https://github.com) account
- Apple Calendar (or any other calendar app that supports iCal subscription URLs)

## Setup

### 1. Clone this repository

```bash
git clone https://github.com/shiermann/notion2cal.git
cd notion2cal
```

### 2. Create your own GitHub repository

> **Important:** The repository must be **public** so that Apple Calendar (and other clients) can fetch the `.ics` file via a stable raw URL without authentication. Private repositories require temporary tokens that expire and break the subscription.

1. Create a new **public** repository on [github.com/new](https://github.com/new) (leave it empty — no README, no .gitignore, no license)
2. Point your local clone to your new repo and push:

```bash
git remote set-url origin https://github.com/YOUR_USERNAME/notion2cal.git
git push -u origin main
```

### 3. Create a Notion integration

1. Go to [notion.so/my-integrations](https://www.notion.so/my-integrations)
2. Click **New integration**
3. Give it a name — **note:** the name must **not contain "Notion"** (Notion forbids this). Use something like `Calendar Sync`, `CalExporter`, or `MyCalSync`
4. Select the workspace that contains your database
5. Under **Capabilities**, **Read content** is sufficient
6. Click **Submit** and copy the **Internal Integration Secret** (starts with `ntn_`)

### 4. Connect the integration to your database

1. Open your Notion database
2. Click the **...** menu in the top-right corner
3. Choose **Connections** > **Connect to** and pick the integration you just created
4. Confirm access

### 5. Find your database ID

The database ID is part of your Notion database URL:

```
https://www.notion.so/workspace/DATABASE_ID?v=...
                                ^^^^^^^^^^^
```

It is the 32-character hex string between the last `/` and the `?`. Example:

```
https://www.notion.so/workspace/a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4?v=...
→ Database ID: a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4
```

### 6. Configure GitHub Secrets

1. Go to your repository on GitHub
2. **Settings** > **Secrets and variables** > **Actions**
3. Click **New repository secret** and add:
   - Name: `NOTION_TOKEN` — Value: the integration secret from step 3
   - Name: `NOTION_DATABASE_ID` — Value: the database ID from step 5

### 7. Run the workflow

1. Go to **Actions** in your repository
2. Select the **Sync Notion to Calendar** workflow
3. Click **Run workflow** to trigger it manually
4. After a successful run, a `notion_calendar.ics` file will be committed to the repository

### 8. Subscribe in Apple Calendar

1. Copy the raw URL of your `.ics` file (replace `YOUR_USERNAME`):

```
https://raw.githubusercontent.com/YOUR_USERNAME/notion2cal/main/notion_calendar.ics
```

> Make sure the URL has **no `?token=...` query parameter**. Tokens are temporary and will cause the subscription to break.

2. Open Apple Calendar
3. **File** > **New Calendar Subscription...**
4. Paste the URL and click **Subscribe**
5. Set the refresh frequency (e.g. **Weekly**)

## Supported Notion properties

| Property type | Usage |
|---|---|
| **Title** | Used as the event title |
| **Date** | Start and optional end date. Supports both date-only (all-day) and date+time entries |
| **Rich Text** | Used as the event description (looks for: Description, Beschreibung, Notes, Notizen, Text) |

## Run locally

```bash
export NOTION_TOKEN="ntn_..."
export NOTION_DATABASE_ID="a1b2c3..."
pip install -r requirements.txt
python notion2cal.py
```

The file `notion_calendar.ics` will be written to the current directory.

## Configuration

| Environment variable | Description | Default |
|---|---|---|
| `NOTION_TOKEN` | Notion integration secret | (required) |
| `NOTION_DATABASE_ID` | ID of the Notion database | (required) |
| `OUTPUT_FILE` | Output file name | `notion_calendar.ics` |

## Change the schedule

The workflow runs weekly on Mondays at 06:00 UTC by default. To change the schedule, edit the `cron` expression in [.github/workflows/sync.yml](.github/workflows/sync.yml):

```yaml
schedule:
  - cron: "0 6 * * 1"  # minute hour day month weekday
```

Cron syntax reference:

```
0 6 * * 1
│ │ │ │ │
│ │ │ │ └─ weekday (0–6, Sunday=0, Monday=1)
│ │ │ └─── month (1–12 or *)
│ │ └───── day of month (1–31 or *)
│ └─────── hour (0–23, UTC)
└───────── minute (0–59)
```

Note that times are in **UTC** — GitHub Actions does not support time zones in cron expressions.

## Privacy considerations

Because the repository is public, the contents of your `notion_calendar.ics` file (event titles, dates, descriptions, and Notion page URLs) are publicly visible to anyone who knows the URL. Only sync databases whose contents you are comfortable making public.

Your `NOTION_TOKEN` and `NOTION_DATABASE_ID` stay private — they are stored as GitHub Secrets and never appear in the repository.
