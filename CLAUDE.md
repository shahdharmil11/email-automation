# bulk-draft-mailer

Personal Python CLI that reads Apollo recruiter contacts from a CSV, writes a personalized
cold-outreach email per contact using an AI middleware layer, and creates Gmail drafts
(or sends them throttled) across ~10 Google Workspace accounts. A per-company tailored
resume PDF (skills section only) is attached to every draft.

**Repo:** git@github.com:shahdharmil11/email-automation.git
**Local:** /Users/dharmilshah/Downloads/bulk-draft-mailer
**Branch:** main
**Design spec:** docs/superpowers/specs/2026-07-26-bulk-draft-mailer-design.md
**Status:** Design approved. No code written yet. Start with milestone 1.

---

## What this tool does end-to-end

1. User drops a real Apollo CSV export into the project root (`contacts.csv`).
2. User adds one `.md` file per target company in `roles/` with the job description + role.
3. User runs `python -m mailer preflight` → tool validates every row, surfaces missing data.
4. User runs `python -m mailer generate` → AI writes a personalized draft per contact,
   Gmail drafts are created across accounts, a tailored resume PDF is attached.
5. Optionally: `python -m mailer deliver` in send mode → throttled, confirm-first sending.
6. `python -m mailer status` → counts by state at any point.

---

## Real input data (already available on disk)

**contacts.csv** — Apollo export at `/Users/dharmilshah/Downloads/apollo-contacts-export.csv`
Key columns used: First Name, Last Name, Title, Company Name, Email, City, State,
Person Linkedin Url, Keywords, Technologies. The CSV has ~70 columns total; we only
normalize the ones above into the `Contact` domain model. Everything else is ignored.

Example row: Angela Song | Senior Recruiter | Waymo | songangela@waymo.com | San Francisco CA

**baseline.docx** — `/Users/dharmilshah/Downloads/RESUMES/v15/archives/jan/resume jan 7.docx`
53 paragraphs, no tables. The Technical Skills lines are at indices 11–14:
- 11: `Languages:	Java · Scala · Python · TypeScript · JavaScript`
- 12: `Frameworks:	Spring MVC · React · Node.js · FastAPI · Microservices`
- 13: `Databases:	PostgreSQL · PgVector · NoSQL · MongoDB · Redis · MySQL`
- 14: `Tools & Technologies:	AWS · Azure · GCP · Terraform · JUnit · MCPs · ...`

Resume owner: Dharmil Shah — MS CS (Northeastern, GPA 3.5), 3+ years in Finance & Healthcare.
Current employers: Cigna Healthcare (Software Engineer AI), State Street, Thomson Reuters.
Tech: Java, Scala, Python, TypeScript, Spring Boot, React, FastAPI, AWS, Azure, GCP,
Terraform, Kafka, Airflow, LangGraph, Docker, PostgreSQL, PgVector, MongoDB, Elasticsearch.

**MS Word** is installed at `/Applications/Microsoft Word.app` — use `docx2pdf` for
high-fidelity Word→PDF conversion. Do NOT use LibreOffice (not installed).

---

## Architecture: 3 stages + SQLite state store

```
contacts.csv ─┐
roles/*.md   ─┤   preflight ──▶ generate ──▶ deliver ──▶  Gmail drafts / sends
prompts/     ─┤   (validate)    (AI+resume)  (draft|send)
style/       ─┤        │             │            │
resume/      ─┤        └──────── out/state.db ◀──┘
config.yaml  ─┤         row status: pending→drafted→sent | failed
.env         ─┘
accounts/
```

**preflight** — parse CSV + roles, check every row for required fields (email, company,
matching role file), collect ALL missing params into `out/gaps.csv`, print a summary,
stop. No AI or Gmail calls. User fills gaps then re-runs.

**generate** — for each `pending` row:
  1. Build prompt (system + style refs + few-shot + contact fields + JD + resume text)
  2. Call LLM → get subject + personalized_pitch
  3. Validate output (no unfilled `{{}}`, name + company present, length in bounds)
  4. Ensure this company's tailored resume PDF exists (build it if not, integrity-check it)
  5. Create Gmail draft via API, assign account round-robin, attach resume PDF
  6. Mark row `drafted` in state.db

