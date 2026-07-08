# AI Recruitment Screener & Auto-Scheduler

An end-to-end n8n automation that reads job applications straight from Gmail, extracts and scores resumes against a job description using AI, logs everything to Google Sheets, and auto-sends acceptance or rejection emails — with zero manual screening.

n8n + Google Gemini + Google Workspace

> Alt names for this project: ScreenerAI · HireFlow AI · TalentGate · RecruitBot Pro

---

## What It Does

1. Watches a Gmail inbox for new emails every minute.
2. Uses an AI classifier to check if the email is actually a job application (filters out spam, newsletters, unrelated mail).
3. Extracts the resume text from the attached CV — works with PDF, JPEG, and PNG formats.
4. Logs the applicant's details into a Google Sheet.
5. Reads the current Job Description from a Google Doc.
6. Scores the applicant (0–100) by comparing their resume against the job requirements.
7. Updates the same sheet row with the score.
8. Automatically sends an **interview invitation** if the score is 80+, or a **polite rejection email** if it's below 80.

## Workflow Diagram

```
Gmail Trigger
     │
     ▼
AI Email Classifier (Job Application? YES/NO)
     │  YES
     ▼
Attach Resume Binary (Code Node)
     │
     ▼
Analyze Document (Gemini — extract resume text from PDF/image)
     │
     ▼
AI Agent (Recruitment Screening Agent)
     ├─ Tool: Log applicant → Google Sheets
     ├─ Tool: Read Job Description → Google Docs
     └─ Tool: Update score → Google Sheets
     │
     ▼
Extract Score (Code Node — regex parses score from AI output)
     │
     ▼
IF Score >= 80 ?
     ├─ YES → Send Interview Invitation (Gmail)
     └─ NO  → Send Rejection Email (Gmail)
```

## Tech Stack

| Component | Tool |
|---|---|
| Automation platform | [n8n](https://n8n.io) |
| Email trigger & sending | Gmail (OAuth2) |
| Resume parsing | Google Gemini (multimodal document analysis) |
| Screening & scoring agent | Google Gemini via n8n AI Agent (LangChain) |
| Applicant tracking | Google Sheets |
| Job description source | Google Docs |
| Score extraction | Custom JavaScript (Code node) |

## Node-by-Node Breakdown

### 1. Email Scraper
`Gmail Trigger` → `Basic LLM Chain` → `If1`
Polls Gmail every minute. Every new email is passed to an AI classifier that responds with a strict `YES`/`NO` on whether it's a job application (looking for cues like "resume," "CV," "applying," "position," etc.). Only `YES` emails proceed.

### 2. Resume Reader
`Code in JavaScript1` → `Analyze document`
Reattaches the binary CV attachment (n8n loses binary data through LLM chain nodes, so this node manually reattaches it) and passes it to Gemini, which can read PDF, JPEG, or PNG resumes and extract clean plain-text candidate details — name, phone, email, skills, education, experience.

### 3. Screening Agent
`AI Agent` (with `Google Gemini Chat Model`)
A single AI Agent runs the full screening logic with three tools:
- **Log tool** (Google Sheets) — appends a new row with applicant name, email, subject, resume summary, and date received. Never guesses missing fields.
- **JD tool** (Google Docs) — reads the live job description (skills, experience, qualifications required).
- **Update tool** (Google Sheets) — writes the score back into the same row (matched by name/email), without touching other columns.

The agent is instructed to score strictly 0–100 based only on what's explicitly stated in the resume — no assumptions about unlisted skills.

### 4. Decision & Auto-Response
`Code in JavaScript` → `If` → `Send a message` / `Send a message1`
A regex-based Code node extracts the numeric score and applicant name from the agent's raw output (handles formats like `88/100`, `Score: 88`, `scored 88`, etc.). An `IF` node then routes:
- **Score ≥ 80** → sends a personalized interview invitation email.
- **Score < 80** → sends a respectful rejection email.

## Setup Requirements

- n8n instance (cloud or self-hosted)
- Gmail account connected via OAuth2 (with read + send scopes)
- Google Gemini API credentials
- A Google Sheet for applicant tracking (columns: Applicant Name, Email, Subject, Resume Summary, Date Received, Score)
- A Google Doc containing the current Job Description
- Update the hardcoded job title and email copy in the "Send a message" / "Send a message1" nodes to match your role

## Known Limitations / Notes

- Score threshold (80) and job title text are currently hardcoded — swap these out per role/client.
- Score parsing relies on regex matching common AI output formats; if the agent's phrasing drifts, the pattern list may need an update.
- Currently built for a single open role at a time (one Job Description doc). Multi-role support would need a lookup step to match applicants to the correct JD.

## Business Use Case

Designed for HR teams or hiring managers who get high email volume and want first-pass screening automated — cutting manual resume review time while keeping a full audit trail (sheet log) of every applicant and their score.

