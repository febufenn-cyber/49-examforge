This repo is product #49 of Febin's 50-SaaS challenge. Nothing is built yet — docs/ is the source of truth. Read docs/LLD.md then docs/PLAN.md and execute tasks in order with TDD.

Stack conventions + the shipped reference implementation live in the private repo febufenn-cyber/50-saas (contract-reviewer/) — same Worker+Supabase+auth patterns. This product has two auth models: teachers use Supabase JWT + org-scoped RLS (like tuitionloop, #48); students take tests via an opaque `access_token`, never a Supabase account — do not merge these into one auth path.

Grading is deterministic (no LLM), content is user-authored only (no past-paper import, ever), and the multi-tenant `org_members` RLS design in the LLD are verified decisions — do not re-litigate without evidence.
