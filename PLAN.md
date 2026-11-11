# PLAN.md: Phased build checklist

Rules: one phase at a time. Ask the AI for a plan first, approve it, then code.
Commit to Git after every working step. Do not start a phase until the previous one passes its "Done when".

## Phase 0: Setup and spec (Week 1-2)

- Tools installed and verified (Git, VS Code, Python 3.12 via uv, Node, Docker)
- Repo created at `C:\dev\medagent` with `.gitignore` (includes `.env`)
- `PROJECT_SPEC.md`, `PLAN.md`, `.github/copilot-instructions.md` committed
- Research questions approved by tutor
- Guideline PDFs collected into `data/documents/` with `sources.csv` (title, url, year, tier)
- API keys in `.env` (Gemini, Tavily, NCBI, Langfuse)
**Done when:** repo is on GitHub and tutor has signed off on RQs.

## Phase 1: Foundation (Week 3-4)

- Docker Compose: Postgres + pgvector
- FastAPI skeleton, `/health` endpoint, config via env
- Alembic migrations for all tables in the spec
- Auth: register, login, JWT, role guard (patient / doctor)
- Next.js app with login page and two empty dashboards
- Basic tests for auth
**Done when:** a patient and a doctor can log in and see only their own dashboard.

## Phase 2: RAG and baselines (Week 5-6)

- Document ingestion: parse, chunk, embed, store with metadata
- Retrieval function with source filters
- **S1 baseline:** plain LLM endpoint
- **S2 baseline:** LLM + RAG endpoint
- Langfuse tracing connected
**Done when:** S1 and S2 answer a test query and traces are visible.

## Phase 3: Multi-agent graph (Week 7-9)

- GraphState and node skeletons
- Triage agent (rule-based red flags first, LLM second) with unit tests
- Query understanding agent
- RAG node + web research node (whitelist enforced)
- Evidence grader
- Synthesizer with structured claim output
- Verifier + bounded retry loop
- Safety/policy agent
- Explanation builder
- Every node writes to `agent_traces`
**Done when:** a query runs end to end and produces claims, citations and an explanation (S3).

## Phase 4: Human in the loop (Week 10-11)

- Postgres checkpointer and `interrupt` before publish
- Doctor queue sorted by risk tier
- Review screen: claims, evidence, verification, agent trace
- Approve / reject (with reason + feedback) / edit
- Resume graph with feedback, bounded rounds
- Patient dashboard shows only approved answers
- Real-time status updates (SSE or WebSocket)
- Audit log and answer versioning
**Done when:** the full loop works: patient asks, doctor rejects with feedback, revised answer appears, doctor approves, patient sees it.

## Phase 5: Evaluation (Week 12-13)

- Build the ~150-query set with reference answers and labels
- Script to run S1, S2, S3, S4 over the set and store results in `eval_runs`
- Compute all metrics from the spec
- Doctor review sessions (with vs without explanations, time recorded)
- Tables and charts for the report
**Done when:** results table for all four systems exists.

## Phase 6: Report and polish (Week 14)

- Paper / report: intro, related work, method, results, limitations, ethics
- Demo video and seeded demo data
- README with setup steps
- Viva preparation: be able to explain every agent and metric

## Stretch (only if ahead of schedule)

- Hindi patient-language output
- Drug interaction check (RxNorm / openFDA)
- Doctor analytics dashboard
