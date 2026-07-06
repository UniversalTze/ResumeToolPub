---
name: process-job
description: Runs the full job-application pipeline end to end for a single job posting. Use when the user wants everything done at once — phrases like "process the job in jobs/...", "run the full pipeline on jobs/acme.md", "do the new job", "tailor everything for this posting", or "generate a resume and cover letter for jobs/foo.md". Orchestrates three skills in order — extract-job, then tailor-resume, then tailor-coverletter — producing a parsed job file, a tailored one-page resume PDF, and a one-page cover letter PDF in the output/ folder.
---

# Process Job (Pipeline Orchestrator)

## Overview

This skill ties the whole workflow together. Given one job posting in `jobs/`, it runs three skills in sequence:

1. **extract-job** — parses the posting into `parsed/<name>.parsed.md` and archives the source to `completed/`.
2. **tailor-resume** — produces `output/<UserToken>_<Company>_Resume.tex` + `.pdf`, tailored to the parsed job, guaranteed one page.
3. **tailor-coverletter** — produces `output/<UserToken>_<Company>_CoverLetter.tex` + `.pdf`, following the rules file, as close to one page as possible.

This skill does **not** re-implement any of that logic. It invokes each skill in turn, checks that each produced its expected output before moving on, and reports a consolidated result at the end. The three skills remain the source of truth for their own behaviour.

The applicant's `UserToken` lives in `context/user.md`. Step 2 loads it into a shell variable; the orchestrator uses that value when checking for the downstream skills' outputs (they read the same file, so tokens always match).

All required directories (`jobs/`, `parsed/`, `completed/`, `output/`, `context/`, `templates/`) already exist. Do **not** create directories — never run `mkdir`. This applies to every sub-skill invoked as well. If an expected directory is missing, stop and report it rather than creating it.

## Inputs

- A single job posting path under `jobs/`, e.g. `jobs/acme-backend.md`. If the user names a company instead of a path, list `jobs/` and resolve it; confirm with the user if ambiguous.

## Expected end-state outputs

After a successful run, the following exist:

- `parsed/<name>.parsed.md`
- `completed/<name>.md` (source moved here; no longer in `jobs/`)
- `output/<UserToken>_<Company>_Resume.tex` and `.pdf` (1 page)
- `output/<UserToken>_<Company>_CoverLetter.tex` and `.pdf` (1 page)

## Orchestration rules

