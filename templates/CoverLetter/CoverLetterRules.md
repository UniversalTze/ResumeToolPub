# Cover Letter Rules

These rules describe how cover letters are structured and written. The `tailor-coverletter` skill reads this file and follows it when generating a tailored letter from a parsed job posting.

The rules capture an established voice, not a generic template. Follow them faithfully; only deviate when the situation genuinely demands it (and flag the deviation if so).

All personal specifics — the applicant's name and contact details, degree, employers, and past experience — come from the applicant's own files, never from this rules file:

- Identity and contact details → `context/user.md`
- Work history → `context/workexp/experience.md`
- Education → `context/education/courses.md`
- Projects → `context/projects/projects.md`

This file governs *how* the letter reads and is laid out, not *who* the applicant is.

---

## 1. Hard constraints

These are non-negotiable.

- **Length: as close to one full page as possible, but never over.** Aim for 480–510 words. Use the bottom of the page; do not leave large empty space, but do not spill onto a second page under any circumstances.
- **First person, understated tone.** No hype words: avoid "passionate", "thrilled", "rockstar", "ninja", "world-class", "leverage" (as a verb), "synergy".
- **Specificity over generality.** Every technical or experiential claim must include at least one of: a tech name, a number/metric, or a concrete artefact (e.g. "Terraform-provisioned cloud infrastructure", "20–50% runtime improvements", "automated test suites").
- **Truth only.** Never claim tech, projects, metrics, or experience not present in the source files (`context/projects/projects.md`, `context/workexp/experience.md`, `context/education/courses.md`). Tailoring means selecting and reframing real material, not embellishing.
- **Australian English.** Spellings: organised, optimising, behaviour, analyse, recognised, programme (for non-software contexts), centre. Date format: `7 May 2026`, not `May 7, 2026`.
- **Explicit bridge between past and role.** Every body paragraph must connect what the applicant has done to what the role needs — don't leave the reader to infer the link.

---

## 2. Document structure

A complete cover letter has these blocks, in order.

The applicant's name and contact details in the header come from `context/user.md`. Do not hardcode them here.

### 2.1 Header block

The header is a **two-column layout** with a vertical divider running between the columns. Both columns sit above the same baseline; the layout spans the full text width.

**Left column** (left-aligned):
```
<Full name — from context/user.md>
<Role title applied for, exactly as in the posting>
```

**Vertical divider** — a thin vertical rule running between the two columns, aligned with the height of the column contents.

**Right column** (right-aligned, or left-aligned within its column — match the source template):
```
<Location — from context/user.md>
<Phone — from context/user.md>
<Email — from context/user.md>
```

**Below both columns**, on its own line: the date in `D MMMM YYYY` format (e.g. `7 May 2026`).

**A horizontal rule sits directly under the entire header block**, spanning the full text width, separating the header from the salutation.

So the visual order top-to-bottom is:
1. The two-column block (name+role on left, contact details on right, with a vertical divider between)
2. The date line
3. A horizontal rule across the full text width
4. The salutation begins

LaTeX implementation note for the skill: this is most cleanly built with a `tabularx` or `minipage` pair for the two columns, with a `|` column separator (or `\vrule` between minipages) for the vertical divider, and `\noindent\rule{\textwidth}{0.4pt}` for the horizontal rule beneath.

### 2.2 Salutation + Re: line

```
Dear Hiring Manager,

Re: <Role Title> – <Company>
```

Use an en dash (–) between role and company in the Re: line, not a hyphen (-). If the role has a parent team or programme name, include it: `Re: <Team or Programme> – <Role Title>, <Company>`.

### 2.3 Opening paragraph (required)

Three jobs in one paragraph:

1. State the role being applied for.
2. Name something specific about *this* company or team that draws the applicant to it — pulled from the parsed file's company values, mission, or role description. Avoid generic praise.
3. Establish credentials in a single sentence, drawn from `context/education/courses.md` and `context/workexp/experience.md` — the applicant's degree plus the most relevant professional experience for this role.

