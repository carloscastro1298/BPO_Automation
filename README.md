# BPO Automation with n8n and AI

Two [n8n](https://n8n.io/) workflows that bring AI-assisted decision making to a BPO (Business Process Outsourcing) environment:

| Workflow | What it does |
|----------|--------------|
| **BPO Email Analysis** | Monitors a Gmail inbox, analyzes each incoming email with Google Gemini, and classifies it as **Critical** or **Non-Critical**. |
| **BPO Decision Making** | Reads agent decisions from an Excel file, scores risk and resolution time (TMO), analyzes each case with AI, and emails reports for critical and non-critical cases. |

Both workflows use **Google Gemini** (`gemini-3.5-flash-lite`) through n8n's AI Agent nodes.

---

## Repository Structure

```
.
├── BPO_Email_Analysis.json              # n8n workflow: email analysis
├── BPO_Decision_Making.json             # n8n workflow: decision analysis & reports
├── BPO_Decision_Making_Expanded.xlsx    # Sample dataset (50 agent decisions)
└── README.md
```

---

## 1. BPO Email Analysis

### How it works

```
Gmail Trigger (every minute, -in:sent)
   └─> Get Email Details
        └─> Prepare Email Data      (clean HTML, extract resolution time, TMO category)
             └─> Check if already processed (Data Table: Processed_Emails)
                  └─> [new email] Save email ID
                       └─> Save original email as .txt
                            └─> AI Agent: Analyze Email (Gemini + structured output)
                                 └─> Format Analysis
                                      └─> Critical?  ── yes ─> Write critical analysis (.txt)
                                                     └─ no  ─> Write non-critical analysis (.txt)
```

### Key features

- **Gmail trigger** polling every minute, ignoring sent mail.
- **Duplicate protection:** each processed email ID is stored in an n8n Data Table (`Processed_Emails`), so the same email is never analyzed twice.
- **HTML cleaning:** strips styles, scripts, tags, and entities to leave plain text for the AI.
- **TMO extraction:** a regex finds the expected resolution time in the email (supports English and Spanish: `resolution time`, `TMO`, `tiempo de resolución`).
- **Structured AI output:** the agent must return exactly five fields:

```json
{
  "email_summary": "The email promotes an online English course.",
  "email_category": "Marketing / Promotional",
  "criticality": "Critical | Non-Critical",
  "recommended_action": "Review the offer if interested.",
  "reason": "The email does not contain urgent information."
}
```

- **Prompt-injection safety:** the prompt tells the model to treat email content as untrusted data and never follow instructions found inside it.
- **Criticality rules:** emails are Critical when urgent action is needed to prevent a serious problem, financial loss, security risk, service interruption, or missed deadline. Angry tone alone does not make an email Critical.
- **File output:** the original email and the analysis are saved as `.txt` files in `/files/`.

### TMO categories

| Resolution time | Category |
|-----------------|----------|
| ≤ 5 minutes | Within Target |
| 6 – 30 minutes | Delayed - Within Limit |
| > 30 minutes | Critical |
| Not found | Not Specified |

---

## 2. BPO Decision Making

### How it works

```
Manual Trigger
   └─> Read BPO_Decision_Making_Expanded.xlsx
        └─> Extract from File
             └─> Code: classify decision type, risk, quality, criticality, TMO
                  └─> Critical?
                       ├─ Critical     ─> Loop ─> AI Agent (Critical)     ─> Wait ─> Aggregate ─> Excel + HTML email
                       └─ Non-Critical ─> Loop ─> AI Agent (Non-Critical) ─> Wait ─> Aggregate ─> Excel + HTML email
```

### Input data

`BPO_Decision_Making_Expanded.xlsx` (sheet `BPO_Decisions`) contains 50 sample cases with these columns:

`Employee_ID`, `Employee_Name`, `Situation`, `Decision`, `Reason`, `Result`, `Confidence`, `Resolution_Time_Minutes`

### Classification logic

| Decision | Decision type | Risk level |
|----------|---------------|------------|
| Listen first | Customer Support | Low |
| Inform the customer | Communication | Low |
| Explain the process | Guidance | Low |
| Check policy | Policy | Medium |
| Investigate the problem | Investigation | Medium |
| Ask a supervisor | Escalation | High |

- **Criticality:** `High` risk → **Critical**; everything else → **Non-Critical**.
- **Decision quality:** `Effective` when the result matches the expected outcomes for its risk level (for example, *Problem solved* or *Customer understood* for low risk; *Case solved* or *Exception approved* for high risk). Otherwise `Needs Review`.
- **TMO status:** same thresholds as the email workflow (≤ 5 / ≤ 30 / > 30 minutes). Cases above 30 minutes set `Alert_Required = true`.

### AI analysis

Two separate Gemini agents (one for critical cases, one for non-critical) analyze each case in simple English and explain:

- Why the case is critical or non-critical
- The possible risk or impact
- Whether the decision was good or needs review
- Whether the resolution time meets the target and any TMO risk
- One clear action (immediate action for critical cases, a practical suggestion for non-critical ones)

A **Wait** node between iterations helps avoid API rate limits.

### Outputs

For each group (critical and non-critical), the workflow sends an HTML email with:

- A styled report with the AI analysis for every case
- An Excel attachment: `Critical_Cases_Report.xlsx` or `Non_Critical_Cases_Report.xlsx`

---

## Setup

### Requirements

- An n8n instance (self-hosted or cloud) with **AI Agent**, **Data Table**, and **Read/Write Files from Disk** nodes available
- A **Google Gemini (PaLM) API** key
- A **Gmail OAuth2** credential

### Installation

1. In n8n, go to **Workflows → Import from File** and import `BPO_Email_Analysis.json` and `BPO_Decision_Making.json`.
2. Create your credentials and assign them to the nodes:
   - Google Gemini Chat Model nodes
   - Gmail nodes (trigger, get message, and send)
3. **Email Analysis:** create an n8n Data Table named `Processed_Emails` with two columns, `email_id` (string) and `processed_at` (date/time), and select it in the Data Table nodes.
4. **Decision Making:** place `BPO_Decision_Making_Expanded.xlsx` in the n8n `/files/` directory (or update the path in the *Read/Write Files from Disk* node).
5. In the **Critical Cases** and **Non-Critical Cases** Gmail nodes, replace the recipient with your own email address.
6. Activate **BPO Email Analysis** to start monitoring. Run **BPO Decision Making** manually with *Execute workflow*.

### Notes

- Files are written to `/files/` inside the n8n environment. Make sure that folder exists and is writable.
- The Email Analysis prompt contains a fixed "current date" used for testing deadline logic. Update it (or replace it with `{{ $now }}`) for real use.
- Never commit your API keys or credentials. n8n exports do not include credential secrets, but check the JSON before publishing.

---

## Tech Stack

- n8n (workflow automation)
- Google Gemini (LLM)
- Gmail API
- JavaScript (Code nodes)
- Excel (`.xlsx`) for input and reporting

---

## Team

- Carlos Castro
- Vanderley Castro

---
