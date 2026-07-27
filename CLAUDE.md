# bulk-draft-mailer

Personal Python CLI — Apollo contacts CSV → ~2000 personalized Gmail drafts across ~10 Workspace accounts.
AI writes the email body, per-company resume PDF (skills section only) is attached.

**Repo:** git@github.com:shahdharmil11/email-automation.git
**Spec:** docs/superpowers/specs/2026-07-26-bulk-draft-mailer-design.md

## Where we are

Design spec done and pushed. No code written yet.
Next step: write the implementation plan (invoke superpowers:writing-plans skill), then start milestone 1.

## Build order

1. Skeleton + models (Pydantic) + config loading + SQLite repo + `status` command
2. Contacts + roles loaders (Adapter) with unit tests
3. LLM provider + PromptBuilder + `--dry-run` (renders drafts to out/, no Gmail)
4. Gmail draft creation + round-robin accounts — **shippable core**
5. Resume tailoring + integrity gate (1 page, skills-only diff, Word→PDF)
6. Preflight/gaps flow
7. Send mode (throttled, confirm-first, resumable)

## Key rules

- Commit messages: short plain English ("add contact loader", "fix resume page overflow")
- Code comments: simple, say WHY not WHAT, one line max
- Never add Claude as co-author (`includeCoAuthoredBy: false` in global settings)
- Resume edits: ONLY the four Technical Skills paragraphs (lines 11–14 in baseline.docx)
- Sending is NEVER automatic — always throttled (3–5/min/account) + explicit confirmation

## Stack

Python · google-api-python-client · pydantic · python-docx · docx2pdf · anthropic/openai SDKs · SQLite

## Patterns

Pipeline (3 stages) · Strategy (LLM provider, delivery, account selector) · Factory · Repository · Adapter · Builder · Command
