# AI CV Screening Workflow (n8n)

A prototype n8n workflow that helps a recruiter shortlist CVs. It removes personal details from a CV, asks an LLM to score it against a fixed set of role criteria using only evidence in the CV, and saves a ranked result.

> Built as a learning project on **synthetic CVs only**. It supports a recruiter; it does not make hiring decisions.

## What it does

1. **Form intake**: upload a CV (PDF), enter the job title and paste a job description.
2. **Text extraction**: pulls the text out of the PDF.
3. **Anonymisation**: a code step removes the candidate's name, email, phone number and links *before* the model sees the CV. This reduces bias and keeps personal data away from the LLM.
4. **LLM scoring**: Google Gemini scores the CV against the role's required skills and returns structured JSON (score, recommendation, matched skills, missing skills, reasoning).
5. **Routing and saving**: every result is saved to an n8n Data table. Scores of 75 or above also trigger a shortlist email alert.
6. **Error handling**: if scoring fails for one CV, the run continues and that CV is saved with a score of 0 and a "scoring failed" note.

```
Form -> Extract PDF -> Anonymise -> Gemini scoring (structured output)
     -> Flatten -> IF score >= 75 -> Data table -> (shortlist) email alert
```

## Design choices

- **Anonymise first**: personal identifiers are stripped before scoring.
- **Evidence only**: the model is told to use only what the CV states, never to invent experience, and never to infer anything from name, gender, age, nationality or university.
- **Fixed, auditable criteria**: the required skills for the role are defined in the model's instructions rather than left to the model's guesswork.
- **Structured output**: a JSON schema keeps results machine-readable and consistent.
- **Low temperature** for more stable scores.

## About the prompt

The exact prompt wording is not published in this repo and is marked `[PROMPT REDACTED]` in `workflow.json`. In summary, it tells the model to act as a fair recruiter, assess only the listed required skills, list each as matched or missing, score mainly on the share of required skills evidenced, and keep its reasoning short.

## Stack

n8n (self-hosted with Docker), Google Gemini API, n8n Data tables, SMTP (optional).

## How to run

1. Start n8n: `docker run -it --rm -p 5678:5678 -v n8n_data:/home/node/.n8n n8nio/n8n`
2. Import `workflow.json` (Workflows, then Import from file).
3. Add your Gemini API key under Credentials and select the Google Gemini Chat Model.
4. Write your own scoring instructions in the **Score CV with LLM** node (the prompt in this repo is redacted).
5. Create a Data table `cv_screening_results` with columns: `candidateId`, `jobTitle`, `score` (number), `recommendation`, `matched_skills`, `missing_skills`, `reasoning`, `processedAt`, then select it in both save nodes.
6. Optional: connect SMTP for the shortlist alert, or delete that node.
7. Click **Execute workflow**, open the form, and upload one of the sample CVs.
8. If the PDF step fails, set the **Extract CV text** input field to the binary name shown in the form node's output.

## Test data

The three sample CV PDFs in this repo are invented (strong, partial and weak match for the sample role in `sample-job-description.txt`). Expected behaviour: the strong CV (Alex Morgan) scores highest and is shortlisted, the partial one (Sam Patel) lands in the middle, and the weak one (Jordan Lee) scores lowest.

## Limitations

- This is decision support. A human must make the final call.
- Scores are an LLM's judgement, so they can vary slightly between runs and depend on how a CV is written. Unusual formats or wording can be missed.
- Not tested for fairness or bias; this would be needed before any real use.
- Anonymisation uses simple pattern matching and can miss identifiers such as school names or unusual formats.
- Uses synthetic data only. Real candidate data would need proper GDPR handling (lawful basis, retention, transparency about automated processing).

## Possible next steps

- Per-role criteria entered by the recruiter in the form
- Batch upload and duplicate detection
- A human review step before any decision
- Bias testing across varied CVs and a scoring rubric with a visible breakdown
