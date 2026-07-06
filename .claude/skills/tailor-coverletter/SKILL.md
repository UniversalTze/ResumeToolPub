---
name: tailor-coverletter
description: Generates a tailored cover letter as a PDF for a specific parsed job posting. Use when user asks to "write a cover letter", "draft the cover letter for jobs/...", "generate a cover letter for the Acme role", or as the third step in the resume-tailoring pipeline after extract-job and tailor-resume. Reads parsed/<name>.parsed.md, the tailored resume at output/<UserToken>_<Company>_Resume.tex, everything in context/, and the rules file at templates/CoverLetter/CoverLetterRules.md. Writes output/<UserToken>_<Company>_CoverLetter.tex and the compiled PDF, as close to one page as possible without exceeding it.
---

# Tailor Cover Letter

## Overview

Generate a job-tailored cover letter as a LaTeX file and compile it to PDF. The letter's structure, tone, paragraph rules, lexicon, and formatting are defined in `templates/CoverLetter/CoverLetterRules.md` — **that file is the source of truth for what the letter looks like and how it reads**. This skill handles the mechanics: reading inputs, applying the rules, generating LaTeX, compiling, verifying page count, and cleaning up.

The applicant's identity and contact details — `UserToken`, full name, location, phone, email — live in `context/user.md`. Step 1 loads them; use them in the header block and output filenames. Never hardcode any personal details in this skill.

All required directories (`parsed/`, `context/`, `templates/`, `output/`) already exist. Do **not** create directories — never run `mkdir`. The template files under `templates/` are read-only and must never be modified. If an expected directory is missing, stop and report it rather than creating it.

## Inputs

- `parsed/<name>.parsed.md` — output of `extract-job`. If missing, stop and tell the user to run extract-job first.
- `context/user.md` — the applicant's identity and contact details (name, location, phone, email) for the header, and `UserToken` for filenames. If missing, tell the user to copy `context/user.example.md` to `context/user.md` and fill it in.
- `output/<UserToken>_<Company>_Resume.tex` — the tailored resume from `tailor-resume`. Used to ensure the cover letter reinforces (rather than duplicates) what the resume already says. If missing, warn the user but proceed using only context/ — the cover letter is still generatable, just without the cross-check.
- `context/projects/projects.md` — full project pool.
- `context/workexp/experience.md` — full work experience pool (full bullet pool per role).
- `context/education/courses.md` — courses studied.
- Any other `.md` files in `context/` — read all of them; they may contain values, motivations, or background the rules file references.
- `templates/CoverLetter/CoverLetterRules.md` — the authoritative style and structure guide. Read and follow exactly. Read-only.

## Outputs

- `output/<UserToken>_<Company>_CoverLetter.tex` — the LaTeX source.
- `output/<UserToken>_<Company>_CoverLetter.pdf` — the compiled PDF, as close to one page as possible without exceeding it.

`<Company>` is taken from the parsed file's company name, with spaces and punctuation stripped, original casing preserved (e.g. `Acme Corp` → `AcmeCorp`). Must match the casing used by `tailor-resume` so the two files pair cleanly.

## Hard rules

1. **CoverLetterRules.md is authoritative.** If anything in this skill conflicts with the rules file, the rules file wins. Re-read it every run — do not rely on prior session memory.
2. **Never fabricate.** Every concrete claim (tech, metrics, projects, role facts) must trace back to `context/`. Tailoring is selection and reframing, not invention.
3. **One page maximum.** The compiled PDF must be exactly 1 page. If it overflows, tighten and recompile (see Step 8).
4. **Fill the page, don't waste it.** Target the upper end of the word range from the rules file (~500 words). Letters under ~400 words on a full-page-target layout will look thin.
5. **Do not duplicate the resume verbatim.** The cover letter can reference projects and roles from the resume, but it tells a connected story — it doesn't restate bullets word-for-word.
6. **Never modify template files.** Everything under `templates/` is read-only — read from it, never write to it.
7. **Personal details come from `context/user.md`.** Name, location, phone, and email are read from there, never hardcoded in this skill.

## Step 1: Verify inputs, load user details, set paths

