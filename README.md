# rxauto

Automated prescription refill management — reads your Google Sheet daily, texts you when a refill is due, classifies your reply with Claude AI, and makes the outbound calls via Vapi.ai.

> **Getting started?** See [SETUP.md](SETUP.md) for the complete step-by-step guide.

## Project Structure

```
rxauto/
├── .env.example              # All required secrets (copy to .env, never commit .env)
├── SETUP.md                  # Complete setup guide — start here
├── prompts/
│   ├── system.md             # Claude agent system prompt
│   ├── tools.json            # Claude tool definitions (route_prescription_request)
│   └── user_message_template.md  # How to build the user message in Make.com
├── vapi/
│   ├── doctor_agent.json     # Vapi doctor call agent config
│   └── trigger_call_payload.json  # Make.com HTTP body template for Vapi calls
├── make/
│   └── scenario_guide.md     # Step-by-step Make.com scenario build guide
└── sheets/
    ├── prescription_template.csv  # Import this to create your Google Sheet
    └── README.md             # Sheet setup and column reference
```

## How Claude Is Used (Agent with Tools)

Claude receives the prescription details + your SMS reply and must call the `route_prescription_request` tool — it cannot reply in plain text. The tool forces a structured output:

```json
{
  "action": "DOCTOR | SKIP",
  "confidence": "HIGH | MEDIUM | LOW",
  "confirmation_sms": "Got it — calling Dr. Smith now...",
  "doctor_call_notes": "Patient says prescription expired",
  "outcome_sms_template": "Done! {{CALL_RESULT}} They said 1–2 business days."
}
```

Make.com parses the tool call output and routes the Make.com scenario accordingly. See [prompts/tools.json](prompts/tools.json) and [make/scenario_guide.md](make/scenario_guide.md).

---

# Prescription Reorder Automation Plan

## What This System Does

A no-code automation pipeline that:
1. Reads your prescriptions from a Google Sheet daily
2. Texts you via SMS when a refill date is approaching
3. Lets you reply in plain English to decide what action to take
4. Uses Claude AI to classify your reply (Doctor call / Skip)
5. Triggers Vapi.ai to call the doctor's office on your behalf
6. Texts you back with the outcome

---

## Full System Flow

```
📋 Google Sheet checked daily
  → ⏰ Refill date approaching?
    → 📱 SMS sent to you

💬 You reply freely (e.g. "my doctor hasn't sent the script yet")
  → 🤖 Claude API classifies intent
    → Doctor call / Skip

📞 Vapi calls doctor's office
  → 📱 SMS outcome summary sent to you
```

---

## Example SMS Exchange

**System → You:**
> Hi! Your Lisinopril refill is due in 5 days. Do you need a reorder?

**You → System:**
> Yeah my doctor hasn't sent a new script yet

**Claude classifies:** call doctor only

**System → You:**
> Got it — calling Dr. Smith's office now. I'll text you when done.

**After call — System → You:**
> Done! Reached Dr. Smith's office and requested a refill for Lisinopril 10mg. They said 1–2 business days. 🟢

---

## Google Sheet Structure

| Drug Name | Rx # | Refill Date | Days Notice | Doctor Name | Doctor Phone | Supply Days | Last Notified | Notify Attempt | Reply Received | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| Lisinopril 10mg | RX-1234 | 2026-06-01 | 5 | Dr. Smith | 312-555-0100 | 30 | | | | |
| Metformin 500mg | RX-5678 | 2026-06-10 | 7 | Dr. Jones | 312-555-0101 | 90 | | | | |

You edit the first 7 columns. `Last Notified`, `Notify Attempt`, and `Reply Received` are auto-managed by Make.com. No app or database setup required.

---

## Tech Stack

### Google Sheets — Prescription Data
- You manually enter and maintain all prescription info
- The backend reads it daily via the Google Sheets API
- **Cost:** Free

