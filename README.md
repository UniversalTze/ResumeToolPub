# ResumeTool

An agentic resume-and-cover-letter tailoring pipeline built on **Claude Code**. Drop a job posting into a folder, and the tool reads it, tailors your resume to match, and drafts a one-page cover letter — all from your own experience, projects, and education. No web service, no API glue: everything runs locally through Claude Code's skills, project memory, and permission system.

---

## What it does

Given a single job posting (a markdown file in `jobs/`), the pipeline runs four steps in order:

1. **Extract** — Scans the posting and writes a structured summary to `parsed/<name>.parsed.md`: company, role, responsibilities, tech stack, must-have vs nice-to-have requirements, values, and keywords. The original posting is archived to `completed/`.
2. **Tailor the resume** — Reads the parsed job plus your source material in `context/` and fills a LaTeX resume template, updating only the Projects, Work Experience, and Coursework sections. It selects the most relevant projects, chooses the strongest bullets per role, and picks the most relevant courses — then compiles to PDF and **guarantees the result fits on exactly one page**.
3. **Tailor the cover letter** — Drafts a cover letter driven by a rule book (`templates/CoverLetter/`), matching the tone and structure to the role, and compiles it to PDF **as close to one full page as possible without ever spilling to a second**.
4. **Orchestrate** — A single command runs all three end to end.

Everything the tool writes is grounded in your real material — it selects and reframes, it never fabricates experience.

---

## How it uses Claude Code

This project is a worked example of building a real workflow entirely out of Claude Code's building blocks:

- **Skills** (`.claude/skills/<name>/SKILL.md`) — Each step is a self-contained skill: `extract-job`, `tailor-resume`, `tailor-coverletter`, and `process-job` (the orchestrator that chains the other three). Claude Code auto-discovers them by their `description` and runs them when your request matches. The procedural logic lives in the skill files, so every run is consistent and the only thing that varies is the job posting.
- **Project memory** (`CLAUDE.md`) — A single file at the project root gives every session shared context: the folder layout, how files move through the pipeline, the one-page rule, and the hard constraints (never modify templates, never fabricate, never create directories).
- **Permissions** (`.claude/settings.json`) — The agent's capabilities are scoped down to exactly what the pipeline needs. It can read project files, run the specific commands the skills use (`pdflatex`, `pdfinfo`, `cp`, and friends), and write only to `output/`. It is explicitly denied write access to your templates and source data, and web access is switched off entirely — the whole workflow is local, so it never needs the network. The same file also pins the model to `opusplan` (Opus for the planning and selection reasoning, a faster model for the mechanical LaTeX assembly).

The result is an agent that can do real, multi-step work unattended while staying inside boundaries you've set.

---

## Project structure

```
ResumeTool/
├── jobs/                        Drop new job postings (.md) here.
├── parsed/                      Structured summaries land here.
├── completed/                   Original postings are archived here after processing.
├── output/                      Final deliverables: tailored resume + cover letter PDFs.
├── context/
│   ├── projects/                Your software projects, strongest first.
│   ├── education/               Courses studied.
│   └── workexp/                 Work experience: the full pool of bullets per role.
├── templates/
│   ├── Resume/                  LaTeX resume template. (Read-only.)
│   └── CoverLetter/             Cover letter rule book. (Read-only.)
└── .claude/
    ├── skills/                  The four pipeline skills.
    └── settings.json            Model + permission configuration.
```

Example data files (`*.example.md`, `template.example.tex`) show the expected format. Copy them to their real names and fill them with your own details — your real files stay local and out of version control.

---

## Requirements

