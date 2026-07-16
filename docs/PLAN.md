# examforge — Build Plan

TDD throughout: write the failing test first, then the minimum code to pass it. Each task is sized for one focused agent session. The teacher (org-scoped, JWT) and student (access-token, no JWT) paths are independent enough to build in separate tasks — do the teacher-authoring path first so there's content to test the student flow against.

**T1 — Scaffold + migration.**
Files: `package.json`, `wrangler.jsonc`, `tsconfig.json`, `supabase/migrations/0001_init.sql`, `.gitignore`, `.dev.vars` (gitignored).
Create every table from `docs/LLD.md` including the `topic_analytics` view; RLS on org-scoped tables via `org_members`; RLS enabled but zero grants on `test_attempts`/`attempt_answers`.
Test: two test orgs, prove cross-org read of questions is blocked; prove `select` on `test_attempts` fails even for an authenticated user (service-role only by design).
Done: migration applies; both isolation properties verified with real queries.

**T2 — Supabase client + org-membership guard.**
Interface: `class Supa { ...; isOrgMember(userId, orgId): Promise<boolean>; createOrg(...); listOrgsForUser(userId); addMember(orgId, userId, role); createBank(...); listBanks(orgId); createQuestion(...); listQuestions(bankId); createTest(...); publishTest(testId); getTestByAccessCode(code); createAttempt(testId, studentName, studentIdentifier?); getAttemptByToken(token); recordAnswers(attemptId, graded[]); finalizeAttempt(attemptId, status, score, maxScore); getTopicAnalytics(orgId, testId); }`
Test: `isOrgMember` mocked-fetch tests for member/non-member; every other method asserts correct PostgREST call shape.
Done: `isOrgMember` is the single choke point every teacher-side write handler calls first.

**T3 — Router + worker entrypoint + auth.**
Files: `src/router.ts` (must route both `/api/*` JWT paths and `/take/*` token paths), `src/worker.ts`, `handlers.ts` skeleton, `handleMe`.
Test: router path-matching unit tests for both route families.
Done: `wrangler dev` boots; magic-link login works; `/api/orgs` lists the caller's orgs.

**T4 — Org + question bank + question CRUD.**
Files: `handlers.ts` (`handleCreateOrg`, `handleAddMember`, `handleCreateBank`, `handleListBanks`, `handleCreateQuestion`, `handleListQuestions`), frontend bank/question editor.
Test: question creation validates `correct_answer` shape against `question_type` (reject an `mcq_single` with a numeric `correct_answer`); non-member gets 403 before any write (assert via mock call count, not just status code).
Done: a teacher can build a question bank with mixed question types.

**T5 — Grading function (pure, no I/O).**
Files: `src/grading.ts` (`gradeAnswer(question, selectedAnswer): {isCorrect: boolean, marksAwarded: number}`).
Test: table-driven tests per `question_type` — exact match, partial mcq_multi, numeric within/outside tolerance, missing answer (zero marks, not negative), negative-marks deduction on a wrong answer.
Done: `gradeAnswer` is fully unit-tested in isolation before any handler calls it — this is the correctness-critical function in the whole product.

**T6 — Test assembly + publish.**
Files: `handlers.ts` (`handleCreateTest`, `handlePublishTest`), frontend test builder (pick questions, set duration, get link).
Test: `access_code` collision handling (regenerate on unique-constraint violation); publish rejects an already-published/closed test with 409; empty question list rejected at creation.
Done: a teacher assembles a test from bank questions and gets a shareable `access_code`.

**T7 — Student take-flow: start.**
Files: `handlers.ts` (`handleTakeStart` — validates `access_code`, test is `published`, creates attempt, returns `access_token` + question list minus `correct_answer`).
Test: unpublished/unknown code returns 404; questions payload never contains `correct_answer` (explicit assertion, not just "looks right").
Done: a student can start a test with only the access code, no account.

**T8 — Student take-flow: submit + server-enforced timing.**
Files: `handlers.ts` (`handleTakeSubmit` — computes `now() > started_at + duration_minutes` server-side, calls `gradeAnswer` per answer, writes `attempt_answers`, finalizes score).
Test: on-time submission grades correctly and sums to the right score; late submission (mock clock past deadline) is rejected/marked expired regardless of what the client claims; resubmission on an already-`submitted` attempt returns 409.
Done: grading is server-authoritative; a manipulated client clock cannot extend time or resubmit.

**T9 — Analytics + subscription gating.**
Files: `handlers.ts` (`handleTestAnalytics` reading `topic_analytics`), gate `handleCreateTest`/`handlePublishTest` on `subscription_status !== 'expired'`, admin script for manual UPI-proof → `active` flip.
Test: analytics endpoint returns per-topic accuracy for a test with multiple attempts; expired-org test creation/publish returns 402, reads (including analytics) still succeed.
Done: a teacher sees which topics a batch is weakest on after a handful of attempts.

**T10 — Deploy + live smoke test + launch checklist.**
Files: `scripts/smoke.ts` (create org → bank → questions → test → publish → take as a student via `access_code` → submit → check analytics, against the deployed Worker).
Checklist: `SUPABASE_SERVICE_ROLE_KEY` set via `wrangler secret put`; manual UPI top-up flow working; pricing page live; custom domain routed; confirm in production (not just tests) that `test_attempts` has zero anon/authenticated grants.
Done: `npm run smoke` passes end to end against the live deployment, teacher path and student path both exercised.

## Notes for whoever executes this plan
- T5 (`gradeAnswer`) has no database or network dependency — it should be the fastest, most thoroughly tested unit in the whole build. Do not let it leak into a handler untested.
- The access-token student flow (T7/T8) is unauthenticated by design; do not add Supabase auth to it later without re-reading why it was built this way in `docs/LLD.md`.
- No provider file, no `fetchFn` injection pattern needed anywhere in this plan — there is no external API call in the product's request paths, which is unusual for this challenge and worth keeping that way.
- If a future version adds descriptive/essay grading, treat it as a new, clearly-labeled LLM provider module with its own cost/latency section in the LLD — do not quietly fold it into `gradeAnswer`.