### Make.com — Automation Backbone
- No-code visual workflow builder
- Connects every piece: reads the Sheet → sends SMS → waits for reply → calls Claude → triggers Vapi → sends outcome SMS
- **Cost:** Free tier (1,000 ops/month, personal use stays well under this)

### Twilio — SMS Send & Receive
- Sends the check-in text to your number
- Receives your reply via webhook back into Make.com
- **Cost:** ~$1–2/month ($1/mo phone number + ~$0.01 per message)

### Claude API — Intent Classification
- Make.com passes your SMS reply to Claude
- Prompt: "Is this asking to call the doctor, or skip?"
- Returns a clean routing decision (DOCTOR / SKIP)
- **Cost:** <$0.01 per reply

### Vapi.ai — Outbound AI Voice Calls
- AI voice agent calls the doctor's office
- Navigates IVR menus, speaks naturally with staff
- Leaves a voicemail if no answer
- **Cost:** ~$0.05–$0.20 per call (~$0.10 for a 2-minute call)

### Vapi Webhook → Twilio — Outcome Notification
- After the call ends, Vapi fires a webhook to Make.com with the result
- Make.com sends you a final SMS confirming what happened
- **Cost:** Included in above

---

## Build Phases

### Phase 1 — Day 1–2: Data Foundation
- Set up Google Sheet with all prescription data
- Connect Google Sheets to Make.com via Google Sheets API

### Phase 2 — Day 3–4: SMS Check-in
- Set up Twilio phone number
- Build Make.com scenario: daily cron → read sheet → check refill dates → send SMS

### Phase 3 — Day 5–6: Reply Handling
- Add inbound SMS webhook in Make.com
- Pass reply text to Claude API for DOCTOR / SKIP classification

### Phase 4 — Day 7–8: Outbound Calls
- Configure Vapi.ai agent script for doctor conversations
- Wire Make.com to trigger Vapi doctor call based on Claude's classification

### Phase 5 — Day 9–10: Outcome Loop & Testing
- Add Vapi webhook → Make.com → outcome SMS to user
- End-to-end test with real phone numbers
- Tune Vapi agent script based on actual call behavior

---

## Monthly Cost Estimate

| Service | Cost |
|---|---|
| Google Sheets | Free |
| Make.com | Free (under 1,000 ops/month) |
| Twilio SMS | ~$1–2/month |
| Claude API | <$0.01 per reply |
| Vapi.ai | ~$0.05–$0.20 per call |
| **Total (typical)** | **~$3–5/month** |

> **Worst case** (5 medications, all needing doctor calls in the same month): ~$5–7/month.
> 
> **Note:** Vapi costs can increase if calls run long (e.g. placed on hold). Set a call timeout in Vapi to cap this.

---

## Key Risks to Plan For

- **IVR variance:** Every doctor's office has a different phone tree. Vapi needs robust prompting to navigate common menus. Test with your actual contacts first.
- **Business hours:** Calls should only fire Mon–Fri, 9am–5pm local time. Add timezone-aware scheduling logic to avoid off-hours calls.
- **Voicemail scripts:** Voicemail messages must include patient name, DOB, Rx number, and callback number so offices can action the refill without calling back.
- **No-reply handling:** If you don't reply to the SMS check-in, the system retries every 3 hours for up to 4 total attempts before giving up. See Scenario 4 in `make/scenario_guide.md`.

---

## Features

- [x] Retry SMS if no reply after 3 hours — up to 4 total attempts (Scenario 4)
- [x] Auto-update refill date in Sheet after successful reorder (Scenario 3, Module 4)

## Enhancements for later
- [ ] **Pharmacy notifications** — add a Vapi pharmacy agent, a `Pharmacy Phone` column to the Sheet, and PHARMACY / BOTH routing in Claude and Make.com to call the pharmacy directly for refills already on file
- [ ] Batch multiple medications into one SMS check-in
- [ ] Log call transcripts back to Google Sheet
- [ ] Enforce business-hours call window per timezone