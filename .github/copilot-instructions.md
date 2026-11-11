# Copilot instructions for MedAgent

Read `README.md` (the project spec) and `PLAN.md` before any task. They are the source of truth.

## How to work

- Do only the task I name. Never build ahead into later phases.
- For anything larger than one file, first propose a short plan and wait for my approval.
- Keep changes small. Explain what changed and why in plain language, because I must explain this code in my viva.
- If the spec is unclear or something conflicts, ask instead of guessing.
- Do not add libraries without telling me why.

## Backend (Python 3.14 target, FastAPI)

- Python is managed with `uv`. Use `uv add` and `uv run`, never plain `pip`.
- Type hints everywhere. Use Pydantic models for API schemas and for structured LLM outputs.
- Layout: `backend/app/{api,agents,graph,services,models,schemas,core}`; tests in `backend/tests`.
- One file per agent in `agents/`. Each agent is a pure function: takes state, returns a partial state update.
- The LangGraph graph is defined in one place: `graph/builder.py`.
- Every agent must append an entry to `state["trace"]` and write to `agent_traces`.
- Use async for I/O. Database access goes through the service layer, never directly in route handlers.
- Use Alembic for every schema change.

## Safety rules in code (never weaken these)

- The doctor-approval check is enforced on the server. A patient endpoint returns only answers with status `approved`.
- Emergency triage runs first and has rule-based keyword checks that do not depend on the LLM.
- Web retrieval accepts only domains from the allow-list config. Reject all others.
- Every claim must have at least one `evidence_id`, or it is dropped or flagged.
- Loop limits (verifier retries, review rounds) come from config and are always enforced.
- Never write code that gives diagnosis, prescriptions or dose changes.

## Secrets and config

- Secrets only in `.env`, loaded via settings class. Never hardcode keys. Never commit `.env`.
- The model name comes from env `GEMINI_MODEL`. Do not hardcode model names.

## Testing

- Add tests with each feature (pytest). Triage and safety logic need unit tests with edge cases.
- Run the tests before saying a task is done.
- LLM calls are mocked in unit tests.

## Frontend (Next.js, TypeScript)

- Tailwind and shadcn/ui. Strict TypeScript, no `any`.
- API calls go through one client module. Handle loading and error states.
- Role-based route protection on both patient and doctor pages.
- Always show the medical disclaimer on patient answers.

## Git

- Small commits with clear messages, one per working step.
