# Make.com Scenario Guide

Build three scenarios in Make.com. Each is described below with exact module settings.

---

## Before You Start: Store Your Secrets

In Make.com, go to **Variables** (left sidebar) and create these global variables.  
Never hardcode keys in module fields — always reference variables.

| Variable Name              | Value Source                          |
|---------------------------|---------------------------------------|
| `ANTHROPIC_API_KEY`       | Anthropic console                     |
| `TWILIO_ACCOUNT_SID`      | Twilio console                        |
| `TWILIO_AUTH_TOKEN`       | Twilio console                        |
| `TWILIO_PHONE_NUMBER`     | Your Twilio number (+1XXXXXXXXXX)     |
| `YOUR_PHONE_NUMBER`       | Your personal number (+1XXXXXXXXXX)   |
| `VAPI_API_KEY`            | Vapi dashboard                        |
| `VAPI_DOCTOR_ASSISTANT_ID`| Vapi dashboard → Assistants           |
| `VAPI_PHONE_NUMBER_ID`    | Vapi dashboard → Phone Numbers        |
| `GOOGLE_SPREADSHEET_ID`   | From Google Sheets URL                |
| `PATIENT_NAME`            | Your full name                        |
| `PATIENT_DOB`             | Your date of birth (MM/DD/YYYY)       |
| `CALLBACK_NUMBER`         | Your personal phone number            |

---

## Scenario 1: Daily Prescription Check

**Trigger:** Schedule — runs every day at 9:00 AM

### Modules

**Module 1 — Schedule**
- Type: `Schedule`
- Interval: Every day at 09:00 (your local timezone)

**Module 2 — Get Prescriptions**
- Type: `Google Sheets → Get Spreadsheet Rows`
- Spreadsheet ID: `{{GOOGLE_SPREADSHEET_ID}}`
- Sheet Name: `Sheet1`
- Table contains headers: Yes
- Range: Leave empty (reads all rows)

**Module 3 — Iterate Rows**
- Type: `Flow Control → Iterator`
- Array: `{{2.rows}}`

**Module 4 — Check Refill Date**
- Type: `Flow Control → Filter` (add a filter after the iterator)
- Label: `Refill due soon?`
- Condition: `{{formatDate(addDays(now; 0); "YYYY-MM-DD")}}` ≥ `{{formatDate(addDays(parseDate(4.Refill Date; "YYYY-MM-DD"); multiply(-1; 4.Days Notice)); "YYYY-MM-DD")}}`
- This fires when: today is on or after (Refill Date minus Days Notice)

**Module 5 — Send SMS Alert**
- Type: `Twilio → Send an SMS`
- From: `{{TWILIO_PHONE_NUMBER}}`
- To: `{{YOUR_PHONE_NUMBER}}`
- Body:
  ```
  Hi! Your {{4.Drug Name}} refill is due in {{dateDifference(parseDate(4.Refill Date; "YYYY-MM-DD"); now; "days")}} days (Rx {{4.Rx #}}). Do you need a reorder? Reply YES, NO, or tell me what's going on.
  ```

**Module 6 — Log Sent**
- Type: `Google Sheets → Update a Row`
- Spreadsheet ID: `{{GOOGLE_SPREADSHEET_ID}}`
- Sheet Name: `Sheet1`
- Row Number: `{{3.rowNumber}}` (the row number returned by the Iterator)
- Fields to update:
  - `Last Notified` → `{{formatDate(now; "YYYY-MM-DD HH:mm:ss")}}`
  - `Notify Attempt` → `1`
  - `Reply Received` → *(leave blank / clear the field)*
- This prevents duplicate alerts and seeds the retry counter for Scenario 4

---

## Scenario 2: Inbound SMS Handler (Claude Agent)

**Trigger:** Twilio inbound SMS webhook

### Setup

1. In Make.com, create a new scenario
2. Add a **Webhook** module as the trigger (type: Custom Webhook)
3. Copy the webhook URL Make.com gives you
4. In Twilio console → Phone Numbers → your number → Messaging → "A message comes in" → set to that webhook URL

### Modules

**Module 1 — Webhook Trigger**
- Type: `Webhooks → Custom Webhook`
- This receives the Twilio SMS payload
- Key fields: `Body` (the patient's reply), `From` (patient's number)