- **[Claude Code](https://docs.claude.com/en/docs/claude-code)** — the agent that runs the pipeline. Requires a paid Claude plan (Pro, Max, Team, Enterprise, or API/Console).
- **A LaTeX distribution providing `pdflatex` and `pdfinfo`** — the resume and cover letter are LaTeX documents that get compiled to PDF. This is the one real dependency.

The core pipeline needs **no Python packages** — the skills use only standard command-line tools plus LaTeX, so there is nothing to install with `pip` and no virtual environment to set up.

---

## Setup

### 1. Install Claude Code

Native installer (no Node.js required):

```bash
# macOS / Linux / WSL
curl -fsSL https://claude.ai/install.sh | bash

# Windows (PowerShell)
irm https://claude.ai/install.ps1 | iex
```

Or via npm (needs Node.js 18+):

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:

```bash
claude --version
```

### 2. Install LaTeX (`pdflatex`)

**Linux (Debian/Ubuntu):**
```bash
sudo apt update
sudo apt install texlive-latex-base texlive-fonts-recommended texlive-latex-extra
```

**macOS:** install [MacTeX](https://www.tug.org/mactex/) (full) or BasicTeX (smaller):
```bash
brew install --cask mactex
# or the lighter option:
brew install --cask basictex
```

**Windows:** install [MiKTeX](https://miktex.org/download).

Verify both tools are available:
```bash
pdflatex --version
pdfinfo -v
```

### 3. Git clone the directory
Clone the repository to get the full setup — Claude Code skills, permissions, and project structure.
```bash
git clone https://github.com/UniversalTze/ResumeToolPublic.git
```

### 4. Fill in your details

Copy each example file to its real name and replace the placeholder content with your own:

```bash
cp context/projects/projects.example.md   context/projects/projects.md
cp context/workexp/experience.example.md  context/workexp/experience.md
cp context/education/courses.example.md    context/education/courses.md
cp templates/Resume/template.example.tex   templates/Resume/YourResume.tex
```

Then edit them:
- **`context/workexp/experience.md`** — the full pool of tasks/achievements for each role (the tool selects the most job-relevant ones per application).
- **`context/projects/projects.md`** — your projects, ordered strongest to weakest.
- **`context/education/courses.md`** — the courses you've taken.
- **`templates/Resume/YourResume.tex`** — your resume, with your real name and contact details in the header.

> If you rename the resume template or change any resume titles, update the path referenced / titles in the `tailor-resume` skill to match.

Your real files are ignored by git (see `.gitignore`), so they never leave your machine.

---

## Adding a job posting

**Format: use `.md`.** The pipeline reads postings as plain text, so a markdown file works directly with no conversion. Markdown is preferred over a plain `.txt` because job postings have structure — *About the Role*, *Responsibilities*, *Requirements*, *Bonus Points* — and the `extract-job` skill uses those section headings to separate must-have from nice-to-have requirements. Adding a few `##` headings gives the parser clean seams to work with.

Avoid PDFs: they aren't plain text, so the tool can't read them without an extra extraction step, and PDF-to-text conversion tends to scramble columns and bullet points. If a posting only exists as a PDF, open it, copy the text out, and save it as `.md` yourself — do the conversion once at the input stage, and strip out page numbers and legal footers while you're there.

Most postings live on a web page (LinkedIn, Seek, a company careers site). The workflow is simply: copy the posting text, paste it into a new `.md` in `jobs/`, add `##` headings if it doesn't already have clear sections, and you're done.

**Naming: `company-role.md`.** Name each file after the company and the role, with the role shortened where sensible. This keeps `jobs/` readable and flows through into the output filenames. Examples:

```
jobs/accenture-jr-dev.md
jobs/atlassian-backend.md
jobs/canva-fullstack.md
jobs/nab-data-eng.md
```

Use lowercase with hyphens, keep it short, and make the company clear — it's the first thing you'll scan when the folder fills up.

---

## Usage

From the project root:

```bash
cd ResumeTool/
claude
```

Then, in the session:

```
> process the job in jobs/accenture-jr-dev.md
```

The pipeline parses the posting, tailors your resume, writes a matching cover letter, and leaves both PDFs in `output/`:

- `output/<You>_<Company>_Resume.pdf`
- `output/<You>_<Company>_CoverLetter.pdf`

You can also run the steps individually — e.g. *"tailor my resume for jobs/accenture-jr-dev.md"* or *"write a cover letter for the Accenture role"*.

On first launch, run `/status` to confirm the model loaded, and `/permissions` to see the active allow/deny rules. Claude Code may ask you to approve a command the first time it runs; choose the "don't ask again" option to make it persistent for future sessions.

---

## Notes

- The tool never modifies your templates or source data — it only reads them and writes finished documents to `output/`.
- Both documents are constrained to a single page; if content can't fit after tightening, the tool reports it rather than silently producing a second page.
- All written output uses Australian English spelling and date formatting by default (configurable in `CLAUDE.md` and the cover letter rule book).
- It will always ensure your resume and Cover Letter are no longer than 1 page. 
