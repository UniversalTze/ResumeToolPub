---
name: tailor-resume
description: Tailors a LaTeX resume to a specific parsed job posting by updating only the Projects, Work Experience, and Education coursework sections. Use when user asks to "tailor my resume", "customize my resume for this job", "fit my resume to jobs/...", "make a resume for the Acme role", or similar — after the job has been parsed by extract-job. Reads parsed/<name>.parsed.md, context/projects/projects.md, context/workexp/experience.md, context/education/courses.md, and the resume template in templates/Resume/. Writes output/<UserToken>_<Company>_Resume.tex and the compiled PDF, guaranteeing exactly one page.
---

# Tailor Resume

## Overview

Generate a job-tailored LaTeX resume by modifying only three sections of the resume template in `templates/Resume/`: Projects, Work Experience, and the Relevant Coursework line under Education. All other sections (header, contact, Skills, Additional Experience, document preamble, formatting) must remain byte-identical to the template. The final output must compile to **exactly one page** — this is a hard constraint, not a preference.

The applicant's `UserToken` (used in output filenames) lives in `context/user.md`. Step 1 loads it into a shell variable; use that variable everywhere the filename is built. Never hardcode a name here.

All required directories (`parsed/`, `context/`, `templates/`, `output/`) already exist. Do **not** create directories — never run `mkdir`. The template file under `templates/` is read-only and must never be modified — always read from it and write the result to `output/`. If an expected directory is missing, stop and report it rather than creating it.

## Inputs

- `parsed/<name>.parsed.md` — produced by the `extract-job` skill. If the user names a job but the parsed file doesn't exist, stop and tell the user to run extract-job first.
- `context/user.md` — the applicant's identity; `UserToken` is read from here for the output filename. If missing, tell the user to copy `context/user.example.md` to `context/user.md` and fill it in.
- `context/projects/projects.md` — projects ordered strongest to weakest.
- `context/workexp/experience.md` — work history with the full pool of tasks/achievements per role.
- `context/education/courses.md` — list of courses studied.
- The resume template in `templates/Resume/` (a single `.tex` file) — the LaTeX template. Treat as read-only; never modify in place.

## Outputs

- `output/<UserToken>_<Company>_Resume.tex` — the tailored LaTeX source.
- `output/<UserToken>_<Company>_Resume.pdf` — the compiled PDF, exactly one page.

`<Company>` is taken from the parsed file's company name, with spaces removed and original casing preserved (e.g. `Acme Corp` → `AcmeCorp`).

## Hard rules

1. **Never invent facts.** Don't claim tech you haven't used, inflate metrics, or fabricate achievements. Tailoring means selecting and rephrasing real material, not embellishing.
2. **Modify only three sections:** Projects, Work Experience, Relevant Coursework. Everything else is preserved character-for-character — header, contact line, Skills section, Additional Experience, all spacing, all macros, all `\vspace` adjustments.
3. **One page after compile.** If pdflatex produces >1 page, tighten and recompile. Up to 3 retries before reporting failure to the user.
4. **Keep bullet counts.** Each work experience role keeps the same number of bullets as the template (currently 2-3 per role). Each project keeps the same number of bullets (currently 2-3 per project). Do not add or remove bullets to "make space" — tighten wording instead.
5. **Never modify the template.** The resume template in `templates/Resume/` is read-only.
6. **UserToken comes from `context/user.md`.** Never hardcode a name in this skill.

## Step 1: Verify inputs, load the user token, set paths

```bash
PARSED="parsed/<name>.parsed.md"
[ -f "$PARSED" ] || { echo "ERROR: $PARSED not found. Run extract-job first."; exit 1; }
[ -f "context/user.md" ] || { echo "ERROR: context/user.md missing. Copy context/user.example.md to context/user.md and fill it in."; exit 1; }
[ -f "context/projects/projects.md" ] || { echo "ERROR: context/projects/projects.md missing"; exit 1; }
[ -f "context/workexp/experience.md" ] || { echo "ERROR: context/workexp/experience.md missing"; exit 1; }
[ -f "context/education/courses.md" ] || { echo "ERROR: context/education/courses.md missing"; exit 1; }

# Locate the single resume template in templates/Resume/
TEMPLATE=$(find templates/Resume -maxdepth 1 -name '*.tex' | head -1)
[ -f "$TEMPLATE" ] || { echo "ERROR: no .tex template found in templates/Resume/"; exit 1; }

# Load the UserToken from context/user.md (parses the "UserToken:" line, strips label/formatting/spaces)
USER_TOKEN=$(grep -m1 'UserToken:' context/user.md | sed -E 's/.*UserToken:\*{0,2} *//' | tr -d ' ')
[ -n "$USER_TOKEN" ] || { echo "ERROR: could not read UserToken from context/user.md"; exit 1; }

# Derive a filesystem-safe company token from the parsed file
COMPANY_RAW=$(grep -m1 '^- \*\*Name:\*\*' "$PARSED" | sed 's/- \*\*Name:\*\* //')
COMPANY=$(echo "$COMPANY_RAW" | sed 's/[^A-Za-z0-9]//g')   # strip spaces/punct, keep casing

OUT_TEX="output/${USER_TOKEN}_${COMPANY}_Resume.tex"
OUT_PDF="output/${USER_TOKEN}_${COMPANY}_Resume.pdf"
echo "Target: $OUT_TEX"
```

