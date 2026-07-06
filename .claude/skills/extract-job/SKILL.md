---
name: extract-job
description: Extracts structured information from a job posting markdown file in the jobs/ directory. Use when user asks to "extract the job in jobs/...", "parse this posting", "pull the details from jobs/foo.md", or as the first step when the user asks to "tailor my resume" or "write a cover letter" for a job file. Writes a structured summary to parsed/<name>.parsed.md and moves the original posting from jobs/ to completed/.
---

# Extract Job Posting

## Overview

Read a raw job posting from `jobs/<name>.md`, write a structured summary to `parsed/<name>.parsed.md`, then move the source posting from `jobs/` to `completed/`. The parsed output is the input for downstream tailor-resume and write-cover-letter skills, so the schema below must be followed exactly. Do not infer, embellish, or pull facts from outside the posting — missing fields are written as `_Not specified_`.

All required directories (`jobs/`, `parsed/`, `completed/`) already exist. Do **not** create directories — never run `mkdir`. If an expected directory is missing, stop and report it rather than creating it.

## Step 1: Locate the job file and prepare paths

If the user named a path, use it. If they named only a company or role, list `jobs/` to find a match:

```bash
ls -la jobs/
```

If multiple files could match, ask the user which one before continuing.

Set up the paths:

```bash
JOB="jobs/<name>.md"
BASENAME="$(basename "${JOB%.md}")"
PARSED="parsed/${BASENAME}.parsed.md"
COMPLETED="completed/${BASENAME}.md"
```

Skip parsing if `$PARSED` already exists and is newer than `$JOB`, unless the user explicitly asked to re-extract:

```bash
[ -f "$PARSED" ] && [ "$PARSED" -nt "$JOB" ] && echo "Already parsed, newer than source"
```

## Step 2: Read and analyze the posting

Read the full source file:

```bash
cat "$JOB"
```

Extract the following from the text alone:

- Company name, industry, size/stage, location or remote policy
- Role title, seniority level, team/reporting line
- Responsibilities (in the posting's order)
- Tech stack split into: languages, frameworks/libraries, infrastructure/tools, other
- Requirements split into **must-have** and **nice-to-have** (signal words: "required" / "must have" vs "preferred" / "bonus" / "plus")
- Company values, only if the posting explicitly states them
- Compensation, career growth, benefits — only what's stated
- 10-20 distinctive keywords for ATS matching (specific tech, methodologies, domain words — skip filler like "team player", "passionate")

Preserve exact wording for tech names (`PostgreSQL` not `Postgres database`, `Next.js` not `Next`). Paraphrase responsibilities into compact bullets.

## Step 3: Write the parsed file

Use this exact structure and these exact headings — downstream skills depend on the format:

```bash
cat > "$PARSED" << 'EOF'
# <Role Title> — <Company>

## Company
- **Name:** <company name>
- **Industry / domain:** <one line, or _Not specified_>
- **Size / stage:** <e.g. "Series B", "500+ employees", or _Not specified_>
- **Location / remote policy:** <e.g. "Remote (US)", "Hybrid, Sydney 2 days/week">

## Role
- **Title:** <exact title from posting>
- **Seniority:** <Junior / Mid / Senior / Staff / Lead / Principal>
- **Team / reporting line:** <if stated, else _Not specified_>
- **One-line summary:** <single sentence on what the role does>

## Responsibilities
- <bullet per responsibility, in the posting's order>

## Tech stack
- **Languages:** <list, or _Not specified_>
- **Frameworks / libraries:** <list>
- **Infrastructure / tools:** <cloud, CI, observability, etc.>
- **Other:** <databases, queues, etc.>

## Requirements
### Must-have
- <bullet per requirement>
### Nice-to-have
- <bullet per requirement>

## Company values
- <bullet per stated value; _Not specified_ if the posting doesn't list any>

## Benefits & growth
- **Compensation:** <salary range, or _Not specified_>
- **Career growth:** <mentorship, promotion path, learning budget, etc.>
- **Other benefits:** <equity, PTO, health, remote stipend, etc.>

## Keywords
<comma-separated line of 10-20 distinctive terms>

## Notes
<Anything notable: unusual application instructions, ambiguity in seniority, red flags. Omit this section if nothing to add.>
EOF
```

Confirm the parsed file was written:

```bash
echo "Parsed → $PARSED ($(wc -l < "$PARSED") lines)"
```

## Step 4: Move the source posting to completed/

Only do this after Step 3 has succeeded — never move the source before the parsed output exists on disk, or a failure mid-parse would lose the original.

```bash
if [ -s "$PARSED" ]; then
  cp "$JOB" "$COMPLETED"
  rm "$JOB"
  echo "Moved $JOB → $COMPLETED"
else
  echo "ERROR: parsed file missing or empty; leaving $JOB in place"
  exit 1
fi
```

## Step 5: Print a one-line summary

Reply to the user with both file locations and a brief gist:

> Parsed → `parsed/acme-backend.parsed.md`
> Archived → `completed/acme-backend.md`
> Acme Corp · Senior Backend Engineer · Go, Kubernetes, PostgreSQL; 5+ yrs distributed systems required; remote (US).

## Step 6: Quality check

Before finishing, verify:

- [ ] `parsed/<name>.parsed.md` exists and is non-empty
- [ ] `completed/<name>.md` exists and matches the original posting byte-for-byte
- [ ] `jobs/<name>.md` no longer exists
- [ ] No directories were created
- [ ] Every heading from the schema is present, in order, even if the value is `_Not specified_`
- [ ] No content was invented — every fact traces back to the source posting
- [ ] Tech names use exact casing (`PostgreSQL`, `Next.js`, `TypeScript`)
- [ ] Must-have and nice-to-have are correctly separated; when in doubt, default to must-have
- [ ] The Keywords line is comma-separated on a single line, not bulleted
- [ ] Only one posting was processed; if the user asked for multiple, the skill ran once per file