**Module 1b — Mark Reply Received**
- Type: `Google Sheets → Search Rows`
- Find the row where `Last Notified` is not blank and `Reply Received` is blank (the active pending refill)
- Then immediately follow with `Google Sheets → Update a Row` on that row:
  - `Reply Received` → `{{formatDate(now; "YYYY-MM-DD HH:mm:ss")}}`
  - `Notify Attempt` → `0` (stops Scenario 4 from sending further retries)
- Do this **before** calling Claude so retries are halted even if Claude or Vapi fail

**Module 2 — Get Prescription Context**
- Type: `Google Sheets → Search Rows`
- Search for rows where the refill is upcoming (same filter as Scenario 1)
- Or: `Get Spreadsheet Rows` and filter to the most recently notified prescription
- This gives you the prescription details to pass to Claude

**Module 3 — Call Claude Agent**
- Type: `HTTP → Make a Request`
- URL: `https://api.anthropic.com/v1/messages`
- Method: POST
- Headers:
  ```
  x-api-key: {{ANTHROPIC_API_KEY}}
  anthropic-version: 2023-06-01
  content-type: application/json
  ```
- Body (JSON):
  ```json
  {
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "system": "<paste contents of prompts/system.md here>",
    "tools": <paste contents of prompts/tools.json here>,
    "tool_choice": {"type": "auto"},
    "messages": [
      {
        "role": "user",
        "content": "Prescription details:\n- Drug: {{2.Drug Name}}\n- Rx #: {{2.Rx #}}\n- Refill due: {{2.Refill Date}}\n- Doctor: {{2.Doctor Name}} — {{2.Doctor Phone}}\n\nPatient's reply:\n\"{{1.Body}}\""
      }
    ]
  }
  ```

**Module 4 — Parse Claude's Tool Call**
- Type: `JSON → Parse JSON`
- JSON string: `{{3.content[]}}`
- You need to extract `content[1].input` where `content[1].type == "tool_use"`
- Parse the `action`, `confirmation_sms`, `doctor_call_notes`, `outcome_sms_template` fields