## Step 2: Read inputs

```bash
cat "$PARSED"
cat context/projects/projects.md
cat context/workexp/experience.md
cat context/education/courses.md
cat "$TEMPLATE"
```

From the parsed file, identify:
- The full **tech stack** (languages, frameworks, infrastructure, other)
- The **Keywords** line
- **Must-have** requirements
- **Responsibilities**

These four feed every selection decision below.

## Step 3: Select 3 projects

From `context/projects/projects.md`:

1. Score each project by overlap with the job's tech stack + keywords + responsibilities. Direct tech name match is the strongest signal (`PostgreSQL` in job → project using `PostgreSQL` scores high).
2. Pick the 3 highest-scoring projects.
3. **Fallback rule:** if fewer than 3 projects have meaningful matches (i.e. at least one tech overlap), fill the remaining slots from the top of `projects.md` in order, because projects are ranked strongest-to-weakest in that file.
4. Preserve the order strongest → weakest in the output (best match first).

For each selected project, generate bullets matching the template structure:

```latex
\noindent
\textbf{<Name>} (\href{<url>}{<short-url>})\\
<tech stack line>
\begin{itemize}
    \item <bullet 1>
    \item <bullet 2>
    \item <bullet 3 if present>
\end{itemize}
```

Bullet rules:
- Same count as the template's existing project bullets (2-3).
- Lead each bullet with an action verb and a concrete outcome.
- Mirror the job's terminology where it's truthful (e.g. if the job says "RESTful APIs" and the project did exactly that, use "RESTful APIs", not "web endpoints").
- Pull tech names verbatim from `projects.md` — don't restyle.

## Step 4: Rewrite Work Experience bullets

