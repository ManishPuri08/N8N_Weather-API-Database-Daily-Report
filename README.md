# n8n Weather API → Database → Daily Report

> A scheduled workflow that pulls current weather data for a fixed location, logs it to a Google Sheet as a running record, and emails a daily update — automating a "check and log" task that would otherwise mean manually hitting a weather API and updating a spreadsheet.

## Problem & Goal

Keeping a running log of weather conditions for a location and notifying someone about it each day is a small, repetitive task if done by hand — open the API/site, note the numbers, update a sheet, write an email. This workflow removes that manual step: on a schedule, it calls a weather API, extracts the location fields, appends them as a new row in a Google Sheet, and emails a formatted summary of exactly what was just logged.

## Architecture

**Schedule Trigger → HTTP Request → Edit Fields → Append Row in Sheet → Gmail**

1. **Schedule Trigger** — kicks off the workflow on a timed interval (the interval isn't explicitly configured in this version and should be set before use).
2. **HTTP Request** — calls `GET http://api.weatherapi.com/v1/current.json?q=London&aqi=no` against WeatherAPI.com, with the API key passed directly in the URL query string.
3. **Edit Fields (Set)** — pulls five fields off the response into a `location.*` namespace: `location.country`, `location.lat`, `location.lon`, `location.tz_id`, `location.localtime`.
4. **Append Row in Sheet (Google Sheets)** — appends a new row to the **Weather** tab of a spreadsheet, mapping `Date` ← `location.localtime`, `Lat` ← `location.lat`, `Lon` ← `location.lon`, `City` ← `location.tz_id`. Note: the spreadsheet itself is titled **"Lead form"** — this looks like a sheet originally set up for something else (lead capture) and repurposed here for weather logging, worth renaming for clarity. Also note the `City` column is actually populated from `tz_id` (a timezone identifier like `Europe/London`), not a city name field — functional, but a slightly misleading column label.
5. **Gmail — Send a message** — sends an email to a fixed recipient with subject `"Weather"` and a message body built from `{{ $json.Date }}`, `{{ $json.Lat }}`, `{{ $json.Lon }}`, `{{ $json.City }}`. Because this node runs immediately after the Google Sheets append (which returns the written row using those same column names), these references resolve correctly — the email genuinely reflects what was just logged. This is a meaningful improvement over Project 5, where the equivalent formatted output was never actually wired into the email.

## Tools & Integrations Used

- **n8n** (Schedule Trigger, HTTP Request, Set/Edit Fields, Google Sheets, Gmail nodes)
- **WeatherAPI.com** — current weather data for a fixed location (`London`)
- **Google Sheets API** — OAuth2-connected, appends to a "Weather" tab in a spreadsheet currently named "Lead form"
- **Gmail API** — OAuth2-connected for sending the daily summary

## Setup Instructions

1. Import `Project-6 :Weather/API → Database → Daily Report.json` into your n8n instance (Workflows → Import from File).
2. Open the **Schedule Trigger** and set an explicit interval (e.g., once daily) — it's currently unconfigured/default.
3. Move the WeatherAPI key out of the hardcoded URL and into n8n credentials or an expression referencing a credential, rather than leaving it in plain text in the node's URL field. Also consider parameterizing the `q=London` location if this needs to track more than one place.
4. Rename the target spreadsheet (or point the **Append Row in Sheet** node at a dedicated spreadsheet) so it isn't still labeled "Lead form".
5. In the **Gmail** node, connect your own **Gmail OAuth2** credentials and update the hardcoded recipient.
6. Activate the workflow.

## Product Decisions

- **Flatten before writing, not raw response into the sheet:** The **Edit Fields** step extracts just the five location fields needed, rather than appending the full nested weather API response into the spreadsheet.
- **Email sourced from the sheet write, not the raw API call:** Placing **Gmail** after **Append Row in Sheet** (rather than branching both off the API response directly) means the email always reflects exactly what was persisted to the log — the two can't drift apart.
- **Append-only log, not overwrite:** Using the Sheets `append` operation builds a historical record over time rather than replacing a single "latest reading" cell — supports trend-checking later, not just a snapshot.

## How I Checked Accuracy

Based on what's in the workflow file itself, the things worth verifying before relying on this in production:

- **Confirm the field mapping survives a real run:** `City` is sourced from `tz_id`, not an actual city name — worth checking the sheet output looks right, or remapping to a real city field if the API response has one.
- **Confirm the Gmail expressions resolve:** the message body depends on the Google Sheets append node returning the row with `Date`/`Lat`/`Lon`/`City` keys — worth a live test run to confirm the email isn't silently sending blank values.
- **Confirm the API key isn't exposed anywhere it shouldn't be:** it's currently plain text in the HTTP Request URL, which means it's visible in this JSON export and in n8n's execution logs — worth rotating and moving to credentials before sharing this workflow further.
- **Confirm schedule cadence:** the trigger interval isn't set explicitly, so actual run frequency needs to be configured and verified.

## AI Evaluation

- Reviewed the node graph and connections (Claude) and confirmed this workflow, unlike Project 5, correctly threads its transformed output through to the email — the Gmail node's expressions match the column names the Sheets node just wrote, so the "report" step is functionally connected to the "log" step.
- Flagged a credential-hygiene issue: the WeatherAPI key is embedded directly in the HTTP Request node's URL rather than stored as a credential, which means it's exposed in plain text in the exported JSON.
- Noted a naming mismatch between the destination spreadsheet ("Lead form") and its actual current use (a weather log), and between the `City` column and the `tz_id` value actually stored in it — neither breaks the workflow, but both would likely confuse anyone reading the sheet without the workflow's logic in front of them.

## Limitations & Next Steps

- **Hardcoded API key in the URL:** should be moved to n8n credentials rather than left in plain text.
- **Fixed single location (`London`):** not parameterized — extending to multiple locations would need a loop or a list input.
- **No error handling:** a failed API call, sheet write, or send has no fallback or notification.
- **Mislabeled destination spreadsheet and column:** "Lead form" as the spreadsheet name and `City` holding a timezone ID are both worth cleaning up for anyone reading the log directly.
- **Unconfigured schedule interval:** needs an explicit cadence set before activation.
- **Single hardcoded recipient:** not yet parameterized for multiple recipients or environments.
