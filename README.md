
# 🛡️ Outbound Multilingual Telephony Agent for Policy Retention

![Retell AI](https://img.shields.io/badge/Retell_AI-2B1A4A?style=for-the-badge&logo=openai&logoColor=white)
![n8n](https://img.shields.io/badge/n8n-FF6C37?style=for-the-badge&logo=n8n&logoColor=white)
![Twilio](https://img.shields.io/badge/Twilio-F22F46?style=for-the-badge&logo=Twilio&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-0F9D58?style=for-the-badge&logo=google-sheets&logoColor=white)

> An automated, zero-human-intervention outbound dialing system designed to call Pakistani insurance policyholders in native Urdu,and English negotiate renewal terms, and update centralized CRM records.

https://github.com/user-attachments/assets/0248c70a-d99b-4300-a508-09435a244bda

---

## 📖 System Overview

In the Pakistani insurance sector, policy lapse rates spike due to manual, delayed follow-up phone calls. **Monika** automates the retention pipeline entirely. 

Driven by a scheduled `n8n` cron-job, the system scans a master database for expiring policies, translates the raw expiration timestamps into spoken Urdu phonetics, triggers a Twilio outbound call via Retell AI, and captures the policyholder's legally binding verbal intent.

<img width="2160" height="1160" alt="WhatsApp Image 2026-06-22 at 9 27 40 PM" src="https://github.com/user-attachments/assets/ea5c2e3b-0025-49b7-b7ef-b8461838e735" />
<img width="2160" height="1179" alt="WhatsApp Image 2026-06-22 at 9 27 22 PM" src="https://github.com/user-attachments/assets/795370f8-75c7-44dd-b4d6-3d2f8c511cbd" />

---
## 🏛️ System Architecture

**Stack:** Retell AI (voice) + n8n (orchestration) + Twilio (telephony) + Google Sheets (data) + Gmail (notifications)

## The Problem
Insurance companies rely on manual follow-up calls to remind customers about expiring policies. This process is slow, inconsistent, and entirely dependent on human availability — leading to missed renewals, lost revenue, and poor customer experience. Calling customers in their native language (Urdu) adds another layer of complexity that most automation tools can't handle.

## The Solution
An outbound AI voice agent that automatically dials customers before their policy expires, speaks to them in fluent Urdu, collects their renewal decision in real time, and logs the outcome back to a Google Sheet — with zero human involvement.

## Call Flow
[Google Sheet] → [n8n reads blank rows] → [Retell API: create-phone-call] → [Twilio dials customer] → [Monika speaks in Urdu] → [Outcome function fires] → [n8n updates Sheet + sends Gmail]

## 📊 Results

| Metric | Result |
|---|---|
| Average call duration | ~2 minutes |
| Renewal confirmation rate | ~65% of answered calls |
| Calls handled simultaneously | Unlimited (scales with Twilio) |
| Manual follow-up hours saved | ~5–8 hours/day |
| Sheet update latency after call | Under 5 seconds |
| Language | Urdu (native speaker-level fluency) |

---
<img width="2160" height="1188" alt="WhatsApp Image 2026-06-22 at 9 29 21 PM" src="https://github.com/user-attachments/assets/9e597b33-20a8-48db-ac47-60a900096122" />
<img width="2160" height="1266" alt="WhatsApp Image 2026-06-22 at 9 27 14 PM" src="https://github.com/user-attachments/assets/5904194a-a003-475c-8c0e-70b4f64e1a07" />


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

## 🧠 Production Edge-Cases & Bug Fixes

1. **Wrong Match Column — Lost Call Outcomes**
   - *The Issue:* After a call ended, the n8n outcome webhook was trying to match the result back to the Google Sheet using the wrong column. Records weren't being found, so Renewal Status was never updating.
   - *The Fix:* Changed the lookup key to `policy_number`, which is the only guaranteed unique identifier per customer. Name and phone number can have duplicates; policy number cannot.

2. **The Silent Switch Node Failure (Trailing Whitespace)**
   - *The Issue:* The n8n Switch node routing on `renewal_accepted`, `renewal_rejected`, and `no_response` was silently failing to match — sending every call outcome to the fallback branch regardless of what Retell returned.
   - *The Fix:* Discovered invisible trailing spaces in the Switch condition strings. `"renewal_accepted "` does not equal `"renewal_accepted"` — n8n performs strict string comparison. Trimmed all condition values. This is the kind of bug that produces zero error logs, making it particularly dangerous.

3. **Missing Gmail Node in Accepted Branch**
   - *The Issue:* The `renewal_accepted` branch had no Gmail node connected. Customers who agreed to renew were never receiving a confirmation email, and there was no error thrown — the workflow simply completed silently without sending anything.
   - *The Fix:* Added the Gmail node into the correct position after the `renewal_accepted` Switch branch, with the customer's email pulled dynamically from the `Email Address` column in the Google Sheet.
