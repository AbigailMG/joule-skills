---
name: weekly-customer-effort-tracker
description: >-
  Scans Outlook calendar meetings and work blocks, sent and received emails, and desktop files (Desktop and Documents folders, up to 2 levels deep) for the current working week to estimate hours spent per customer. Infers customer names automatically from calendar titles, email subjects, bodies, and sender/recipient domains. Produces a Markdown summary table saved to Documents. Use whenever the user asks to run the weekly effort tracker, track customer hours, generate an effort report, or asks how much time they spent on customers this week or last week.
allowed-tools: list_calendar_events list_emails search_emails get_email write_file execute read_file
metadata:
  author: Abi / Joule Skill Creator
  version: 1.2.0
  tags: productivity time-tracking customer weekly-report outlook
---

# Weekly Customer Effort Tracker

Scans Outlook calendar, sent and received emails, and desktop files for the current working week to estimate hours spent per customer. Generates a downloadable Markdown report saved to Documents, with effort broken down by customer and by day.

---

## Activation

Activate for requests like:
- "Run my weekly effort tracker"
- "How much time did I spend on customers this week / last week"
- "Generate my weekly effort report"
- "Track my customer hours"
- "Run the effort tracker"

---

## Step 1 — Determine the reporting week

Calculate the Monday and Friday of the target week based on today's date.

- Default: the current working week (Mon–Fri). If today is a weekend, use the most recently completed week.
- If the user specifies a past week (e.g. "last week", "the week of Aug 17"), adjust accordingly.

Store as `WEEK_START` (Monday 00:00:00) and `WEEK_END` (Friday 23:59:59) in ISO 8601 format.
Example: `2026-08-24T00:00:00` and `2026-08-28T23:59:59`.

Also store the date for each weekday as MON, TUE, WED, THU, FRI (YYYY-MM-DD format) — you will use these to tag each activity item with its day.

Tell the user: *"Scanning your week of [Mon D MMM] to [Fri D MMM YYYY]…"*

---

## Step 2 — Gather all data sources in parallel

Make these 3 tool calls **in a single message**. They have no dependencies on each other.

### A — Calendar
Call `list_calendar_events` with:
- `from`: WEEK_START
- `to`: WEEK_END
- `limit`: 50

### B — Sent emails
Call `list_emails` with:
- `folder`: `"sentitems"`
- `top`: 50

### C — Received emails
Call `list_emails` with:
- `folder`: `"inbox"`
- `top`: 50

Wait for all 3 to return, then proceed.

---

## Step 3 — Scan Desktop and Documents files

Write the following Python script to `_scan_files.py` in the working directory using `write_file`. Substitute the actual Monday and Friday dates in YYYY, MM, DD integer format before writing.

```python
import os, json
from datetime import datetime

week_start = datetime(YYYY, MM, DD, 0, 0, 0)    # Monday
week_end   = datetime(YYYY, MM, DD, 23, 59, 59) # Friday

home = os.path.expanduser("~")
base_folders = {
    "Desktop":   os.path.join(home, "Desktop"),
    "Documents": os.path.join(home, "Documents"),
}

MAX_DEPTH = 2
results = []

for label, base_path in base_folders.items():
    if not os.path.isdir(base_path):
        continue
    base_depth = base_path.rstrip(os.sep).count(os.sep)
    for root, dirs, files in os.walk(base_path):
        current_depth = root.count(os.sep) - base_depth
        if current_depth >= MAX_DEPTH:
            dirs[:] = []  # stop descending further
        for fname in files:
            fpath = os.path.join(root, fname)
            try:
                mtime = datetime.fromtimestamp(os.path.getmtime(fpath))
            except OSError:
                continue
            if week_start <= mtime <= week_end:
                relative = os.path.relpath(fpath, base_path)
                results.append({
                    "folder": label,
                    "filename": fname,
                    "path": relative,
                    "modified": mtime.strftime("%Y-%m-%d %H:%M"),
                    "day": mtime.strftime("%Y-%m-%d")
                })

print(json.dumps(results, indent=2))
```

Run the script with `python _scan_files.py`. Note the file list returned. Folders that do not exist are silently skipped. Delete `_scan_files.py` after running.

---

## Step 4 — Infer customers and estimate effort

For every activity item, record both the **customer name** and the **day** (YYYY-MM-DD) it occurred. You will use day to populate the daily columns in Step 5.

### 4A — Calendar events (meetings and work blocks)

Filter events from Step 2A to only those within the reporting week. Skip:
- Personal blocks with no customer reference (e.g. "Lunch", "OOO", "Break", "Commute")
- All-day events

For each qualifying event, classify it as a **meeting** or a **work block**:

**Meetings** — events with external or multiple attendees, or titles suggesting a call or meeting (e.g. "Contoso Quarterly Review", "ACME kick-off call", "TechCorp discovery"):
1. **Day**: date of the meeting start time (YYYY-MM-DD)
2. **Infer customer** from the meeting title
3. **Calculate duration** in minutes from start and end times
4. **Add prep time**: meetings ≤ 30 min → +5 min; meetings > 30 min → +10 min
5. **Hours** = (duration_min + prep_min) / 60