```bash
PARSED="parsed/<name>.parsed.md"
[ -f "$PARSED" ] || { echo "ERROR: $PARSED not found. Run extract-job first."; exit 1; }
[ -f "context/user.md" ] || { echo "ERROR: context/user.md missing. Copy context/user.example.md to context/user.md and fill it in."; exit 1; }
[ -f "templates/CoverLetter/CoverLetterRules.md" ] || { echo "ERROR: CoverLetterRules.md missing"; exit 1; }
[ -d "context" ] || { echo "ERROR: context/ directory missing"; exit 1; }

# Load applicant identity + contact details from context/user.md.
# Each line is parsed by its label; labels must match user.example.md exactly.
USER_TOKEN=$(grep -m1 'UserToken:' context/user.md | sed -E 's/.*UserToken:\*{0,2} *//'  | tr -d ' ')
FULL_NAME=$(grep -m1 'FullName:'  context/user.md | sed -E 's/.*FullName:\*{0,2} *//'   | sed 's/ *$//')
LOCATION=$(grep -m1 'Location:'   context/user.md | sed -E 's/.*Location:\*{0,2} *//'   | sed 's/ *$//')
PHONE=$(grep -m1 'Phone:'         context/user.md | sed -E 's/.*Phone:\*{0,2} *//'      | sed 's/ *$//')
EMAIL=$(grep -m1 'Email:'         context/user.md | sed -E 's/.*Email:\*{0,2} *//'      | sed 's/ *$//')
[ -n "$USER_TOKEN" ] && [ -n "$FULL_NAME" ] || { echo "ERROR: could not read UserToken/FullName from context/user.md"; exit 1; }

# Derive the company token — must match tailor-resume's convention exactly
COMPANY_RAW=$(grep -m1 '^- \*\*Name:\*\*' "$PARSED" | sed 's/- \*\*Name:\*\* //')
COMPANY=$(echo "$COMPANY_RAW" | sed 's/[^A-Za-z0-9]//g')

OUT_TEX="output/${USER_TOKEN}_${COMPANY}_CoverLetter.tex"
OUT_PDF="output/${USER_TOKEN}_${COMPANY}_CoverLetter.pdf"
RESUME_TEX="output/${USER_TOKEN}_${COMPANY}_Resume.tex"

# Resume is a nice-to-have, not required — warn if missing
if [ ! -f "$RESUME_TEX" ]; then
  echo "WARNING: $RESUME_TEX not found. Generating cover letter without resume cross-reference."
fi

echo "Target: $OUT_TEX  (applicant: $FULL_NAME)"
```

## Step 2: Read all inputs

```bash
cat context/user.md
cat templates/CoverLetter/CoverLetterRules.md
cat "$PARSED"
[ -f "$RESUME_TEX" ] && cat "$RESUME_TEX"
ls context/
find context -name "*.md" -exec cat {} \;
```

**The rules file must be read in full every run** — paragraph structure, lexicon, conditional paragraph rules, and page-styling requirements all live there. The applicant's name and contact details for the header come from `context/user.md` (loaded in Step 1).

From the parsed file, extract the four signals that drive tailoring decisions:
- Role title (exact wording for the Re: line and header)
- Company name (exact wording for the Re: line)
- Tech stack (drives experience selection)
- Must-have requirements (drives gap-paragraph decision)
- Company values / culture signals (drives closing-paragraph emphasis)

## Step 3: Apply the tailoring decisions from the rules file

Following §5 of `CoverLetterRules.md`, decide:

1. **Lead-paragraph framing**: dominant-requirement vs broad-relevance — based on whether the parsed job has one clearly dominant skill requirement or several balanced ones.
2. **Lead anchor**: work experience or projects — whichever has the strongest direct overlap with must-haves. Tie-break in favour of work experience.
3. **Gap paragraph**: include only if a real, material gap exists. Never invent one.
4. **Closing emphasis**: trajectory / cultural / domain — based on what the parsed job emphasises.
5. **Logistics sentence in opener**: include only if geography, security clearance, visa status, or availability is non-obvious and relevant to this role.

