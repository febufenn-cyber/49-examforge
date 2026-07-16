# examforge

Mock-test builder with auto-grading and per-topic analytics — teachers author their own question bank, assemble a timed test, share a link, and get a per-topic breakdown of where their students are actually weak.

**Status: planned — not yet built (50-SaaS challenge #49)**

## The problem
Coaching institutes and exam-prep creators run mock tests through Google Forms, WhatsApp PDFs, or paper — none of which auto-grade objectively, enforce a time limit properly, or tell the teacher *which topics* a batch is weak on versus just a final score.

## Target buyer
Coaching institutes and independent exam-prep creators (India-first) who already have their own question material and want it turned into a timed, auto-graded, analytics-backed test.

## Pricing hypothesis
Rs799/month per institute (multi-tenant: one subscription covers an institute's teachers and question banks).

## Stack
Cloudflare Worker (TypeScript, Workers Assets + `/api/*`) serving the teacher dashboard and a separate unauthenticated test-taking flow for students. Supabase for magic-link teacher auth and Postgres — question banks and tests are `org_id`-scoped with RLS; student attempts are token-gated through the Worker, not Supabase auth. Analytics are Postgres views, not an LLM.

## How to continue this build
Read `docs/LLD.md` for the multi-tenant data model, the access-token student flow, and the grading design, then `docs/PLAN.md` for the ordered, TDD-sized task list. `CLAUDE.md` points an agent session at both. Nothing is implemented yet — these three files are the source of truth.

## Risks / constraints (verified — do not soften)
- **User-authored content only.** This product must never scrape, bundle, or offer to import copyrighted past-paper content (UGC NET, board exams, NCERT-adjacent material) — that content is government/publisher copyright, not freely reusable, per the licensing research for this challenge. Teachers author or manually upload their own questions; there is no "import from past papers" feature in any version.
- **Auto-grading is deterministic in v1**, not LLM-graded — objective question types only (MCQ, true/false, numeric). Descriptive/essay grading needs a human or an LLM step and is explicitly out of scope for v1.
- **Multi-tenant RLS**: every question-bank/test table is `org_id`-scoped via an `org_members` table, the same deviation from this challenge's single-owner reference used in tuitionloop (#48).
- **Timed-test correctness must be server-enforced**: a client-side countdown is UX only — the Worker must reject a submission that arrives after `started_at + duration_minutes`, or a student can screenshot the questions and stall.
