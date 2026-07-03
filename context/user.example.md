# Applicant Details

This file supplies the applicant's identity and contact details to the tailor
skills. Copy it to `context/user.md` (which is gitignored) and replace the
placeholder values with your own. The skills read from `context/user.md`, never
from this example and never hardcoded.

Keep the field labels exactly as written — the skills parse these lines by their
label (`UserToken:`, `FullName:`, etc.), so renaming a label will break parsing.

- **UserToken:** JordanMercer
- **FullName:** Jordan Avery Mercer
- **Location:** Brisbane, QLD
- **Phone:** 0400 000 000
- **Email:** jordan.mercer@example.com

## Field notes

- **UserToken** — a short, filesystem-safe form of the name with no spaces or
  punctuation (e.g. `JordanMercer`). Used only in output filenames, e.g.
  `output/JordanMercer_<Company>_Resume.pdf`.
- **FullName** — the name exactly as it should appear in the resume and cover
  letter headers (spaces, middle names, brackets all fine), e.g.
  `Jordan Avery Mercer`.
- **Location** — city and state/region for the cover letter header, e.g.
  `Brisbane, QLD`.
- **Phone** — contact number for the cover letter header.
- **Email** — contact email for the cover letter header.
