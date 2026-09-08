# AI Resume ATS — Automated Resume Scoring & Candidate Routing

An automated Applicant Tracking System (ATS) workflow built in **n8n** that monitors a Google Drive folder for new resumes, extracts resume content, evaluates candidates against predefined scoring parameters using an OpenAI model, logs results in Google Sheets, and automatically routes candidates based on their total score.

## Workflow Overview

![Resume ATS Workflow](resume-ats-workflow.png)

The workflow:

1. Detects a newly uploaded resume in Google Drive.
2. Finds and loops through resume files.
3. Downloads each resume.
4. Extracts text from the PDF.
5. Sends the extracted resume content to an AI agent.
6. Scores the candidate against the configured evaluation parameters.
7. Parses the AI response into structured JSON.
8. Logs the analysis in Google Sheets.
9. Routes the candidate based on the final score.
10. Sends the appropriate email automatically.

## Candidate Routing Logic

The main decision rule is:

```text
Total Score > 65
    → Candidate passes
    → Add candidate to the Passed sheet
    → Send shortlisted email
    → Include the link for the second-round interview

Total Score <= 65
    → Candidate is rejected
    → Add candidate to the Failed/Rejected sheet
    → Send rejection email
```

The workflow also contains a **Needs Review** route that can be used for resumes that require manual verification, such as incomplete information, parsing issues, or uncertain AI output.

## Workflow Architecture

```text
Google Drive Trigger
        │
        ▼
Search Files and Folders
        │
        ▼
Loop Over Items
        │
        ▼
Download Resume
        │
        ▼
Extract Text from PDF
        │
        ▼
AI Agent + OpenAI Chat Model
        │
        ▼
Parse AI JSON + Route
        │
        ├──────────────► Move / Organize File
        │
        └──────────────► Append Analysis to Google Sheets
                                │
                                ▼
                         Candidate Status Switch
                         ┌───────┼────────┐
                         │       │        │
                     Rejected  Review   Passed
                         │       │        │
                         ▼       ▼        ▼
                      Failed   Needs    Passed
                       Sheet   Review    Sheet
                         │                │
                         ▼                ▼
                   Rejection Email   Shortlist Email
                                      + Round 2 Link
```

## Main Integrations

- **n8n** — workflow automation
- **Google Drive** — resume upload source and file handling
- **PDF/Text Extraction** — converts resumes into text for analysis
- **OpenAI Chat Model** — evaluates resumes against the scoring criteria
- **Google Sheets** — stores candidate details, scores, status, and analysis
- **Gmail** — sends rejection and shortlisted-candidate emails

## Example Scoring Parameters

The exact scoring model can be changed depending on the role. A typical setup may include:

| Parameter | Example Weight |
|---|---:|
| Relevant work experience | 25 |
| Required technical skills | 25 |
| Education / certifications | 15 |
| Projects / achievements | 15 |
| Role-specific keywords | 10 |
| Resume quality / clarity | 10 |
| **Total** | **100** |

> Replace these example weights with the actual criteria used for the position.

## Recommended AI Output Format

The AI agent should return a predictable JSON object so the routing node can process the result reliably.

```json
{
  "candidate_name": "Candidate Name",
  "email": "candidate@example.com",
  "experience_score": 20,
  "skills_score": 22,
  "education_score": 12,
  "projects_score": 13,
  "keyword_score": 8,
  "resume_quality_score": 8,
  "total_score": 83,
  "status": "passed",
  "summary": "Strong match for the role based on experience and required skills."
}
```

Suggested values for `status`:

- `passed`
- `rejected`
- `needs_review`

The workflow should validate the JSON before using it for candidate routing.

## Threshold Configuration

The current pass condition is:

```text
total_score > 65
```

A score of exactly **65** is treated as rejected under this rule.

If you want 65 to pass as well, change the condition to:

```text
total_score >= 65
```

## Google Sheets Structure

A useful sheet structure is:

| Column | Description |
|---|---|
| Candidate Name | Extracted candidate name |
| Email | Candidate email address |
| File Name / URL | Resume reference |
| Total Score | Final ATS score |
| Status | Passed, Rejected, or Needs Review |
| AI Summary | Short explanation of the evaluation |
| Processed At | Date/time the resume was processed |

You can maintain separate sheets/tabs for:

- `Analyzed`
- `Passed`
- `Rejected`
- `Needs Review`

## Email Automation

### Passed Candidate

Candidates with a score above 65 receive a shortlisted email containing the **second-round interview link**.

Example:

```text
Subject: You have been shortlisted for the next round

Hi {{candidate_name}},

Thank you for applying.

Your profile has been shortlisted for the second round of our hiring process.

Please use the link below to continue:

{{second_round_interview_link}}

Best regards,
Hiring Team
```

### Rejected Candidate

Candidates scoring 65 or below receive a rejection email.

```text
Subject: Update on your application

Hi {{candidate_name}},

Thank you for taking the time to apply.

After reviewing your profile against the requirements for this position,
we will not be moving forward with your application at this time.

We appreciate your interest and wish you the best in your job search.

Best regards,
Hiring Team
```

## Setup

Before running the workflow, configure credentials for:

1. Google Drive
2. Google Sheets
3. Gmail
4. OpenAI

Then configure:

- Resume upload folder
- Destination/processed folder
- Google Sheets spreadsheet and tab names
- ATS scoring parameters
- Passing threshold (`65`)
- Second-round interview URL
- Email sender details and templates

## Suggested Configuration Variables

```text
ATS_PASS_SCORE=65
SECOND_ROUND_INTERVIEW_LINK=https://your-interview-link.example
RESUME_FOLDER_ID=your_google_drive_folder_id
SPREADSHEET_ID=your_google_sheet_id
```

For n8n, these values can also be stored as workflow variables, credentials, or environment variables instead of being hard-coded inside nodes.

## Reliability Recommendations

- Force the AI output to follow a strict JSON schema.
- Validate that `total_score` is numeric and within the expected range.
- Prevent duplicate processing by storing the resume file ID.
- Add an error workflow for failed PDF extraction or API calls.
- Route malformed or incomplete AI responses to `needs_review`.
- Keep the scoring rubric consistent for all applicants for the same role.
- Avoid including protected or irrelevant personal characteristics in scoring.
- Keep a human-review path available for hiring decisions.

## Privacy & Security

Resumes contain personal information. When deploying this workflow:

- Restrict access to Google Drive and Sheets.
- Do not expose API keys inside workflow exports or Git repositories.
- Store credentials using n8n's credential system.
- Limit stored candidate information to what is necessary.
- Define a retention/deletion policy for resumes and evaluation data.
- Review applicable employment, privacy, and automated-decision requirements for your jurisdiction.

## Use Case

This workflow is useful for teams that receive a large number of resumes and want to automate the first screening stage while keeping the scoring logic consistent.

The automation handles repetitive screening tasks, while recruiters can focus on shortlisted candidates and any applications routed for manual review.

## Disclaimer

AI-generated resume scores should be treated as decision-support signals rather than the sole basis for employment decisions. The scoring criteria should be job-related, periodically reviewed for bias, and paired with appropriate human oversight.
