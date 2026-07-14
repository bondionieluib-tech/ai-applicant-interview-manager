# Applicant Interview Manager

An n8n workflow that screens job applicants with AI, sends recruiters a scored recommendation to approve or reject, then automatically handles scheduling, rejection emails, and internal notifications — including escalation if a recruiter doesn't respond in time.

## What it does

Applicants submit a Google Form, which triggers an AI agent to assess their experience, skills, and stated interest against a specific job posting. The AI returns a score (1–10), a classification ("Strong Match," "Good Match," or "Do Not Match"), and a written recommendation — all sent to the recruiter in an approval email alongside a link to the applicant's resume.

From there:

- ✅ **Approved** → applicant receives a Calendly link to schedule their interview; internal team is notified
- ❌ **Rejected** → applicant receives a rejection email; internal team is notified
- ⏳ **No response from recruiter** → internal alert is sent, a second approval request goes out with a 48-hour window, and if that also goes unanswered, the workflow escalates to a higher-level contact

Every applicant's status is logged and updated in Google Sheets throughout.

## Architecture

<img width="1324" height="582" alt="image" src="https://github.com/user-attachments/assets/70e4e0d7-df08-4022-9665-a9b652934dae" />


## Tools

n8n (self-hosted, Docker) · Groq & OpenRouter · Gmail · Telegram · Google Sheets · Google Forms · Calendly

## Notes

- Full write-up, including bugs encountered and how they were fixed, is in [`documentation.md`](./documentation.md)
- Known limitations: no duplicate-detection at submission time, resume content isn't AI-evaluated (link only, for manual review), and escalation currently ends at one fallback contact