Record these decisions before writing — they shape every paragraph.

## Step 4: Draft the letter content

Write each paragraph in plain prose following the structure in §2 of the rules file. Target word counts from §3:

- Opening: 70–100 words
- Lead body: 120–160 words
- Supporting body: 100–130 words each
- Gap paragraph (if included): 70–100 words
- Closing: 60–90 words

**Total target: 480–510 words.** This is the upper end of the comfortable range and is what fills a one-page letter cleanly.

Apply the lexicon rules from §4: use the approved phrases when they fit naturally, avoid the banned ones, follow Australian English spelling, and use en dashes for ranges and the Re: line.

After drafting, run the grammar pass from §6 of the rules file before generating LaTeX.

## Step 5: Generate LaTeX source

Use the structure below. Fill the header block with the applicant details loaded in Step 1 — the placeholders `<FULL_NAME>`, `<LOCATION>`, `<PHONE>`, `<EMAIL>` are taken from `context/user.md`, never hardcoded. The only per-application variables are the date, role title, company name, body paragraphs, and sign-off choice.

```latex
\documentclass[a4paper,11pt]{article}
\usepackage[margin=2.0cm]{geometry}
\usepackage[hidelinks]{hyperref}
\usepackage{array}
\usepackage{tabularx}
\usepackage{parskip}
\pagestyle{empty}

\begin{document}

%==================== HEADER BLOCK ====================%
\noindent
\begin{tabularx}{\textwidth}{@{} X !{\vrule} X @{}}
    \textbf{<FULL_NAME>} & <LOCATION> \\
    <ROLE TITLE>         & <PHONE> \\
                         & <EMAIL> \\
\end{tabularx}

\vspace{0.4ex}
\noindent <DATE>

\vspace{0.6ex}
\noindent\rule{\textwidth}{0.4pt}
\vspace{0.4ex}

%==================== SALUTATION ====================%
\noindent Dear Hiring Manager,

\vspace{0.6ex}
\noindent Re: <ROLE TITLE> -- <COMPANY NAME>

\vspace{0.8ex}

%==================== BODY ====================%
<OPENING PARAGRAPH>

<LEAD BODY PARAGRAPH>

<SUPPORTING BODY PARAGRAPH>

<GAP PARAGRAPH IF INCLUDED>

<CLOSING PARAGRAPH>

\vspace{1.2ex}
\noindent <SIGN-OFF>,

\vspace{2.0ex}
\noindent <FULL_NAME>

\end{document}
```

Notes on the LaTeX:

- The `tabularx` with `!{\vrule}` produces the two-column header with vertical divider between left (name + role) and right (location, phone, email).
- The date sits below the table, then the horizontal rule, matching the layout from the rules file §2.1.
- The Re: line uses `--` which LaTeX renders as an en dash (–).
- `parskip` gives paragraph spacing without indents — clean modern letter look.
- Margins are slightly tighter than default (2.0 cm) to give room for ~500 words on one page without crowding.

## Step 6: Escape LaTeX special characters

In all generated content (paragraphs, company name, role title, and the applicant details from `context/user.md`), escape:

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

Particularly watch the company name — names like `AT&T`, `P&G`, or anything with `&` will break the LaTeX silently without escaping.

## Step 7: Write the .tex file

```bash
cat > "$OUT_TEX" << 'EOF'
<full LaTeX content from Step 5, with all placeholders filled and characters escaped>
EOF
echo "Wrote $OUT_TEX ($(wc -l < "$OUT_TEX") lines)"
```

## Step 8: Compile to PDF and verify page count

```bash
pdflatex -interaction=nonstopmode -halt-on-error -output-directory=output "$OUT_TEX" > output/cl_compile.log 2>&1
COMPILE_STATUS=$?

if [ $COMPILE_STATUS -ne 0 ]; then
  echo "ERROR: pdflatex failed. Tail of log:"
  tail -30 output/cl_compile.log
  exit 1
fi

PAGES=$(pdfinfo "$OUT_PDF" 2>/dev/null | awk '/^Pages:/ {print $2}')
[ -z "$PAGES" ] && PAGES=$(grep -oE 'Output written.*\(([0-9]+) page' output/cl_compile.log | grep -oE '[0-9]+' | head -1)
echo "Page count: $PAGES"
```

