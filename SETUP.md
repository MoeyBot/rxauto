# Setup Guide

Complete these steps in order. Estimated total time: 2–4 hours.

---

## Step 0: Clone & Configure Secrets

```bash
git clone <your-repo-url>
cd rxauto
cp .env.example .env
# Edit .env and fill in all values as you complete the steps below
```

The `.env` file is gitignored. You will never commit it. For Make.com, you will mirror each secret as a Make.com variable instead.

---

## Step 1: Google Sheets

1. Create a new Google Sheet at [sheets.google.com](https://sheets.google.com)
2. Import `sheets/prescription_template.csv` — File → Import → Upload
3. Replace sample data with your real prescription information
4. See `sheets/README.md` for column definitions and access setup
5. Copy your Spreadsheet ID from the URL and add to `.env` and Make.com variables

**APIs to enable:** [Google Sheets API](https://console.cloud.google.com/apis/library/sheets.googleapis.com)

---

## Step 2: Twilio

1. Sign up at [twilio.com](https://www.twilio.com)
2. Buy a phone number (~$1/month) with SMS capability
3. Copy your Account SID, Auth Token, and phone number to `.env` and Make.com variables
4. Leave the inbound SMS webhook URL blank for now — you'll fill it in after creating Scenario 2

---

## Step 3: Anthropic (Claude)

1. Sign up at [console.anthropic.com](https://console.anthropic.com)
2. Create an API key under Settings → API Keys
3. Copy to `.env` as `ANTHROPIC_API_KEY` and to Make.com variables
4. The Claude agent prompt is in `prompts/system.md`
5. The tool definitions are in `prompts/tools.json`

**Note:** Claude is called by Make.com via HTTP — no additional setup needed beyond the API key.

---

## Step 4: Vapi.ai

1. Sign up at [vapi.ai](https://vapi.ai)
2. **Create the Doctor Call Agent:**
   - Go to Assistants → Create Assistant
   - Name: `RxAuto Doctor Caller`
   - System Prompt: copy the `model.systemPrompt` value from `vapi/doctor_agent.json`
   - Voice: choose a natural-sounding voice (11labs recommended)
   - Max call duration: 300 seconds
   - Save and copy the Assistant ID → add to `.env` as `VAPI_DOCTOR_ASSISTANT_ID`
3. **Get a Vapi phone number:**
   - Go to Phone Numbers → Buy Number
   - Copy the Phone Number ID → you'll need it in Make.com
4. **Configure webhook:**
   - Go to Settings → Server URL
   - Leave blank for now — fill in after creating Make.com Scenario 3

---

## Step 5: Make.com

1. Sign up at [make.com](https://make.com) (free tier is sufficient)
2. **Store variables:** Go to Variables in the left sidebar and add all keys from `.env.example`
3. **Build Scenario 1** (Daily Check): follow `make/scenario_guide.md` → Scenario 1
4. **Build Scenario 2** (SMS Handler + Claude Agent): follow `make/scenario_guide.md` → Scenario 2
   - After creating the Scenario 2 webhook, copy its URL
   - Paste into Twilio: Phone Numbers → your number → Messaging → "A message comes in"
5. **Build Scenario 3** (Vapi Outcome): follow `make/scenario_guide.md` → Scenario 3
   - After creating the Scenario 3 webhook, copy its URL
   - Paste into Vapi: Settings → Server URL
6. **Build Scenario 4** (SMS Retry): follow `make/scenario_guide.md` → Scenario 4

---

## Step 6: End-to-End Test

Test each piece individually before a full test:

### Test 1: Google Sheets connection
- In Make.com Scenario 1, click "Run once"
- Verify it reads your prescriptions without error

### Test 2: Twilio SMS
- In Make.com Scenario 1, manually trigger it with a prescription due today
- Verify you receive the SMS on your phone

### Test 3: Claude classification
- Send a test reply from your phone (reply to the SMS, or manually post to the Scenario 2 webhook)
- Verify Make.com calls Claude and returns a `route_prescription_request` tool call
- Check the `action` and `confirmation_sms` fields

### Test 4: Vapi calls
- Use a test phone number (e.g. your own number, or a Google Voice number)
- Trigger a Vapi call manually via Make.com Scenario 2 with `action = DOCTOR`
- Listen to the call and verify the script is correct

### Test 5: Full end-to-end
- Set one prescription's refill date to tomorrow
- Wait for Scenario 1 to run (or trigger it manually)
- Reply to the SMS
- Verify the call fires, completes, and you receive the outcome SMS

---

## Ongoing Maintenance

- **Update refill dates** after each successful reorder (manual or via Make.com Scenario 3)
- **Add new prescriptions** directly to the Google Sheet — no code changes needed
- **Tune Vapi scripts** based on real call behavior (edit `vapi/doctor_agent.json` and update the Vapi dashboard)
- **Adjust Days Notice** per-prescription in the sheet (e.g. 7 days for a controlled substance that takes longer)

---

## Troubleshooting

| Issue | Check |
|-------|-------|
| No SMS received | Twilio account balance, phone number SMS settings, Make.com scenario is active |
| Claude returns plain text | Make sure `tool_choice` is set in the HTTP body, tools JSON is valid |
| Vapi call doesn't trigger | Vapi API key, phone number ID, `customer.number` must be E.164 format |
| Vapi webhook not firing | Vapi Server URL is set, Make.com scenario 3 is active and webhook is listening |
| Google Sheets 403 | Sheet is shared with service account email, or API key has Sheets API enabled |

---

## Cost Monitoring

| Service | Where to Watch |
|---------|---------------|
| Twilio | Console → Monitor → Usage |
| Vapi | Dashboard → Usage |
| Anthropic | Console → Usage |
| Make.com | Account → Operations |

Set billing alerts in each service to avoid surprise charges.
