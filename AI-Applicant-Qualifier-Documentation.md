# AI Applicant Qualifier + Scheduling Management

## 1. Problem

Reviewing job applicants manually is slow — recruiters often have to read through every submission one by one just to figure out who's actually worth prioritizing, and applications can sit unreviewed for days while other tasks take priority. On top of that, once a candidate is approved, scheduling the interview and keeping applicants informed adds another round of manual back-and-forth.

This project automates both halves of that process. It uses an AI agent to screen incoming applicants against a specific job's requirements — scoring them, classifying their fit, and explaining why — so a recruiter can make a faster, better-informed decision. Once a decision is made, the workflow automatically handles the scheduling link, the rejection or approval email, and internal team notifications, including a built-in escalation path if a recruiter doesn't respond in time.

## 2. Architecture

```
Google Form → Webhook → AI Agent (screen + score) → Gmail (approval request to recruiter)
                                                              ↓
                                          Switch: Approved / Reject / No Update
                                                              ├─ Approved  → Sheets lookup → Dedup check → Calendly link email → Telegram (internal) → Sheets (Done)
                                                              ├─ Reject    → Sheets lookup → Dedup check → Rejection email    → Telegram (internal) → Sheets (Done)
                                                              └─ No Update → Telegram alert → 2nd approval request (48h) → escalation email
```

**Flow breakdown:**

The workflow triggers when an applicant submits the Google Form. A **Webhook** node captures the response, and an **AI Agent** assesses the applicant's tenure/years of experience, relevant skills, and stated interest in the role — scoring them from 1–10 and classifying them as "Strong Match," "Good Match," or "Do Not Match," along with a written reason for the recommendation. This is sent to the recruiter via **Gmail** as an approval request, including the applicant's name, score, reasoning, classification, and a link to their resume — the recruiter can approve or reject directly from that email, using their own judgment alongside the AI's assessment.

The flow then branches through a **Switch** node into three paths:

**If APPROVED:**
- A Google Sheets lookup retrieves the applicant's full details.
- An If node checks whether the applicant has already been processed, to prevent duplicate handling.
- A second Switch confirms the approve/reject outcome.
- The applicant receives an email with a Calendly link to schedule their interview.
- An internal Telegram message notifies the team to confirm the process.
- Google Sheets updates the applicant's STATUS column to "Done."

**If REJECTED:**
- Follows the same lookup and duplicate-check steps as the Approved path.
- The applicant receives a rejection email instead of a scheduling link.
- An internal Telegram message notifies the team.
- Google Sheets updates the STATUS column to "Done."

**If NO UPDATE (recruiter hasn't responded within the time limit):**
- A Telegram message alerts the internal team that an applicant hasn't been reviewed.
- A second Gmail approval request is sent, with a new 48-hour response window.
- If the recruiter responds within that window, the flow continues down the Approved or Reject path as normal.
- If there's still no response after 48 hours, the workflow escalates via a third Gmail message to a higher-level contact.

## 3. Tools Used

- **n8n** (self-hosted via Docker)
- **Groq / OpenRouter** (AI applicant screening and scoring)
- **Gmail** (approval requests, applicant emails, escalation notices)
- **Telegram** (internal team notifications)
- **Google Sheets** (applicant logging and status tracking)
- **Google Forms** (application intake, including resume upload)
- **Calendly** (interview scheduling, handled outside n8n)

## 4. Problems Solved

**Problem:** The AI's structured output was missing required fields due to a schema mismatch.
**Cause:** The JSON Schema's `required` array referenced field names that didn't exactly match the properties defined above it (a typo in one field name, and a reference to a field that didn't exist under that name).
**Fix:** Corrected the `required` array to match the actual property names exactly, which resolved the validation mismatch.

**Problem:** The resume link included in the recruiter's approval email led to a broken page ("This site can't be reached").
**Cause:** Google Forms' file upload only returns the file's internal Drive ID, not a full usable URL — and that ID was also being passed through as an array rather than a plain string.
**Fix:** Added a Set node to convert the field from Array to String, and concatenated it into a proper Google Drive URL (`https://drive.google.com/file/d/{id}/view`) before it reached the AI prompt and the email template.

**Problem:** The Structured Output Parser intermittently failed with "Model output doesn't fit required format."
**Cause:** The AI's raw response occasionally didn't match the expected JSON structure exactly — a mismatch between what the model returned and the schema's expected types/format.
**Fix:** Reviewed the raw model output via the parser's error details to identify the exact mismatch, and adjusted the prompt/schema accordingly to keep output consistent.

**Problem:** Early test runs showed vague AI reasoning that didn't clearly reference applicant skills.
**Cause:** The Google Form hadn't yet included a dedicated Skills field, so the AI had no structured skill data to evaluate — despite the prompt asking it to weigh skills at 40% of the score.
**Fix:** Added a checkboxed Skills field to the form, mirroring the job posting's Requirements and Nice-to-have sections, giving the AI concrete, comparable data to score against instead of inferring skills from unrelated fields.

## 5. Limitations & Next Steps

- **No duplicate-applicant detection at the form-submission stage** — the current dedup check only prevents re-processing an already-approved/rejected applicant, not a person submitting the form multiple times before a decision is made.
- **Resume content itself isn't evaluated by the AI** — screening is based only on structured form fields (skills, experience, interest). The resume is provided as a link for the recruiter to review manually; no parsing/extraction has been built yet.
- **Escalation currently ends at a single higher-level contact** — there's no further fallback if that escalation also goes unanswered; at that point it requires manual follow-up.
- **Skills are collected via a fixed checklist tailored to one job posting** — scaling this to multiple open roles would require either a dynamic skills list per role or a more general free-text field with added scoring complexity.