1. **Run in strict order.** extract-job → tailor-resume → tailor-coverletter. Each stage depends on the previous one's output.
2. **Gate each stage.** Do not start a stage until the previous stage's expected output exists on disk. If a stage's output is missing, stop and report — do not silently proceed.
3. **extract-job is a hard dependency.** If it fails, abort the whole pipeline — neither downstream skill can run without the parsed file.
4. **Resume and cover letter are sequential but the cover letter degrades gracefully.** tailor-coverletter prefers the resume as a cross-reference but can run without it. If tailor-resume fails, surface the failure clearly, then still attempt tailor-coverletter (it will warn about the missing resume and proceed from parsed/ + context/).
5. **Never fabricate to keep the pipeline moving.** If a sub-skill reports it cannot complete truthfully (e.g. resume won't fit one page after retries), report that honestly rather than papering over it.
6. **One posting per run.** If the user asks for several, run this pipeline once per posting, sequentially.

## Step 1: Resolve the job file

If the user gave a path, use it. Otherwise locate it:

```bash
ls -la jobs/
```

Confirm the target before proceeding if multiple files could match. Set the basename for tracking:

```bash
JOB="jobs/<name>.md"
[ -f "$JOB" ] || { echo "ERROR: $JOB not found in jobs/."; exit 1; }
BASENAME="$(basename "${JOB%.md}")"
echo "Processing: $JOB"
```

## Step 2: Stage 1 — Extract the job

Invoke the **extract-job** skill on `$JOB`. Follow that skill's full procedure (parse, write `parsed/<name>.parsed.md`, move source to `completed/`).

After it runs, gate on its output and load the user token:

```bash
PARSED="parsed/${BASENAME}.parsed.md"
if [ ! -s "$PARSED" ]; then
  echo "ABORT: extract-job did not produce $PARSED. Pipeline cannot continue."
  exit 1
fi
echo "Stage 1 OK → $PARSED"

# Load the UserToken from context/user.md (same source the downstream skills use)
[ -f "context/user.md" ] || { echo "ABORT: context/user.md missing. Copy context/user.example.md to context/user.md and fill it in."; exit 1; }
USER_TOKEN=$(grep -m1 'UserToken:' context/user.md | sed -E 's/.*UserToken:\*{0,2} *//' | tr -d ' ')
[ -n "$USER_TOKEN" ] || { echo "ABORT: could not read UserToken from context/user.md"; exit 1; }

# Derive the company token the downstream skills will use (must match their convention)
COMPANY_RAW=$(grep -m1 '^- \*\*Name:\*\*' "$PARSED" | sed 's/- \*\*Name:\*\* //')
COMPANY=$(echo "$COMPANY_RAW" | sed 's/[^A-Za-z0-9]//g')
echo "Company token: $COMPANY"
```

## Step 3: Stage 2 — Tailor the resume

Invoke the **tailor-resume** skill against `$PARSED`. Follow that skill's full procedure (select projects, rewrite work experience bullets, select 7 courses, generate LaTeX, compile, enforce one page, clean up).

Gate on its output:

```bash
RESUME_PDF="output/${USER_TOKEN}_${COMPANY}_Resume.pdf"
RESUME_TEX="output/${USER_TOKEN}_${COMPANY}_Resume.tex"

RESUME_OK=0
if [ -s "$RESUME_TEX" ] && [ -s "$RESUME_PDF" ]; then
  RESUME_OK=1
  echo "Stage 2 OK → $RESUME_PDF"
elif [ -s "$RESUME_TEX" ] && [ ! -s "$RESUME_PDF" ]; then
  echo "Stage 2 PARTIAL → $RESUME_TEX written but PDF missing (pdflatex issue?). Continuing to cover letter."
else
  echo "Stage 2 FAILED → resume not produced. Will still attempt cover letter (it degrades gracefully)."
fi
```

Note: a resume failure does **not** abort the pipeline (per orchestration rule 4) — record the state and continue. But do surface it prominently in the final report.

## Step 4: Stage 3 — Tailor the cover letter

Invoke the **tailor-coverletter** skill against `$PARSED`. Follow that skill's full procedure (read CoverLetterRules.md, read context/, optionally cross-reference the resume, generate LaTeX, compile, enforce one page, clean up).

Gate on its output:

```bash
COVER_PDF="output/${USER_TOKEN}_${COMPANY}_CoverLetter.pdf"
COVER_TEX="output/${USER_TOKEN}_${COMPANY}_CoverLetter.tex"

COVER_OK=0
if [ -s "$COVER_TEX" ] && [ -s "$COVER_PDF" ]; then
  COVER_OK=1
  echo "Stage 3 OK → $COVER_PDF"
elif [ -s "$COVER_TEX" ] && [ ! -s "$COVER_PDF" ]; then
  echo "Stage 3 PARTIAL → $COVER_TEX written but PDF missing (pdflatex issue?)."
else
  echo "Stage 3 FAILED → cover letter not produced."
fi
```

## Step 5: Print completion

Print a short completion line with the paths to the produced files:

```bash
echo "Process completed."
echo "Resume:       $RESUME_PDF"
echo "Cover letter: $COVER_PDF"
```

If a stage failed, replace that file's path with a brief "NOT PRODUCED — <stage> failed" note so the user knows which output is missing. Keep it to these few lines — no extended summary.

## Step 6: Quality check

Before declaring done, verify:

- [ ] The job source is no longer in `jobs/` and now exists in `completed/`
- [ ] `parsed/<name>.parsed.md` exists and is non-empty
- [ ] No directories were created
- [ ] The `UserToken` was read from `context/user.md`, and the same token was used for both the resume and the cover letter (so the two PDFs pair correctly)
- [ ] If the resume succeeded: `output/<UserToken>_<Company>_Resume.pdf` exists and is one page
- [ ] If the cover letter succeeded: `output/<UserToken>_<Company>_CoverLetter.pdf` exists and is one page
- [ ] Any stage failure is clearly reported, not hidden — the user knows exactly what to fix
- [ ] No sub-skill was asked to fabricate content to keep the pipeline moving
- [ ] Only one posting was processed this run (if the user asked for more, the pipeline ran once per posting)
