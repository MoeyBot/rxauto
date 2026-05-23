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
| `VAPI_PHARMACY_ASSISTANT_ID`| Vapi dashboard → Assistants         |
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

**Module 6 — Log Sent (Optional)**
- Type: `Google Sheets → Update a Row`
- Update a "Last Notified" column in your sheet to track when you sent the alert
- Useful to prevent duplicate alerts if the scenario runs multiple times

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
        "content": "Prescription details:\n- Drug: {{2.Drug Name}}\n- Rx #: {{2.Rx #}}\n- Refill due: {{2.Refill Date}}\n- Doctor: {{2.Doctor Name}} — {{2.Doctor Phone}}\n- Pharmacy: {{2.Pharmacy Phone}}\n\nPatient's reply:\n\"{{1.Body}}\""
      }
    ]
  }
  ```

**Module 4 — Parse Claude's Tool Call**
- Type: `JSON → Parse JSON`
- JSON string: `{{3.content[]}}`
- You need to extract `content[1].input` where `content[1].type == "tool_use"`
- Parse the `action`, `confirmation_sms`, `doctor_call_notes`, `pharmacy_call_notes`, `outcome_sms_template` fields

**Module 5 — Send Confirmation SMS**
- Type: `Twilio → Send an SMS`
- From: `{{TWILIO_PHONE_NUMBER}}`
- To: `{{YOUR_PHONE_NUMBER}}`
- Body: `{{4.confirmation_sms}}` (from Claude's tool output)

**Module 6 — Router**
- Type: `Flow Control → Router`
- Route 1: `action == "DOCTOR"` → go to Module 7a
- Route 2: `action == "PHARMACY"` → go to Module 7b
- Route 3: `action == "BOTH"` → go to both 7a and 7b (use Repeater or two parallel paths)
- Route 4: `action == "SKIP"` → end (no calls needed)

**Module 7a — Trigger Doctor Call**
- Type: `HTTP → Make a Request`
- URL: `https://api.vapi.ai/call/phone`
- Method: POST
- Headers: `Authorization: Bearer {{VAPI_API_KEY}}`
- Body: See `vapi/trigger_call_payload.json` — use doctor assistant ID, doctor phone from Google Sheets
- Set `CALL_REASON` to `{{4.doctor_call_notes}}`

**Module 7b — Trigger Pharmacy Call**
- Same as 7a but use `VAPI_PHARMACY_ASSISTANT_ID` and pharmacy phone number
- Set `CALL_REASON` to `{{4.pharmacy_call_notes}}`

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

**Module 3 — Send Outcome SMS**
- Type: `Twilio → Send an SMS`
- From: `{{TWILIO_PHONE_NUMBER}}`
- To: `{{YOUR_PHONE_NUMBER}}`
- Body: Build a message from the outcome. Example:
  ```
  Update on your {{DRUG_NAME}} refill: {{OUTCOME_TEXT}} 
  Transcript summary: {{truncate(1.message.transcript; 200)}}
  ```

**Module 4 — Update Sheet (Optional)**
- Type: `Google Sheets → Update a Row`
- Update the "Refill Date" column to +30 or +90 days when a call succeeds
- Prevents the system from re-alerting immediately after a successful reorder

---

## Tips

- **Test mode:** In Make.com, use "Run once" to test each scenario manually before activating the schedule
- **Error handling:** Add error handlers to the Twilio and Vapi HTTP modules to catch failures
- **Dedup protection:** Add a "Last Notified" date column to the sheet and filter it in Scenario 1 to avoid sending multiple alerts per day
- **Business hours:** In Scenario 2, add a filter before Module 7a/7b to check that the current time is Mon–Fri, 9am–5pm. If not, store the pending action and trigger it at 9am the next business day using a scheduled scenario
