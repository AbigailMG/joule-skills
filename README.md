# Weekly Customer Effort Tracker

A Joule Work Desktop skill that scans your Outlook calendar, emails, and desktop files to estimate time spent per customer each week. Outputs a formatted Markdown report saved to your Documents folder.

---

## What It Does

- Scans **calendar meetings**, **sent emails**, **received emails**, and **recently modified desktop files** for the reporting week
- **Infers customer names** from meeting titles, email subjects, sender/recipient domains, and filenames
- **Estimates effort** per customer per day using meeting durations, email word counts, and file activity
- Produces a **Markdown report** (`effort-YYYY-MM-DD.md`) broken down by customer and day

---

## How to Activate

Say any of the following to Joule:

- *"Run my weekly effort tracker"*
- *"How much time did I spend on customers this week?"*
- *"Generate my weekly effort report"*
- *"Track my customer hours"*
- *"Run the effort tracker for last week"*

---

## Output

The report is saved to your **Documents** folder as `effort-YYYY-MM-DD.md` (Monday date of the reporting week).

### Example Report Structure

```
# Weekly Customer Effort Report
Week of Mon 24 Aug 2026 to Fri 28 Aug 2026

| Customer          | Mon 24 | Tue 25 | Wed 26 | Thu 27 | Fri 28 | Total  |
|-------------------|--------|--------|--------|--------|--------|--------|
| Contoso           |  2.0   |  1.5   |  —     |  1.0   |  0.5   |  5.0   |
| Fabrikam          |  —     |  0.5   |  2.0   |  —     |  —     |  2.5   |
| Internal / Unknown|  1.0   |  0.5   |  0.5   |  1.0   |  0.5   |  3.5   |
| TOTAL             |  3.0   |  2.5   |  2.5   |  2.0   |  1.0   |  11.0  |
```

---

## Effort Estimation Rules

| Activity | Estimation Method |
|---|---|
| Calendar meeting ≤ 30 min | Duration + 5 min prep |
| Calendar meeting > 30 min | Duration + 10 min prep |
| Sent email with attachment/link | 1.0 hour |
| Sent or received email (no attachment) | `max(5, min(45, word_count / 8)) / 60` hours |
| Desktop file modified this week | 0.5 hours per file |

Activities with no identifiable customer are grouped under **Internal / Unknown**.

---

## Configuration

Customise the tracker's behaviour by editing **`effort-tracker-config.json`** in your Documents folder.

### Key Settings

| Setting | What to Edit |
|---|---|
| `internal_domains` | Add your organisation's email domains — emails from these are treated as internal and excluded |
| `known_customers` | List your active customers with their email domains and meeting title keywords |
| `customer_aliases` | Map name variations (e.g. "AcmeCorp") to a canonical name ("Acme Corp") |
| `excluded_meeting_keywords` | Add meeting title keywords to skip (e.g. "Team Standup", "Coffee") |
| `effort_estimation` | Adjust prep time thresholds or desktop file hours |

### Recommended First-Time Setup

1. Open `effort-tracker-config.json`
2. Confirm `internal_domains` includes your organisation's domain(s)
3. Add your active customers to `known_customers` — include their email domains and any shorthand names used in meeting titles
4. Add any customer name variations you've seen to `customer_aliases`

---

## Files

| File | Description |
|---|---|
| `effort-tracker-config.json` | Configuration file — customise this to improve accuracy |
| `effort-YYYY-MM-DD.md` | Generated weekly reports (one per week, saved to Documents) |

---

## Notes & Limitations

- The tracker scans up to **50 calendar events** and **50 emails per folder** per run
- Desktop files are scanned from `~/Desktop`, `~/Desktop/Customer`, and `~/Desktop/CEP`
- All-day calendar events are skipped
- If a customer name cannot be inferred, the activity is logged under **Internal / Unknown**
- Estimates are approximations — you can ask Joule to adjust any figure that looks off after the report is generated

---

## Skill Details

| | |
|---|---|
| **Skill name** | `weekly-customer-effort-tracker` |
| **Version** | 1.1.0 |
| **Author** | Abi / Joule Skill Creator |
| **Tags** | productivity, time-tracking, customer, weekly-report, outlook |