**Work blocks** — solo calendar blocks where the user has set aside time to work on a customer deliverable. Identify these by the presence of task-oriented keywords in the title (e.g. *prep, draft, deliverable, review, proposal, document, write, analysis, scope, prepare, estimate, design, build, workshop prep, slides*) **AND** an inferable customer name in the title. Examples: "Contoso scope doc prep", "ACME proposal review", "Draft TechCorp workshop slides":
1. **Day**: date of the block start time (YYYY-MM-DD)
2. **Infer customer** from the block title
3. **Hours** = full blocked duration in hours (no prep time added — the time is already dedicated work time)

Skip internal work blocks with no customer reference (e.g. "Focus time", "Admin", "Deep work" with no customer name). Only include blocks where a customer name can be clearly inferred from the title.

### 4B — Sent emails

Filter emails from Step 2B to those sent within the reporting week. Discard any outside the week.

For each qualifying sent email:
1. **Day**: date the email was sent (YYYY-MM-DD)
2. **Infer customer** from recipient email domains and/or subject line
3. **Estimate time**:
   - Subject or preview contains attachment or link indicators ("attached", "attachment", "please find", "see attached", "enclosed", "http", "https", "www", "click here") → **1.0 hour**
   - Otherwise → word-count estimate from the preview text:
     `time_hrs = max(5, min(45, preview_word_count / 8)) / 60`

### 4C — Received emails

Filter emails from Step 2C to those received within the reporting week. Discard others.

Skip emails where the sender domain matches your own organisation (internal emails).

For each qualifying received email:
1. **Day**: date the email was received (YYYY-MM-DD)
2. **Infer customer** from sender email domain or subject
3. **Estimate time**: `time_hrs = max(5, min(45, preview_word_count / 8)) / 60`

### 4D — Desktop and Documents files

For each file returned in Step 3:
1. **Day**: use the `day` field from the script output (YYYY-MM-DD)
2. **Infer customer** from the filename or subfolder path (e.g. "Contoso_ScopeDoc_v2.docx" or "Projects/Contoso/proposal.docx" → Contoso)
3. **Assign time**: **0.5 hours** (30 minutes) per file

---

## Step 5 — Consolidate by customer and day

Group all activity items by **customer** and **day**. Sum hours within each customer-day cell.

- Normalise obvious name variations (e.g. "Acme" and "AcmeCorp" → "Acme Corp")
- Activities with no identifiable customer → group under **"Internal / Unknown"**
- Round all values to 1 decimal place

Build a grid: rows = customers, columns = Mon / Tue / Wed / Thu / Fri, plus a Total column (sum across all days). Also add a TOTAL row summing each column.

---

## Step 6 — Save the report

Save the report as `effort-[YYYY-MM-DD].md` (using the Monday date) in the working directory using `write_file`. Use this exact structure:

```
# Weekly Customer Effort Report
**Week of [Mon D MMM YYYY] to [Fri D MMM YYYY]**
*Generated [today's date]*

---

## Summary (hours per customer per day)

| Customer | Mon [D] | Tue [D] | Wed [D] | Thu [D] | Fri [D] | Total |
|---|---|---|---|---|---|---|
| [Customer A] | x.x | x.x | x.x | x.x | x.x | **x.x** |
| [Customer B] | x.x | x.x | x.x | x.x | x.x | **x.x** |
| Internal / Unknown | x.x | x.x | x.x | x.x | x.x | **x.x** |
| **TOTAL** | **x.x** | **x.x** | **x.x** | **x.x** | **x.x** | **x.x** |

---

## Activity Detail

### [Customer A]
**Meetings:** [Meeting title] on [Day] — [duration + prep] = x.x hrs
**Work blocks:** [Block title] on [Day] — [duration] = x.x hrs
**Emails:** [x] sent, [y] received across the week — x.x hrs total
**Files:** [filename] modified [Day] — 0.5 hrs

### [Customer B]
...
```

Use the day number (date) in the column header, e.g. "Mon 24", "Tue 25".

After saving, tell the user:
1. The filename saved to their Documents folder
2. A quick inline summary of the top 3 customers by total hours
3. A reminder that they can ask to adjust any estimate that looks off

---

## Edge Cases

- **No meetings / emails / files found**: Note which source returned nothing and continue with the others.
- **Ambiguous customer name**: Use the most specific name you can infer. When truly ambiguous, use "Internal / Unknown".
- **Emails outside the reporting week**: Silently discard — `list_emails` returns the latest N items and may include older ones.
- **Desktop or Documents folders not found**: Note in the report: "Desktop and/or Documents folders were not found or had no files modified this week."
- **All-day events**: Skip entirely — these are not customer-facing activities.
- **Zero hours on a day**: Show `—` in that cell rather than `0.0` for readability.
- **Work block with no inferable customer name**: Skip — do not guess; only include blocks where a customer can be clearly identified from the title.