Optional add-on: if logistics matter and aren't obvious from the resume (different city, relocation willingness, security clearance held, visa status), add one sentence at the end of this paragraph. Example pattern: "I am <city>-based and comfortable commuting to or relocating to <location> for the role."

### 2.4 Lead body paragraph (required)

This paragraph carries the strongest match to the role's core requirement. **Lead with whichever experience is the strongest fit — work OR projects, whichever maps best.**

Two framing options:

- **Dominant-requirement framing**: when the role has one clearly dominant requirement (e.g. a SQL-heavy data role), open with a direct claim about that skill. Example pattern: *"Strong SQL is the centre of my experience."*
- **Broad-relevance framing**: when the role has several roughly equal requirements, use the standard opener: *"My most directly relevant experience is..."*

This paragraph should be the longest and densest in the letter. Pack it with specifics: tech names, metrics, concrete outcomes, and an explicit bridge to the role — all drawn from the source files.

### 2.5 Supporting body paragraph(s) (required, 1–2)

Adjacent experience that rounds out the picture. Could cover:

- Projects (if the lead was work experience) or work experience (if the lead was projects)
- Daily-use tools and habits
- Relevant coursework concepts where they translate to the role
- Domain context from the applicant's degree, when applicable

Each supporting paragraph ends with an explicit role-bridge: "the same fundamentals the role lists", "directly relevant to the [X] responsibilities the role describes", "habits I would bring to [Company]'s [Y] responsibilities".

### 2.6 Honest gap paragraph (conditional)

**Include only when there is a real, material gap between the applicant's experience and the role's requirements.** For example, a role asking for a specific cloud platform or BI tool the applicant hasn't used directly. Do not invent a gap to seem humble; if the applicant's experience genuinely covers the role, skip this paragraph entirely.

When present, the paragraph does three things:

1. Name the gap directly: *"One honest gap to acknowledge: I have not yet..."* or *"On [X], my hands-on experience is with [Y] rather than [Z]..."*
2. Reframe with transferable concepts or adjacent experience: *"the concepts translate directly"*, *"I am confident I can pick up X quickly given my comfort with Y"*.
3. Signal eagerness to learn — without overclaiming speed.

