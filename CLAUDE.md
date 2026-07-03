# CLAUDE.md

## What this project does

This project tailors a resume and cover letter to a specific job posting. For each job, the work is drawn from a fixed set of source material — projects, education, and work experience — and shaped to fit what the posting asks for. The output is a one-page tailored resume and a one-page cover letter, both as PDFs.

The pipeline is: read a job posting → extract its key details → tailor the resume → write a matching cover letter. Four skills handle this (`extract-job`, `tailor-resume`, `tailor-coverletter`, and `process-job`, which runs the other three in order).

## Folder structure

```
ResumeTool/
├── jobs/                            New job postings (.md) go here, one per posting.
├── parsed/                          Parsed job details land here (<name>.parsed.md).
├── completed/                       Archived original postings after processing.
├── output/                          Final deliverables: tailored resume + cover letter PDFs.
├── context/
│   ├── projects/                    Software projects, ordered strongest → weakest.
│   ├── education/                   Courses studied.
│   └── workexp/                     Work experience: full pool of tasks/achievements per role.
├── templates/
│   ├── Resume/                      Resume template (LaTeX). READ-ONLY.
│   └── CoverLetter/                 Cover letter rule book (.md). READ-ONLY.
└── .claude/
    └── skills/                      The four pipeline skills.
```

All directories above already exist. **Never create directories — do not run `mkdir`.** If an expected directory is missing, stop and report it rather than creating it.

### Where to find information

- **Projects** → `context/projects/`. Ordered most impressive first; when fewer projects match a job than the resume has slots, fill remaining slots from the top.
- **Work experience** → `context/workexp/`. Holds the full pool of bullets per role; the resume selects the most job-relevant ones rather than using all of them.
- **Education / coursework** → `context/education/`. The resume selects the 7 most relevant courses per job.
- **Resume template** → `templates/Resume/`.
- **Cover letter rules** → `templates/CoverLetter/`.

## How the pipeline moves files

1. A new posting starts in `jobs/`.
2. After extraction, its parsed details are written to `parsed/<name>.parsed.md`.
3. The original posting is then **removed from `jobs/` and archived in `completed/`** — `jobs/` only ever holds unprocessed postings.
4. The tailored resume and cover letter PDFs are written to `output/`.

So after a successful run: the posting lives in `completed/`, its parsed form in `parsed/`, and the two finished PDFs in `output/`.

## Output requirements

- The tailored resume and the cover letter must **each fit on exactly one page** — as close to a full page as possible without ever spilling onto a second page. The resume template is already perfectly one page; tailoring must preserve that.
- Output file naming:
  - Resume → `output/TzeKheng_<Company>_Resume.pdf`
  - Cover letter → `output/TzeKheng_<Company>_CoverLetter.pdf`
  - `<Company>` is the posting's company name with spaces and punctuation stripped, original casing kept. The resume and cover letter must use the identical company token so the pair matches.

## Hard rules

- **Never modify the template files.** Everything under `templates/` — the resume template in `templates/Resume/` and the cover letter rule book in `templates/CoverLetter/` — is read-only and must not be edited, reformatted, or overwritten in any way. Always read from them; never write back.
- **Never create directories.** All folders already exist; do not run `mkdir`.
- **Never fabricate.** Every concrete claim in a resume or cover letter (tech, metrics, projects, roles, courses) must trace back to `context/projects/`, `context/workexp/`, or `context/education/`. Tailoring means selecting and reframing real material — not inventing experience, inflating numbers, or claiming unused tech.
- **One posting per run.** Process a single job at a time. If asked for several, run the pipeline once per posting.
- **The cover letter follows the rule book exactly.** The `.md` in `templates/CoverLetter/` is the source of truth for the cover letter's structure, tone, and formatting; read it fresh each time rather than relying on memory.
- **Source material is read-only too.** The skills read from `context/` but never modify it.

## Australian English

All written output uses Australian English spelling (organised, behaviour, optimising, analyse) and date format (e.g. `7 May 2026`).