For each role already in the template (don't add or remove roles — these are real jobs):

1. Find the same role in `context/workexp/experience.md`, which contains the full pool of tasks/achievements for that role.
2. From that pool, select the bullets that best match the job's must-haves, responsibilities, and tech stack.
3. Keep the **same bullet count** as the template currently shows for that role.
4. Light rewrites are allowed to mirror the job's language — but only if the underlying fact is true. Reordering and selection is the primary lever, not rewriting.
5. Preserve the exact role header structure (the `tabular*` block with company, dates, and role title — those don't change).

Each role's bullet block follows this template:

```latex
\begin{itemize}
    \item <bullet 1>
    \item <bullet 2>
    \item <bullet 3 if originally present>
\end{itemize}
```

## Step 5: Select 7 relevant coursework items

From `context/education/courses.md`:

1. Score each course by relevance to the job's tech stack and responsibilities.
2. Pick the 7 most relevant.
3. Replace only the coursework line — the rest of the Education block (university, degree, dates, GPA) stays identical.

The replaced line looks like:

```latex
    \item \textbf{Relevant Coursework:} <Course 1>, <Course 2>, <Course 3>, <Course 4>, <Course 5>, <Course 6>, <Course 7>.
```

## Step 6: Escape LaTeX special characters

Before writing into the .tex, escape these in any selected content (project names, bullets, course names):

| Char | Replace with |
|------|--------------|
| `&`  | `\&`         |
| `%`  | `\%`         |
| `_`  | `\_`         |
| `#`  | `\#`         |
| `$`  | `\$`         |
| `{`  | `\{`         |
| `}`  | `\}`         |
| `~`  | `\textasciitilde{}` |
| `^`  | `\textasciicircum{}` |
| `\`  | `\textbackslash{}` |

URLs inside `\href{...}{...}` are exempt — `\href` handles them. `C++` → `C\texttt{++}` is fine if it appears.

## Step 7: Assemble and write the tailored .tex

Read the template (`$TEMPLATE`) and produce `$OUT_TEX` with three substitutions:

1. **Projects section**: replace everything between `\resheading{Projects}` and the next `\resheading{...}` (Education) with the 3 rendered project blocks.
2. **Work Experience section**: replace everything between `\resheading{Work Experience}` and the next `\resheading{...}` (Projects) with the rewritten role blocks. Role headers stay identical; only the bullets inside each role change.
3. **Education coursework line**: replace only the single `\item \textbf{Relevant Coursework:} ...` line with the new 7-course version. The surrounding Education block is untouched.

Every `\vspace`, `\noindent`, `\\`, and blank line from the template's structure must be preserved.

```bash
# After generating the modified content in memory, write it:
cat > "$OUT_TEX" << 'EOF'
<full modified .tex content here>
EOF
echo "Wrote $OUT_TEX ($(wc -l < "$OUT_TEX") lines)"
```

## Step 8: Compile to PDF and verify one page

Run pdflatex from the project root directly — do not prepend `cd`, as chaining `cd` with output redirection requires shell approval. All intermediate files land in `output/` via the `-output-directory` flag.

```bash
pdflatex -interaction=nonstopmode -halt-on-error -output-directory=output "$OUT_TEX" > output/compile.log 2>&1
COMPILE_STATUS=$?

if [ $COMPILE_STATUS -ne 0 ]; then
  echo "ERROR: pdflatex failed. See output/compile.log for details."
  tail -30 output/compile.log
  exit 1
fi

# Determine page count
PAGES=$(pdfinfo "$OUT_PDF" 2>/dev/null | awk '/^Pages:/ {print $2}')
if [ -z "$PAGES" ]; then
  # Fallback: parse from .log if pdfinfo unavailable
  PAGES=$(grep -oE 'Output written.*\(([0-9]+) page' output/compile.log | grep -oE '[0-9]+' | head -1)
fi
echo "Page count: $PAGES"
```

If `$PAGES` ≠ 1, do **one** tightening pass and recompile:

1. Shorten the longest project bullets first (compress phrasing, keep facts).
2. Then shorten longest work experience bullets.
3. Never delete bullets — only tighten.
4. Never reduce coursework below 7 unless the user is asked and approves.

Allow up to 3 tightening passes. If still >1 page after 3 passes, stop and report:

> Could not fit on one page after 3 tightening passes. Current page count: N.
> Suggested fixes: reduce coursework from 7 to 6, or remove the lowest-priority bullet from project #3.

If `pdflatex` isn't installed, skip compilation but warn the user explicitly:

> pdflatex not found — wrote $OUT_TEX but could not generate PDF or verify one-page constraint.
> Install pdflatex (e.g. `sudo apt install texlive-latex-base` on Linux, MacTeX/BasicTeX on macOS) and run: `pdflatex -output-directory=output "$OUT_TEX"`

## Step 9: Clean up auxiliary files

Keep only the `.tex` and `.pdf` in `output/`; remove pdflatex's intermediate files:

```bash
rm -f "output/${USER_TOKEN}_${COMPANY}_Resume.aux" \
      "output/${USER_TOKEN}_${COMPANY}_Resume.log" \
      "output/${USER_TOKEN}_${COMPANY}_Resume.out" \
      output/compile.log
ls -la output/ | grep "${USER_TOKEN}_${COMPANY}"
```

## Step 10: Print confirmation

```
Tailored resume completed, ready for review.
```

## Step 11: Quality check

Before finishing, verify:

- [ ] `output/<UserToken>_<Company>_Resume.tex` exists and is non-empty
- [ ] `output/<UserToken>_<Company>_Resume.pdf` exists and is exactly 1 page (or user has been explicitly warned about a missing pdflatex)
- [ ] The `UserToken` was read from `context/user.md`, not hardcoded
- [ ] The resume template in `templates/Resume/` was not modified
- [ ] No directories were created
- [ ] Header, contact line, Skills section, and Additional Experience section are byte-identical to the template
- [ ] Exactly 3 projects in the Projects section
- [ ] Same number of work experience roles as the template (none added, none removed)
- [ ] Bullet counts per role and per project match the template
- [ ] Exactly 7 courses in the Relevant Coursework line
- [ ] No fabricated tech, metrics, or achievements — every claim traces back to context/projects, context/workexp, or context/education
- [ ] LaTeX special characters in content are escaped
- [ ] Auxiliary files (.aux, .log, .out) cleaned from output/