**deliver** — draft mode: no-op. send mode: read all `drafted` rows, show confirmation
(count, accounts, rate), wait for explicit "yes", then send at throttled rate with jitter,
mark `sent`. Ctrl-C safe — leaves state consistent for resume.

---

## Component map

| File | Class / function | Pattern | Does |
|------|-----------------|---------|------|
| `mailer/cli.py` | `cli()` Click group | Command | Entry point, subcommands |
| `mailer/config.py` | `load_config()` → `RunConfig` | — | Reads config.yaml + .env |
| `mailer/models.py` | `Contact`, `RoleProfile`, `DraftContent`, `Account`, `RunConfig` | — | Pydantic domain models |
| `mailer/state.py` | `Repository` | Repository | SQLite CRUD for row state |
| `mailer/contacts.py` | `ContactLoader` | Adapter | Apollo CSV → `Contact` list |
| `mailer/roles.py` | `RoleLoader` | Adapter | roles/*.md → `RoleProfile` dict |
| `mailer/llm/base.py` | `LLMProvider` ABC | Strategy | `generate(prompt) -> str` |
| `mailer/llm/anthropic_provider.py` | `AnthropicProvider` | Strategy impl | Calls Anthropic API |
| `mailer/llm/openai_provider.py` | `OpenAIProvider` | Strategy impl | Calls OpenAI API |
| `mailer/llm/factory.py` | `get_provider(config)` | Factory | Returns correct impl |
| `mailer/prompt.py` | `PromptBuilder` | Builder | Assembles final prompt |
| `mailer/email_validator.py` | `validate_draft()` | — | Checks AI output quality |
| `mailer/resume/tailor.py` | `ResumeTailor` | Adapter | Edits docx skills lines, Word→PDF |
| `mailer/resume/checker.py` | `IntegrityChecker` | — | 1-page + skills-only diff gate |
| `mailer/gmail/client.py` | `GmailClient` | Adapter | Creates drafts / sends via API |
| `mailer/accounts.py` | `AccountSelector` | Strategy | Round-robin across accounts |
| `mailer/deliver.py` | `DraftOnly`, `ThrottledSend` | Strategy | Delivery modes |
| `mailer/pipeline.py` | `run_preflight()`, `run_generate()`, `run_deliver()` | Pipeline | Wires stages |
| `tests/` | pytest suite | — | All components mocked |

---

## Domain models (Pydantic — mailer/models.py)

```python
class Contact(BaseModel):
    first_name: str
    last_name: str
    title: str
    company: str          # normalized from "Company Name"
    email: str
    linkedin: str
    city: str
    state: str
    raw: dict             # full original row for anything else

class RoleProfile(BaseModel):
    company: str          # must match Contact.company (case-insensitive)
    role: str
    apply_url: str
    location: str
    jd_text: str
    extras: str = ""

class DraftContent(BaseModel):
    subject: str
    body: str
    contact_email: str
    account_email: str
    resume_pdf_path: str

class Account(BaseModel):
    email: str
    auth_ref: str         # path to accounts/<name>.json OAuth token
    daily_cap: int = 300

class RunConfig(BaseModel):
    mode: Literal["draft", "send"]
    provider: Literal["anthropic", "openai"]
    model: str
    accounts: list[Account]
    resume_allow_unlisted: bool
    resume_max_pages: int = 1
    send_per_account_per_minute: int = 4
    send_jitter: tuple[int, int] = (3, 12)
    send_daily_cap: int = 300
    style_refs: list[str] = []
```

---

## Folder layout (final)

```
bulk-draft-mailer/
  contacts.csv                    # Apollo export (user drops here)
  config.yaml                     # mode, provider, model, accounts, throttle
  .env                            # ANTHROPIC_API_KEY / OPENAI_API_KEY
  CLAUDE.md                       # this file

  roles/
    waymo.md                      # one file per company
    stripe.md

  prompts/
    system_prompt.md              # AI persona + hard rules
    template.md                   # subject + body skeleton with {{placeholders}}
    examples/
      ex_01.md                    # few-shot: (context → ideal email)

  style/
    warm_concise.md               # tone-only reference (voice/psychology, never facts)
    bold_direct.md

  resume/
    baseline.docx                 # source of truth — never edit directly
    skills_master.txt             # full truthful superset of Dharmil's skills
    generated/                    # tailored per-company PDFs (auto-generated, gitignored)

  accounts/                       # OAuth token JSON per Workspace account (gitignored)

  out/                            # runtime output (gitignored)
    state.db
    gaps.csv
    failed.csv
    run_report.md

  mailer/                         # all source code
    __init__.py
    __main__.py
    cli.py
    config.py
    models.py
    state.py
    contacts.py
    roles.py
    prompt.py
    email_validator.py
    accounts.py
    deliver.py
    pipeline.py
    llm/
      __init__.py
      base.py
      anthropic_provider.py
      openai_provider.py
      factory.py
    resume/
      __init__.py
      tailor.py
      checker.py
    gmail/
      __init__.py
      client.py
      auth.py

  tests/
    conftest.py
    test_contacts.py
    test_roles.py
    test_prompt.py
    test_email_validator.py
    test_resume_tailor.py
    test_resume_checker.py
    test_state.py
    test_pipeline.py

  requirements.txt
  .gitignore
```

---

## roles/*.md format

```markdown
---
company: Waymo
role: Software Engineer
apply_url: https://waymo.com/joinus/...
location: Mountain View, CA
---
## Job Description
<paste full JD text here>

## Emphasis
<optional: specific keywords or requirements to lean on>
```

Company name must match the "Company Name" column in the Apollo CSV (case-insensitive match).

---

## prompts/template.md format

```
Subject: {{role}} opportunity — quick intro

Hi {{first_name}},

{{personalized_pitch}}

Would love to connect if there's a fit.

Best,
Dharmil Shah
```

`{{first_name}}`, `{{company}}`, `{{role}}` are merged directly from data.
`{{personalized_pitch}}` is the only part the AI writes (2–3 sentences max).

---

## AI middleware — how the prompt is assembled

PromptBuilder stacks these in order:
1. `prompts/system_prompt.md` — persona + constraints (never invent facts, stay concise)
2. Selected style refs from `style/` (tone/voice only — no factual claims borrowed)
3. Few-shot examples from `prompts/examples/` (shows ideal output format)
4. Contact context: first_name, last_name, title, company, city
5. Role context: role title, location, JD text
6. Resume text (extracted from baseline.docx — the AI uses it to write honest claims)

LLM returns JSON: `{"subject": "...", "personalized_pitch": "..."}`.
Validator checks: no `{{` in output, first_name in body, company in body, body ≤ 200 words.

---

## Resume tailoring — step by step

1. Copy `resume/baseline.docx` to a temp file.
2. Ask the LLM (same provider) to pick the most relevant skills from `skills_master.txt`
   given the company's JD. If `allow_unlisted_keywords: true`, may also add JD keywords
   not in master list — every addition is logged to `out/run_report.md`.
3. Open the temp docx with `python-docx`. Rewrite ONLY paragraphs 11–14 (the four
   Technical Skills lines). Every other paragraph: untouched.
4. Save temp docx, convert to PDF using `docx2pdf` (MS Word at `/Applications/Microsoft Word.app`).
5. IntegrityChecker:
   - Open the PDF with `pypdf`, assert page count == 1.
   - Extract text from tailored PDF and baseline PDF, diff every non-skills paragraph,
     assert they are identical.
   - If either check fails: drop the lowest-priority added keyword, re-render, retry
     (max 3 attempts).
   - If still failing after fit-loop: discard tailored PDF, use `baseline.docx`→PDF,
     log fallback to run_report.
6. Cache result at `resume/generated/<company>.pdf`. Same PDF reused for every contact
   at that company — never rebuilt unless the role file changes.

---

## Gmail auth setup (user does this once before first run)

Each Workspace account needs a one-time OAuth consent. The tool calls
`gmail/auth.py` which opens a browser tab per account and saves a token to
`accounts/<account-slug>.json`. After that, all runs are fully background.

`config.yaml` lists each account:
```yaml
accounts:
  - email: dharmil@domain1.com
    auth_ref: accounts/domain1.json
```

Round-robin assignment: contact row 0 → account 0, row 1 → account 1, etc., wrapping.

---

## Throttle & safety (send mode)

- Default: 4 emails/min/account, jitter 3–12s between sends.
- Daily cap: 300/account (well under Gmail's 2000/day limit).
- Before sending: print a summary table (count, accounts, estimated time), require
  user to type "yes" to proceed.
- Ctrl-C at any point: state.db already has `sent` for completed rows.
  Re-running `deliver` skips those and continues from where it stopped.
- Sending is NEVER triggered automatically — only when `mode: send` in config
  AND user explicitly runs `deliver` AND types "yes" at the prompt.

---

## Error handling rules

- Every row failure is caught at the row level. The run never crashes mid-batch.
- Failed rows: written to `out/failed.csv` with reason (LLM error / Gmail API error /
  validation failure / integrity check fallback).
- Transient API errors: retry 3 times with exponential backoff, then mark failed.
- After the run: `status` shows counts. User can re-run `generate` or `deliver` —
  completed rows are skipped automatically via state.db.

---

## Testing approach

- All Gmail API calls and LLM calls are **mocked** in tests (never hit real APIs).
- Fixtures use the real Apollo CSV column names so tests break if the Adapter breaks.
- `--dry-run` flag: in generate mode, skips Gmail API call and writes draft content
  to `out/dry_run/<contact_email>.txt` instead. Lets the user read real AI output
  before touching Gmail.
- Run tests: `pytest tests/ -v`

---

## Coding conventions (important — do not drift from these)

- **Commit messages:** short, plain English. "add contact loader", "fix page overflow check",
  "wire gmail draft creation". No feat/fix/chore prefixes, no long descriptions in title.
- **Code comments:** only when the WHY is non-obvious. One line max. No docstring essays.
  Example: `# Gmail API requires base64url encoding, not standard base64`
- **Never add Claude as co-author.** `includeCoAuthoredBy: false` is set in global settings.
- **No fabricated skills on resume.** The tailor logs every added keyword. The user reviews.
- **Keys only from .env.** Never hard-code or log any API key.

---

## Dependencies (requirements.txt — to be created in milestone 1)

```
click
pydantic
pyyaml
pandas
python-docx
docx2pdf
pypdf
google-api-python-client
google-auth-oauthlib
anthropic
openai
pytest
pytest-mock
python-frontmatter
```

---

## Build milestones — start here

### Milestone 1 (start now)
- `mailer/models.py` — all Pydantic models
- `mailer/config.py` — load config.yaml + .env → RunConfig
- `mailer/state.py` — SQLite repository (create table, upsert row, query by status)
- `mailer/__main__.py` + `mailer/cli.py` — Click CLI skeleton with preflight/generate/deliver/status subcommands (stubs ok)
- `mailer/pipeline.py` — stub stage functions wired to CLI
- `requirements.txt`, `config.yaml` (example), `.gitignore`
- `pytest tests/test_state.py` — Repository unit tests

### Milestone 2
- `mailer/contacts.py` — ContactLoader Adapter (Apollo CSV → Contact list)
- `mailer/roles.py` — RoleLoader Adapter (roles/*.md → RoleProfile dict, company join)
- `tests/test_contacts.py`, `tests/test_roles.py` — unit tests against real column names

### Milestone 3
- `mailer/llm/` — LLMProvider ABC, AnthropicProvider, OpenAIProvider, Factory
- `mailer/prompt.py` — PromptBuilder
- `mailer/email_validator.py` — validate_draft()
- Wire into generate pipeline with `--dry-run` writing to `out/dry_run/`

### Milestone 4 (shippable core)
- `mailer/gmail/auth.py` — OAuth flow, token storage
- `mailer/gmail/client.py` — create_draft(), send_message()
- `mailer/accounts.py` — AccountSelector (round-robin)
- Wire into generate pipeline — full end-to-end draft creation

### Milestone 5
- `mailer/resume/tailor.py` — ResumeTailor
- `mailer/resume/checker.py` — IntegrityChecker
- Wire into generate pipeline before draft creation

### Milestone 6
- Flesh out preflight stage — gap detection, out/gaps.csv, summary

### Milestone 7
- `mailer/deliver.py` — ThrottledSend strategy
- Wire into deliver stage with confirmation prompt and kill-switch safety
