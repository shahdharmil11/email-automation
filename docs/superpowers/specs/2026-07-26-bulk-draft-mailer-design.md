# Bulk Draft Mailer — Design Spec

**Date:** 2026-07-26
**Owner:** Dharmil Shah
**Status:** Approved for planning

## 1. Goal

A personal Python CLI that turns an Apollo contacts CSV into ~2000 personalized,
review-ready Gmail drafts (optionally sent, throttled) across ~10 Google Workspace
accounts. Each draft is written by an AI middleware layer using the contact's data,
a per-company job role + job description, tone/style references, and the user's resume.
A per-company tailored resume PDF (skills section only) is attached.

**Top priority: ship a working product.** Structure for maintainability, but do not
over-build. Every component below earns its place; anything not listed is out of scope.

## 2. Non-goals (YAGNI)

- No web UI, no hosted service, no multi-user support.
- No CRM / reply tracking / open tracking / analytics.
- No OCR of image PDFs — job descriptions are pasted as text.
- No automatic sending without explicit per-run confirmation.
- No resume editing beyond the Technical Skills lines.

## 3. Success criteria

1. Run `generate` on a real Apollo CSV → correct number of Gmail drafts appear,
   each addressed to the right contact, personalized, with a tailored resume PDF attached.
2. Re-running after a crash never creates duplicates and skips completed rows.
3. `deliver` in send mode sends at a safe throttle, resumable, only after confirmation.
4. No tailored resume that is 2 pages or has non-skills text drift is ever attached.
5. The user is shown, per run: what was generated, what was skipped, and every
   resume keyword that was added.

## 4. Architecture

Three CLI stages over a shared SQLite state store. Each stage is independently
runnable and resumable.

```
Inputs (files)                Stages                         Output
--------------                ------                         ------
contacts.csv  ─┐
roles/*.md    ─┤   preflight ──▶ generate ──▶ deliver ──▶  Gmail drafts / sends
prompts/      ─┤   (validate)    (AI + resume) (draft|send)
style/        ─┤        │             │            │
resume/       ─┤        └──────── state.db (SQLite) ────────┘
config.yaml   ─┤        row status: pending→generated→drafted→sent | failed
.env          ─┘
accounts/
```

- **preflight** — parse + validate rows, collect all missing params into `out/gaps.csv`,
  ask once. No AI/Gmail calls.
- **generate** — for each `pending` row: build prompt, call LLM, validate output,
  ensure the company's tailored resume exists (build + integrity-check if not),
  create a Gmail draft round-robin across accounts, mark `drafted`.
- **deliver** — draft mode: no-op (drafts already exist). send mode: throttled,
  confirm-first, resumable send of `drafted` rows; mark `sent`.

## 5. Components & responsibilities

| Component | Responsibility | Pattern |
|-----------|----------------|---------|
| `cli` | Subcommands `preflight/generate/deliver/status`, `--dry-run` | Command |
| `config` | Load + validate `config.yaml` and `.env` into `RunConfig` | — |
| `contacts.ContactLoader` | Read Apollo CSV, normalize the 70-column schema into `Contact` | Adapter |
| `roles.RoleLoader` | Load `roles/*.md` (frontmatter + JD) into `RoleProfile`, join by company | Adapter |
| `state.Repository` | SQLite persistence of per-row status + generated content | Repository |
| `llm.LLMProvider` | `generate(prompt) -> text`; Anthropic + OpenAI impls | Strategy + Factory |
| `prompt.PromptBuilder` | Assemble system + style + few-shot + contact + JD + resume | Builder |
| `email.DraftContent` validation | No leftover `{{}}`, name/company present, length sane | — |
| `resume.ResumeTailor` | Edit skills lines in `baseline.docx`, Word→PDF, cache per company | Adapter |
| `resume.IntegrityChecker` | 1 page, only skills changed, valid PDF; fit-loop or fallback | — |
| `gmail.GmailClient` | Create draft / send message per account (google-api-python-client) | Adapter |
| `accounts.AccountSelector` | Round-robin account assignment | Strategy |
| `deliver.DeliveryStrategy` | `DraftOnly` vs `ThrottledSend` | Strategy |
| `pipeline` | Wire stages, drive rows through the store | Pipeline |

## 6. Domain models (pydantic)

- `Contact` — `first_name, last_name, title, company, email, linkedin, city, state, raw`
- `RoleProfile` — `company, role, apply_url, location, jd_text, extras`
- `DraftContent` — `subject, body, contact_email, account, resume_pdf_path`
- `Account` — `email, auth_ref, daily_cap`
- `RunConfig` — mode, throttle, provider+model, accounts, flags

Normalization happens once at the loaders; the rest of the code sees clean models.

## 7. Inputs & formats

```
bulk-draft-mailer/
  contacts.csv                 # Apollo export (as provided)
  roles/
    waymo.md                   # frontmatter (company, role, apply_url, location) + JD text
  prompts/
    system_prompt.md           # AI persona + hard rules (never invent facts)
    template.md                # email skeleton; AI fills {{personalized_pitch}}
    examples/ex_01.md ...       # few-shot (context -> ideal email)
  style/
    warm_concise.md ...         # 6+ tone-only references
  resume/
    baseline.docx              # source of truth (Word)
    skills_master.txt          # truthful superset of skills
    generated/<company>.pdf    # tailored, cached (auto)
  config.yaml
  .env                          # ANTHROPIC_API_KEY / OPENAI_API_KEY, etc.
  accounts/                     # one OAuth token file per Workspace account
  out/                          # gaps.csv, failed.csv, run_report.md, state.db
```