**If pages > 1**: tighten and recompile. Tightening priority order:

1. Trim the longest supporting paragraph by 10–15 words (compress phrasing, keep facts).
2. Trim the lead body paragraph by 10 words.
3. Drop a soft sentence from the closing or opening (the role-bridge sentence is non-negotiable; trim around it).
4. As a last resort: remove the gap paragraph entirely if it's present and the letter is close to but over one page.

Allow up to 3 tightening passes. If still > 1 page, stop and report:

> Could not fit on one page after 3 tightening passes. Current page count: N words: M.
> Recommend manual review of $OUT_TEX.

**If pages = 1 but word count is well under 400**: the letter is too sparse for a full-page-target layout. Do one expansion pass — add a supporting sentence to the lead or supporting paragraph (always truthful, always specific) to fill the page more cleanly. Recompile and verify still 1 page.

**If pdflatex isn't installed**: write the .tex anyway and warn:

> pdflatex not found. Wrote $OUT_TEX but could not generate PDF or verify page count.
> Install texlive-latex-base (Linux) or MacTeX (Mac), then run: pdflatex -output-directory=output "$OUT_TEX"

## Step 9: Cross-check against the resume

If `$RESUME_TEX` exists, do a final consistency pass:

- Cover letter must not contradict the resume (different metrics, different tech names, different role titles).
- Cover letter should reference at least one project or experience that appears in the resume — the two documents tell one consistent story.
- Cover letter should NOT verbatim copy resume bullets — the letter reframes and connects, the resume lists.

If a contradiction is found, fix the cover letter (the resume was generated first and is the canonical version for this application).

## Step 10: Clean up auxiliary files

```bash
rm -f "output/${USER_TOKEN}_${COMPANY}_CoverLetter.aux" \
      "output/${USER_TOKEN}_${COMPANY}_CoverLetter.log" \
      "output/${USER_TOKEN}_${COMPANY}_CoverLetter.out" \
      output/cl_compile.log
ls -la output/ | grep "${USER_TOKEN}_${COMPANY}_CoverLetter"
```

## Step 11: Print confirmation

```
Cover letter ready.
```

## Step 12: Quality check

Before finishing, verify:

- [ ] `output/<UserToken>_<Company>_CoverLetter.tex` exists and is non-empty
- [ ] `output/<UserToken>_<Company>_CoverLetter.pdf` exists and is exactly 1 page (or user has been warned about missing pdflatex)
- [ ] No directories were created and no template files were modified
- [ ] Applicant name and contact details were read from `context/user.md`, not hardcoded
- [ ] `templates/CoverLetter/CoverLetterRules.md` was read and followed — not skipped or summarized from memory
- [ ] Header block uses the two-column layout with vertical divider (left: name + role; right: location, phone, email)
- [ ] Date sits below the two-column block, above the horizontal rule
- [ ] Horizontal rule renders below the date, above the salutation
- [ ] Re: line uses an en dash (`--` in source) between role and company
- [ ] Word count is in the 480–510 range (or close, with documented reason for variance)
- [ ] No banned words from the rules file lexicon present ("passionate", "leverage" as verb, "synergy", "rockstar", "team player", etc.)
- [ ] Australian English spelling throughout (organised, behaviour, optimising, analyse)
- [ ] Every concrete claim traces back to context/projects, context/workexp, or context/education
- [ ] Lead paragraph leads with the strongest direct match — work OR projects — per §5 of the rules
- [ ] Gap paragraph included only if a real, material gap exists
- [ ] Closing requests the conversation and matches the chosen emphasis
- [ ] Sign-off is one of "Yours sincerely," or "Kind regards," with the full name below
- [ ] Cover letter does not contradict the resume (if resume exists)
- [ ] Cover letter does not verbatim duplicate resume bullets
- [ ] LaTeX special characters escaped in all generated content (especially company name)
- [ ] Auxiliary files (`.aux`, `.log`, `.out`) cleaned from output/
