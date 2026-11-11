# MedAgent: Human-in-the-Loop Evidence-Grounded Medical Q&A (Diabetes + Hypertension)

> Source of truth for the project. AI coding tools must read this before every task.
> If code and this spec disagree, ask before changing either.

## 1. Purpose

A web platform where patients ask questions about **type 2 diabetes and essential hypertension**.
A multi-agent pipeline (LangGraph + LangChain) retrieves evidence, drafts an answer made of
**atomic claims**, verifies each claim, and builds an evidence-based explanation.
A **doctor reviews** the draft (approve / reject with feedback / edit). Only approved answers
become visible to the patient. Rejection feedback goes back to the pipeline for revision.

The system provides **education and information only**. It never diagnoses, prescribes or changes doses.

## 2. Research framing

**Core question:** Does a human-in-the-loop, evidence-traceable pipeline make patient-facing
medical AI safer and more trustworthy than AI alone, and at what cost in doctor time?

| ID | Research question | Hypothesis |
|---|---|---|
| RQ1 | Does RAG + multi-agent verification reduce unsupported claims versus a plain LLM? | H1: Unsupported-claim rate drops significantly from S1 to S3 |
| RQ2 | Does the human review stage catch errors the automatic verifier misses? | H2: Doctors reject/edit a non-zero share of claims that the verifier marked "supported" |
| RQ3 | Do evidence-grounded explanations reduce doctor review time without lowering error detection? | H3: Review time with explanations < without, with equal or better error detection |
| RQ4 | Does doctor feedback improve the revised answer? | H4: Approval rate after revision round 1 is higher than first-pass approval |

**Systems compared (ablation), same query set:**

- S1: Plain LLM
- S2: LLM + RAG
- S3: Multi-agent + Verifier (no human)
- S4: S3 + Human-in-the-loop

## 3. Scope

**In scope:** adults, type 2 diabetes, essential hypertension, comorbid cases, lifestyle, diet,
monitoring, general medicine information, when-to-see-a-doctor guidance.

**Out of scope:** pregnancy, children, type 1 diabetes, diagnosis, prescriptions, dose changes,
real patient data, EHR integration, voice, mobile app.

**Terminology note:** "Explainable AI" in this project means **evidence-grounded explainability**
(claim-to-source mapping, agent trace, confidence, verification status). We do NOT claim to explain
the internal workings of the LLM. "Learning from feedback" means feedback is injected into
the next revision and stored for analysis. It is NOT model retraining.

## 4. Non-negotiable safety rules

1. **Emergency bypass:** if triage detects red flags, skip the answer pipeline and show
   an urgent-care message immediately. Still log the query and notify the doctor queue.
   Red flags include: chest pain, sudden weakness or numbness, speech/vision problems,
   severe headache with very high BP, confusion/vomiting/breathlessness with high or low sugar,
   fainting, suicidal thoughts.
2. **No diagnosis, no prescriptions, no dosage change advice.**
3. **Every factual claim must cite at least one retrieved source.** Claims without evidence are dropped or flagged.
4. **Nothing reaches the patient without doctor approval** (status must be `approved`).
5. **Fixed disclaimer** on every patient answer.
6. **Only whitelisted, tiered sources** for web retrieval (see section 8).
7. **Synthetic or public data only.** No real patient data.
8. Loops are bounded: max verifier retries = 2, max doctor-feedback rounds = 3.

## 5. Pipeline (LangGraph)

```text
START
  -> triage_agent ----(emergency)----> emergency_response -> notify doctor -> END
  -> query_understanding_agent
  -> [rag_agent || web_research_agent]       (parallel)
  -> evidence_grader_agent
  -> synthesizer_agent                        (outputs claims + citations)
  -> verifier_agent ----(unsupported claims, retries < 2)----> synthesizer_agent
  -> safety_policy_agent
  -> explanation_builder
  -> human_review (INTERRUPT: pause until doctor decides)
        approve -> publish -> END
        reject  -> synthesizer_agent (with doctor feedback, rounds < 3)
        edit    -> save edited version -> publish -> END
```

| Agent | Job | Output |
|---|---|---|
| Triage | Red-flag / scope detection (rules + LLM) | `is_emergency`, `in_scope` |
| Query Understanding | Rewrite query, extract entities (condition, drug, symptom) | `rewritten_query`, `entities` |
| RAG | Retrieve from local guideline documents (pgvector) | `rag_chunks` |
| Web Research | Search whitelisted domains and PubMed | `web_docs` |
| Evidence Grader | Score relevance and credibility, drop weak evidence | `graded_evidence` |
| Synthesizer | Write patient-friendly answer as atomic claims with citations | `draft_claims` |
| Verifier | Label each claim supported / unsupported / contradicted | `verification` |
| Safety/Policy | Remove diagnosis, dosage advice; add disclaimer | `safety_flags` |
| Explanation Builder | Evidence map, confidence, agent trace | `explanation` |

