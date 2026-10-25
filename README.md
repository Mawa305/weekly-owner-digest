# Weekly Owner Digest

An n8n workflow that sends an HVAC business owner a short weekly summary every Monday at 08:00. It reads the past seven days from the missed calls log, the lead sheets, the renewals and reviews tables in Supabase, and writes a plain-text email with OpenAI. If the AI step fails, a basic version of the numbers is sent instead. Every send is recorded in a log table.

## Architecture

```
Schedule Trigger (every Monday, 08:00)
   ├── Get Missed Calls (Google Sheets)
   ├── Get Qualified Leads (Google Sheets)
   ├── Get Unqualified Leads (Google Sheets)
   ├── Get Invalid Submissions (Google Sheets)
   ├── Get Renewals (Supabase: renewals)
   └── Get Reviews (Supabase: reviews)
          ↓ all six feed
   Wait For All Data (Merge)
          ↓
   Calculate Weekly Stats (last 7 days, counts and review sentiment)
          ↓
   Write Digest (OpenAI: email text)
        ├── success → Send Digest (Gmail)
        │               ├── success → Log Sent (Supabase: digest_log)
        │               └── error   → Log Failed → Alert Failed (Telegram)
        └── error   → Send Fallback Digest (Gmail: plain numbers)
                        ├── success → Log Fallback (Supabase: digest_log)
                        └── error   → Log Failed → Alert Failed (Telegram)
```

## Features

* Runs every Monday at 08:00
* Counts missed calls, new leads (qualified and unqualified), invalid form submissions, renewals, and reviews from the last seven days
* Shows positive and negative review counts from the reviews table
* AI writes a short summary with one suggested action for the coming week
* The AI prompt uses only the counted numbers and is told not to invent anything
* Falls back to a plain-number email if the AI step fails
* Logs each digest as `sent`, `sent_fallback`, or `failed`
* Telegram alert when a digest cannot be sent at all

## Tech Stack

* n8n 2.x (workflow automation, Merge node)
* Google Sheets (missed calls and leads)
* Supabase (renewals, reviews, digest log)
* OpenAI (digest text)
* Gmail API (owner email)
* Telegram Bot API (failure alert)

## Setup

1. Prepare these data sources:
   * **Missed calls sheet**: a column named `timestamp` with the date and time of each missed call.
   * **Leads sheet**: two tabs, `Qualified leads` and `Unqualified leads`, plus a tab `Invalid Submissions`. Each row needs a `timestamp` column.
   * **Supabase `renewals`**: a `created_at` column.
   * **Supabase `reviews`**: columns `sentiment` (values `positive` or `negative`) and `created_at`. This is the table written by the Post-Job Review workflow.
   * **Supabase `digest_log`**: columns `period` and `status`.
2. Import `weekly_owner_digest.json` into n8n.
3. Add credentials: Google Sheets OAuth2, Supabase, OpenAI, Gmail OAuth2, and Telegram. Assign them to the matching nodes.
4. Replace these placeholders:
   * `YOUR_MISSED_CALLS_SHEET_ID`: missed calls sheet ID (Get Missed Calls node)
   * `YOUR_LEADS_SHEET_ID`: leads sheet ID (the three lead sheet nodes)
   * `YOUR_EMAIL@example.com`: the owner's email (Send Digest and Send Fallback Digest nodes)
   * `YOUR_TELEGRAM_CHAT_ID`: your chat ID (Alert Failed node)
5. In the three lead sheet nodes, check that the tab names match your sheet. The tab IDs in the file are from the original sheet, so select the correct tab in each node if needed.
6. Run the workflow once by hand and check the email and the `digest_log` row.
7. Activate the workflow.

## Example Output

Sample numbers only, not real data:

```
Weekly Owner Digest: 2026-10-18 to 2026-10-25

Summary: Busy week with 12 missed calls and 9 new leads.
- Missed calls: 12
- New leads: 9 (qualified 6, unqualified 3)
- Invalid submissions: 2
- Renewals: 4
- Reviews: 5 (positive 4, negative 1)

Suggested action: Call back the 12 missed calls first thing Monday.
```

## Project Notes

* Built as part of my AI Automation portfolio.
* All numbers are counted in code. The AI only writes the wording around them, so the model cannot change the figures.
* Only the OpenAI digest node uses `gpt-4o-mini`. Change its model in that node if you want another model.
