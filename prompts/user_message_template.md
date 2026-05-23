# User Message Template

This is the message body to send to Claude from Make.com.
Replace `{{ }}` placeholders with data from Google Sheets and the patient's SMS reply.

---

**Prescription details:**
- Drug: {{drug_name}}
- Rx #: {{rx_number}}
- Refill due: {{refill_date}}
- Doctor: {{doctor_name}} — {{doctor_phone}}
- Pharmacy: {{pharmacy_phone}}

**Patient's reply:**
"{{patient_sms_reply}}"

---

## Make.com: How to Build This Message

In Make.com's HTTP module (the Claude API call), set the `user` message content to:

```
Prescription details:
- Drug: {{1.drug_name}}
- Rx #: {{1.rx_number}}
- Refill due: {{1.refill_date}}
- Doctor: {{1.doctor_name}} — {{1.doctor_phone}}
- Pharmacy: {{1.pharmacy_phone}}

Patient's reply:
"{{3.Body}}"
```

Where `{{1.*}}` values come from the Google Sheets module and `{{3.Body}}` is the Twilio inbound SMS body.