This is also the natural place to mention genuine, related interests (e.g. security, AI tooling, or domain knowledge from the applicant's degree) when they fit.

### 2.7 Closing paragraph (required)

Must do at least one of these three, and must request the conversation:

- Map the role's structure back to the applicant's career trajectory ("The structure of this role — [enumerate elements] — is exactly the path I want at this stage.")
- Signal cultural / collaborative motivation ("I am genuinely motivated by the kind of [X] environment [Company] describes...")
- Express domain or impact interest ("the chance to apply technical skills to data assets that drive real operational decisions")

End with: *"I would welcome the opportunity to discuss how I could contribute."* (or a close variant). Optionally add: *"Thank you for considering my application."*

### 2.8 Sign-off

Acceptable closings (use either consistently within a letter):

- `Yours sincerely,`
- `Kind regards,`

Then a blank line, then the applicant's full name (from `context/user.md`) on its own line:

```
Yours sincerely,

<Full name — from context/user.md>
```

---

## 3. Paragraph and word counts (guidance)

- Opening paragraph: ~70–100 words
- Lead body paragraph: ~120–160 words (the densest)
- Supporting paragraph(s): ~100–130 words each
- Honest gap paragraph (if present): ~70–100 words
- Closing paragraph: ~60–90 words
- **Target total: 480–510 words.** Above 530 is approaching the page limit and should be tightened.

Total paragraph count in the body (excluding header, salutation, Re: line, sign-off):

- With gap paragraph: 5 body paragraphs (opening + lead + 1 supporting + gap + closing)
- Without gap paragraph: 4 body paragraphs (opening + lead + 1 supporting + closing)
- If experience genuinely needs two supporting paragraphs, that's fine — total stays 5 paragraphs and word count stays in range.

---

## 4. Lexicon

### Phrases that consistently work (use when they fit naturally)

- "My most directly relevant experience is..." — lead-paragraph opener (broad-relevance variant)
- "directly relevant to the [X] responsibilities the role describes"
- "maps closely onto the [X] this role calls for"
- "the kind of [X] that feed directly into [Y]"
- "the same fundamentals the role lists"
- "the concepts translate directly"
- "habits I would bring to [Company]'s [Y] responsibilities"
- "I am genuinely motivated by..."
- "is exactly the path I want at this stage"
- "I would welcome the opportunity to discuss how I could contribute."

### Words and phrases to avoid

- "passionate", "thrilled", "excited" (use "interested", "motivated", or omit)
- "leverage" as a verb (use "use" or "apply")
- "synergy", "rockstar", "ninja", "guru", "world-class"
- "I believe I would be a great fit" (vague; replace with a specific bridge)
- "team player", "go-getter", "fast-paced" (cliché; replace with concrete behaviours)
- "Please find attached..." (the resume is referenced by being attached, not narrated)

---

## 5. Tailoring rules per application

These are decisions the generating agent makes for each job:

1. **Pick the lead paragraph anchor** by reading the parsed job file:
    - If one requirement dominates (e.g. "5+ years SQL" is repeated, called out as essential, and other requirements are softer) → use dominant-requirement framing and lead with the matching experience.
    - Otherwise → use broad-relevance framing.

2. **Pick which experience leads** by comparing the job's must-have requirements against:
    - `context/workexp/experience.md` — for work experience leads
    - `context/projects/projects.md` — for project leads

   Choose whichever has the strongest direct overlap (tech names, domain, role type). When tied, prefer work experience (it's a stronger signal than projects to a hiring manager).

3. **Decide whether to include a gap paragraph**:
    - Scan the role's must-haves and tech stack against the source files.
    - If there's a named requirement the applicant doesn't have direct experience in, AND there is adjacent/transferable experience worth pointing to → include the paragraph.
    - If the applicant's experience genuinely covers the role → omit the paragraph.
    - Never fabricate a gap.

4. **Choose the closing emphasis** based on what the parsed job emphasises:
    - Job describes mentoring / career path / progression → trajectory closing
    - Job emphasises team culture, collaboration, values → cultural closing
    - Job emphasises business impact or specific domain → domain/impact closing

5. **Mirror the role's terminology** where truthful. If the job says "ETL pipelines", use "ETL pipelines" not "data flows". If the job says "containerised", use "containerised" not "Dockerised". Do not adopt terminology for tech the applicant hasn't actually used.

6. **Quote tech names from the source files**, not from the job posting's casing. If a project source says `PostgreSQL` but the job says `postgres`, write `PostgreSQL`. (Exception: the Re: line uses the role title exactly as the job posting writes it.)

---

## 6. Grammar and final pass

Before finalizing, scan the draft for:

- Awkward phrasings (e.g. sentences that begin "As I am to...") — rewrite for natural flow
- Two consecutive sentences starting with "I" (vary openings)
- Repeated key phrases in adjacent paragraphs ("directly relevant" twice in two paragraphs)
- Missing en dashes in the Re: line (must be – not -)
- US spellings that slipped in (organize → organise, behavior → behaviour)
- Numbers written inconsistently ("20-50%" should be "20–50%" with en dash for ranges)
- Do not overstate the nature of any experience beyond what the source files support — represent each role exactly as `context/workexp/experience.md` describes it, without inflating its scope or context.

---

## 7. What this file does NOT cover

These are decided by the `tailor-coverletter` skill, not by these rules:

- File naming and output paths
- LaTeX class, packages, and rendering setup
- PDF compilation and page-count verification
- The mechanics of reading the parsed job file and source materials

The skill consumes these rules as content guidance and applies its own technical rules for everything else.
