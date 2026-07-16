# examforge — Low-Level Design

## Architecture

```
  ┌──────────────────┐            ┌───────────────────────┐
  │ Teacher browser   │            │ Student browser        │
  │ (JWT, org-scoped)  │            │ (no account — access   │
  │                    │            │  code + token only)    │
  └─────────┬──────────┘            └──────────┬─────────────┘
            │ fetch (Bearer JWT)                │ fetch (access_code / access_token)
            ▼                                   ▼
  ┌────────────────────────────────────────────────────────┐
  │            Cloudflare Worker (TS, ESM, strict)          │
  │            Workers Assets + /api/* + /take/*             │
  └───────────────────────────┬───────────────────────────┘
                               │ plain fetch (PostgREST + GoTrue), service-role key
                               ▼
                   ┌──────────────────────────┐
                   │ Supabase                  │
                   │ - GoTrue auth (teachers)   │
                   │ - Postgres + RLS            │
                   │   org-scoped bank/test data │
                   │   attempts: service-role     │
                   │   only, no client RLS path   │
                   └──────────────────────────┘
```

Two distinct request flows share one Worker. **Teacher flow**: magic-link JWT, org-membership checked per request (same pattern as tuitionloop), full CRUD on banks/questions/tests, read access to analytics views. **Student flow**: no Supabase account at all — a published test has an `access_code`; hitting `/take/:accessCode/start` creates a `test_attempts` row and returns an opaque `access_token` (stored client-side, not a JWT); every subsequent read/submit call for that attempt must present the token. Grading is a deterministic Worker-side function run at submission time — there is no LLM call anywhere in this request path.

## Data model