**`roles/<company>.md`**
```markdown
---
company: Waymo
role: Software Engineer
apply_url: https://...
location: Mountain View, CA
---
## Job Description
<pasted JD text>
## Emphasis / keywords
<optional>
```

**`prompts/template.md`:** fixed email skeleton (subject + body); AI fills only
`{{personalized_pitch}}`. Deterministic fields (`{{first_name}}`, `{{company}}`,
`{{role}}`) are merged directly.

## 8. Config schema (`config.yaml`)

```yaml
mode: draft                 # draft | send
provider: anthropic         # anthropic | openai
model: <model-id>
resume:
  allow_unlisted_keywords: true   # user chose auto-add; every add is logged
  max_pages: 1
send:
  per_account_per_minute: 4
  jitter_seconds: [3, 12]
  daily_cap_per_account: 300
accounts:
  - email: a1@domain.com
    auth_ref: accounts/a1.json
  # ... up to 10
style:
  use: [warm_concise, bold_direct]   # which style refs to apply
```

## 9. AI middleware (generate)

Prompt = system prompt + selected style references (tone only, never facts) +
few-shot examples + this `Contact` + `RoleProfile.jd_text` + resume text.
Output = `subject` + `personalized_pitch`, merged into the template skeleton.

**Output validation (auto):** no leftover `{{...}}`; contact first name and company
present; length within bounds. Failures → row marked `failed` with reason, run continues.

Provider is chosen from config via a Factory; both Anthropic and OpenAI implement
`LLMProvider`. Keys come only from `.env`.

## 10. Resume tailoring + integrity gate

Per company (cached), before attaching:

1. **Tailor:** copy `baseline.docx`; rewrite only the four skills paragraphs
   (Languages / Frameworks / Databases / Tools & Technologies) to surface relevant
   skills from `skills_master.txt` and add JD keywords (if `allow_unlisted_keywords`).
   All other paragraphs untouched.
2. **Render:** Word automation (`docx2pdf`, MS Word installed) → high-fidelity PDF.
3. **Integrity check (gate):**
   - page count == 1;
   - every non-skills paragraph identical to baseline (text diff);
   - valid, renderable PDF.
4. **On failure:** fit-loop drops lowest-priority added keywords and re-renders;
   if still failing, discard tailored PDF, **attach clean baseline**, and log it.

Every added keyword and every fallback is written to `out/run_report.md`.

## 11. Delivery & safety

- **draft mode (default):** drafts created during `generate`; `deliver` is a no-op.
- **send mode:** `deliver` sends `drafted` rows at `per_account_per_minute` with jitter,
  round-robin across accounts, respecting daily caps. Requires an explicit confirmation
  prompt showing counts + accounts before the first send. Resumable; kill switch (Ctrl-C
  leaves state consistent). Sending is never automatic.
- Keys only in `.env`; no secrets in code or logs.

## 12. Error handling & resumability

- SQLite is the single source of truth for row status; every stage is idempotent.
- Transient LLM/Gmail errors: retry with backoff, then mark `failed` (row-level, run continues).
- `status` command prints counts by state; `out/failed.csv` lists failures + reasons.

## 13. Testing

- Unit tests with Gmail + LLM **mocked**: CSV normalization, role join, gap detection,
  template merge, output validation, skills-line rewrite, integrity check (page/diff).
- `--dry-run`: render drafts + one sample tailored resume to `out/` with **zero** Gmail/LLM
  side effects (LLM stubbed or a single live call gated behind a flag), so real output can
  be eyeballed before touching Gmail.

## 14. Gmail auth

Per-account OAuth token in `accounts/` (one-time browser consent per account). If the
user is Workspace admin, a service account with domain-wide delegation can replace
per-account consent — supported but optional; OAuth is the default path.

## 15. Build order (milestones to a working product)

1. **Skeleton + models + config loading** + SQLite repository + `status`.
2. **Contacts + roles loaders** (Adapter) with unit tests on the real CSV shape.
3. **LLM provider (one, e.g. via config) + PromptBuilder** + output validation; `--dry-run`
   renders drafts to `out/` (no Gmail yet). ← first visible value.
4. **Gmail draft creation** + round-robin accounts; `generate` end-to-end in draft mode.
5. **Resume tailoring + integrity gate**; attach per-company PDF.
6. **preflight/gaps** flow.
7. **send mode** (throttled, confirm-first, resumable).

Stop at any milestone and still have something usable; milestone 4 is a shippable core.

## 16. Assumptions

- JDs are pasted as text (no OCR).
- MS Word present for high-fidelity docx→PDF (confirmed on this machine).
- User supplies Anthropic and/or OpenAI keys at run time; provider chosen in config.
- Cold outreach volume/patterns are the user's responsibility; the tool defaults to
  drafts and safe throttles to reduce account risk.