## 6. LangGraph state (draft)

```python
class ClaimDraft(BaseModel):
    claim_id: str
    text: str
    evidence_ids: list[str]
    verification: Literal["supported", "unsupported", "contradicted", "unchecked"] = "unchecked"
    confidence: float = 0.0

class GraphState(TypedDict):
    query_id: str
    patient_query: str
    is_emergency: bool
    in_scope: bool
    rewritten_query: str
    entities: dict
    rag_chunks: list[dict]
    web_docs: list[dict]
    graded_evidence: list[dict]
    draft_claims: list[ClaimDraft]
    safety_flags: list[str]
    explanation: dict
    verifier_retries: int
    review_round: int
    doctor_decision: Literal["approve", "reject", "edit", "pending"]
    doctor_feedback: str | None
    final_answer: str | None
    trace: list[dict]          # per-agent log: name, input summary, output summary, latency, tokens
```

## 7. Database schema (PostgreSQL + pgvector)

| Table | Key columns |
|---|---|
| `users` | id, email, password_hash, role (`patient`/`doctor`), name, created_at |
| `queries` | id, patient_id, text, status (`submitted`/`processing`/`emergency`/`awaiting_review`/`approved`/`rejected`/`revising`), risk_tier, created_at |
| `answer_versions` | id, query_id, version_no, full_text, source (`ai`/`doctor_edit`), created_at |
| `claims` | id, answer_version_id, text, verification, confidence, doctor_decision, rejection_reason |
| `evidence` | id, query_id, source_type (`rag`/`web`/`pubmed`), source_name, url, credibility_tier, snippet, relevance_score |
| `claim_evidence` | claim_id, evidence_id |
| `reviews` | id, query_id, doctor_id, decision, feedback, rejection_reason, review_seconds, created_at |
| `agent_traces` | id, query_id, agent_name, input_summary, output_summary, latency_ms, tokens, created_at |
| `documents` | id, title, source_url, year, publisher, credibility_tier |
| `document_chunks` | id, document_id, content, embedding (vector), page |
| `audit_log` | id, actor_id, action, entity, entity_id, timestamp |
| `eval_runs` | id, system (S1-S4), query_set_id, metrics (jsonb), created_at |

**Rejection reasons (enum):** `wrong_fact`, `outdated`, `unsafe`, `unclear`, `missing_context`, `off_scope`.

## 8. Source whitelist and credibility tiers

- **Tier 1:** ADA Standards of Care, WHO, ICMR, RSSDI, ESC/ESH, ACC/AHA, NICE guidelines
- **Tier 2:** PubMed systematic reviews / RCTs, NIH, CDC, MedlinePlus
- **Tier 3:** Hospital patient-education pages (only if explicitly added)
- **Blocked:** blogs, forums, social media, commercial supplement sites

## 9. Tech stack

| Layer | Choice |
|---|---|
| Frontend | Next.js, Tailwind, shadcn/ui |
| Backend | FastAPI (Python 3.12, managed with `uv`) |
| Orchestration | LangGraph + LangChain |
| LLM | Gemini via API (model name from env var `GEMINI_MODEL`) |
| Embeddings | Configurable via env; pgvector for storage |
| DB | PostgreSQL + pgvector (Docker) |
| Graph persistence | LangGraph Postgres checkpointer |
| Auth | JWT, role-based (patient / doctor) |
| Real-time | SSE or WebSocket for status and approval notifications |
| Web retrieval | PubMed E-utilities + Tavily with domain allow-list |
| Tracing | Langfuse or LangSmith |
| Evaluation | RAGAS / DeepEval + custom scripts |

## 10. Evaluation metrics

- **Retrieval:** context precision, context recall, source credibility score
- **Answer quality:** faithfulness, answer relevancy, correctness vs reference
- **Safety:** unsupported-claim rate, contradiction rate, unsafe-advice rate, **emergency red-flag recall**
- **Citations:** citation precision and recall
- **HITL:** first-pass approval rate, avg revision rounds, claims rejected per answer, edit distance,
  time-to-approval, errors caught by doctor but missed by verifier
- **Explainability:** doctor Likert rating (1-5), review time with vs without explanations
- **Agreement:** Cohen's kappa between 2 reviewers
- **System:** latency per agent, tokens and cost per query

**Query set (~150):** 50 diabetes, 50 hypertension, 30 comorbid, 20 emergency/trap queries.
Sources: PubMedQA, MedQA (subset), HealthSearchQA, plus hand-written patient-style queries.

## 11. Limitations to state in the report

- Small number of doctor reviewers (1-3)
- Two conditions only
- LLM output varies between runs
- Not clinically validated; not a medical device
- Synthetic / public data only