**Module 5 — Send Confirmation SMS**
- Type: `Twilio → Send an SMS`
- From: `{{TWILIO_PHONE_NUMBER}}`
- To: `{{YOUR_PHONE_NUMBER}}`
- Body: `{{4.confirmation_sms}}` (from Claude's tool output)

**Module 6 — Router**
- Type: `Flow Control → Router`
- Route 1: `action == "DOCTOR"` → go to Module 7
- Route 2: `action == "SKIP"` → end (no call needed)

**Module 7 — Trigger Doctor Call**
- Type: `HTTP → Make a Request`
- URL: `https://api.vapi.ai/call/phone`
- Method: POST
- Headers: `Authorization: Bearer {{VAPI_API_KEY}}`
- Body: See `vapi/trigger_call_payload.json` — use doctor assistant ID, doctor phone from Google Sheets
- Set `CALL_REASON` to `{{4.doctor_call_notes}}`

---

## Scenario 3: Vapi Call Outcome Webhook

**Trigger:** Vapi fires a webhook when a call ends

### Setup

1. In Make.com, create a new scenario with a **Custom Webhook** trigger
2. Copy the webhook URL
3. In Vapi dashboard → Settings → Server URL → paste the webhook URL
4. Vapi will POST call results here when each call ends

### Modules

**Module 1 — Webhook Trigger**
- Type: `Webhooks → Custom Webhook`
- Receives Vapi call completion payload
- Key fields: `message.call.id`, `message.transcript`, `message.endedReason`

**Module 2 — Extract Outcome**
- Type: `Flow Control → Set Variable`
- Determine outcome from `message.endedReason`:
  - `assistant-said-end-call-phrase` → success
  - `voicemail` → left voicemail
  - `customer-did-not-answer` → no answer
  - `max-duration-exceeded` → timed out (hung up)
- Also extract from `message.call.metadata`:
  - `row_number` → the Google Sheets row to update
  - `supply_days` → the refill cycle length (e.g. 30 or 90)
  - `drug_name` → used in the outcome SMS

**Module 3 — Send Outcome SMS**
- Type: `Twilio → Send an SMS`
- From: `{{TWILIO_PHONE_NUMBER}}`
- To: `{{YOUR_PHONE_NUMBER}}`
- Body: Build a message from the outcome. Example:
  ```
  Update on your {{2.drug_name}} refill: {{OUTCOME_TEXT}}
  Transcript summary: {{truncate(1.message.transcript; 200)}}
  ```

**Module 4 — Update Refill Date (on success only)**
- Type: `Flow Control → Filter` → only continue if outcome is `success`
- Then: `Google Sheets → Update a Row`
  - Spreadsheet ID: `{{GOOGLE_SPREADSHEET_ID}}`
  - Sheet Name: `Sheet1`
  - Row Number: `{{2.row_number}}` (from call metadata)
  - Fields to update:
    - `Refill Date` → `{{formatDate(addDays(now; toNumber(2.supply_days)); "YYYY-MM-DD")}}`
  - This advances the refill date by the prescription's supply length, preventing re-alerts until the next cycle
- **Do not** update the Refill Date for voicemail/no-answer outcomes — the system should continue alerting until confirmed

---

## Scenario 4: SMS Retry (No Reply)

**Trigger:** Schedule — runs every 3 hours

If the patient has not replied to an SMS alert, this scenario retries up to 4 total attempts (the original send from Scenario 1, plus 3 retries), spaced 3 hours apart.

### Modules

**Module 1 — Schedule**
- Type: `Schedule`
- Interval: Every 3 hours (e.g. 09:00, 12:00, 15:00, 18:00)
- Align start times with Scenario 1's 9am run so the first retry window is 12:00

**Module 2 — Get All Prescriptions**
- Type: `Google Sheets → Get Spreadsheet Rows`
- Spreadsheet ID: `{{GOOGLE_SPREADSHEET_ID}}`
- Sheet Name: `Sheet1`
- Table contains headers: Yes

**Module 3 — Iterate Rows**
- Type: `Flow Control → Iterator`
- Array: `{{2.rows}}`

**Module 4 — Filter: Retry Needed?**
- Type: `Flow Control → Filter`
- Label: `Needs retry?`
- All of the following conditions must be true:
  1. `Notify Attempt` ≥ `1` AND `Notify Attempt` ≤ `3`
     *(at least one SMS was sent and we have retries remaining)*
  2. `Reply Received` is empty
     *(patient has not replied)*
  3. `{{formatDate(addSeconds(parseDate(3.Last Notified; "YYYY-MM-DD HH:mm:ss"); 10800); "YYYY-MM-DD HH:mm:ss")}}` ≤ `{{formatDate(now; "YYYY-MM-DD HH:mm:ss")}}`
     *(at least 3 hours have passed since last notification — 10800 seconds = 3 hours)*

**Module 5 — Send Retry SMS**
- Type: `Twilio → Send an SMS`
- From: `{{TWILIO_PHONE_NUMBER}}`
- To: `{{YOUR_PHONE_NUMBER}}`
- Body (vary message by attempt number for clarity):
  ```
  Reminder: Your {{3.Drug Name}} refill is due in {{dateDifference(parseDate(3.Refill Date; "YYYY-MM-DD"); now; "days")}} days (Rx {{3.Rx #}}). Reply YES, NO, or tell me what's going on. (Attempt {{add(3.Notify Attempt; 1)}} of 4)
  ```

**Module 6 — Increment Attempt Counter**
- Type: `Google Sheets → Update a Row`
- Spreadsheet ID: `{{GOOGLE_SPREADSHEET_ID}}`
- Sheet Name: `Sheet1`
- Row Number: `{{3.rowNumber}}`
- Fields to update:
  - `Last Notified` → `{{formatDate(now; "YYYY-MM-DD HH:mm:ss")}}`
  - `Notify Attempt` → `{{add(3.Notify Attempt; 1)}}`

> **How the 4-attempt cap works:** Scenario 1 sends attempt 1 and writes `Notify Attempt = 1`. Scenario 4 fires when `Notify Attempt` is 1, 2, or 3 (max 3 retries). After incrementing to 4, the filter condition `Notify Attempt ≤ 3` fails and retries stop automatically. The patient receives 4 total SMS messages, each 3 hours apart.

---

## Tips

- **Test mode:** In Make.com, use "Run once" to test each scenario manually before activating the schedule
- **Error handling:** Add error handlers to the Twilio and Vapi HTTP modules to catch failures
- **Dedup protection:** Add a "Last Notified" date column to the sheet and filter it in Scenario 1 to avoid sending multiple alerts per day
- **Business hours:** In Scenario 2, add a filter before Module 7 to check that the current time is Mon–Fri, 9am–5pm. If not, store the pending action and trigger it at 9am the next business day using a scheduled scenario