```sql
create table public.organizations (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  owner_user_id uuid not null references auth.users(id) on delete cascade,
  subscription_status text not null default 'trial' check (subscription_status in ('trial','active','expired')),
  trial_ends_at timestamptz not null default (now() + interval '14 days'),
  created_at timestamptz not null default now()
);
alter table public.organizations enable row level security;

create table public.org_members (
  org_id uuid not null references public.organizations(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  role text not null default 'teacher' check (role in ('owner','teacher')),
  primary key (org_id, user_id)
);
alter table public.org_members enable row level security;
create policy "read own memberships" on public.org_members for select using (auth.uid() = user_id);
create policy "read org if member" on public.organizations for select using (
  exists (select 1 from org_members m where m.org_id = organizations.id and m.user_id = auth.uid()));

create table public.question_banks (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references public.organizations(id) on delete cascade,
  name text not null,
  subject text,
  created_at timestamptz not null default now()
);

create table public.questions (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references public.organizations(id) on delete cascade,
  bank_id uuid not null references public.question_banks(id) on delete cascade,
  question_type text not null check (question_type in ('mcq_single','mcq_multi','true_false','numeric')),
  prompt text not null,
  topic text not null,
  options jsonb, -- [{id,text}], null for numeric
  correct_answer jsonb not null, -- e.g. ["opt_2"] or {"value":42,"tolerance":0.5}
  marks numeric not null default 1,
  negative_marks numeric not null default 0,
  created_at timestamptz not null default now()
);

create table public.tests (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references public.organizations(id) on delete cascade,
  title text not null,
  duration_minutes int not null check (duration_minutes > 0),
  access_code text not null unique, -- random, e.g. 10 chars
  status text not null default 'draft' check (status in ('draft','published','closed')),
  created_at timestamptz not null default now()
);

create table public.test_questions (
  test_id uuid not null references public.tests(id) on delete cascade,
  question_id uuid not null references public.questions(id) on delete cascade,
  order_index int not null,
  primary key (test_id, question_id)
);

-- state machine: in_progress -> submitted | in_progress -> expired (server-detected on late submit)
create table public.test_attempts (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references public.organizations(id) on delete cascade,
  test_id uuid not null references public.tests(id) on delete cascade,
  student_name text not null,
  student_identifier text, -- roll no / email, optional
  access_token uuid not null default gen_random_uuid() unique,
  started_at timestamptz not null default now(),
  submitted_at timestamptz,
  status text not null default 'in_progress' check (status in ('in_progress','submitted','expired')),
  score numeric,
  max_score numeric
);

create table public.attempt_answers (
  id uuid primary key default gen_random_uuid(),
  attempt_id uuid not null references public.test_attempts(id) on delete cascade,
  question_id uuid not null references public.questions(id) on delete cascade,
  selected_answer jsonb,
  is_correct boolean,
  marks_awarded numeric,
  unique (attempt_id, question_id)
);

-- org-scoped tables: RLS mirrors organizations/org_members (bank/question/test/read policies checked via org_members).
-- test_attempts and attempt_answers: enable RLS, grant NOTHING to anon/authenticated — students have no
-- Supabase identity, so every read/write on attempts goes through the Worker's access_token check, not RLS.
alter table public.test_attempts enable row level security;
alter table public.attempt_answers enable row level security;

-- per-topic analytics view (Worker reads this for the teacher analytics endpoint)
create view public.topic_analytics as
  select q.org_id, tq.test_id, q.topic,
    count(*) as answers, avg((aa.is_correct)::int)::numeric as accuracy,
    sum(aa.marks_awarded) as total_marks_awarded
  from public.attempt_answers aa
  join public.questions q on q.id = aa.question_id
  join public.test_questions tq on tq.question_id = q.id and tq.test_id = (
    select test_id from public.test_attempts where id = aa.attempt_id)
  group by q.org_id, tq.test_id, q.topic;
```

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/orgs` | POST, GET | JWT | Create org (caller = owner) / list caller's orgs | 400 missing name |
| `/api/orgs/:id/members` | POST | JWT (owner only) | Add existing user by email as teacher | 403; 404 no account |
| `/api/banks`, `/api/banks/:id/questions` | GET, POST | JWT + org membership | CRUD question banks and questions | 403 not org member; 400 `correct_answer` shape mismatches `question_type` |
| `/api/tests` | POST | JWT + org membership | Assemble a test from question IDs, generate `access_code` | 403; 400 empty question list |
| `/api/tests/:id/publish` | POST | JWT + org membership | `draft` → `published` | 409 already published/closed |
| `/api/tests/:id/analytics` | GET | JWT + org membership | Per-topic accuracy from `topic_analytics` | 403 |
| `/take/:accessCode` | GET | access_code | Test metadata + questions (no `correct_answer`) for a published test | 404 unknown/unpublished code |
| `/take/:accessCode/start` | POST | access_code | Creates `test_attempts` row, returns `access_token` + `started_at` + duration | 409 test not published/closed |
| `/take/:accessCode/submit` | POST | access_token | Grades deterministically, writes `attempt_answers`, sets `status='submitted'` | 409 already submitted; 410 past `started_at + duration_minutes` (server-computed, not client-trusted) → `status='expired'`, zero score |

## LLM strategy
**Not applicable to v1.** Grading is a pure function over `question_type` + `correct_answer` vs `selected_answer` — string/set equality for MCQ and true/false, numeric-with-tolerance for numeric. No model call, no cost, no latency concern, nothing to hallucinate. Descriptive/essay questions (which would need either a human grader or an LLM rubric-scoring step) are explicitly out of scope for v1 — see below.

## Frontend pages
Teacher: Login (magic link) · Org dashboard · Question bank editor · Test builder (pick questions, set duration, generate link) · Test analytics (per-topic accuracy, score distribution) · Settings (staff, subscription). Student: unauthenticated landing (`/take/:accessCode`, enter name) · timed test-taking view (question navigator, countdown, auto-submit at zero) · result screen (score, no correct-answer review in v1 to prevent trivial leak of the bank to future test-takers via the same code).

## Error handling
Submission after the server-computed deadline is rejected and the attempt marked `expired`, not silently accepted — the client countdown is UX, the Worker clock is the source of truth. Duplicate submission (`status` already `submitted`) returns 409, no double-grading. Question authoring validates `correct_answer`'s shape against `question_type` at write time so a malformed question can't reach a live test. No credits/refund flow — flat org subscription, gated the same way as tuitionloop: expired subscription blocks new test creation and publishing but not reading existing analytics.

## Integrations and launch gates
None block launch — no third-party API dependency (no WhatsApp, no LLM, no STT). The only hard constraint is content sourcing: **no feature may ingest, scrape, or bulk-import past exam papers or third-party question banks** (UGC NET, board exams, and similar are under long-duration government/publisher copyright, not freely reusable — this is a verified constraint from this challenge's licensing research, not a style preference). Teachers type or manually upload their own questions only.

## Security notes
Org-scoped RLS on banks/questions/tests, membership re-checked in every write handler (service-role key bypasses RLS, as in tuitionloop). `test_attempts`/`attempt_answers` are locked to service-role-only access — no anon/authenticated grants — because students never get a Supabase identity; the Worker's `access_token` check is the entire security boundary for that path, so `access_token` must be a cryptographically random UUID, not guessable. `access_code` should be long enough (10+ random chars) to resist brute-force guessing of a live test link. Input validation on question authoring and on submitted answers (reject answers referencing a `question_id` not on that test).

## Out of scope for v1
LLM-graded descriptive/essay questions; importing or scraping past papers from any external source; plagiarism/cheating detection; adaptive difficulty; student accounts or login (attempts stay token-based); PDF export of results; Razorpay recurring billing (manual UPI + admin activation, matching this challenge's day-1 approach); negative-marking beyond a flat per-question value.
