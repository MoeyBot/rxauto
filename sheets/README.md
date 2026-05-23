# Google Sheet Setup

## Create Your Sheet

1. Go to [Google Sheets](https://sheets.google.com) and create a new spreadsheet
2. Import `prescription_template.csv`: File → Import → Upload → select the file
3. Replace the sample data with your real prescriptions
4. Copy the Spreadsheet ID from the URL:
   ```
   https://docs.google.com/spreadsheets/d/THIS_IS_YOUR_ID/edit
   ```
5. Paste it into `.env` as `GOOGLE_SPREADSHEET_ID` and into Make.com as a variable

## Column Reference

| Column | Description | Format |
|--------|-------------|--------|
| Drug Name | Full drug name and dosage | `Lisinopril 10mg` |
| Rx # | Prescription number from the label | `RX-1234` |
| Refill Date | When the current supply runs out | `YYYY-MM-DD` |
| Days Notice | How many days before Refill Date to get alerted | `5` |
| Doctor Name | Your prescribing doctor's name | `Dr. Smith` |
| Doctor Phone | Doctor's office direct line | `312-555-0100` |
| Supply Days | Days in the dispensed supply — used to auto-advance Refill Date after a reorder | `30` or `90` |
| Last Notified | Auto-updated by Make.com with the datetime of the most recent SMS alert (don't edit) | `YYYY-MM-DD HH:mm:ss` |
| Notify Attempt | Auto-updated by Make.com — counts SMS attempts (1–4); reset to 0 when reply received (don't edit) | `1` |
| Reply Received | Auto-updated by Make.com with the datetime the patient replied (don't edit) | `YYYY-MM-DD HH:mm:ss` |
| Notes | Any freeform notes for yourself | Optional |

## Access for Make.com

**Option A: API Key (read-only)**
1. Make the sheet publicly readable (Share → Anyone with link → Viewer)
2. Create an API key at [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
3. Enable the Google Sheets API in your project
4. Use the API key in Make.com's Google Sheets connection

**Option B: Service Account (read + write, recommended)**
1. Create a service account at [Google Cloud Console](https://console.cloud.google.com/iam-admin/serviceaccounts)
2. Download the JSON key file — **do not commit this file** (it's in `.gitignore`)
3. Share the Google Sheet with the service account email (give Editor access)
4. Use the service account in Make.com's Google Sheets connection
5. This enables Make.com to write back "Last Notified" and updated refill dates

## Keeping Refill Dates Current

Make.com Scenario 3 automatically updates the "Refill Date" column after a successful call, using the row's `Supply Days` value to compute the next refill date. No manual update needed after a reorder.

A typical refill cycle:
- 30-day supply → `Supply Days = 30`
- 90-day supply → `Supply Days = 90`

If a call outcome is unclear (voicemail, no answer), the Refill Date is left unchanged so the system keeps alerting.
