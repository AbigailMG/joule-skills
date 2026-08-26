Weekly Customer Effort Tracker
Scans Outlook calendar, sent and received emails, and desktop files for the current working week to estimate hours spent per customer. Generates a downloadable Markdown report saved to Documents, with effort broken down by customer and by day.

Activation
Activate for requests like:

"Run my weekly effort tracker"
"How much time did I spend on customers this week / last week"
"Generate my weekly effort report"
"Track my customer hours"
"Run the effort tracker"
Step 1 — Determine the reporting week
Calculate the Monday and Friday of the target week based on today's date.

Default: the current working week (Mon–Fri). If today is a weekend, use the most recently completed week.
If the user specifies a past week (e.g. "last week", "the week of Aug 17"), adjust accordingly.
Store as WEEK_START (Monday 00:00:00) and WEEK_END (Friday 23:59:59) in ISO 8601 format. Example: 2026-08-24T00:00:00 and 2026-08-28T23:59:59.

Also store the date for each weekday as MON, TUE, WED, THU, FRI (YYYY-MM-DD format) — you will use these to tag each activity item with its day.

Tell the user: "Scanning your week of [Mon D MMM] to [Fri D MMM YYYY]…"

Step 2 — Gather all data sources in parallel
Make these 3 tool calls in a single message. They have no dependencies on each other.

A — Calendar
Call list_calendar_events with:

from: WEEK_START
to: WEEK_END
limit: 50
B — Sent emails
Call list_emails with:

folder: "sentitems"
top: 50
C — Received emails
Call list_emails with:

folder: "inbox"
top: 50
Wait for all 3 to return, then proceed.

Step 3 — Scan desktop files
Write the following Python script to _scan_desktop.py in the working directory using write_file. Substitute the actual Monday and Friday dates in YYYY, MM, DD integer format before writing.

import os, json
from datetime import datetime

week_start = datetime(YYYY, MM, DD, 0, 0, 0)    # Monday
week_end   = datetime(YYYY, MM, DD, 23, 59, 59) # Friday

home = os.path.expanduser("~")
folders = {
    "Desktop":          os.path.join(home, "Desktop"),
    "Desktop/Customer": os.path.join(home, "Desktop", "Customer"),
    "Desktop/CEP":      os.path.join(home, "Desktop", "CEP"),
}

results = []
for label, folder in folders.items():
    if not os.path.isdir(folder):
        continue
    for fname in os.listdir(folder):
        fpath = os.path.join(folder, fname)
        if os.path.isfile(fpath):
            mtime = datetime.fromtimestamp(os.path.getmtime(fpath))
            if week_start <= mtime <= week_end:
                results.append({
                    "folder": label,
                    "filename": fname,
                    "modified": mtime.strftime("%Y-%m-%d %H:%M"),
                    "day": mtime.strftime("%Y-%m-%d")
                })

print(json.dumps(results, indent=2))
Run the script with python _scan_desktop.py. Note the file list returned. Folders that do not exist are silently skipped. Delete _scan_desktop.py after running.

Step 4 — Infer customers and estimate effort
For every activity item, record both the customer name and the day (YYYY-MM-DD) it occurred. You will use day to populate the daily columns in Step 5.

4A — Calendar meetings
Filter events from Step 2A to only those within the reporting week. Skip:

Personal blocks with no customer reference (e.g. "Lunch", "Focus time", "OOO")
All-day events
Internal-only meetings with no external customer name in the title
For each qualifying event:

Day: date of the meeting start time (YYYY-MM-DD)
Infer customer from the meeting title (e.g. "Contoso Quarterly Review" → Contoso)
Calculate duration in minutes from start and end times
Add prep time: meetings ≤ 30 min → +5 min; meetings > 30 min → +10 min
Hours = (duration_min + prep_min) / 60
4B — Sent emails
Filter emails from Step 2B to those sent within the reporting week. Discard any outside the week.

For each qualifying sent email:

Day: date the email was sent (YYYY-MM-DD)
Infer customer from recipient email domains and/or subject line
Estimate time:
Subject or preview contains attachment or link indicators ("attached", "attachment", "please find", "see attached", "enclosed", "http", "https", "www", "click here") → 1.0 hour
Otherwise → word-count estimate from the preview text: time_hrs = max(5, min(45, preview_word_count / 8)) / 60
4C — Received emails
Filter emails from Step 2C to those received within the reporting week. Discard others.

Skip emails where the sender domain matches your own organisation (internal emails).

For each qualifying received email:

Day: date the email was received (YYYY-MM-DD)
Infer customer from sender email domain or subject
Estimate time: time_hrs = max(5, min(45, preview_word_count / 8)) / 60
4D — Desktop files
For each file returned in Step 3:

Day: use the day field from the script output (YYYY-MM-DD)
Infer customer from the filename (e.g. "Contoso_ScopeDoc_v2.docx" → Contoso)
Assign time: 0.5 hours (30 minutes) per file
Step 5 — Consolidate by customer and day
Group all activity items by customer and day. Sum hours within each customer-day cell.

Normalise obvious name variations (e.g. "Acme" and "AcmeCorp" → "Acme Corp")
Activities with no identifiable customer → group under "Internal / Unknown"
Round all values to 1 decimal place
Build a grid: rows = customers, columns = Mon / Tue / Wed / Thu / Fri, plus a Total column (sum across all days). Also add a TOTAL row summing each column.

Step 6 — Save the report
Save the report as effort-[YYYY-MM-DD].md (using the Monday date) in the working directory using write_file. Use this exact structure:

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
**Emails:** [x] sent, [y] received across the week — x.x hrs total
**Files:** [filename] modified [Day] — 0.5 hrs

### [Customer B]
...
Use the day number (date) in the column header, e.g. "Mon 24", "Tue 25".

After saving, tell the user:

The filename saved to their Documents folder
A quick inline summary of the top 3 customers by total hours
A reminder that they can ask to adjust any estimate that looks off
Edge Cases
No meetings / emails / files found: Note which source returned nothing and continue with the others.
Ambiguous customer name: Use the most specific name you can infer. When truly ambiguous, use "Internal / Unknown".
Emails outside the reporting week: Silently discard — list_emails returns the latest N items and may include older ones.
Desktop folders not found: Note in the report: "Desktop/Customer and/or CEP folders were not found or had no files modified this week."
All-day events: Skip entirely — these are not customer-facing activities.
Zero hours on a day: Show — in that cell rather than 0.0 for readability.
