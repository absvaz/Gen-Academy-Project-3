# Intelligent Project Status Agent (n8n)

An agent that pulls the current sprint status from Notion, flags risks and blockers, writes a weekly status report, and remembers every week so it can answer questions like **"What's been stuck for more than one sprint?"**

Built for Project 3C (low-code track) using n8n, Notion, Google Sheets, and an OpenAI model.

- **Demo recording:** `<ADD LINK>`
- **Sample report:** [`reports/sample_weekly_report.md`](reports/sample_weekly_report.md)

---

## What it does

1. **Pulls current status** from a Notion task database (Smart Water Valve project, 15 tasks).
2. **Flags risks in code**: blocked, overdue, unassigned, due within 7 days.
3. **Compares against history** saved in Google Sheets to work out how many weeks each task has been in its current status, and which tasks are stuck across more than one sprint.
4. **Writes a weekly report** with risk flags, blockers, week-over-week movement, and recommended actions.
5. **Saves this week's snapshot** so next week's run has more history.
6. **Answers questions in chat** from the saved history, including follow-ups.

## Design principle: code counts, the AI explains

All numbers (days overdue, weeks stuck, sprint spans, status counts) are computed in JavaScript Code nodes. The language model only turns those results into readable text, and is instructed not to recalculate or invent anything. This keeps the report and chat answers reliable.

---

## Architecture

### Workflow 1: Weekly Status Report (`workflows/weekly_status_report.json`)

```
Manual Trigger
  -> Notion: Get many database pages      (15 tasks)
  -> Risk Flags (Code)                    (blocked / overdue / unassigned / due soon, risk level)
  -> Load History (Google Sheets)         (all past snapshots, Execute Once)
  -> Stuck Calc (Code)                    (weeks in status, stuck, sprints spanned)
        |-> Save Snapshot (Google Sheets: Append Row)   (this week's rows -> memory)
        |-> Prepare Summary (Code)                      (one summary object)
              -> Write Report (OpenAI)                  (Markdown weekly report)
```

### Workflow 2: Status Chat Agent (`workflows/status_chat_agent.json`)

```
Chat Trigger
  -> Load History (Google Sheets)
  -> Stuck Report (Code)                  (full week-over-week picture)
  -> AI Agent
        - OpenAI Chat Model
        - Simple Memory (chat context for follow-up questions)
```

---

## Memory

Memory is a Google Sheet (tab `snapshots`) with one row per task per week:

| week | task_id | title | status | assignee | due_date | blocked | sprint |
|------|---------|-------|--------|----------|----------|---------|--------|

- Each run of the weekly workflow appends the current week's rows (`Save Snapshot`).
- The next run reads all prior weeks to compute trends.
- The current week is excluded when counting, and duplicate rows are ignored, so re-running in the same week does not inflate the numbers.
- The chat agent's **Simple Memory** node keeps conversation context so follow-ups such as "Who owns it?" work.

### Definitions

| Term | Rule |
|------|------|
| Weeks in status | Consecutive weeks, ending now, with the same status |
| Stuck | Blocked for 2 or more weeks, or In Progress for 3 or more weeks |
| Stuck over one sprint | A stuck streak that spans more than one sprint |
| HIGH risk | Blocked and overdue |
| MEDIUM risk | Blocked, overdue, unassigned, or stuck (but not HIGH) |
| DUE_SOON | Not done and due within 7 days |

---

## Sample data

The project data is simulated.

- `sample_data/notion_project_tasks.csv`: the 15 current tasks, imported into a Notion database called **Project Tasks**.
- `sample_data/seed_history.csv`: three weeks of past snapshots (2026-W40 to W42, covering Sprint 2 and Sprint 3), loaded into the Google Sheet so the memory feature has history to work with. The current week (2026-W43) is added by the workflow itself.

The seeded history includes T-007 (Select valve supplier) blocked for three consecutive weeks across two sprints, which is the case the chat agent is demonstrated on.

---

## Setup

### Prerequisites

- An n8n instance (n8n Cloud trial or self-hosted Community Edition)
- A Notion account with an integration
- A Google account with a Google Sheet
- An OpenAI-compatible model credential (the demo used n8n's built-in AI credits)

### Steps

1. **Notion**
   - Create a database named `Project Tasks` by importing `sample_data/notion_project_tasks.csv`.
   - Set the property types: Status (Select), Due Date (Date), Blocked (Checkbox), Sprint (Select).
   - Share the database with your n8n integration.
2. **Google Sheet**
   - Create a spreadsheet with a tab named `snapshots`.
   - Import `sample_data/seed_history.csv` into it (uncheck "convert text to numbers, dates, and formulas").
3. **Import the workflows** into n8n: Workflows -> Import from file -> choose each JSON file in `workflows/`.
4. **Credentials**: in each node, select your own Notion, Google Sheets, and OpenAI credentials.
5. **Replace the placeholders**: set the Google Sheets document to your own sheet (`YOUR_GOOGLE_SHEET_URL`) and the Notion data source to your `Project Tasks` database.
6. **Check node names**: the Stuck Calc code looks up the first Code node by the name `Risk Flags`, and the chat agent looks up `Stuck Report`. Keep those names.

### Running it

- **Weekly report:** open *Weekly Status Report* and click **Execute workflow**. Read the report from the *Write Report* node.
- **Chat:** open *Status Chat Agent*, click **Open chat**, and ask questions.

Example questions:

- What's been stuck for more than one sprint?
- What's blocked right now?
- How long has T-008 been blocked?
- Which tasks are unassigned?
- What's the budget? (the agent should say the data does not contain it)

---

## Known limitations

- The weekly workflow is triggered manually. A Schedule Trigger would automate it.
- Risk thresholds are fixed in code (see Definitions) and not configurable from the UI.
- Sprint labels in the Notion data are a single placeholder value; sprint history comes from the saved snapshots.
- Only Notion is connected. Jira or Asana would need a source node that outputs the same fields.
- Google Sheets is used as a simple store. A database would suit a larger project.

## Ideas for next steps

- Schedule the report weekly and deliver it to Slack or email.
- Write the report back into Notion as a page.
- Add Jira or Asana as additional sources.
- Add trend charts of blocked and overdue counts over time.

## Repository layout

```
status-agent-n8n/
├── README.md
├── workflows/
│   ├── weekly_status_report.json
│   └── status_chat_agent.json
├── sample_data/
│   ├── notion_project_tasks.csv
│   └── seed_history.csv
├── reports/
│   └── sample_weekly_report.md
└── screenshots/
```

## Tech

n8n (workflows and AI Agent node), Notion API, Google Sheets API, OpenAI model, JavaScript Code nodes.
