# Weekly Customer Effort Tracker

**Version:** 1.2.0  
**Author:** Abi / Joule Skill Creator  
**Tags:** productivity · time-tracking · customer · weekly-report · outlook

---

## Overview

The Weekly Customer Effort Tracker scans your Outlook calendar, emails, and local files to estimate how much time you spent on each customer during a working week. It produces a Markdown report with hours broken down by customer and by day, saved to your Documents folder.

---

## How to Activate

Say things like:
- "Run my weekly effort tracker"
- "How much time did I spend on customers this week?"
- "Generate my weekly effort report"
- "Track my customer hours"
- "How much time did I spend on customers last week?"

You can also specify a past week:  
> "Run the effort tracker for the week of 17 August"

---

## Data Sources

The skill scans four sources simultaneously:

| Source | What it looks at | Tool used |
|---|---|---|
| Calendar | Meetings and work blocks for the week | `list_calendar_events` |
| Sent emails | Emails sent during the week | `list_emails` (sentitems) |
| Received emails | External emails received during the week | `list_emails` (inbox) |
| Files | Desktop and Documents files modified during the week | `execute` (Python script) |

---

## How Effort is Estimated

### Calendar Meetings
Duration from calendar + prep time:
- Meetings ≤ 30 min → +5 min prep
- Meetings > 30 min → +10 min prep

### Calendar Work Blocks
Solo blocks where you have set aside time to work on a customer deliverable. Detected by task-oriented keywords in the title such as *prep, draft, scope, proposal, review, slides, estimate, design, document, analysis, build, workshop prep*. Counted at **full duration** — no prep time added, since the time is already dedicated work.

### Sent Emails
- Email with attachment or link indicators → **1.0 hour**
- Otherwise → estimated from preview word count: `max(5, min(45, words / 8)) / 60`

### Received Emails
Same word-count formula as sent emails. Internal emails (same organisation domain) are skipped.

### Files (Desktop & Documents)
**0.5 hours (30 min)** per file modified during the week. Scans up to 2 folder levels deep inside Desktop and Documents.

---

## Customer Inference

Customer names are inferred automatically from:
- Meeting and work block titles (e.g. "Contoso Quarterly Review" → Contoso)
- Email recipient/sender domains (e.g. contoso.com → Contoso)
- Email subject lines
- Filenames and subfolder paths (e.g. `Projects/Contoso/proposal.docx` → Contoso)

Activities where no customer can be identified are grouped under **Internal / Unknown**.

---

## Output

A Markdown file named `effort-YYYY-MM-DD.md` (using the Monday date) is saved to your Documents folder.

The report includes:
1. **Summary table** — hours per customer per day, with totals
2. **Activity Detail** — breakdown of meetings, work blocks, emails, and files per customer

---

## Requirements

- Microsoft Outlook calendar and email access (via Joule Desktop)
- Python available in the sandbox (for file scanning)

---

## Limitations & Notes

- Email scanning is limited to the 50 most recent sent and received items — older emails from the same week may be missed if you send or receive a high volume
- Calendar scanning covers up to 50 events per week
- File scanning covers Desktop and Documents, up to 2 subfolder levels deep
- All-day calendar events are skipped
- Work blocks are only counted if a customer name is clearly identifiable in the title
- Estimates are approximations — the report always includes a reminder to adjust any figure that looks off

---

## Changelog

### v1.2.0
- File scanning now covers **Desktop** and **Documents** folders (up to 2 levels deep) — no specific subfolder names assumed, works across different setups
- Calendar step now detects and counts **solo work blocks** (time booked to work on customer deliverables) in addition to meetings
- Work block hours counted at full duration (no prep time added)
- Activity Detail report section now includes a **Work blocks** line per customer
- File customer inference now uses the subfolder path as well as the filename

### v1.1.0
- Initial release with calendar, email, and desktop file scanning
