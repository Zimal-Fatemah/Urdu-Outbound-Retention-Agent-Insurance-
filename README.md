
# 🛡️ Outbound Urdu Telephony Agent for Policy Retention

![Retell AI](https://img.shields.io/badge/Retell_AI-2B1A4A?style=for-the-badge&logo=openai&logoColor=white)
![n8n](https://img.shields.io/badge/n8n-FF6C37?style=for-the-badge&logo=n8n&logoColor=white)
![Twilio](https://img.shields.io/badge/Twilio-F22F46?style=for-the-badge&logo=Twilio&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-0F9D58?style=for-the-badge&logo=google-sheets&logoColor=white)

> An automated, zero-human-intervention outbound dialing system designed to call Pakistani insurance policyholders in native Urdu, negotiate renewal terms, and update centralized CRM records.

**[Watch Live Video Demo](https://youtube.com/your-video-link)** | **[View n8n Architecture](./n8n-workflows/Urdu_Outbound_Dispatcher.json)** *(ID: `NBtB2uvLBZnzrqvP`)*

---

## 📖 System Overview

In the Pakistani insurance sector, policy lapse rates spike due to manual, delayed follow-up phone calls. **Monika** automates the retention pipeline entirely. 

Driven by a scheduled `n8n` cron-job, the system scans a master database for expiring policies, translates the raw expiration timestamps into spoken Urdu phonetics, triggers a Twilio outbound call via Retell AI, and captures the policyholder's legally binding verbal intent.

---


## ⚙️ The Two-Stage Asynchronous Pipeline

Because an outbound phone call can last anywhere from 10 seconds to 10 minutes, the architecture is strictly split into **two decoupled Webhook processes** to keep server memory consumption near zero:

```text
   STAGE 1: THE DISPATCHER                           STAGE 2: THE RESOLVER
   
 ┌─────────────────────────┐                       ┌─────────────────────────┐
 │ G-Sheet (Policy DB)     │                       │ Retell AI Call Hook     │
 └────────────┬────────────┘                       └────────────┬────────────┘
              │ (Scan blank rows)                               │ (Webhook payload)
              ▼                                                 ▼
 ┌─────────────────────────┐                       ┌─────────────────────────┐
 │ n8n Code: Urdu Formatter│                       │  n8n Webhook Listener   │
 └────────────┬────────────┘                       └────────────┬────────────┘
              │                                                 │
              ▼                                                 ▼
 ┌─────────────────────────┐                       ┌─────────────────────────┐
 │ POST /v2/create-call    │                       │ Switch: Intent Parser   │
 └────────────┬────────────┘                       └──────┬───────────┬──────┘
              │                                           │           │
       (Twilio Outbound)                         (Accepted)           (Rejected)
              │                                           ▼           ▼
              ▼                                     ┌───────────┐ ┌───────────┐
     [ Customer Phone ]                             │G-Sheet: OK│ │G-Sheet: NO│
                                                    │ + Gmail   │ └───────────┘
                                                    └───────────┘
```

### The Database Schema (`Google Sheets`)
The sheet acts as both the trigger queue and the source of truth:

| `full_name` | `mobile_number` | `policy_number` | `expiry_date` | `renewal_status` *(Target)* |
| :--- | :--- | :--- | :--- | :--- |
| Tariq Jameel | `+923001234567` | `POL-88192` | 2026-06-28 | *(Blank - Pending Call)* |
| Fatima Noor | `+923339876543` | `POL-44102` | 2026-06-29 | `renewal_accepted` |

---

## 🗣️ Native Urdu Date Synthesis

A major point of failure in Urdu voice bots is the LLM reading an English string `"2026-06-25"` as *"Two thousand twenty-six dash zero six..."* To bypass this, Stage 1 passes the date through a JavaScript transformation node before sending it to Retell's dynamic variables:

```javascript
// Input: "2026-06-25" 
// Spoken output injected into LLM Prompt: "Pachees June, Do Hazaar Chhabees"

const months = ["January", "February", "March", "April", "May", "June", "July", "August", "September", "October", "November", "December"];
const parts = input.expiry_date.split('-');
const urduFormatted = `${parseInt(parts[2])} ${months[parseInt(parts[1]) - 1]}, Do Hazaar ${Chhabees}`; 
```

---

## 🪝 Deterministic Intent Functions 

During the call, Monika listens specifically to trigger one of three hardcoded boolean tool states:

1. `renewal_accepted`: Triggers if the user says words aligning with *"Jee haan karein"* (Yes, do it) or *"Theek hai"*.
2. `renewal_rejected`: Triggers on explicit refusal *"Nahi karwana"* or *"Band kar dein"*.
3. `no_response`: Fired automatically by Twilio if the line returns a `BUSY`, `FAILED`, or `NO_ANSWER` carrier code.

---

## 🚧 Critical Debugging History

* **The `=RENEWAL` Formula Trap:** When passing the string `"renewal_accepted"` back to the Google Sheet via the raw REST API, Google Sheets occasionally interpreted the leading character as an erroneous math formula (`=`), breaking the spreadsheet. Fixed by forcing an explicit `'` literal prefix escape in the JSON body.
* **The Trailing Space Death-Trap:** The Stage 2 `Switch Node` was failing to route accepted calls to the Gmail node because the Retell webhook payload was returning `"renewal_accepted "` (with an invisible trailing space). Sanitized the payload upstream using `{{ $json.outcome.trim() }}`.
* **Primary Key Realignment:** Originally tried matching incoming call resolutions to the spreadsheet's row ID. When the client sorted the Google Sheet by 'Name', the row IDs broke. Re-indexed the entire lookup logic to query strictly against the immutable `policy_number`.
